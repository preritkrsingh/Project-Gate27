### 1. THE 30-SECOND BIG PICTURE (Prime + Dual Coding)

* **1-Sentence Core Idea (Feynman):** An array in C is a contiguous block of identical elements whose name quietly decays into a typed pointer to element 0 in almost every expression, turning every subscript and string operation into scaled address arithmetic.
* **Master Map:**

```text
                    [ ARRAYS & STRINGS (GATE CORE) ]
                                   │
       ┌───────────────────────────┼───────────────────────────┐
       ▼                           ▼                           ▼
[ 1. 1D & DECAY ]          [ 2. 2D & DECLARATIONS ]    [ 3. C STRINGS ]
  ├─ Contiguous RAM          ├─ Row-Major Offset         ├─ '\0' Terminated
  ├─ Decay: a -> &a[0]       │   Base + (i*C + j)*sz     ├─ Stack char s[] (RW)
  │   (Except sizeof, &a)    ├─ 3 Pointer Levels         │   vs .rodata char *p (RO)
  └─ Subscript Rule          │   ├─ a  : int (*)[C]      ├─ sizeof (total B, CT)
      a[i] == *(a+i) == i[a] │   ├─ *a : int *           │   vs strlen (chars, RT)
                             │   └─ &a : int (*)[R][C]   └─ string.h (O(n) scans)
                             └─ int *a[N] (Arr of Ptr)       strcpy, strcmp, strcat
                                vs int (*a)[N] (Ptr->Arr)

```

---

### 2. COMPLETE COMPRESSED MODULES (Chunked + Active Recall)

#### M1. 1D Arrays, Decay Exceptions & Indexing Equivalence

* **Cue:** When does an array name decay into a pointer, when does it refuse to decay, and why is `i[a]` legal C?
* **Intuition (Feynman):** An array name is a fixed street sign for the whole block, which C automatically converts into the address of house #0 unless you ask for the whole block's size or address.
* **Mechanics:**
* **Contiguous Layout:** Element `i` sits strictly at `Base + i * sizeof(T)` (zero gaps between elements).
* **Decay Rule:** In expressions, `a` (`T[N]`) decays to `&a[0]` (`T *`, non-modifiable R-value) → `a++` and `a = p` are **compile errors**.
* **The 3 Decay Exceptions:** `a` stays full type `T[N]` inside: (1) `sizeof(a)` (returns `N * sizeof(T)`), (2) `&a` (yields `T (*)[N]`, where `&a + 1` jumps the **entire array**), and (3) string literal init (`char s[] = "hi"`).
* **Subscript Identity:** `a[i]` is purely syntactic sugar for `*(a + i)` → since `+` is commutative, `a[i] ≡ *(a + i) ≡ *(i + a) ≡ i[a]`.


* **Example:** `int a[4] = {10, 20, 30, 40};` → `2[a]` evaluates to `30`, and `*((int *)(&a + 1) - 1)` steps past `a`, backs up 1 `int`, and yields `40`.
* **Trap / Edge Case:** Inside a function parameter `void f(int a[10])`, `a` decays immediately to `int *a` → `sizeof(a)` gives pointer size (`4B`/`8B`), **not** `40B`, and `a++` becomes legal!

#### M2. 2D & Multi-Dimensional Storage Layout (Row-Major Math)

* **Cue:** How does C flatten a multi-dimensional array into 1D RAM, and what is the exact address formula?
* **Intuition (Feynman):** C stores a 2D table by laying Row 0 flat in memory, followed immediately by Row 1, Row 2, and so on (Row-Major Order).
* **Mechanics:**
* **2D Row-Major (0-indexed `T a[R][C]`):** `Addr(a[i][j]) = Base + (i * C + j) * sizeof(T)`.
* **2D Arbitrary Bounds (`[L1..U1][L2..U2]`):** Let `C = U2 - L2 + 1` → `Base + ((i - L1) * C + (j - L2)) * sizeof(T)`.
* **3D Row-Major (`T a[D1][D2][D3]`):** `Addr(a[i][j][k]) = Base + ((i * D2 + j) * D3 + k) * sizeof(T)` (multiply each index by the product of all **later** dimensions).
* **GATE Contrast (Column-Major — Fortran/MATLAB):** `Addr(a[i][j]) = Base + (j * R + i) * sizeof(T)` (swaps row/col priority).


* **Example:** `int a[4][5]` (4B `int`, Base `1000`): `Addr(a[2][3]) = 1000 + (2 * 5 + 3) * 4 = 1052`.
* **Trap / Edge Case:** Forgetting `R` is unused in 0-indexed Row-Major offset math (`i * C + j` uses only `C`), whereas Column-Major uses only `R` (`j * R + i`).

