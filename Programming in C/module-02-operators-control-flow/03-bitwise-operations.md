### LAYER 1: THE SPEED LAYER

#### 1. Core Intuition (Feynman)

Every integer in C is just a row of tiny on/off light switches (bits). While normal math (`+`, `*`) treats the whole row as one combined number, **bitwise operators** let you reach in and flip, check, copy, or slide individual switches directly in a single CPU clock cycle—making them the backbone of hardware drivers, OS permissions, network packets, and high-speed algorithms.

#### 2. Master Mental Map

```text
                         BITWISE OPERATIONS (1-Cycle CPU Ops)
                                          │
       ┌──────────────────────────────────┼──────────────────────────────────┐
       ▼                                  ▼                                  ▼
  BIT-BY-BIT LOGIC                   BIT SHIFTS                      CORE BIT IDIOMS
  ────────────────                   ──────────                      ───────────────
  &  (AND) : Clear / Check           << (Left)  : x * 2^k            Set bit k   : x |=  (1U << k)
  |  (OR)  : Set                     >> (Right) : x / 2^k            Clear bit k : x &= ~(1U << k)
  ^  (XOR) : Toggle / Diff               │                           Toggle bit k: x ^=  (1U << k)
  ~  (NOT) : Invert all bits             ├── Unsigned: Logical (0s)  Check bit k : (x >> k) & 1U
                                         └── Signed -: Arith   (1s)  Power of 2  : n > 0 && !(n & (n-1))

```

#### 3. 80/20 High-Yield Cheat Sheet

* **The 4 Golden Bit-Masking Formulas (0-indexed from LSB):**
* **Set bit `k`:** `x |= (1U << k);`
* **Clear bit `k`:** `x &= ~(1U << k);`
* **Toggle bit `k`:** `x ^= (1U << k);`
* **Check bit `k`:** `(x & (1U << k)) != 0`  *(or `(x >> k) & 1U`)*


* **Power of 2 Formula:** `n > 0 && (n & (n - 1)) == 0` *(Never omit `n > 0`—otherwise `0` falsely passes!)*
* **Unsigned Rule:** Always perform bit shifting and masking on **`unsigned` types** (`uint32_t`, `1U << k`). Shifting signed integers into the sign bit or shifting negative numbers triggers **Undefined Behavior (UB)** or implementation-defined behavior.
* **Shift Width UB:** Shifting by a negative count (`x << -1`) or by $\ge$ the bit width of the type (`uint32_t x; x << 32`) is **Undefined Behavior**—not zero!
* **Precedence Warning:** Always parenthesize bitwise expressions when mixing with `==`, `!=`, `+`, or `-`: `(x & MASK) == 0` and `(1U << n) - 1`.

---

### LAYER 2: EXHAUSTIVE DEEP-DIVE

#### Module 1: How do the 6 bitwise operators (`&`, `|`, `^`, `~`, `<<`, `>>`) transform bits?

* **Plain-English Intuition (Feynman):**
`&`, `|`, `^`, and `~` apply basic truth-table logic independently to every vertical column of bits—no carries bleed into neighboring columns. `<<` and `>>` physically slide the entire row of bits left or right, dropping bits that fall off the edge.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Bitwise AND (`&`):** Output bit is `1` *only if both* input bits are `1`. Acts as a **filter** (bits ANDed with `1` survive; bits ANDed with `0` are wiped to `0`).
2. **Bitwise OR (`|`):** Output bit is `1` if *either* input bit is `1`. Acts as a **forcer** (bits ORed with `1` become `1`; bits ORed with `0` stay unchanged).
3. **Bitwise XOR (`^`):** Output bit is `1` if the two input bits are *different*. Acts as a **selective inverter** (bits XORed with `1` flip; bits XORed with `0` stay unchanged).
4. **Bitwise NOT (`~`):** Unary operator that flips every `0` to `1` and every `1` to `0` across the entire promoted integer width.
5. **Left Shift (`x << k`):** Slides all bits left by `k` positions, shifting `0`s into the right (LSB). For unsigned integers without overflow, equals $x \times 2^k$.
6. **Right Shift (`x >> k`):** Slides all bits right by `k` positions, discarding the lowest `k` bits. For unsigned integers, equals $\lfloor x / 2^k \rfloor$.


* **Visual / Structure (Dual Coding — 8-Bit Trace):**

