### LAYER 1: THE SPEED LAYER

#### 1. Core Intuition (Feynman)

A CPU cannot directly add a 1-byte `char` to an 8-byte `double`—its arithmetic circuits require both inputs to be the **exact same type and width**. Whenever you mix types in an expression, C silently runs an automatic "leveling-up" protocol behind the scenes to promote smaller or narrower types into a common type before executing the math. Understanding these silent rules prevents C's nastiest bugs: infinite loops, broken security bounds checks, and unexpected negative-to-huge-positive wrap-arounds.

#### 2. Master Mental Map

```text
                  MIXED-TYPE EXPRESSION (e.g., a + b)
                                  │
                                  ▼
           [STEP 1: INTEGER PROMOTION] (Always runs first!)
    Any type smaller than int (char, short, _Bool) ──► int (or unsigned int)
                                  │
                                  ▼
           [STEP 2: USUAL ARITHMETIC CONVERSIONS] (Match ranks)
   ┌────────────────────────────────────────────────────────────────────┐
   │  long double ◄── double ◄── float ◄── unsigned long long ◄── ...   │
   │                                                                    │
   │  If both are integers & mixed signedness:                          │
   │   • rank(unsigned) >= rank(signed)  ──► BOTH become unsigned       │
   │   • signed can hold ALL unsigned    ──► BOTH become signed         │
   │   • otherwise                       ──► BOTH become unsigned(S)    │
   └────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                 [EXPLICIT CASTING: (target_type)expr]
       Overrides implicit rules intentionally (value conversion or bit truncation)

```

#### 3. 80/20 High-Yield Cheat Sheet

* **Integer Promotion Rule:** C **never** does arithmetic on `char` or `short`. Both are silently promoted to `int` first—so `sizeof('a' + 'b')` is **`4`**, not `1`.
* **The `unsigned short` Surprise:** On 32-bit `int` systems, `unsigned char` and `unsigned short` fit completely inside `signed int`, so they promote to **`signed int`**, not `unsigned int`.
* **Same-Rank Mixed Signedness Rule:** Mixing `int` and `unsigned int` (or `long` and `size_t`) converts the **signed** operand to **unsigned**. `-1` becomes `4,294,967,295` (32-bit) or `18,446,744,073,709,551,615` (64-bit).
* **`size_t` Underflow Rule:** `size_t` is unsigned. Never write `for (size_t i = n; i >= 0; i--)` (infinite loop!) and never subtract two `size_t` values without checking which is larger first (`a > b`, not `a - b > 0`).
* **Integer Division Rule:** `5 / 2` is `2`, even if assigned to a `double` (`double d = 5 / 2;` → `2.0`). Cast **before** dividing: `(double)5 / 2` → `2.5`.

---

### LAYER 2: EXHAUSTIVE DEEP-DIVE

#### Module 1: What is Implicit Integer Promotion, and why does `char + char` produce an `int`?

* **Plain-English Intuition (Feynman):**
CPU registers are naturally built for `int`-sized math (32 bits). Rather than forcing the CPU to do awkward 8-bit or 16-bit math, C automatically upgrades every tiny integer (`_Bool`, `char`, `short`) into an `int` the moment you use it in *any* expression.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Trigger:** Applies to any integer type whose **conversion rank is less than `int**`: `_Bool`, `char`, `signed char`, `unsigned char`, `short`, `unsigned short`, and small bit-fields.
2. **The Value-Preserving Rule (ISO C):**
* If `signed int` can represent **all** possible values of the original type → promote to **`signed int`**.
* Otherwise → promote to **`unsigned int`**.


3. **Bit-Level Mechanics:**
* Promoting a `signed char` (`-1` = `0xFF`) uses **Sign Extension**: copies the MSB (`1`) into the new upper 24 bits → `0xFFFFFFFF` (`-1` as `int`).
* Promoting an `unsigned char` (`255` = `0xFF`) uses **Zero Extension**: fills the new upper 24 bits with `0` → `0x000000FF` (`+255` as `int`).




* **Visual / Structure (Dual Coding):**

```text
signed char c = -1;     (0xFF) ──[Sign Extend]──► int: 0xFFFFFFFF (-1)
unsigned char u = 255;  (0xFF) ──[Zero Extend]──► int: 0x000000FF (+255)

```

* **Concrete Examples:**
1. **No Intermediate Overflow on `char`:**
```c
signed char a = 100, b = 100;
signed char c = (a + b) / 2; // Result is 100, NOT overflow!

```


