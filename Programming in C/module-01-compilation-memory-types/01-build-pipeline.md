### LAYER 1: THE SPEED LAYER

#### 1. Core Intuition (Feynman)

A computer chip cannot read C code; it can only execute raw binary numbers representing exact hardware instructions and memory addresses. Turning your C files into a runnable program is a 4-station assembly line where each station strips away one layer of human convenience: first expanding text shortcuts, then checking grammar and translating to chip-specific instructions, then converting those instructions to raw bytes with blank address slots, and finally stitching all files together and filling in the blanks.

#### 2. Master Mental Map

```text
  file1.c (+ headers .h)                  file2.c (+ headers .h)
         │                                       │
         ▼  [1. PREPROCESSOR: cpp / gcc -E]      ▼
     file1.i  (Pure expanded C text)         file2.i
         │                                       │
         ▼  [2. COMPILER: cc1 / gcc -S]          ▼
     file1.s  (CPU-specific Assembly)        file2.s
         │                                       │
         ▼  [3. ASSEMBLER: as / gcc -c]          ▼
     file1.o  (Machine code + blank addrs)   file2.o      +  libc.a / libc.so
         │                                       │                │
         └───────────────────┬───────────────────┴────────────────┘
                             ▼  [4. LINKER: ld / gcc]
                     a.out / executable  (Final patched binary)
                             │
                             ▼  [OS LOADER + ld-linux.so]
                        Running Process

```

#### 3. 80/20 High-Yield Cheat Sheet

* **The GCC Command Ladder:**
* `gcc -E main.c -o main.i` → Stop after **Preprocessing** (inspect macros & includes).
* `gcc -S main.c -o main.s` → Stop after **Compilation** (inspect assembly).
* `gcc -c main.c -o main.o` → Stop after **Assembly** (output relocatable object file).
* `gcc main.o util.o -o app` → Run **Linking** (produce final executable).


* **Error Diagnosis Rule:**
* **Syntax / Type / Undeclared identifier error** → Happened in **Stage 2 (Compilation)**.
* **`undefined reference to X`** → Function was *declared* (Stage 2 passed) but never *defined* or library wasn't linked (**Stage 4: Linking**).
* **`multiple definition of X`** → Two `.o` files exported the same strong global symbol (**Stage 4: Linking**).


* **Translation Unit Rule:** Stages 1, 2, and 3 operate strictly on **one `.c` file at a time** in total isolation. Only Stage 4 (Linker) sees multiple files at once.
* **Header Rule:** Put **declarations** (prototypes, `extern` vars, structs, macros) in `.h` files; put **definitions** (function bodies, global variable storage) in `.c` files.
* **Library Link Order Rule:** Static libraries (`-lm`, `-lfoo`) must appear **after** the `.o` files that call them (`gcc main.o -lm`, never `gcc -lm main.o`).

---

### LAYER 2: EXHAUSTIVE DEEP-DIVE

#### Module 1: What happens to a `.c` file before the compiler ever checks its syntax?