```text
  A = 0101 1100 (92)       A & B = 0001 1000  (Both 1)
  B = 0011 1010 (58)       A | B = 0111 1110  (Either 1)
                           A ^ B = 0110 0110  (Different)
                           ~A    = 1010 0011  (Flipped — on 8-bit)
                           A << 1 = 1011 1000 (92 * 2 = 184)
                           A >> 1 = 0010 1110 (92 / 2 = 46)

```

* **Concrete Examples:**
1. **Fast Even/Odd Check:** `(x & 1)` checks the LSB (weight $2^0 = 1$). Returns `1` for odd, `0` for even—works for both positive and negative 2's complement integers without division/modulo overhead.
2. **Extracting RGB Channels from 24-bit Color (`0xRRGGBB`):**
```c
uint32_t color = 0xADFF2F;
uint8_t r = (color >> 16) & 0xFF; // 0xAD
uint8_t g = (color >> 8)  & 0xFF; // 0xFF
uint8_t b =  color        & 0xFF; // 0x2F

```




* **Traps & Edge Cases (Warning):**
* **Integer Promotion on `~`:** Because `~` promotes `uint8_t` to `signed int` first, `~((uint8_t)0x0F)` is `0xFFFFFFF0` (`-16`), **not** `0xF0` (`240`). Cast back or mask (`~x & 0xFF`) if you only want the low 8 bits.



---

#### Module 2: How do Bit Masking, Setting, Clearing, and Toggling pack multiple flags into one integer?

* **Plain-English Intuition (Feynman):**
Instead of wasting 8 separate `bool` variables (8 bytes) to store 8 on/off settings (like Linux file permissions `rwx`), you pack all 8 settings into a single 1-byte `uint8_t` and use a **bitmask** (a stencil with `1`s only at the target bit positions) to read or modify one switch without disturbing the other 7.
* **Exact Mechanics (Elaborative Interrogation):**
Let bit position `k` be 0-indexed from the right (LSB = bit 0). The single-bit mask is `mask = (1U << k)`:
1. **Set bit `k` (Force to `1`):** `flags |= (1U << k);`
*Why:* `x | 1 = 1` sets bit `k`; `x | 0 = x` leaves all other bits untouched.
2. **Clear bit `k` (Force to `0`):** `flags &= ~(1U << k);`
*Why:* `~(1U << k)` puts a `0` at bit `k` and `1`s everywhere else. `x & 0 = 0` clears bit `k`; `x & 1 = x` preserves all other bits.
3. **Toggle bit `k` (Flip `0 ↔ 1`):** `flags ^= (1U << k);`
*Why:* `x ^ 1 = ~x` flips bit `k`; `x ^ 0 = x` preserves all other bits.
4. **Check bit `k` (Read `0` or `1`):** `(flags & (1U << k)) != 0` or `(flags >> k) & 1U`.


* **Visual / Structure (Dual Coding — Modifying Bit 2):**

```text
Original flags:  0100 1010   (Bit 2 is 0)
Mask (1U << 2):  0000 0100
~Mask:           1111 1011

SET    (flags | Mask):   0100 1110  (Bit 2 forced to 1)
CLEAR  (flags & ~Mask):  0100 1010  (Bit 2 forced to 0)
TOGGLE (flags ^ Mask):   0100 1110  (Bit 2 flipped 0 → 1)

```

* **Concrete Examples:**
1. **POSIX/Linux Permission Flags:**
```c
#define PERM_READ  (1U << 0) // 001 (1)
#define PERM_WRITE (1U << 1) // 010 (2)
#define PERM_EXEC  (1U << 2) // 100 (4)

uint8_t p = PERM_READ | PERM_WRITE; // Grant Read + Write
p &= ~PERM_WRITE;                   // Revoke Write
if (p & PERM_READ) { /* allowed */ }

```


2. **Clearing a Range / Creating an $n$-bit Mask:** `(1U << n) - 1` produces `n` consecutive `1`s at the bottom (e.g., `(1U << 4) - 1` = `16 - 1` = `15` = `0000 1111`).


* **Traps & Edge Cases (Warning):**
* **The `1 << 31` Signed Overflow Bug:** `1` is a **signed `int**` (32-bit). Shifting `1 << 31` shifts a `1` into the sign bit of a signed `int`, which is **Undefined Behavior** in C99/C11! Always use **`1U << 31`** (for 32-bit) or **`1ULL << 63`** (for 64-bit).



---

#### Module 3: How does `n & (n - 1)` check powers of 2, and what are the high-yield XOR properties?