`a` and `b` are promoted to `int` first: `100 + 100 = 200` (fits easily in `int`), `200 / 2 = 100`, then truncated back to `signed char`.
2. **`sizeof` Proves Promotion:** `char a = 1, b = 2; sizeof(a + b)` evaluates to **`4`** (size of `int`), whereas `sizeof(a)` is **`1`**.


* **Traps & Edge Cases (Warning):**
* **Bitwise NOT (`~`) on `unsigned char`:**
```c
unsigned char x = 0x00;
if (~x == 0xFF) { /* FALSE! */ }

```


`x` is promoted to `int` (`0x00000000`) *before* `~` runs. `~x` becomes `0xFFFFFFFF` (`-1`), which is **not equal** to `0x000000FF` (`255`). Fix: `((unsigned char)~x == 0xFF)`.



---

#### Module 2: How do Usual Arithmetic Conversions decide the final type when two operands differ?

* **Plain-English Intuition (Feynman):**
When two different types meet at a binary operator (`+`, `-`, `*`, `/`, `<`, `==`), C promotes the "smaller/narrower" type to match the "larger/wider" type so no precision is lost—moving up a strict ladder from integers to floats, and from signed to unsigned when ranks tie.
* **Exact Mechanics (Elaborative Interrogation):**
For any binary arithmetic or comparison operator, C applies these rules **in strict top-to-bottom order**:
1. **Floating-Point Wins Over Integer:**
* If either operand is `long double` → convert other to `long double`.
* Else if either is `double` → convert other to `double`.
* Else if either is `float` → convert other to `float`.


2. **Integer Promotion First:** If both are integers, run **Integer Promotion** (Module 1) on both.
3. **Same Type:** If they now match → done.
4. **Same Signedness:** Convert the lower-rank type to the higher-rank type (`int` → `long` → `long long`).
5. **Mixed Signedness (1 signed `S`, 1 unsigned `U`):**
* **Case A:** If `rank(U) >= rank(S)` → convert `S` to `U` (e.g., `int` + `unsigned int` → `unsigned int`; `int` + `size_t` → `size_t`).
* **Case B:** Else if `S` can represent **all** values of `U` → convert `U` to `S` (e.g., 64-bit `long` + 32-bit `unsigned int` → **`long`**).
* **Case C:** Else (`rank(S) > rank(U)` but same byte width, e.g., 32-bit `long` + 32-bit `unsigned int`) → convert **both** to `unsigned S` (`unsigned long`).




* **Visual / Structure (Dual Coding):**

```text
Priority Ladder (High → Low):
long double  >  double  >  float  >  (u)long long  >  (u)long  >  (u)int
                                                                    ▲
                                                   [char, short promoted here first]

```

* **Concrete Examples:**
1. **Float + Int:** `5 + 2.5` → `5` converts to `double` (`5.0`), result is `7.5` (`double`).
2. **64-bit `long` + 32-bit `unsigned int` (Case B):** On 64-bit Linux, `long` (8 bytes) can hold every value of `unsigned int` (0 to 4.29B). So `long a = -1; unsigned int b = 1; a < b` converts `b` to `long` → **`-1L < 1L` is TRUE**! (Opposite of `int` + `unsigned int`!)


* **Traps & Edge Cases (Warning):**
* **Assignment Happens LAST:** Usual Arithmetic Conversions only look at the **right-hand side operands**, never the target variable on the left.
```c
double ratio = 1 / 2;        // 1 and 2 are both int -> 1/2 = 0 -> stored as 0.0!
long long big = 100000 * 100000; // int * int overflows 32-bit int (UB) BEFORE assigning to long long!

```


Fix: `1.0 / 2` and `100000LL * 100000`.



---

#### Module 3: How does Signed-to-Unsigned Wrap-Around (`size_t` underflow) cause silent bugs and security flaws?

* **Plain-English Intuition (Feynman):**
Unsigned numbers are like a car odometer that cannot show negative numbers. If you roll `0` backward by `1`, it doesn't show `-1`—it clicks around to `999999999`. Because `sizeof`, `strlen`, and array lengths all return unsigned `size_t`, any negative number or subtraction that dips below zero explodes into a massive positive number.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Modulo $2^n$ Conversion Rule:** Converting a negative signed integer $x$ to an $n$-bit unsigned type adds $2^n$ until it lands in $[0, 2^n - 1]$.
* On 64-bit systems, `(size_t)-1` = $2^{64} - 1 = 18,446,744,073,709,551,615$ (`SIZE_MAX`, or `0xFFFFFFFFFFFFFFFF`).