* **Plain-English Intuition (Feynman):**
The preprocessor is a blind copy-paste robot. It does not know what an `int`, a `for` loop, or a semicolon is. It just reads text, obeys lines starting with `#`, deletes comments, and hands one giant, clean C file to the compiler.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Line Splicing:** Any backslash immediately followed by a newline (`\` + `\n`) is deleted, joining lines together (used for multi-line macros).
2. **Comment Stripping:** Every `/* ... */` and `// ...` comment is replaced by a **single whitespace character** (not empty string—so `int/*x*/a;` becomes `int a;`, not `inta;`).
3. **Header Inclusion (`#include`):** Literally opens the target file and pastes its entire text at that exact line.
* `#include <file.h>` searches standard system directories first (`/usr/include`).
* `#include "file.h"` searches the current source file's directory first, then falls back to system directories.


4. **Macro Expansion (`#define`):** Performs textual token substitution for object-like macros (`#define PI 3.14`) and function-like macros (`#define MAX(a,b) ((a)>(b)?(a):(b))`). Macros are **not** expanded inside string literals (`"PI"` stays `"PI"`).
5. **Conditional Compilation (`#if`, `#ifdef`, `#ifndef`, `#else`, `#endif`):** Evaluates constant integer expressions or macro existence; code in false branches is completely deleted before Stage 2.
6. **Line Markers & Output:** Inserts `# line "filename"` markers so Stage 2 can report accurate original line numbers in error messages. Output is a single **Translation Unit (`.i`)**.


* **Visual / Structure (Dual Coding):**

```text
Source (main.c)                     Expanded Translation Unit (main.i)
─────────────────────────           ────────────────────────────────────
#include "math_util.h"      ──►     # 1 "math_util.h"
#define SQR(x) ((x)*(x))            int add(int a, int b);
                                    # 2 "main.c"
int main() {                        int main() {
    // compute                          int v = ((5)*(5));
    int v = SQR(5);                     return add(v, 1);
    return add(v, 1);               }
}

```

* **Concrete Examples:**
1. **Include Guards:** Preventing double-inclusion when `a.h` and `b.h` both `#include "types.h"`:
```c
#ifndef TYPES_H
#define TYPES_H
struct Point { int x, y; };
#endif

```


First inclusion defines `TYPES_H` and pastes `struct Point`. Second inclusion sees `TYPES_H` is already defined and deletes the entire block, preventing a `redefinition of struct Point` error in Stage 2. (`#pragma once` does the same via file path tracking, though `#ifndef` is strictly ISO C standard.)
2. **Zero-Cost Debug Logging:** Wrapping logs in `#ifdef DEBUG ... #endif` and compiling with `gcc -DDEBUG main.c` enables logs; compiling without `-DDEBUG` strips the code out completely at Stage 1—zero runtime CPU or binary size overhead.


* **Traps & Edge Cases (Warning):**
* **Operator Precedence & Side-Effect Double Evaluation:**
* `#define MUL(a,b) a * b` → `MUL(1+2, 3+4)` expands to `1 + 2 * 3 + 4` = **11** (not 21).
* Even with parentheses, `#define SQR(x) ((x)*(x))` fails on `SQR(i++)` → expands to `((i++)*(i++))`, incrementing `i` **twice** and triggering **Undefined Behavior**. Use `static inline` functions instead of function-like macros whenever type safety or side effects matter.





---

#### Module 2: How does the compiler turn expanded C code (`.i`) into CPU-specific Assembly (`.s`)?

* **Plain-English Intuition (Feynman):**
The compiler is a strict grammar teacher paired with an efficiency-obsessed translator. It checks that every sentence makes legal sense in C, rewrites clumsy logic into faster equivalents, and translates it into the exact vocabulary of your specific processor (x86-64, ARM64, RISC-V).
* **Exact Mechanics (Elaborative Interrogation):**
Compilation (`cc1`, triggered via `gcc -S`) splits into three internal phases:
1. **Front-End (Language-Dependent, Machine-Independent):**
* **Lexical Analysis (Lexer/Scanner):** Breaks raw character streams into **tokens** (`int`, `x`, `=`, `5`, `;`). Whitespace is discarded.
* **Syntax Analysis (Parser):** Checks token order against C grammar rules and builds an **Abstract Syntax Tree (AST)**. Missing semicolons or mismatched braces fail here.
* **Semantic Analysis:** Walks the AST to check types, variable scope, function signatures (number/type of args), and lvalue/rvalue validity. Fails on `undeclared identifier` or incompatible pointer assignments.


2. **Middle-End (Optimizer — Machine-Independent):**
* Converts AST into **Intermediate Representation (IR)** (e.g., GIMPLE/RTL in GCC, LLVM IR in Clang).
* Applies optimization passes based on flags (`-O0` none, `-O1` basic, `-O2` standard production, `-O3` aggressive vectorization/inlining): constant folding (`3 + 4` → `7`), dead code elimination, loop unrolling, function inlining.


3. **Back-End (Machine-Dependent):**
* **Instruction Selection:** Maps IR operations to target ISA instructions (`addl`, `movq`, `imul`).
* **Register Allocation:** Assigns variables to physical CPU registers (`%eax`, `%rdi`) instead of slow RAM stack slots.
* **Output:** Human-readable assembly text (`.s`).




* **Visual / Structure (Dual Coding):**

```text
.i text ──► [Lexer] ──► Tokens ──► [Parser] ──► AST ──► [Semantic] ──► Validated AST
                                                                             │
.s file ◄── [Code Gen & Reg Alloc] ◄── Optimized IR ◄── [Optimizer (-O2)] ◄──┘

```

* **Concrete Examples:**
1. **Constant Folding & Dead Code Elimination (`-O2`):**
```c
int x = 10 * 20;
if (0) { printf("never"); }
return x;

```


At `-O2`, the compiler computes `200` at compile time, deletes the `if` branch entirely, and emits just `movl $200, %eax` followed by `ret`.
2. **Cross-Compilation:** The exact same `main.i` compiled with `gcc -S` on an x86-64 laptop emits `movl`, `addl`, `%rdi`, whereas `aarch64-linux-gnu-gcc -S` emits ARM64 `mov`, `add`, `w0`. Only the back-end changes.


* **Traps & Edge Cases (Warning):**
* **Why `printf` compiles without its body:** Stage 2 only needs a **declaration** (`int printf(const char *, ...);` pasted from `stdio.h`) to verify argument types and generate a `call printf` assembly instruction. It blindly trusts that the actual machine code for `printf` will exist in Stage 4.
* **Undefined Behavior (UB) + Optimization Trap:** At `-O2`, the compiler assumes UB *never happens*. In `if (x + 1 < x)` for signed `int`, signed overflow is UB, so the optimizer legally assumes `x + 1 < x` is always false and deletes your overflow check.



---

#### Module 3: What is inside an Object File (`.o`), and why can't the OS execute it directly?

* **Plain-English Intuition (Feynman):**
The assembler translates readable assembly instructions into raw binary bytes the CPU understands. However, because it only sees one file at a time, it leaves **blank sticky notes** wherever your code calls an external function (like `printf`) or touches an external variable. The OS cannot run a binary full of blank addresses.
* **Exact Mechanics (Elaborative Interrogation):**
The Assembler (`as`, invoked by `gcc -c`) reads `.s` and produces a **Relocatable Object File** (`.o` in ELF format on Linux, COFF on Windows, Mach-O on macOS). An ELF `.o` file is structured into distinct **sections**:
1. **`.text`:** Raw binary machine code for all functions defined in this `.c` file.
2. **`.rodata`:** Read-only data (string literals like `"Hello\n"`, `const` globals).
3. **`.data`:** Initialized global and `static` local variables (e.g., `int count = 5;`). Takes space in the `.o` file on disk.
4. **`.bss` (Block Started by Symbol):** Uninitialized (or zero-initialized) global and `static` variables (e.g., `int buf[10000];`). Takes **0 bytes of data storage on disk**—only stores the *size* so the OS zero-fills it at load time.
5. **`.symtab` (Symbol Table):** Index of every global function/variable **defined** here (with its section & offset) and every external symbol **referenced** here (marked `UND` = Undefined).
6. **`.rela.text` / `.rela.data` (Relocation Entries):** A checklist telling the linker: *"At byte offset `0x14` in `.text`, I put `0x00000000` as a placeholder; patch it with the final address of `printf`."*


* **Visual / Structure (Dual Coding):**

```text
┌──────────────────────────────────────────────────────────┐
│                 main.o (ELF Relocatable)                 │
├──────────────┬───────────────────────────────────────────┤
│ .text        │ 55 48 89 e5 e8 [00 00 00 00] ...          │
│              │                 ▲ placeholder for printf  │
├──────────────┼───────────────────────────────────────────┤
│ .rodata      │ "Hello\0"                                 │
├──────────────┼───────────────────────────────────────────┤
│ .data        │ global_x = 42                             │
├──────────────┼───────────────────────────────────────────┤
│ .bss         │ (metadata only: zero_arr = 4096 bytes)    │
├──────────────┼───────────────────────────────────────────┤
│ .symtab      │ main: T (defined in .text+0x0)            │
│              │ printf: U (UND - external!)               │
├──────────────┼───────────────────────────────────────────┤
│ .rela.text   │ Patch .text+0x5 with address of 'printf'  │
└──────────────┴───────────────────────────────────────────┘

```

* **Concrete Examples:**
1. **Inspecting `.o` Symbols:** Running `nm main.o` prints:
* `0000000000000000 T main` (`T` = defined in `.text`, globally visible)
* `                 U printf` (`U` = undefined, must be resolved in Stage 4)
* `0000000000000000 t helper` (`t` = lowercase means `static` internal linkage—hidden from other `.o` files).


2. **Why `int big[1000000];` doesn't bloat your `.o` file:** Because it is uninitialized, it goes into `.bss`. Changing it to `int big[1000000] = {1};` moves it to `.data`, instantly adding ~4 MB to the `.o` and executable file size on disk.


* **Traps & Edge Cases (Warning):**
* **Local non-static variables never appear in `.symtab`, `.data`, or `.bss`:** Automatic variables inside a function live exclusively on the **runtime stack** (addressed relative to `%rbp`/`%rsp` via assembly instructions in `.text`). Only globals and `static` variables get static section allocations.



---

#### Module 4: How does the Linker merge `.o` files, resolve symbols, and patch addresses?

* **Plain-English Intuition (Feynman):**
The linker is the project manager who collects all the `.o` puzzle pieces and libraries, stacks all their `.text` sections together and `.data` sections together, matches every "I need `foo`" sticky note to the file that says "I have `foo`", and overwrites every `0x00000000` blank with the exact final memory address.
* **Exact Mechanics (Elaborative Interrogation):**
The Linker (`ld`, invoked via `gcc`) executes two core passes:
1. **Pass 1 — Symbol Resolution:**
* Scans input `.o` files and `.a` libraries **strictly left-to-right**, maintaining three sets: **E** (object files to merge), **U** (currently unresolved symbols), **D** (defined symbols).
* When processing a `.o` file: always adds it to **E**, moves its definitions to **D**, adds its unresolved calls to **U**.
* When processing a `.a` static library: pulls in **only** the specific `.o` members inside the archive that resolve a symbol currently sitting in **U**.
* **Strong vs. Weak Symbol Rules:**
* **Strong:** Functions and initialized globals (`int x = 5;`).
* **Weak:** Uninitialized globals (under legacy `-fcommon`, now `-fno-common` by default in GCC 10+) or `__attribute__((weak))` symbols.
* Rule 1: Two strong symbols with the same name → **`multiple definition` error**.
* Rule 2: One strong + one weak → Linker picks the strong one.


* At the end of Pass 1: if **U** is non-empty → **`undefined reference` error**.


2. **Pass 2 — Relocation (Address Patching):**
* **Section Merging:** Concatenates all `.text` sections into one final `.text` segment, all `.data` into one `.data` segment, etc., assigning each symbol its final virtual memory address (e.g., `main` at `0x401126`, `add` at `0x401150`).
* **Patching:** Walks every `.rela.text` / `.rela.data` entry and overwrites the placeholder `0x00000000` bytes with the exact computed absolute or PC-relative offset.
* **Entry Point:** Links the C Runtime startup object (`crt1.o`) so the OS actually starts at `_start`, which initializes the stack/environment and then calls your `main()`.




* **Visual / Structure (Dual Coding):**

```text
   main.o (.text)              util.o (.text)
  ┌──────────────────┐        ┌──────────────────┐
  │ main:            │        │ add:             │
  │   call [00000000]│        │   lea (%rdi,%rsi)│
  └──────────────────┘        └──────────────────┘
             │                          │
             └────────────┬─────────────┘
                          ▼  [LINKER: Merge + Relocate]
  ┌───────────────────────────────────────────────────┐
  │ Final Executable (.text)                          │
  │ 0x401126 <main>:  call 0x401150 <add>  (Patched!) │
  │ 0x401150 <add>:   lea (%rdi, %rsi), %eax          │
  └───────────────────────────────────────────────────┘

```

* **Concrete Examples:**
1. **Math Library Link Order:**
* `gcc -lm main.c` → **Fails** with `undefined reference to 'sqrt'`. When `ld` scans `libm.a` first, **U** is empty, so it pulls nothing from `libm.a`. Then it scans `main.o`, adds `sqrt` to **U**, and finishes with `sqrt` unresolved.
* `gcc main.c -lm` → **Succeeds**. `main.o` puts `sqrt` into **U**, then `libm.a` satisfies it.


2. **Preventing Symbol Collisions with `static`:** If both `net.c` and `db.c` define `void init_buffer()`, linking fails with `multiple definition`. Marking them `static void init_buffer()` gives them **internal linkage** (local symbols in `.symtab`), so the linker ignores them across files.


* **Traps & Edge Cases (Warning):**
* **Putting a function or global definition in a `.h` file:** Even with `#ifndef` include guards, if *two different `.c` files* (`a.c` and `b.c`) both `#include "bad.h"`, each translation unit gets its own copy. Both `a.o` and `b.o` export the strong symbol → Stage 4 crashes with **`multiple definition`**. Include guards only protect *within a single translation unit*, never across multiple `.c` files.



---

#### Module 5: How do Static Linking (`.a`) and Dynamic Linking (`.so`) differ at build time and runtime?

* **Plain-English Intuition (Feynman):**
Static linking photocopies library pages and staples them directly into your book—your book becomes heavy, but works anywhere offline. Dynamic linking leaves a bookmark saying "look up page 42 in the public library"—your book stays tiny and automatically gets library updates, but crashes if the library is missing.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Static Linking (`gcc -static` / `.a` archives):**
* A `.a` file is just a bundled zip-like archive (`ar rcs libfoo.a f1.o f2.o`) of `.o` files.
* At Stage 4, the linker copies the exact machine code of needed `.o` members directly into your executable. Zero external library dependencies at runtime.


2. **Dynamic Linking (Default in GCC / `.so` Shared Objects):**
* Compiled with `-fPIC` (Position-Independent Code) and `-shared` so the library can be loaded at *any* virtual memory address without rewriting its `.text` code.
* At Stage 4, `ld` does **not** copy library code into your executable. It only verifies symbols exist and writes the library's name (`libc.so.6`) into the executable's `.dynamic` section.
* At **runtime**, the OS loads your executable *and* the dynamic linker (`ld-linux-x86-64.so.2`), which maps `libc.so.6` into memory (shared in physical RAM across all running processes).
* **PLT & GOT (Lazy Binding):** Calls to shared functions jump through the **Procedure Linkage Table (PLT)** and **Global Offset Table (GOT)**. On the *first* call to `printf`, the dynamic linker looks up `printf`'s real RAM address and writes it into the GOT; every subsequent call jumps directly to `printf` in 1 instruction.




* **Visual / Structure (Dual Coding):**

```text
STATIC LINKING (.a)                        DYNAMIC LINKING (.so - Default)
────────────────────                       ────────────────────────────────
[main.o] + [libc.a]                        [main.o] + (stub ref to libc.so)
         │                                          │
         ▼ (Copy code at build time)                ▼ (Small executable)
┌─────────────────────┐                    ┌─────────────────────┐
│ Huge Standalone App │                    │ Tiny App (PLT/GOT)  │
│ (main + printf code)│                    └──────────┬──────────┘
└─────────────────────┘                               │ (At Runtime via ld-linux.so)
                                                      ▼
                                           [Shared libc.so in RAM] ◄── shared by all apps

```

* **Concrete Examples:**
1. **Binary Size & Security Patching:** A statically linked "Hello World" (`gcc -static hello.c`) is ~800 KB because it embeds `libc.a`. Dynamically linked (`gcc hello.c`), it is ~16 KB. If OpenSSL has a vulnerability, updating `libssl.so` patches every dynamically linked app immediately without recompiling them.
2. **Inspecting Dynamic Dependencies:** Running `ldd ./a.out` lists every `.so` required at runtime (e.g., `libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6`).


* **Traps & Edge Cases (Warning):**
* **Runtime Linker Error (`error while loading shared libraries: libfoo.so: cannot open shared object file`):** You linked successfully with `-L./libs -lfoo`, but at runtime `ld-linux.so` only searches standard paths (`/lib`, `/usr/lib`), not `./libs`. Fix by either setting `LD_LIBRARY_PATH=./libs` or embedding the search path at link time with `-Wl,-rpath,'$ORIGIN/libs'`.



---

### LAYER 3: SYNTHESIS & INTERLEAVING

#### Compare & Contrast Table

| Confused Pair | Option A | Option B | Decisive Rule / Diagnostic |
| --- | --- | --- | --- |
| **Declaration vs. Definition** | **Declaration:** Announces name & type (`extern int x;`, `int f(int);`). Allocates **0 bytes**. | **Definition:** Creates actual storage or function body (`int x = 0;`, `int f(int a){...}`). | You can **declare** a symbol 100 times (in `.h`); you can **define** a strong symbol only **once** across the whole program (in `.c`). |
| **Compiler Error vs. Linker Error** | **Compiler (Stage 2):** Syntax, type mismatch, `undeclared identifier`. Mentions exact `.c` line number. | **Linker (Stage 4):** `undefined reference`, `multiple definition`. Mentions `.o` file & symbol name. | If the error says `undeclared` → missing `#include` or prototype. If it says `undefined reference` → missing `.o` file or `-l` library flag. |
| **`.data` vs. `.bss` vs. `.rodata**` | **`.data`:** Initialized non-zero globals/statics (takes disk space). | **`.bss`:** Uninitialized/zero globals/statics (0 bytes of data on disk). **`.rodata`:** String literals & `const` globals (read-only page). | Writing to a string literal (`char *s = "hi"; s[0] = 'H';`) crashes with **SIGSEGV** because `"hi"` lives in read-only `.rodata`. |
| **`static` vs. `extern` (at file scope)** | **`static`:** Internal linkage—symbol is hidden in `.symtab` (`t`/`d`/`b`), invisible to other `.o` files. | **`extern`:** External linkage—declares a symbol defined in another `.o` file (`U` in `.symtab`). | Use `static` for file-private helpers/globals; use `extern` in `.h` files to share a global defined in one `.c` file. |
| **Static (`.a`) vs. Dynamic (`.so`)** | **`.a`:** Copied into executable at Stage 4. Larger binary, zero runtime dependency, slightly faster calls. | **`.so`:** Mapped into RAM at runtime via `ld-linux.so` + PLT/GOT. Tiny binary, shared RAM, hot-patchable. | Use `.a` for portable single-binary tools/embedded; use `.so` for OS libraries, plugins, and memory efficiency. |

#### Cross-Connections

* **Stage 1 → Stage 2:** A broken `#define` macro never throws a Stage 1 error; it silently mutates the `.i` text and surfaces as a bizarre syntax error in Stage 2 (which is why `gcc -E` is the ultimate macro debugger).
* **Stage 1 → Stage 4:** Putting a non-`static` function definition inside a `.h` file passes Stage 1, 2, and 3 cleanly for every `.c` file, only to blow up in Stage 4 with `multiple definition`.
* **Stage 2 → Stage 3 → Stage 4:** Declaring `int foo(int);` lets Stage 2 emit `call foo` without knowing `foo`'s body, causes Stage 3 to mark `foo` as `UND` in `.symtab` with a `0x00000000` relocation placeholder, and forces Stage 4 to either find `foo` in another `.o`/library or abort with `undefined reference`.

---

### LAYER 4: LOCK-IN LAYER

#### 1. Blank-Page Self-Test

Try answering all 8 without looking up, then open the key below.

1. **Conceptual:** Why does the preprocessor replace a comment `/* comment */` with a **space** instead of deleting it completely?
2. **Applied Macro:** Given `#define DOUBLE(x) x + x`, what exact value does `10 * DOUBLE(3)` evaluate to, and at which stage does the bug get introduced?
3. **Diagnostic:** You compile with `gcc main.c` and get `error: 'sqrt' undeclared`. Your friend compiles the same code after adding `#include <math.h>` and gets `undefined reference to 'sqrt'`. Name the exact stage each error occurred in and how to fix the second error.
4. **Object File Internals:** You have `int arr[500000];` as a global variable in `a.c`. You change it to `int arr[500000] = {1};`. Which ELF section did it move from/to, and what happens to the size of `a.o` on disk?
5. **Linkage Trap:** You put `int counter = 0;` inside `common.h` with proper `#ifndef` include guards, and include `common.h` in both `main.c` and `util.c`. Why do the include guards fail to stop the build error, and what exact error occurs?
6. **Linker Mechanics:** Why does `gcc -lm main.c` fail while `gcc main.c -lm` succeeds on Linux?
7. **Runtime vs. Build Time:** You compile with `gcc -c main.c` (no errors), then try `./main.o` in the terminal. Why can't the OS run it?
8. **Dynamic Linking:** Why is a shared library (`.so`) compiled with `-fPIC`, and what role do the PLT and GOT play on the first call vs. second call to `printf`?

1. Deleting it completely would accidentally fuse adjacent tokens (`int/*x*/a;` would become `inta;`, breaking lexing in Stage 2).
2. Evaluates to **33** (`10 * 3 + 3`). Introduced in **Stage 1 (Preprocessing)** because of blind text substitution without parentheses.
3. `undeclared` = **Stage 2 (Compilation)** (missing prototype). `undefined reference` = **Stage 4 (Linking)** (prototype existed, but math library machine code wasn't linked). Fix: `gcc main.c -lm`.
4. Moved from **`.bss`** to **`.data`**. `a.o` grows by ~2 MB on disk because `.bss` stores only metadata (size), whereas `.data` stores the actual initialized bytes.
5. Include guards only prevent double-inclusion *within a single translation unit*. Both `main.c` and `util.c` are preprocessed independently, so both `main.o` and `util.o` export a strong symbol `counter`, causing a Stage 4 **`multiple definition of 'counter'`** error. (Fix: `extern int counter;` in `.h`, `int counter = 0;` in one `.c`.)
6. The linker resolves static archives (`.a`) strictly **left-to-right**. When `-lm` comes first, the unresolved symbol set **U** is empty, so nothing is extracted from `libm.a`.
7. `main.o` is a **relocatable object file**, not an executable: it has unresolved `0x00000000` address placeholders in `.text`, no merged libraries, and no `_start` entry point (`crt1.o`).
8. `-fPIC` (Position-Independent Code) allows the OS to map the `.so` at any virtual address without modifying its read-only `.text` pages (enabling RAM sharing). On the **first** call, the PLT invokes the dynamic linker (`ld-linux.so`) to find `printf`'s real address and cache it in the GOT; on the **second** call, the PLT jumps directly to the cached GOT address.

---

#### 2. Anki Flashcards

* **Q:** What GCC flag stops the build after Preprocessing, Compilation, and Assembly respectively?
**A:** `-E` (`.i`), `-S` (`.s`), `-c` (`.o`).
* **Q:** What is a Translation Unit in C?
**A:** The single expanded `.i` file produced by Stage 1 (a `.c` file plus all recursively `#include`d headers, macros expanded, comments stripped).
* **Q:** Why do Stages 1–3 never detect `multiple definition` errors across files?
**A:** Because they process only one translation unit at a time in complete isolation; only Stage 4 (Linker) sees multiple `.o` files together.
* **Q:** What are the 3 internal phases of Stage 2 (Compilation)?
**A:** Front-End (Lexing → Parsing/AST → Semantic analysis), Middle-End (IR + Optimization), Back-End (Instruction selection + Register allocation → `.s`).
* **Q:** What does `.bss` store in an ELF `.o` file, and why does it save disk space?
**A:** Uninitialized/zero-initialized globals & `static` vars; it stores only the total byte size (0 bytes of data on disk) and the OS zero-fills it at load time.
* **Q:** What is the purpose of `.rela.text` in a `.o` file?
**A:** It lists exact byte offsets in `.text` containing placeholder addresses (`0x00000000`) that the linker must patch with final virtual addresses in Stage 4.
* **Q:** In `nm main.o`, what do `T`, `t`, and `U` mean?
**A:** `T` = globally visible function in `.text`; `t` = `static` (file-local) function in `.text`; `U` = undefined external symbol needed by this file.
* **Q:** What is the difference between `undeclared identifier` and `undefined reference`?
**A:** `undeclared` = Stage 2 Compiler error (missing declaration/header); `undefined reference` = Stage 4 Linker error (declaration existed, definition/library missing).
* **Q:** Why do `#ifndef` include guards NOT prevent `multiple definition` errors if you define a global variable in a `.h` file?
**A:** Guards only stop re-inclusion within one `.c` file's preprocessing; each separate `.c` file still gets its own copy and exports a duplicate strong symbol to the linker.
* **Q:** What is the actual first entry point the OS jumps to in a C executable before `main()`?
**A:** `_start` (provided by `crt1.o`), which sets up the runtime environment/args and then calls `main()`.

---

#### 3. 60-Second Revision Block (Day 1 / 3 / 7 / 21)

* **Pipeline:** `.c` —(`-E` Preprocess)→ `.i` —(`-S` Compile)→ `.s` —(`-c` Assemble)→ `.o` —(Link)→ Executable.
* **Stage 1 (`.i`):** Blind text robot. Strips comments (→ space), pastes `#include`, expands `#define`, prunes `#ifdef`. Zero C syntax knowledge. Watch for macro precedence & double-evaluation traps.
* **Stage 2 (`.s`):** Lexer → Parser (AST) → Semantic (types/scope) → Optimizer (IR) → Target Assembly. Needs only *declarations* to pass; emits CPU-specific assembly.
* **Stage 3 (`.o`):** Translates assembly to binary ELF sections (`.text`, `.rodata`, `.data`, `.bss`), builds `.symtab` (`T`/`t`/`U`), and logs placeholder addresses in `.rela.text`.
* **Stage 4 (Executable):** Merges sections across `.o` files, resolves symbols left-to-right (libs go *after* `.o` files), enforces single strong definition, patches relocation placeholders, and attaches `crt1.o` (`_start`).
* **Static (`.a`) vs. Dynamic (`.so`):** `.a` copies code into a large standalone binary; `.so` (`-fPIC`) maps shared code at runtime via `ld-linux.so` using PLT/GOT lazy binding.