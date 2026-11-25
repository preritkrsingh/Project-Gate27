### LAYER 1: THE SPEED LAYER

#### 1. Core Intuition (Feynman)

RAM is just a giant row of raw 0s and 1s—it has no idea whether 4 bytes represent the number `65`, the letter `'A'`, or a decimal fraction. C's primitive types are **lenses** that tell the compiler two things: **how many bytes** to read, and **which math rule** to use to interpret those bits (plain binary, 2's complement, or scientific notation).

#### 2. Master Mental Map

```text
                        C PRIMITIVE TYPES (Bits in RAM)
                                      │
          ┌───────────────────────────┴───────────────────────────┐
          ▼                                                       ▼
   INTEGER TYPES                                        FLOATING-POINT (IEEE 754)
(Exact discrete values)                                (Approximate scientific notation)
          │                                                       │
   ┌──────┴────────────────────┐                           ┌──────┴──────┐
   ▼                           ▼                           ▼             ▼
UNSIGNED                    SIGNED                       float        double
(0 to 2^n - 1)       (-2^(n-1) to 2^(n-1) - 1)          (32-bit)     (64-bit)
Modulo wrap-around      2's Complement                  1+8+23 bits  1+11+52 bits
(Well-defined)          Overflow = UB!                  ~7 digits    ~15-17 digits
   │
   └──► Sizes: char (1B) ≤ short (≥2B) ≤ int (≥2B/4B) ≤ long (≥4B) ≤ long long (≥8B)
               Measured at COMPILE TIME by sizeof() → returns size_t

```

#### 3. 80/20 High-Yield Cheat Sheet