2. **Why `size_t` Underflow Breaks Loops:** Because `size_t` can **never** be less than `0`, any condition `i >= 0` is mathematically **always true**. When `i == 0`, `i--` wraps to `SIZE_MAX`, and the loop runs forever (or segfaults accessing `arr[SIZE_MAX]`).
3. **Why `size_t` Subtraction Breaks Bounds Checks:** `strlen(a) - strlen(b) > 0` computes an unsigned difference. If `strlen(a) == 3` and `strlen(b) == 5`, `3 - 5` wraps to `SIZE_MAX - 1`, which is `> 0` (**TRUE**!).


* **Visual / Structure (Dual Coding):**

```text
size_t i = 2;
i--  ──►  1
i--  ──►  0
i--  ──►  18,446,744,073,709,551,615 (SIZE_MAX)  ◄── Still >= 0! Infinite Loop!

```

* **Concrete Examples:**
1. **Safe Reverse `size_t` Loop:**
```c
// BROKEN: for (size_t i = n - 1; i >= 0; i--)
// FIXED:
for (size_t i = n; i-- > 0; ) {
    process(arr[i]); // i goes n-1, n-2, ..., 0, then exits cleanly
}

```


2. **Real-World Security Vulnerability (Buffer Underflow):**
```c
int len = get_user_input(); // Attacker sends -1
if (len < sizeof(buf)) {    // -1 converts to SIZE_MAX -> FALSE (good? Wait...)
    // What if check is: if (sizeof(buf) - len > 0) -> wraps and bypasses!
}
memcpy(dst, src, len);      // len (-1) converts to size_t SIZE_MAX -> Massive Heap Overflow!

```




* **Traps & Edge Cases (Warning):**
* **Compiler Flags Save You:** GCC does *not* warn about signed/unsigned comparisons by default. Always compile with `-Wall -Wextra -Wsign-conversion` to catch silent signed-to-unsigned promotions at Stage 2.



---

#### Module 4: What does Explicit Casting `(type)expr` actually do, and when is it dangerous?

* **Plain-English Intuition (Feynman):**
An explicit cast tells the compiler: *"Shut up, I know the types don't match—convert this value anyway."* For numbers, it safely translates the value (with rounding/truncation). For pointers, it tells the compiler to reinterpret the raw memory address through a different lens—which can easily crash or trigger Undefined Behavior if misused.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Numeric Value Casts (Changes bit representation to preserve value):**
* **Int → Float/Double:** Converts integer to IEEE 754 representation (e.g., `(double)5` changes bits from `0x00000005` to `0x4014000000000000`).
* **Float/Double → Int:** **Truncates toward zero** (drops the fractional part: `(int)3.9 == 3`, `(int)-3.9 == -3`). If the float exceeds the target integer range (or is `NaN`/`Inf`), it is **Undefined Behavior**.
* **Narrowing Cast (e.g., `int` → `uint8_t`):** Keeps the lowest $n$ bits (well-defined $\bmod 2^n$ for unsigned; implementation-defined for signed).


2. **Pointer Casts (Does NOT change the address bits—only changes the lens):**
* **`void *` ↔ Any Object Pointer:** Completely legal in C (in fact, C doesn't even require casting `malloc`'s `void *` return).
* **Incompatible Type Pointers (`float *` → `int *`):** Violates C's **Strict Aliasing Rule** if dereferenced. The compiler assumes an `int *` and a `float *` can *never* point to the same memory and may reorder/optimize away your reads.
* **Exception:** Casting any pointer to `unsigned char *` (or `char *`) to inspect raw bytes is **explicitly allowed** by the C standard.




* **Visual / Structure (Dual Coding):**

```text
Value Cast:   int x = 5;  double d = (double)x;  ──► Bits CHANGED (int → IEEE 754)
Pointer Cast: float f = 1.0f;
              int *p = (int *)&f;  // *p is UB (Strict Aliasing Violation!)
              // Legal Type Punning:
              int bits; memcpy(&bits, &f, sizeof(bits)); // Safe + compiles to 1 mov!

```

* **Concrete Examples:**
1. **Preventing Integer Division:**
```c
int total = 7, count = 2;
double avg = (double)total / count; // total → 7.0, count promoted → 2.0, avg = 3.5

```