#### M3. 2D Array Pointer Hierarchy (`a`, `*a`, `&a`) & Function Passing

* **Cue:** For `int a[R][C]`, why do `a`, `*a`, `&a`, and `&a[0][0]` print the exact same address yet jump wildly different distances on `+ 1`?
* **Intuition (Feynman):** All four point to the same starting byte, but each wears a different "stride lens": whole matrix, single row, or single integer.
* **Mechanics:**
* **`a` (decayed):** Type `int (*)[C]` → `a + 1` jumps **1 full row** (`C * sizeof(int)` bytes).
* **`*a` (or `a[0]`):** Decays from `int[C]` to `int *` → `*a + 1` jumps **1 element** (`sizeof(int)` bytes).
* **`&a`:** No decay → Type `int (*)[R][C]` → `&a + 1` jumps the **entire 2D array** (`R * C * sizeof(int)` bytes).
* **Double Dereference & Passing:** `a[i][j] ≡ *(*(a + i) + j)`; passing `a` to a function requires all dimensions except the first (`void f(int a[][C])` or `void f(int (*a)[C])`) so the compiler knows the row stride `C`.


* **Example:** `int a[3][4]` at `1000` (4B `int`): `a + 1 = 1016` (+16B), `*a + 1 = 1004` (+4B), `&a + 1 = 1048` (+48B), `*(*(a + 1) + 2) = a[1][2]`.
* **Trap / Edge Case:** `a[i]` is an address (pointer to row `i`'s first element), **not** an integer value; also `int **p = a;` is a compile error because `int (*)[C]` is incompatible with `int **`.

#### M4. Declaration Disambiguation (`int *a[N]` vs `int (*a)[N]`)

* **Cue:** How do you instantly distinguish an array of pointers from a pointer to an array?
* **Intuition (Feynman):** Bracket `[]` outranks star `*`, so without parentheses you get an array holding pointers; parentheses force `*` first to make one pointer aiming at a whole array.
* **Mechanics:**
* **Array of Pointers (`int *a[N]`):** `a` is an array of `N` elements, each of type `int *` → `sizeof(a) = N * sizeof(int *)`; `a + 1` steps by `sizeof(int *)` (`4B`/`8B`). Ideal for jagged/variable-length rows (`char *argv[]`).
* **Pointer to an Array (`int (*a)[N]`):** `a` is a **single pointer** pointing to an `int[N]` block → `sizeof(a) = 4B/8B`, `sizeof(*a) = N * sizeof(int)`; `a + 1` steps by `N * sizeof(int)`. Ideal for traversing contiguous 2D arrays.
* **Binding Rule:** Read from the identifier outward: go **right** first (`[]`, `()`) until blocked by `)`, then go **left** (`*`).


* **Example:** `int mat[3][5]; int (*p)[5] = mat;` (`p + 1` moves to `mat[1]`) vs `char *days[3] = {"Mon", "Wed", "Fri"};` (`days` stores 3 separate string addresses).
* **Trap / Edge Case:** Confusing `sizeof(p)` with `sizeof(*p)` for `int (*p)[5]` — `p` itself is just one pointer (`8B` on 64-bit), while `*p` is the 5-int array (`20B`).

#### M5. Strings in C: Stack Arrays vs Literals & `sizeof` vs `strlen`

* **Cue:** How do `char s[] = "abc"` and `char *p = "abc"` differ in memory, mutability, and size measurement?
* **Intuition (Feynman):** A C string is just a `char` array ending with an invisible `'\0'` stop sign; `s[]` makes your own editable copy on the stack, while `*p` only points at a read-only master copy.
* **Mechanics:**
* **Null Termination:** Every valid C string ends with `'\0'` (ASCII `0`, 1 byte) → storing `k` visible characters requires at least `k + 1` bytes.
* **Mutable Stack Array (`char s[] = "abc";`):** Allocates 4 writable bytes `{'a','b','c','\0'}` on Stack → `s[0] = 'X'` is **valid**, `s++` is a **compile error**.
* **Read-Only Literal (`char *p = "abc";`):** Literal lives in `.rodata` (Read-Only Data); `p` sits on Stack → `p++` is **valid**, `*p = 'X'` is **Undefined Behaviour** (segfault).
* **`sizeof` vs `strlen`:** `sizeof` is a **compile-time** `O(1)` operator returning allocated bytes (includes `'\0'`; gives `4B`/`8B` for `char *`); `strlen` is a **runtime** `O(n)` function counting chars strictly before the **first** `'\0'`.


* **Example:** `char s[10] = "GATE";` → `sizeof(s) == 10`, `strlen(s) == 4`; for `char *p = "GATE";` (64-bit) → `sizeof(p) == 8`, `strlen(p) == 4`.
* **Trap / Edge Case:** `char s[3] = "abc";` compiles in C but drops `'\0'` → passing `s` to `printf("%s", s)` or `strlen(s)` reads past the array into garbage (**UB**).

#### M6. Core `string.h` Mechanics (`strcpy`, `strcmp`, `strcat`)

* **Cue:** What exact work, return value, time complexity, and failure mode govern `strcpy`, `strcmp`, and `strcat`?
* **Intuition (Feynman):** Standard C string functions have zero length metadata—they blindly march byte-by-byte until they hit `'\0'`.
* **Mechanics:**
* **`strcpy(dst, src)`:** Copies bytes from `src` to `dst` **including `'\0'**`, returns `dst`, runs in `O(|src|)` → **UB** if `dst` overflows or `src`/`dst` overlap.
* **`strcmp(s1, s2)`:** Compares unsigned-char values left-to-right until first mismatch or `'\0'` → returns `0` if equal, **negative** if `s1` precedes `s2`, **positive** if `s1` follows `s2` (C guarantees only the **sign**, not exact `-1`/`+1`).
* **`strcat(dst, src)`:** Walks `dst` to its `'\0'`, overwrites it with `src` + new `'\0'`, returns `dst` → takes `O(|dst| + |src|)` per call.
* **Pointer Idioms:** `while (*dst++ = *src++);` is the classic 1-line `strcpy` (stops when `'\0'` is copied and evaluates to `0`).


* **Example:** `strcmp("Apple", "Banana")` returns negative (`'A'` is 65, `'B'` is 66); `if (strcmp(a, b) == 0)` checks equality.
* **Trap / Edge Case:** Writing `if (strcmp(a, b))` to test equality is backwards (`0` is falsy in C, so the `if` enters when strings are **unequal**); also `strncpy(dst, src, n)` does **not** append `'\0'` if `strlen(src) ≥ n`.

---

### 3. CONFUSION-KILLER TABLE (Interleaving)

| Concept | Core Difference | When to Use / Trigger | Common Trap |
| --- | --- | --- | --- |
| **`a + 1` vs `&a + 1` (1D `int a[N]`)** | `a + 1` steps by 1 element (`sizeof(int)`); `&a + 1` steps past whole array (`N * sizeof(int)`). | Element traversal vs finding one-past-the-end array boundary. | Thinking `&a` and `a` have identical types (same value, different type!). |
| **`a` vs `*a` vs `&a` (2D `int a[R][C]`)** | `a`: `int (*)[C]` (+1 row); `*a`: `int *` (+1 elem); `&a`: `int (*)[R][C]` (+whole matrix). | Tracing 2D pointer arithmetic in GATE NATs. | Assuming `a + 1` moves to `a[0][1]` (it actually moves to `a[1][0]`). |
| **`int *a[N]` vs `int (*a)[N]**` | `int *a[N]` = array of `N` pointers; `int (*a)[N]` = 1 pointer to an `N`-int array. | Jagged arrays (`char *argv[]`) vs contiguous 2D matrix traversal. | Miscomputing `sizeof`: `N * ptr_size` vs `ptr_size`. |
| **`int a[R][C]` vs `int **p**` | `a` is 1 contiguous block (`R*C` ints); `p` points to an array of separate row pointers. | Compile-time matrix vs dynamically allocated jagged rows. | Passing `int a[3][4]` into `void f(int **p)` (compile error). |
| **`char s[] = "hi"` vs `char *p = "hi"**` | `s[]` = writable Stack array (`s++` illegal); `*p` = pointer to `.rodata` (`*p = 'X'` is UB). | In-place string edits vs read-only constant messages. | Mutating a string literal via `char *p` (causes segfault/UB). |
| **`sizeof(s)` vs `strlen(s)**` | `sizeof` = compile-time total bytes (incl. `'\0'`, or ptr size); `strlen` = runtime chars before first `'\0'`. | Buffer capacity check vs logical string length. | Calling `sizeof(p)` on `char *p` expecting string length (returns `4`/`8`). |
| **`strcmp(a, b) == 0` vs `a == b**` | `strcmp` compares character contents; `a == b` compares starting **memory addresses**. | Checking whether two strings hold identical text. | Using `if (s1 == s2)` on two stack arrays (always false!). |
| **`strcpy` vs `strncpy**` | `strcpy` always copies `'\0'` (overflow risk); `strncpy` bounds to `n` bytes (may omit `'\0'`). | Safe vs unsafe string duplication. | Forgetting to manually set `dst[n - 1] = '\0'` after `strncpy`. |

---

### 4. RAPID LOCK-IN (Active Recall + Spaced Repetition)

#### 60-Second Self-Test

1. **Whole-Array Jump:** `int a[] = {10, 20, 30, 40}; int *p = (int *)(&a + 1); printf("%d", *(p - 1));` — What prints?
2. **2D Pointer Strides:** `int a[4][5];` starts at address `2000` (4-byte `int`). Give the exact addresses of **(i) `a + 1**`, **(ii) `*a + 1**`, and **(iii) `&a + 1**`.
3. **Commutative Subscript:** `char s[] = "GATE"; printf("%c", 2[s]);` — Valid or error? If valid, what prints?
4. **Declaration Size Check (64-bit, 4B `int`):** Given `int *x[5];` and `int (*y)[5];`, what are `sizeof(x)`, `sizeof(y)`, and `sizeof(*y)`?
5. **Embedded Null & Pointer Size (64-bit):** Given `char s[] = "AB\0DEF"; char *p = "AB\0DEF";`, compute `sizeof(s)`, `strlen(s)`, and `sizeof(p)`.
6. **Mutability & L-Value Audit:** Given `char a[] = "hi"; char *p = "hi";`, classify each as **Valid**, **Compile Error**, or **UB**: `(1) a[0] = 'H';` `(2) a++;` `(3) p[0] = 'H';` `(4) p++;`.
7. **`string.h` Complexity & Logic:** Why does `if (strcmp(a, b))` execute its body when `a` and `b` are **different**, and what is the time complexity of calling `strcat(buf, "x")` inside a loop `n` times?

#### Spaced-Repetition Flashcards (Day 1 / 3 / 7)

* `Q: List the 3 contexts in C where an array name 'a' does NOT decay to &a[0]. | A: Inside sizeof(a), inside &a, and when initializing a char array with a string literal.`
* `Q: Why is i[a] strictly equivalent to a[i] in C? | A: By definition a[i] == *(a + i), and pointer-integer addition is commutative: *(a + i) == *(i + a) == i[a].`
* `Q: What is the 0-indexed Row-Major address formula for a[i][j] in T a[R][C]? | A: Base + (i * C + j) * sizeof(T).`
* `Q: State the exact types and +1 byte strides of a, *a, and &a for int a[R][C]. | A: a: int (*)[C] (+C*4B); *a: int * (+4B); &a: int (*)[R][C] (+R*C*4B).`
* `Q: Which dimensions must be specified when passing a 2D array to a C function? | A: Every dimension except the first (e.g., void f(int a[][C]) or void f(int (*a)[C])).`
* `Q: Decode int *a[N] vs int (*a)[N]. | A: int *a[N] is an array of N int pointers; int (*a)[N] is a single pointer to an array of N ints.`
* `Q: Where do char s[] = "abc" and char *p = "abc" store their characters, and which is mutable? | A: s[] copies chars onto Stack (mutable); p points to Read-Only Data / .rodata (modifying *p is UB).`
* `Q: How do sizeof and strlen differ in timing and handling of '\0'? | A: sizeof is compile-time total allocated bytes (includes '\0'); strlen is runtime O(n) char count before first '\0'.`
* `Q: What does strcmp(s1, s2) return on equality, and what does it actually guarantee on inequality? | A: 0 on equality; only the sign (negative if s1 precedes s2, positive if s1 follows s2), not strictly -1/+1.`
* `Q: Why is char s[3] = "abc"; dangerous when passed to strlen or printf("%s")? | A: It fits only 3 chars and omits the terminating '\0', causing out-of-bounds memory reads (UB).`

#### Answer Key

1. **`40`** (`&a + 1` points right past `a[3]`; after casting to `int *`, `p - 1` backs up 1 `int` to `a[3]`).
2. **(i) `2020**` (`+5*4B`, 1 row); **(ii) `2004**` (`+4B`, 1 int); **(iii) `2080**` (`+4*5*4B`, entire 2D array).
3. **Valid; prints `'T'**` (`2[s]` translates to `*(2 + s)` ≡ `s[2]`).
4. **`sizeof(x) = 40`** (`5 * 8B`), **`sizeof(y) = 8`** (1 pointer), **`sizeof(*y) = 20`** (`5 * 4B`).
5. **`sizeof(s) = 7`** (`'A','B','\0','D','E','F','\0'`), **`strlen(s) = 2`** (stops at first `'\0'`), **`sizeof(p) = 8`** (64-bit pointer).
6. `(1)` **Valid**, `(2)` **Compile Error** (`a` is not a modifiable L-value), `(3)` **UB** (writing to `.rodata`), `(4)` **Valid**.
7. `strcmp` returns `0` (falsy) on match and non-zero (truthy) on mismatch; `n` repeated `strcat` calls scan from index 0 each time → `1 + 2 + ... + n = O(n^2)`.