* **The Size Hierarchy Guarantee:** `1 == sizeof(char) <= sizeof(short) <= sizeof(int) <= sizeof(long) <= sizeof(long long)`.
* **Modern 64-bit Linux (LP64) Sizes:** `char` = 1 B, `short` = 2 B, `int` = 4 B, `long` = 8 B, `long long` = 8 B, `float` = 4 B, `double` = 8 B. *(Warning: On 64-bit Windows/LLP64, `long` is **4 B**! Use `<stdint.h>` types like `int32_t`, `uint64_t` for portable code.)*
* **Range Formulas ($n$ bits):**
* **Unsigned:** $0 \text{ to } 2^n - 1$
* **Signed (2's Complement):** $-2^{n-1} \text{ to } 2^{n-1} - 1$


* **The Overflow Split:** Unsigned overflow wraps around cleanly ($\bmod 2^n$). **Signed overflow is Undefined Behavior (UB)**—the compiler can legally break your logic if you rely on it.
* **`sizeof` Rule:** `sizeof` is a **compile-time operator** (not a function) that returns `size_t` (unsigned). Expressions inside `sizeof(x++)` are **never executed** at runtime (except for C99 Variable-Length Arrays).

---

### LAYER 2: EXHAUSTIVE DEEP-DIVE

#### Module 1: Why aren't C integer sizes fixed across platforms, and what does the C standard actually guarantee?

* **Plain-English Intuition (Feynman):**
C was designed to run fast on everything from 8-bit microcontrollers to 64-bit servers. Instead of forcing a tiny chip to emulate huge numbers, C lets `int` match the CPU's "natural" register size and only enforces *minimum* widths.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Minimum Widths (ISO C Standard):**
* `char`: **exact 1 byte** (guaranteed $\ge 8$ bits, `CHAR_BIT` in `<limits.h>`).
* `short`: at least **16 bits** (2 bytes).
* `int`: at least **16 bits** (typically **32 bits / 4 bytes** on modern 32-bit and 64-bit CPUs).
* `long`: at least **32 bits** (**8 bytes** on 64-bit Linux/macOS LP64; **4 bytes** on 64-bit Windows LLP64).
* `long long`: at least **64 bits** (8 bytes).


2. **The `char` Signedness Quirk:** Unlike `int` (which is always `signed int` by default), plain `char` is **implementation-defined**—it is `signed` on x86-64 Linux, but `unsigned` on ARM64 Linux.
3. **Fixed-Width Portability (`<stdint.h>`):** To eliminate platform guesswork in networking, file formats, or binary protocols, C99 provides exact-width typedefs: `int8_t`, `uint8_t`, `int32_t`, `uint64_t`.


* **Visual / Structure (Dual Coding):**

```text
Type          Min Bits   Linux x86-64 (LP64)   Windows x64 (LLP64)
──────────    ────────   ───────────────────   ───────────────────
char          8          1 byte                1 byte
short         16         2 bytes               2 bytes
int           16         4 bytes               4 bytes
long          32         8 bytes  ◄── DIFF!    4 bytes  ◄── DIFF!
long long     64         8 bytes               8 bytes
void *        -          8 bytes               8 bytes

```

* **Concrete Examples:**
1. **Cross-Platform Bug:** Storing a 64-bit Unix timestamp in a `long` works on Linux (`8 bytes`) but silently truncates/overflows when compiled on Windows (`4 bytes`). Using `int64_t` guarantees 8 bytes everywhere.
2. **`getchar()` Return Type:** `getchar()` returns `int`, **not** `char`, because it must return all 256 possible unsigned byte values (`0`–`255`) *plus* `EOF` (`-1`). Storing `getchar()` in a `char` breaks `EOF` detection on ARM (where `char` is unsigned and can never equal `-1`).


* **Traps & Edge Cases (Warning):**
* **Assuming `sizeof(long) == sizeof(void *)`:** True on Linux x86-64, **false** on Windows x64. Always use `uintptr_t` or `intptr_t` from `<stdint.h>` if you must hold a pointer address as an integer.



---

#### Module 2: How does 2's Complement represent signed vs. unsigned integers, and why is the signed range asymmetric?

* **Plain-English Intuition (Feynman):**
In unsigned binary, every bit adds a positive power of 2. In 2's complement (signed), the **leftmost bit (MSB)** is simply given a **negative weight** ($-2^{n-1}$). This clever trick lets the CPU use the *exact same hardware adder circuit* for both positive and negative numbers with zero duplicate zeros.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Unsigned ($n$-bit):** Value $= \sum_{i=0}^{n-1} b_i \cdot 2^i$. Range: **$0$ to $2^n - 1$**.
2. **Signed 2's Complement ($n$-bit):** Value $= -b_{n-1} \cdot 2^{n-1} + \sum_{i=0}^{n-2} b_i \cdot 2^i$.
* **MSB = 0** → Non-negative ($0$ to $2^{n-1} - 1$).
* **MSB = 1** → Negative ($-2^{n-1}$ to $-1$).
* **Negation Shortcut:** To compute $-x$, **flip all bits and add 1** (`~x + 1`).


3. **Why Asymmetric (`-128` to `+127`, not `-127` to `+127`):** There are $2^n$ (an even number of) bit patterns. Zero (`00000000`) uses up one pattern from the non-negative half (MSB = `0`), leaving $2^{n-1} - 1$ positive numbers, while the entire negative half (MSB = `1`) holds $2^{n-1}$ negative numbers.
4. **Overflow Rules:**
* **Unsigned:** Strictly defined modulo $2^n$. For 8-bit `uint8_t`: `255 + 1 == 0`, and `0 - 1 == 255`.
* **Signed:** **Undefined Behavior (UB)**. While x86 hardware wraps `INT_MAX + 1` to `INT_MIN`, the C compiler optimizer assumes signed overflow never happens and may delete your overflow checks.




* **Visual / Structure (Dual Coding — 8-bit Example):**

```text
Bit Pattern   Weight: -128  64  32  16   8   4   2   1    Unsigned   Signed (2's Comp)
───────────           ────────────────────────────────    ────────   ─────────────────
0000 0000                0   0   0   0   0   0   0   0           0                   0
0000 0001                0   0   0   0   0   0   0   1           1                  +1
0111 1111                0   1   1   1   1   1   1   1         127                +127 (INT8_MAX)
1000 0000                1   0   0   0   0   0   0   0         128                -128 (INT8_MIN)
1111 1111                1   1   1   1   1   1   1   1         255                  -1

```

* **Concrete Examples:**
1. **Computing `-5` in 8-bit 2's Complement:**
* `+5` = `0000 0101`
* Flip bits (`~5`) = `1111 1010`
* Add 1 = `1111 1011` → $-128 + 64 + 32 + 16 + 8 + 2 + 1 = -5$.


2. **The `INT_MIN` Negation Bug:** For 32-bit `int`, `INT_MIN` is `-2,147,483,648`, while `INT_MAX` is `+2,147,483,647`. Calling `abs(INT_MIN)` or `-INT_MIN` overflows because `+2,147,483,648` does not fit in a 32-bit signed `int`.


* **Traps & Edge Cases (Warning):**
* **Signed-to-Unsigned Promotion Trap:**
```c
int a = -1;
unsigned int b = 1;
if (a < b) { /* NEVER EXECUTES! */ }

```


When mixing `int` and `unsigned int` of the same rank, C's **Usual Arithmetic Conversions** silently convert `a` (`-1`) to `unsigned int` (`4,294,967,295`). Since `4294967295 < 1` is **false**, the `if` block is skipped. This is a classic source of buffer-size security vulnerabilities.



---

#### Module 3: How do `float` and `double` store decimals in binary (IEEE 754), and why are they imprecise?

* **Plain-English Intuition (Feynman):**
Floating-point numbers are just **binary scientific notation** ($\pm 1.\text{fraction} \times 2^{\text{exponent}}$). Just as $\frac{1}{3}$ repeats forever in base-10 (`0.3333...`), simple base-10 fractions like `0.1` ($\frac{1}{10}$) repeat forever in base-2 (`0.0001100110011...`). Because a `float` or `double` has finite bits, it has to chop and round the repeating tail.
* **Exact Mechanics (Elaborative Interrogation):**
IEEE 754 packs three fields into the bits:

$$\text{Value} = (-1)^{\text{sign}} \times (1 + \text{mantissa}) \times 2^{(\text{exponent} - \text{bias})}$$


1. **Sign Bit (1 bit):** `0` = positive, `1` = negative.
2. **Biased Exponent:** Stored as an unsigned integer shifted by a **bias** (`127` for `float`, `1023` for `double`) so exponents can be compared quickly without 2's complement.
3. **Mantissa / Significand (Fraction):** Stores only the fractional bits *after* the binary point; the leading `1.` is **implicit** (hidden bit), giving 1 extra bit of free precision.
4. **Special Bit Patterns:**
* Exponent all `0`s + Mantissa `0` → `+0.0` or `-0.0`.
* Exponent all `1`s + Mantissa `0` → `+Inf` or `-Inf` (e.g., `1.0 / 0.0`).
* Exponent all `1`s + Mantissa non-zero → **`NaN`** (Not a Number, e.g., `0.0 / 0.0` or `sqrt(-1.0)`).




* **Visual / Structure (Dual Coding):**

```text
float (32 bits, ~7 decimal digits, Bias = 127):
┌───┬────────────┬───────────────────────────────────┐
│ S │  Exponent  │             Mantissa              │
│ 1 │   8 bits   │             23 bits               │
└───┴────────────┴───────────────────────────────────┘

double (64 bits, ~15–17 decimal digits, Bias = 1023):
┌───┬───────────────┬────────────────────────────────────────────────────┐
│ S │   Exponent    │                      Mantissa                      │
│ 1 │    11 bits    │                      52 bits                       │
└───┴───────────────┴────────────────────────────────────────────────────┘

```

* **Concrete Examples:**
1. **Why `0.1 + 0.2 == 0.3` is False:** `0.1` and `0.2` cannot be represented exactly as sums of powers of $2^{-k}$. Their rounded binary sum equals `0.3000000000000000444...`, which differs from the binary representation of `0.3`. Always compare floats using an epsilon tolerance: `fabs(a - b) < 1e-9`.
2. **Financial Systems Rule:** Never store currency (₹ or $) in `float`/`double`. Store exact integer **paise/cents** in `int64_t` (e.g., ₹199.50 → `19950`).


* **Traps & Edge Cases (Warning):**
* **`NaN != NaN`:** By IEEE 754 mandate, `NaN` is never equal to anything—including itself. `if (x == NaN)` is always **false**. Use `isnan(x)` from `<math.h>`.
* **Large Integer Loss of Precision:** A 32-bit `float` only has 24 bits of significand precision (23 stored + 1 implicit). Above $2^{24}$ (`16,777,216`), `float` can no longer represent every consecutive integer—`16777216.0f + 1.0f == 16777216.0f`!



---

#### Module 4: How does the `sizeof` operator work at compile time, and what are its hidden traps?

* **Plain-English Intuition (Feynman):**
`sizeof` looks like a function, but it's actually a **compile-time ruler**. During Stage 2 (Compilation), the compiler looks at the *type* of whatever you pass to `sizeof`, swaps the entire `sizeof(...)` expression with a hard-coded integer constant (like `4` or `40`), and never runs any code inside the parentheses.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Compile-Time Evaluation:** Because C is statically typed, the compiler knows the byte size of every type, struct, and fixed-size array at Stage 2. `sizeof(int)` becomes the literal constant `4` in the generated `.s` assembly—zero runtime CPU instructions.
2. **Return Type:** Always returns **`size_t`** (an `unsigned long` on 64-bit Linux). Print it with `%zu` in `printf`.
3. **Syntax Rule:** Parentheses are **required** for types (`sizeof(int)`), but **optional** for expressions/variables (`sizeof x` is valid C).
4. **The One Runtime Exception (C99 VLAs):** If you pass a **Variable-Length Array** whose size isn't known until runtime (`int arr[n];`), `sizeof(arr)` *must* be computed at runtime (`n * sizeof(int)`).


* **Visual / Structure (Dual Coding):**

```text
C Source Code                       Generated Assembly Concept (Stage 2)
─────────────────────────────       ────────────────────────────────────
int x = 5;                          movl  $5, -4(%rbp)
size_t s = sizeof(x++);     ──►     movq  $4, -16(%rbp)   // x++ is DELETED!
printf("%d %zu", x, s);             // Prints: 5 4  (x was NEVER incremented!)

```

* **Concrete Examples:**
1. **Array Element Count Idiom:**
```c
int arr[10];
size_t count = sizeof(arr) / sizeof(arr[0]); // 40 / 4 = 10

```


Works cleanly in the scope where `arr` is declared because `arr` is one of the few places where an array does **not** decay into a pointer.
2. **Pointer vs. Pointee Size:**
```c
int *p = malloc(100 * sizeof(int));
sizeof(p);  // Returns 8 (size of the pointer on 64-bit), NOT 400!
sizeof(*p); // Returns 4 (size of one int)

```




* **Traps & Edge Cases (Warning):**
* **Array Decay Inside Functions:**
```c
void print_len(int arr[]) {
    printf("%zu\n", sizeof(arr)); // Prints 8 (pointer size), NOT array size!
}

```


In a function parameter list, `int arr[]` is silently rewritten by the compiler to `int *arr`. `sizeof(arr)` inside the function always returns the size of a pointer (8 bytes on 64-bit), never the caller's array size. Always pass array length as a separate `size_t len` parameter.



---

### LAYER 3: SYNTHESIS & INTERLEAVING

#### Compare & Contrast Table

| Confused Pair | Option A | Option B | Decisive Rule / Diagnostic |
| --- | --- | --- | --- |
| **`signed` vs. `unsigned` Overflow** | **Signed:** Undefined Behavior (UB). Optimizer may break checks. | **Unsigned:** Well-defined wrap-around ($\bmod 2^n$). | Never test `if (a + b < a)` on signed `int`s; check `if (a > INT_MAX - b)` before adding. |
| **`float` vs. `double**` | **`float`:** 4 bytes, 23-bit mantissa (~7 digits). | **`double`:** 8 bytes, 52-bit mantissa (~15–17 digits). | Default to `double` in C; only use `float` when memory bandwidth/SIMD throughput matters (graphics, ML tensors). |
| **`sizeof(arr)` vs. `sizeof(ptr)**` | **`arr` (in declaring scope):** Total bytes of entire array (`N * elem_size`). | **`ptr` (or array passed to func):** Always pointer width (`8` bytes on 64-bit). | Arrays decay to pointers in function parameters; `sizeof` cannot recover heap (`malloc`) or parameter array sizes. |
| **`sizeof` vs. `strlen()**` | **`sizeof("abc")`:** Compile-time operator → **`4`** (includes `\0`). | **`strlen("abc")`:** Runtime O(n) function → **`3`** (excludes `\0`). | `sizeof` measures allocated storage capacity; `strlen` counts characters until the first `\0` byte. |

#### Cross-Connections

* **Build Pipeline (Stage 2) → `sizeof`:** Because `sizeof` resolves during **Compilation (Stage 2)** using the AST's semantic type info, `sizeof(x++)` produces zero assembly instructions for `x++` in `.s`.
* **Process Memory → `sizeof` & Types:** Every primitive type's size directly determines how many bytes are carved out of `.data`, `.bss`, Stack (`subq`), or Heap (`malloc(n * sizeof(T))`). Mixing signed/unsigned sizes in `malloc(len)` (where `len` is negative and converts to a massive `size_t`) is a textbook heap overflow vulnerability.

---

### LAYER 4: LOCK-IN LAYER

#### 1. Blank-Page Self-Test

Answer all 7 before opening the key.

1. **Range Math:** What are the exact min and max values of a 16-bit `signed short` vs. a 16-bit `unsigned short`?
2. **2's Complement:** Convert `-1` and `-128` into 8-bit binary (2's complement).
3. **Conversion Trap:** What does this print, and why?
```c
int x = -5;
unsigned int y = 2;
printf("%s\n", (x > y) ? "YES" : "NO");

```