2. **Inspecting Raw Bytes Safely:**
```c
uint32_t val = 0x12345678;
unsigned char *bytes = (unsigned char *)&val; // Legal! Checks endianness via bytes[0]

```




* **Traps & Edge Cases (Warning):**
* **Casting `malloc()` in C:** Writing `int *p = (int *)malloc(10 * sizeof(int));` is unnecessary in C (unlike C++) and historically hid a fatal bug if you forgot `#include <stdlib.h>` (C89 assumed `malloc` returned `int`, and the cast silenced the compiler warning while truncating the 64-bit pointer). Prefer `int *p = malloc(10 * sizeof(*p));`.



---

### LAYER 3: SYNTHESIS & INTERLEAVING

#### Compare & Contrast Table

| Confused Pair | Option A | Option B | Decisive Rule / Diagnostic |
| --- | --- | --- | --- |
| **Integer Promotion vs. Usual Arithmetic Conversion** | **Integer Promotion:** Unary/binary upgrade of types smaller than `int` (`char`, `short`) → `int`. | **Usual Arithmetic Conversion:** Binary alignment of two *different* types (`int` + `double`, `int` + `unsigned`). | Integer Promotion **always runs first**; Usual Arithmetic Conversion runs second to match the two promoted operands. |
| **Sign Extension vs. Zero Extension** | **Sign Extension:** Copies MSB (`0` or `1`) into upper bits (preserves negative value). | **Zero Extension:** Fills upper bits with `0`s (for unsigned types). | `signed char` `-1` (`0xFF`) → `int` `0xFFFFFFFF`; `unsigned char` `255` (`0xFF`) → `int` `0x000000FF`. |
| **Float → Int Cast vs. `floor()**` | **`(int)-3.9`:** Truncates **toward zero** → **`-3`**. | **`floor(-3.9)`:** Rounds **toward $-\infty$** → **`-4.0`**. | Casts always chop the decimal part; they never round to nearest or floor negatives. |
| **Strict Aliasing (`(int *)&f`) vs. `memcpy**` | **`*(int *)&f`:** **Undefined Behavior** (violates strict aliasing under `-O2`). | **`memcpy(&i, &f, 4)`:** **100% Standard-Compliant** and optimized by GCC into a single register move. | Always use `memcpy` (or `unsigned char *`) to inspect/reinterpret raw bits of another type. |

#### Cross-Connections

* **Primitive Representation → Type Conversion:** Because `int` uses 2's complement and `double` uses IEEE 754, casting `(double)x` forces the CPU to execute a hardware conversion instruction (`cvtsi2sd` in Stage 2 assembly), whereas casting `(unsigned int)x` on a 2's complement machine emits **zero instructions**—it just reinterprets the exact same bits as unsigned!
* **Process Memory → `size_t` Underflow:** Passing a negative signed `int` to `malloc(len)` or `memcpy(dst, src, len)` silently promotes `len` to `size_t` (`SIZE_MAX`), causing `malloc` to return `NULL` or `memcpy` to smash across the Heap/Stack until hitting an unmapped page (`SIGSEGV`).

---

### LAYER 4: LOCK-IN LAYER

#### 1. Blank-Page Self-Test

Answer all 7 before opening the key.

1. **Integer Promotion:** On a 32-bit `int` system, what type does `unsigned short a = 5; unsigned short b = 10; a + b` evaluate to—`unsigned short`, `unsigned int`, or `signed int`? Why?
2. **Bitwise Trap:** What does this print?
```c
unsigned char a = 0x00;
printf("%d\n", ~a == 0xFF);

```


3. **Mixed Signedness (32-bit vs 64-bit):** On 64-bit Linux (LP64), what do these two expressions evaluate to?
* `int a = -1; unsigned int b = 1; a < b`
* `long c = -1; unsigned int d = 1; c < d`


4. **`size_t` Underflow:** Why is `if (strlen(s1) - strlen(s2) > 0)` a bug when `s1 = "hi"` and `s2 = "hello"`? How do you fix it?
5. **Assignment Timing:** Why does `double x = 5 / 2;` store `2.0` instead of `2.5`?
6. **Float-to-Int Truncation:** What are the exact values of `(int)2.9` and `(int)-2.9`?
7. **Type Punning:** Why is `float f = 3.14f; int bits = *(int *)&f;` dangerous at `-O2`, and what is the standard-compliant replacement?