* **Plain-English Intuition (Feynman):**
Subtracting `1` from any binary number borrows down to the **rightmost `1` bit**, turning that `1` into a `0` and flipping every trailing `0` after it into a `1`. When you AND `n` with `n - 1`, that rightmost `1` and all trailing bits annihilate each other—meaning `n & (n - 1)` **deletes the lowest set bit** in a single step.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Power of 2 Check (`n > 0 && (n & (n - 1)) == 0`):**
* A positive integer is a power of 2 ($1, 2, 4, 8, 16, \dots$) **if and only if** it has **exactly one `1` bit** in binary.
* Deleting that single `1` bit via `n & (n - 1)` leaves `0`.
* If `n` has two or more `1` bits (e.g., `6` = `0110`), `n & (n - 1)` (`0110 & 0101 = 0100`) is non-zero.


2. **Brian Kernighan's Bit-Count Algorithm (Popcount):**
* Looping `while (n) { n &= (n - 1); count++; }` counts the number of `1` bits in $O(\text{number of set bits})$ time instead of $O(32)$.


3. **Core XOR (`^`) Algebraic Identities:**
* **Self-Inverse:** `x ^ x == 0`
* **Identity:** `x ^ 0 == x`
* **Commutative & Associative:** `(a ^ b) ^ c == a ^ (b ^ c)` → order doesn't matter; pairs cancel out to `0`.




* **Visual / Structure (Dual Coding):**

```text
Case 1: Power of 2 (n = 8)             Case 2: Not Power of 2 (n = 6)
  n     = 0000 1000                      n     = 0000 0110
  n - 1 = 0000 0111                      n - 1 = 0000 0101
  ─────────────────                      ─────────────────
  &     = 0000 0000 (== 0 -> TRUE!)      &     = 0000 0100 (!= 0 -> FALSE!)

```

* **Concrete Examples:**
1. **Finding the Single Unique Number in an Array (DSA Classic):** Given an array where every element appears twice except one (`{4, 1, 2, 1, 2}`), XORing all elements cancels every duplicate (`1^1=0`, `2^2=0`), leaving `4 ^ 0 = 4` in $O(n)$ time and $O(1)$ space.
2. **Isolating the Lowest Set Bit:** `n & (-n)` (or `n & (~n + 1)` for unsigned) isolates *only* the rightmost `1` bit (used heavily in Fenwick / Binary Indexed Trees).


* **Traps & Edge Cases (Warning):**
* **`n == 0` and Negative `n` in Power-of-2 Check:**
* For `n = 0`: `0 & (0 - 1)` = `0 & 0xFFFFFFFF` = **`0`**! Without `n > 0`, `0` falsely reports as a power of 2.
* For `n = INT_MIN` (`-2147483648`): `n - 1` causes **signed underflow (UB)**. Always guard with `n > 0` first (short-circuiting protects `n - 1`!).


* **XOR Swap Trap (`a ^= b; b ^= a; a ^= b;`):** If `a` and `b` alias the **same memory address** (`swap(&arr[i], &arr[i])`), `a ^= a` zeroes out the value and destroys the data! Use a temp variable in production code.



---

#### Module 4: What is the difference between Arithmetic and Logical Right Shift on signed integers?

* **Plain-English Intuition (Feynman):**
When you slide bits to the right (`>>`), an empty slot opens up on the far left (the MSB). You have two choices for what to fill that empty slot with: always fill with **`0`** (Logical Shift), or **copy the existing sign bit** so negative numbers stay negative (Arithmetic Shift).
* **Exact Mechanics (Elaborative Interrogation):**
1. **Logical Right Shift (Always used for `unsigned`):**
* Shifts bits right and fills vacated leftmost bits with **`0`**.
* Exact equivalent of unsigned floor division by $2^k$.


2. **Arithmetic Right Shift (Used on x86/ARM for `signed`):**
* Shifts bits right and fills vacated leftmost bits with a **copy of the MSB (sign bit)**: `0` for positive numbers, `1` for negative numbers.
* Preserves the sign of negative 2's complement numbers.


3. **Rounding Difference vs. Division (`/`):**
* C's integer division `/` **truncates toward zero**: `-5 / 2 == -2`.
* Arithmetic right shift `>>` **floors toward $-\infty$**: `-5 >> 1 == -3`!


4. **ISO C Standard Caveat:**
* Right-shifting a **negative signed integer** (`-8 >> 1`) is **implementation-defined** in C (GCC/Clang on x86-64 and ARM64 perform arithmetic shift, filling with `1`s).
* Left-shifting a **negative signed integer** (`-1 << 1`) is **Undefined Behavior** in C11.