4. **IEEE 754:** Why does `float` lose the ability to represent every consecutive integer starting at $16,777,216$ ($2^{24}$)?
5. **Compile-Time Evaluation:** What does this print?
```c
int a = 10;
size_t s = sizeof(a = 99);
printf("a=%d, s=%zu\n", a, s);

```


6. **Array Decay Trap:** On a 64-bit Linux machine, `int nums[20];` is passed to `void f(int nums[20])`. What is `sizeof(nums)` in `main()` vs. inside `f()`?
7. **`sizeof` vs. `strlen`:** For `char str[10] = "hi";`, what do `sizeof(str)` and `strlen(str)` return?

1. `signed short`: **$-32,768$ to $+32,767$** ($-2^{15}$ to $2^{15}-1$). `unsigned short`: **$0$ to $65,535$** ($0$ to $2^{16}-1$).
2. `-1` = **`1111 1111`**; `-128` = **`1000 0000`**.
3. Prints **`YES`**. Mixing `int` and `unsigned int` promotes `x` (`-5`) to `unsigned int` ($2^{32} - 5 = 4,294,967,291$), which is greater than `2`.
4. `float` has 23 explicit mantissa bits + 1 implicit leading `1` bit = **24 bits of significand precision**. Beyond $2^{24}$, the gap between adjacent representable floats jumps from `1` to `2`.
5. Prints **`a=10, s=4`** (assuming 4-byte `int`). `sizeof` is evaluated at compile time, so `a = 99` is never executed.
6. In `main()`: **`80`** ($20 \times 4$). Inside `f()`: **`8`** (the parameter decays to `int *`, so `sizeof` measures the 64-bit pointer).
7. `sizeof(str)` = **`10`** (total allocated array capacity). `strlen(str)` = **`2`** (characters before `\0`).