1. **`signed int`**. Integer Promotion upgrades any type smaller than `int` to `signed int` whenever `signed int` can represent *all* values of that type (0..65535 fits easily in 32-bit `signed int`).
2. Prints **`0` (False)**. `a` is promoted to `int` (`0x00000000`) before `~`, making `~a` equal to `0xFFFFFFFF` (`-1`), which is not equal to `0x000000FF` (`255`).
3. * `a < b` → **`0` (False)**: Same rank, so `a` (`-1`) converts to `unsigned int` (`4294967295`).
* `c < d` → **`1` (True)**: `long` (64-bit) can represent all values of `unsigned int` (32-bit), so `d` converts to `long` (`-1L < 1L`).


4. `strlen` returns unsigned `size_t`. `2 - 5` underflows to `SIZE_MAX - 2` (a huge positive number), making `> 0` evaluate to **True**. Fix: `if (strlen(s1) > strlen(s2))`.
5. Usual Arithmetic Conversions only evaluate the right-hand side (`5 / 2`, both `int` → integer division = `2`) *before* converting the result to `double` for assignment.
6. `(int)2.9` = **`2`**; `(int)-2.9` = **`-2`** (always truncates toward zero).
7. It violates C's **Strict Aliasing Rule** (Undefined Behavior); `-O2` may optimize away or reorder reads/writes. Standard replacement: `memcpy(&bits, &f, sizeof(bits));`.

---

#### 2. Anki Flashcards

* **Q:** Which types undergo Implicit Integer Promotion in C expressions?
**A:** `_Bool`, `char`, `signed char`, `unsigned char`, `short`, `unsigned short` (all integer types with rank less than `int`).
* **Q:** On a 32-bit `int` system, what type does `unsigned char` promote to during Integer Promotion?
**A:** **`signed int`** (because `signed int` can represent all values 0–255).
* **Q:** What is the bit-level difference between promoting `signed char` vs. `unsigned char` to `int`?
**A:** `signed char` uses **Sign Extension** (copies MSB); `unsigned char` uses **Zero Extension** (fills upper bits with `0`).
* **Q:** In Usual Arithmetic Conversions, what happens when you mix `int` and `unsigned int`?
**A:** Same rank → the `int` is silently converted to **`unsigned int`** (negative numbers wrap to ~4.29 billion).
* **Q:** In Usual Arithmetic Conversions on 64-bit Linux, what happens when you mix `long` (64-bit) and `unsigned int` (32-bit)?
**A:** `unsigned int` is converted to **`signed long`** (because 64-bit `long` can hold all 32-bit unsigned values).
* **Q:** Why is `for (size_t i = n - 1; i >= 0; i--)` an infinite loop?
**A:** `size_t` is unsigned; when `i == 0`, `i--` wraps around to `SIZE_MAX`, so `i >= 0` is always true.
* **Q:** What is the safe idiomatic way to loop backward with `size_t` from `n - 1` down to `0`?
**A:** `for (size_t i = n; i-- > 0; )`
* **Q:** How does `(int)x` handle negative floating-point numbers like `-3.9`?
**A:** Truncates **toward zero** → `-3` (not `-4`). Out-of-range float-to-int cast is Undefined Behavior.
* **Q:** Which pointer cast is explicitly exempt from C's Strict Aliasing Rule for inspecting raw bytes?
**A:** Casting to **`char *`** or **`unsigned char *`**.
* **Q:** Which GCC flags catch dangerous implicit signed/unsigned conversions at compile time?
**A:** `-Wall -Wextra -Wsign-conversion`.

---

#### 3. 60-Second Revision Block (Day 1 / 3 / 7 / 21)

* **Integer Promotion (Step 1):** `char` and `short` (even `unsigned` ones on 32-bit `int` systems) always promote to `int` before any math or bitwise op (`~`). Signed = Sign Extend; Unsigned = Zero Extend.
* **Usual Arithmetic Conversions (Step 2):** Float beats Int (`long double > double > float`). For mixed signed/unsigned integers: if `rank(U) >= rank(S)` → both become `U`; if `S` can hold all values of `U` → both become `S`.
* **`size_t` Underflow:** Unsigned `0 - 1` wraps to `SIZE_MAX`. Never use `i >= 0` on `size_t` loops (use `for (size_t i = n; i-- > 0; )`) and never compare `a - b > 0` on `size_t` (use `a > b`).
* **Explicit Casting:** Cast *before* dividing (`(double)a / b`). Float→Int truncates toward zero (UB if out of range). Never type-pun via `*(int *)&f` (Strict Aliasing UB); use `memcpy` or `unsigned char *`.