* **Visual / Structure (Dual Coding — 8-Bit `-8` = `1111 1000`):**

```text
Value: 1111 1000  (Signed = -8, Unsigned = 248)

LOGICAL RIGHT SHIFT (unsigned >> 1):
  Fill with 0 ──► [0]111 1100  = +124  (248 / 2)

ARITHMETIC RIGHT SHIFT (signed >> 1):
  Copy MSB 1  ──► [1]111 1100  = -4    (-8 / 2, sign preserved!)

```

* **Concrete Examples:**
1. **Floor vs. Truncate Mismatch:**
```c
int a = -1;
int d = a / 2;   // 0   (truncated toward zero)
int s = a >> 1;  // -1  (arithmetic shift: 1111...1111 >> 1 stays 1111...1111!)

```


Never blindly replace `x / 2` with `x >> 1` if `x` can be negative!
2. **Forcing a Logical Right Shift in C:** Cast to unsigned first: `((unsigned int)x) >> k` guarantees `0`-fill regardless of platform or sign.


* **Traps & Edge Cases (Warning):**
* **Shift Count $\ge$ Type Width (`x << 32` on 32-bit `int`):** On x86-64 CPUs, the `shl` hardware instruction masks the shift count with `& 31` (`32 & 31 == 0`), so `x << 32` actually evaluates to **`x`**, not `0`! Because it is Undefined Behavior in C, never shift by a variable `k` without guaranteeing `0 <= k < width`.



---

### LAYER 3: SYNTHESIS & INTERLEAVING

#### Compare & Contrast Table

| Confused Pair | Option A | Option B | Decisive Rule / Diagnostic |
| --- | --- | --- | --- |
| **Logical Right Shift vs. Arithmetic Right Shift** | **Logical (`unsigned >> k`):** Fills left with `0`s. | **Arithmetic (`signed >> k`):** Copies sign bit (MSB) into left bits. | Unsigned = always Logical. Signed negative = Arithmetic on x86/ARM (implementation-defined in C). |
| **`x >> 1` vs. `x / 2**` | **`x >> 1`:** Floors toward $-\infty$ (`-1 >> 1 == -1`). | **`x / 2`:** Truncates toward `0` (`-1 / 2 == 0`). | Identical for non-negative integers; **different** for odd negative integers. |
| **`1 << 31` vs. `1U << 31**` | **`1 << 31`:** **Undefined Behavior** (overflows 32-bit signed `int` sign bit). | **`1U << 31`:** **Well-defined** (`0x80000000U`). | Always use unsigned literals (`1U`, `1ULL`) when building bitmasks. |
| **Bitwise `~x` vs. Logical `!x**` | **`~x`:** Flips *every bit* (`~0 == -1`, `~5 == -6`). | **`!x`:** Boolean invert (`!0 == 1`, `!5 == 0`). | `~` is for masks (`x &= ~MASK`); `!` is for truth values. |

#### Cross-Connections

* **2's Complement → Bitwise Math:** In 2's complement, `-x == ~x + 1`. That's why `~0 == -1` (all `1`s) and why `n & (-n)` isolates the lowest set bit.
* **Precedence → Bitwise Masks:** Because `==` beats `&` and `+`/`-` beat `<<`, both `(x & (1U << k)) != 0` and `(1U << n) - 1` require parentheses to parse correctly in Stage 2.
* **Short-Circuiting → Power of 2:** In `n > 0 && (n & (n - 1)) == 0`, short-circuiting on `n > 0` doesn't just reject `0`—it also prevents `INT_MIN - 1` signed underflow UB from ever executing!

---

### LAYER 4: LOCK-IN LAYER

#### 1. Blank-Page Self-Test

Answer all 7 before opening the key.

1. **Masking Drill:** Write the exact 1-line C expressions to **(a)** set bit 5, **(b)** clear bit 5, and **(c)** toggle bit 5 of `uint32_t reg`.
2. **Power of 2 Trap:** Why does `(n & (n - 1)) == 0` alone fail as a power-of-2 check? Name the exact input that breaks it.
3. **Shift vs. Division:** On an x86-64 machine using arithmetic right shift, what are the values of `(-5) / 2` vs. `(-5) >> 1`?
4. **Signed Shift UB:** Why is `uint32_t mask = 1 << 31;` Undefined Behavior on a 32-bit `int` platform, and how do you fix it?
5. **Shift Width UB:** On a 32-bit `uint32_t x = 1;`, is `x << 32` guaranteed to evaluate to `0`? Why or why not?
6. **Logical vs. Arithmetic:** Given `int8_t a = -16;` (`1111 0000`) and `uint8_t b = 240;` (`1111 0000`), what are `a >> 2` (arithmetic) and `b >> 2` (logical) in decimal?
7. **XOR Trick:** What does `x ^ x ^ y` evaluate to, and why does XOR-swapping `a` and `b` fail when `&a == &b`?