---

#### 2. Anki Flashcards

* **Q:** What is the exact C standard size hierarchy for integer types?
**A:** `1 == sizeof(char) <= sizeof(short) <= sizeof(int) <= sizeof(long) <= sizeof(long long)`.
* **Q:** Why is `long` dangerous for cross-platform 64-bit code?
**A:** It is **8 bytes** on 64-bit Linux/macOS (LP64) but **4 bytes** on 64-bit Windows (LLP64). Use `int64_t` from `<stdint.h>`.
* **Q:** What is the range of an $n$-bit unsigned vs. signed (2's complement) integer?
**A:** Unsigned: $0 \text{ to } 2^n - 1$. Signed: $-2^{n-1} \text{ to } 2^{n-1} - 1$.
* **Q:** How does signed integer overflow differ from unsigned integer overflow in C?
**A:** Unsigned wraps around modulo $2^n$ (well-defined); Signed is **Undefined Behavior (UB)**.
* **Q:** How do you negate a number in 2's complement binary?
**A:** Flip all bits and add 1 (`~x + 1`).
* **Q:** What are the bit splits (Sign / Exponent / Mantissa) and biases for IEEE 754 `float` and `double`?
**A:** `float`: 1 / 8 / 23 (bias 127). `double`: 1 / 11 / 52 (bias 1023).
* **Q:** Why is `0.1 + 0.2 == 0.3` false in C?
**A:** `0.1` and `0.2` are infinite repeating fractions in base-2 and must be rounded to fit finite mantissa bits.
* **Q:** What does `sizeof(x++)` do to `x`, and what type does `sizeof` return?
**A:** It does **not** increment `x` (unevaluated at compile time); it returns **`size_t`** (print with `%zu`).
* **Q:** What is the ONLY case in C where `sizeof` is evaluated at runtime?
**A:** C99 **Variable-Length Arrays (VLAs)** (e.g., `int arr[n]; sizeof(arr)`).
* **Q:** What is the difference between `sizeof("abc")` and `strlen("abc")`?
**A:** `sizeof("abc")` = `4` (compile-time, includes `\0`); `strlen("abc")` = `3` (runtime, excludes `\0`).

---

#### 3. 60-Second Revision Block (Day 1 / 3 / 7 / 21)

* **Sizes:** `char` (1B) ≤ `short` (2B) ≤ `int` (4B) ≤ `long` (8B Linux / 4B Windows) ≤ `long long` (8B). Use `<stdint.h>` (`int32_t`, `uint64_t`) for exact widths.
* **2's Complement:** MSB has weight $-2^{n-1}$. Range is asymmetric ($-2^{n-1}$ to $2^{n-1}-1$). Unsigned overflow wraps ($\bmod 2^n$); signed overflow is **UB**. Beware mixing signed & unsigned in comparisons (`-1 > 1u` is true!).
* **IEEE 754:** `float` (1+8+23, ~7 digits) and `double` (1+11+52, ~15–17 digits). Base-10 fractions like `0.1` repeat in binary—never use `==` on floats (use `fabs(a-b) < eps`), and never use floats for money.
* **`sizeof`:** Compile-time operator returning `size_t` (`%zu`). Operands like `x++` are **never executed** (except VLAs). Arrays decay to pointers in function params, so `sizeof(arr)` inside a function returns pointer size (8B).