1. **(a)** `reg |= (1U << 5);`  **(b)** `reg &= ~(1U << 5);`  **(c)** `reg ^= (1U << 5);`
2. Fails for **`n = 0`** (`0 & -1 == 0`, falsely returning true) and causes signed underflow UB for `INT_MIN`. Fix: `n > 0 && (n & (n - 1)) == 0`.
3. `(-5) / 2` = **`-2`** (truncates toward zero). `(-5) >> 1` = **`-3`** (arithmetic shift floors toward $-\infty$).
4. `1` is a signed `int`; shifting into the sign bit (bit 31) is **Undefined Behavior** in C. Fix: `uint32_t mask = 1U << 31;`.
5. **No.** Shifting by $\ge$ the bit width of the type is **Undefined Behavior**. On x86-64, the CPU masks the shift count modulo 32 (`32 % 32 = 0`), leaving `x` unchanged as `1`.
6. `a >> 2` = **`-4`** (`1111 1100`, sign-extended with `1`s). `b >> 2` = **`60`** (`0011 1100`, zero-filled with `0`s).
7. Evaluates to **`y`** (`x ^ x == 0`, `0 ^ y == y`). When `&a == &b`, the first step `a ^= b` becomes `a ^= a`, immediately zeroing out the shared memory location and erasing the original value.

---

#### 2. Anki Flashcards

* **Q:** Write the 4 standard bit-manipulation formulas for bit `k` of `x`.
**A:** Set: `x |= (1U << k)` | Clear: `x &= ~(1U << k)` | Toggle: `x ^= (1U << k)` | Check: `(x >> k) & 1U`.
* **Q:** What does `n & (n - 1)` do to the binary representation of `n`?
**A:** Clears (deletes) the **lowest (rightmost) set `1` bit** of `n`.
* **Q:** Write the bug-free C expression to check if a signed `int n` is a power of 2.
**A:** `n > 0 && (n & (n - 1)) == 0`.
* **Q:** What is the difference between Logical Right Shift and Arithmetic Right Shift?
**A:** Logical fills vacated MSBs with **`0`s** (used for `unsigned`); Arithmetic copies the **sign bit** into vacated MSBs (used for `signed` on x86/ARM).
* **Q:** Why can't you safely replace `x / 2` with `x >> 1` for negative signed integers?
**A:** `/` truncates toward zero (`-1 / 2 == 0`), while arithmetic `>>` floors toward $-\infty$ (`-1 >> 1 == -1`).
* **Q:** What are the 3 main Undefined Behavior (UB) traps with bit shifts (`<<`, `>>`) in C?
**A:** (1) Shifting by a negative count, (2) Shifting by $\ge$ the bit width of the type (`x << 32` on 32-bit), (3) Left-shifting a negative signed int or shifting into a signed int's sign bit (`1 << 31`).
* **Q:** How do you force a Logical Right Shift on a signed `int x` in C?
**A:** Cast to unsigned first: `((unsigned int)x) >> k`.
* **Q:** What does `(1U << n) - 1` generate (for $n < 32$)?
**A:** A bitmask with the lowest **`n` bits set to `1**` (e.g., `n = 4` → `0000 1111`).

---

#### 3. 60-Second Revision Block (Day 1 / 3 / 7 / 21)

* **Operators:** `&` (clear/check), `|` (set), `^` (toggle/cancel pairs), `~` (invert all bits—watch integer promotion!), `<<` ($\times 2^k$), `>>` ($\div 2^k$).
* **4 Masking Idioms:** Set `x |= (1U << k)`, Clear `x &= ~(1U << k)`, Toggle `x ^= (1U << k)`, Check `(x >> k) & 1U`.
* **Bit Tricks:** `n & (n - 1)` clears lowest set bit → Power of 2 is `n > 0 && (n & (n - 1)) == 0`. `x ^ x == 0` cancels duplicates.
* **Shifts & UB:** Unsigned `>>` = **Logical** (`0`-fill). Signed negative `>>` = **Arithmetic** (sign-bit fill, implementation-defined, floors toward $-\infty$ unlike `/`). Always use unsigned types (`1U << k`) and ensure `0 <= k < width`.