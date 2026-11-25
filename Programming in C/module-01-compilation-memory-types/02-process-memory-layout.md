### LAYER 1: THE SPEED LAYER

#### 1. Core Intuition (Feynman)

When the OS runs your compiled C executable, it hands the program its own private, illusionary strip of memory called a **Virtual Address Space**. Instead of dumping everything into one messy pile, the OS partitions this strip into 5 strict zones based on **lifetime**, **permissions** (read/write/execute), and **growth behavior**—putting fixed-size instructions and global variables at the bottom, and letting dynamic memory (Heap) and function calls (Stack) grow toward each other from opposite ends.

#### 2. Master Mental Map (Linux x86-64 Virtual Address Space)

```text
High Address (0xFFFFFFFFFFFFFFFF)
┌───────────────────────────────────────────────────────────┐
│                 KERNEL SPACE (OS Only)                    │  (Inaccessible to user code)
├───────────────────────────────────────────────────────────┤  0x7FFFFFFFFFFF (approx)
│             Command-Line Args (argv) & envp               │
├───────────────────────────────────────────────────────────┤
│                    STACK  (rw-)                           │  ◄── Local vars, return addrs,
│                      │  (Grows Downward: High → Low)      │      saved registers (LIFO)
│                      ▼                                    │
│                                                           │
│          [ Unmapped Free Virtual Address Space ]          │
│                                                           │
│       Memory Mapping Segment (mmap / Shared Libs .so)     │  ◄── libc.so, large mallocs
│                                                           │
│                      ▲                                    │
│                      │  (Grows Upward: Low → High)        │
│                     HEAP  (rw-)                           │  ◄── malloc / calloc / realloc
├───────────────────────────────────────────────────────────┤  ◄── Program Break (brk)
│        UNINITIALIZED DATA / BSS (.bss)  (rw-)             │  ◄── Uninit/zero globals & statics
├───────────────────────────────────────────────────────────┤
│         INITIALIZED DATA (.data)  (rw-)                   │  ◄── Init non-zero globals & statics
├───────────────────────────────────────────────────────────┤
│          READ-ONLY DATA (.rodata)  (r--)                  │  ◄── String literals, const globals
├───────────────────────────────────────────────────────────┤
│              TEXT / CODE (.text)  (r-x)                   │  ◄── Compiled machine instructions
└───────────────────────────────────────────────────────────┘  0x00400000 (typical non-PIE)
Low Address  (0x0000000000000000 - Unmapped NULL Guard Page)

```

#### 3. 80/20 High-Yield Cheat Sheet

* **Variable Placement Rule (Memorize this):**
* **Local non-static variable** (`int x = 5;` inside a function) → **Stack**.
* **`malloc()` / `calloc()` block** → **Heap** (the pointer variable itself is on the Stack/Data; the *allocated bytes* are on the Heap).
* **Global or `static` variable (initialized non-zero)** → **Initialized Data (`.data`)**.
* **Global or `static` variable (uninitialized or initialized to `0`)** → **BSS (`.bss`)**.
* **String literal (`"hello"`)** → **Read-Only Data (`.rodata`)**.
* **Function machine code** → **Text (`.text`)**.


* **Hardware Permission Rule (MMU Enforced):**
* `.text`: **`r-x`** (Read + Execute, never Write → prevents self-modifying code).
* `.rodata`: **`r--`** (Read only → writing triggers `SIGSEGV`).
* `.data`, `.bss`, Heap, Stack: **`rw-`** (Read + Write, never Execute → **NX / W^X** security bit prevents executing injected shellcode).


* **Inspection Command:** Run `size ./a.out` in Linux to see exact byte sizes of `text`, `data`, and `bss` before execution.

---

### LAYER 2: EXHAUSTIVE DEEP-DIVE

#### Module 1: What lives in the Text (`.text`) and Read-Only (`.rodata`) segments, and why are they write-protected?

* **Plain-English Intuition (Feynman):**
The Text segment is the recipe book—the raw binary instructions your CPU fetches and runs. Read-Only Data is the printed constants page (like `"Error: 404"`). Both are laminated in glass so buggy code can't accidentally scribble over its own instructions.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Direct Mapping from ELF:** When the OS loader (`execve`) starts your program, it maps the `.text` and `.rodata` sections directly from the executable file on disk into the lowest usable virtual addresses (above the unmapped `0x0` NULL guard page).
2. **Memory Sharing:** Because `.text` is marked **Read + Execute (`r-x`)** and can never be modified, if you run 50 instances of `/bin/bash`, the OS keeps **only one physical RAM copy** of bash's `.text` pages and maps all 50 processes' virtual page tables to it.
3. **`.rodata`:** Stores string literals (`"Hello"`), `const`-qualified global/static variables, and compiler-generated jump tables (`switch` statements). Marked **`r--`**.


* **Visual / Structure (Dual Coding):**

```text
char *p = "hello";   // p (8-byte pointer) -> Stack | "hello\0" -> .rodata (r--)
char s[] = "hello";  // s (6-byte array)   -> Stack (copied from .rodata at runtime)

```

* **Concrete Examples:**
1. **Legal vs. Illegal String Mutation:**
* `char s[] = "abc"; s[0] = 'X';` → **Valid.** `s` is a mutable copy on the **Stack**.
* `char *p = "abc"; p[0] = 'X';` → **Crashes (`SIGSEGV`).** `p` points directly into write-protected **`.rodata`**.


2. **NULL Pointer Dereference:** Dereferencing `*(int *)0` crashes immediately because address `0x00000000` is intentionally left unmapped below `.text`.


* **Traps & Edge Cases (Warning):**
* **`const` local variables do NOT go to `.rodata`:** `const int x = 10;` inside a function still lives on the **Stack** (or gets folded into an immediate instruction operand at `-O2`). `const` is a compile-time type-checker rule; only `const` globals/statics reliably land in `.rodata`.



---

#### Module 2: How do Initialized Data (`.data`) and Uninitialized Data (`.bss`) differ in RAM and on disk?

* **Plain-English Intuition (Feynman):**
Both segments hold variables that live for the **entire lifetime of the program** (globals and `static` locals). The only difference is whether they start with a custom value (`.data`) or start at zero (`.bss`). Splitting them stops your executable file on disk from bloating with millions of zeros.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Initialized Data (`.data`):**
* Holds global variables and `static` local variables explicitly initialized to a **non-zero** value (e.g., `int max_conn = 100;`, `static int calls = 1;`).
* The exact initial bytes **must be stored inside the executable file on disk** and copied into RAM at startup.


2. **Uninitialized Data (`.bss` — Block Started by Symbol):**
* Holds global variables and `static` local variables that are **uninitialized** or explicitly initialized to **`0` / `NULL**` (e.g., `int cache[100000];`, `static int count = 0;`).
* Takes **0 bytes of actual data storage on disk**—the ELF header simply records the total `.bss` size.
* At load time, the OS maps anonymous **zero-filled physical pages** (often lazily via Copy-on-Write from a single system zero page) so every `.bss` variable is **guaranteed by the C standard to start at `0**`.




* **Visual / Structure (Dual Coding):**

```text
Code Declaration                  Segment   Disk Space?   Initial Value
───────────────────────────────   ───────   ───────────   ─────────────
int g1 = 42;                      .data     Yes (4 B)     42
int g2;                           .bss      No  (0 B)     0 (Guaranteed)
int g3 = 0;                       .bss      No  (0 B)     0 (Guaranteed)
static int s1 = 9;  (in func)     .data     Yes (4 B)     9
static int s2;      (in func)     .bss      No  (0 B)     0 (Guaranteed)

```

* **Concrete Examples:**
1. **Executable Size Optimization:** Declaring `char buffer[10 * 1024 * 1024];` globally keeps your binary at ~16 KB (`.bss`). Changing it to `char buffer[10 * 1024 * 1024] = {1};` instantly balloons the binary on disk to **10 MB** (`.data`).
2. **Persistent State Across Function Calls:** A `static int counter = 0;` inside `next_id()` lives in `.bss`, not the Stack. When `next_id()` returns, its stack frame is destroyed, but `counter` retains its value in `.bss` until the process exits.


* **Traps & Edge Cases (Warning):**
* **Uninitialized Local vs. Global:** An uninitialized global (`int x;`) goes to `.bss` and is guaranteed to be `0`. An uninitialized local (`int x;` inside `main()`) goes to the **Stack** and contains **indeterminate garbage bytes**—reading it is Undefined Behavior.



---

#### Module 3: How does the Heap manage dynamic memory, and why does it grow upward?

* **Plain-English Intuition (Feynman):**
The Heap is a flexible warehouse for data whose size you only know at runtime (or data that must outlive the function that created it). You manually rent chunks (`malloc`) and manually return them (`free`). Unlike the Stack, you can return chunks in any order.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Upward Growth (`brk` / `sbrk`):** The Heap starts right above `.bss`. The top boundary of the Heap is called the **program break (`brk`)**. When `malloc` runs out of free space in its internal free-list, glibc's allocator asks the OS kernel to push `brk` **upward** (toward higher addresses).
2. **Large Allocations (`mmap`):** For large requests (typically $\ge 128\text{ KB}$ in glibc), `malloc` bypasses `brk` and calls `mmap()` to grab an anonymous memory region in the **Memory Mapping Segment**. When `free()`d, `munmap()` immediately returns that RAM to the OS.
3. **Allocator Overhead:** Every `malloc(n)` actually allocates slightly more than `n` bytes to store a hidden **chunk header** (size + flags) right before the pointer returned to you, so `free(ptr)` knows how many bytes to reclaim without you passing the size.
4. **Performance Cost:** Slower than the Stack because `malloc`/`free` must search free-lists, split/coalesce blocks, and acquire thread locks (or manage per-thread arenas).


* **Visual / Structure (Dual Coding):**

```text
Higher Addresses
   ▲  brk (Program Break moves UP as heap grows)
┌──┴─────────────────────────────────────────┐
│ [Header: 32B | Free Chunk]                 │
├────────────────────────────────────────────┤
│ [Header: 64B | User Data (ptr2)]           │ ◄── returned by malloc(56)
├────────────────────────────────────────────┤
│ [Header: 32B | Freed Hole (Fragmentation)] │ ◄── freed out of order
├────────────────────────────────────────────┤
│ [Header: 32B | User Data (ptr1)]           │ ◄── returned by malloc(24)
└────────────────────────────────────────────┘
Lower Addresses (right above .bss)

```

* **Concrete Examples:**
1. **Dynamic Sizing:** Reading `n` from user input and allocating `int *arr = malloc(n * sizeof(int));`—impossible to size cleanly in `.data`/`.bss` at compile time.
2. **Outliving a Function:** A factory function `Node *create_node(int val)` allocates a `Node` on the Heap and returns its pointer; the `Node` remains valid after `create_node` returns until `free(node)` is called.


* **Traps & Edge Cases (Warning):**
* **Memory Leak:** Losing the pointer to a `malloc`'d block without calling `free()` leaves it permanently occupied until process termination.
* **Dangling Pointer / Use-After-Free (UAF):** Calling `free(p)` does **not** set `p = NULL` or wipe the bytes. Dereferencing `p` afterward reads/corrupts recycled heap metadata (a prime cybersecurity exploit vector). Always set `p = NULL;` after `free(p)`.
* **Heap Fragmentation:** Allocating and freeing mixed sizes leaves scattered small holes; a large `malloc` can fail even when *total* free bytes are sufficient.



---

#### Module 4: How does the Stack work, why is it so fast, and why does it grow downward?

* **Plain-English Intuition (Feynman):**
The Stack is a spring-loaded plate dispenser (LIFO: Last-In, First-Out). Every time you call a function, a new tray (**Stack Frame**) is slapped on top holding that function's local variables and a sticky note saying where to return. When the function finishes, the tray is popped off in a single CPU instruction.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Downward Growth (`%rsp`):** The Stack starts near the top of user virtual memory and grows **downward** (from high addresses to low addresses).
2. **1-Instruction Allocation:** To allocate 64 bytes of local variables, the CPU literally executes `subq $64, %rsp`. To free them on function exit, it executes `addq $64, %rsp` (or `leave`). Zero free-lists, zero fragmentation, zero syscalls.
3. **Anatomy of a Stack Frame (x86-64):**
* **Return Address:** Pushed onto the stack automatically by the `call` instruction so `ret` knows where to resume in the caller.
* **Saved Base Pointer (`%rbp`):** Marks the fixed anchor of the current frame.
* **Local Variables & Spilled Registers:** Accessed at negative offsets from `%rbp` (e.g., `-4(%rbp)`, `-8(%rbp)`).


4. **Strict Size Limit:** On Linux, the default stack limit is **8 MB** (`ulimit -s`). Exceeding it hits an unmapped **guard page** and triggers `SIGSEGV` (**Stack Overflow**).


* **Visual / Structure (Dual Coding):**

```text
High Address
┌────────────────────────────────────────┐
│ Caller Frame: main()                   │
├────────────────────────────────────────┤
│ Return Address (back to main)          │ ◄── Pushed by `call foo`
├────────────────────────────────────────┤
│ Saved %rbp (main's frame pointer)      │ ◄── Current %rbp points here
├────────────────────────────────────────┤
│ Local var: int a = 10   (-4(%rbp))     │
│ Local var: char buf[16] (-20(%rbp))    │ ◄── Current %rsp points here
└────────────────────────────────────────┘
                   │
                   ▼  (Next function call grows DOWNWARD)
Low Address

```

* **Concrete Examples:**
1. **Deep Recursion Crash:** Calling a recursive function without a base case pushes a new stack frame on every call until `%rsp` crosses the 8 MB limit → **Stack Overflow (`SIGSEGV`)**.
2. **Large Local Array Crash:** Declaring `int huge[3000000];` (12 MB) *inside* `main()` overflows the 8 MB stack immediately on function entry. Moving it to global scope (`.bss`) or `malloc` (Heap) fixes it instantly.


* **Traps & Edge Cases (Warning):**
* **Returning a Pointer to a Local Variable:**
```c
int *bad() {
    int x = 42;
    return &x; // BUG: x's stack frame is popped on return!
}

```


The memory address still exists, so it might *appear* to work briefly until the next function call overwrites that exact stack slot.
* **Stack Buffer Overflow (Security):** Because arrays grow *upward* (`buf[0]` → `buf[15]`) while the stack grows *downward*, writing past the end of `buf` overwrites the **Saved `%rbp**` and **Return Address** sitting right above it—allowing attackers to hijack control flow (mitigated by Stack Canaries, ASLR, and NX).



---

#### Module 5: Why do Heap and Stack grow toward each other, and what sits between them?

* **Plain-English Intuition (Feynman):**
Before running a program, the OS has no idea whether it will need a huge Heap and tiny Stack (like a database) or a tiny Heap and huge Stack (like a recursive parser). Placing them at opposite ends of the empty space lets both share the same pool of virtual addresses without wasting space.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Maximizing Free Space:** In 32-bit systems (only 4 GB total virtual space), opposite growth prevented artificial fixed-size partitions from running out early. In 64-bit systems (128 TB user space on x86-64), they virtually never collide, but the convention remains.
2. **Memory Mapping Segment (`mmap`):** Positioned in the middle gap between Heap and Stack. Holds:
* Dynamically linked shared libraries (`libc.so.6`, `libm.so.6`).
* Memory-mapped files (`mmap()`).
* Large `malloc()` allocations.


3. **ASLR (Address Space Layout Randomization):** Modern OS kernels (Linux default) randomize the exact starting base addresses of the **Stack**, **Heap**, **mmap segment**, and **PIE executable (`.text`)** on every single run so attackers cannot hardcode jump addresses.


* **Visual / Structure (Dual Coding):**

```text
Run 1: Stack @ 0x7ffd8a2... | mmap @ 0x7f1a4b... | Heap @ 0x55a1c2...
Run 2: Stack @ 0x7ffe19b... | mmap @ 0x7f9c02... | Heap @ 0x5610f8...
       (Exact same binary, randomized base addresses via ASLR)

```

* **Concrete Examples:**
1. **Live Memory Map Inspection:** Running `cat /proc/<pid>/maps` on Linux prints the exact virtual address ranges, permissions (`r-xp`, `rw-p`), and backing files (`[heap]`, `[stack]`, `libc.so.6`) of any running process.
2. **Multi-Threaded Stacks:** In a multi-threaded C program (`pthread_create`), all threads **share** one `.text`, `.data`, `.bss`, and **Heap**, but each thread gets its own separate **Stack** allocated in the `mmap` region.


* **Traps & Edge Cases (Warning):**
* **Virtual Address $\neq$ Physical RAM:** Two processes can both have a variable at virtual address `0x55a1c20010` simultaneously. The CPU's **MMU (Memory Management Unit)** and OS page tables translate each process's virtual address to completely different physical RAM frames.



---

### LAYER 3: SYNTHESIS & INTERLEAVING

#### Compare & Contrast Table

| Segment | What Lives Here | Lifetime | Permissions | Disk Space in Binary? | Managed By |
| --- | --- | --- | --- | --- | --- |
| **Text (`.text`)** | Compiled machine instructions | Entire process | **`r-x`** | Yes | OS Loader |
| **Read-Only (`.rodata`)** | String literals, `const` globals | Entire process | **`r--`** | Yes | OS Loader |
| **Initialized Data (`.data`)** | Non-zero globals & `static` vars | Entire process | **`rw-`** | Yes | OS Loader |
| **BSS (`.bss`)** | Uninit / zero globals & `static` vars | Entire process | **`rw-`** | **No** (size only) | OS Loader (zero-filled) |
| **Heap** | `malloc` / `calloc` / `realloc` blocks | Until `free()` | **`rw-`** | No | Programmer + Allocator (`brk`/`mmap`) |
| **Stack** | Local vars, params, return addrs | Function call (LIFO) | **`rw-`** | No | Compiler + CPU (`%rsp`/`%rbp`) |

#### Cross-Connections

* **Build Pipeline → Process Memory:** The ELF sections created in **Stage 3 (Assembly)** and merged in **Stage 4 (Linking)**—`.text`, `.rodata`, `.data`, `.bss`—are literally the exact static segments mapped into the lower half of Process Memory by the OS loader. The **Heap** and **Stack** do *not* exist in the ELF file; they are created dynamically at runtime.
* **Stack/Heap → Systems Security:** Understanding that buffers grow *upward* toward the Return Address on a *downward*-growing Stack explains classic **Stack Smashing**; understanding **`r-x` vs `rw-**` explains why **NX (Non-Executable Stack/Heap)** stops shellcode injection; understanding dynamic base addresses explains **ASLR**.

---

### LAYER 4: LOCK-IN LAYER

#### 1. Blank-Page Self-Test

Answer all 8 before checking the key.

1. **Placement Trace:** For the code below, map every single symbol/storage (`a`, `b`, `c`, `p`, `"hi"`, `*p`) to its exact memory segment:
```c
int a = 5;
int b;
void f() {
    static int c = 0;
    char *p = "hi";
    int *q = malloc(sizeof(int));
}

```


2. **Disk vs. RAM:** Why does `int arr[1000000];` take 0 bytes of data in the executable file on disk, yet 4 MB of RAM when used?
3. **Crash Diagnosis:** Explain why `char s[] = "abc"; s[0] = 'z';` works, while `char *s = "abc"; s[0] = 'z';` crashes with `SIGSEGV`.
4. **Performance:** Why is allocating 100 bytes on the Stack orders of magnitude faster than `malloc(100)` on the Heap?
5. **Growth Direction:** Why does a Stack buffer overflow overwrite the function's **return address** instead of overwriting unallocated free space?
6. **Multi-Threading:** If Thread A and Thread B run in the same C process, which memory segments do they share, and which do they NOT share?
7. **Large Allocations:** What happens differently under the hood in glibc when you call `malloc(64)` vs. `malloc(10 * 1024 * 1024)` (10 MB)?
8. **Lifetime Bug:** A function declares `int res = 99;` and returns `&res`. Why does the compiler warn you, and what happens to that memory address?

1. `a` → **`.data`**; `b` → **`.bss`**; `c` → **`.bss`** (static initialized to 0); `p` (the pointer variable) → **Stack**; `"hi"` (the string bytes) → **`.rodata`**; `q` (the pointer variable) → **Stack**; `*q` (the allocated `int`) → **Heap**.
2. It lives in **`.bss`**. The ELF binary only stores its size metadata (0 data bytes on disk); the OS allocates zero-filled RAM pages at runtime.
3. `char s[]` copies `"abc"` into a writable **Stack** array (`rw-`). `char *s` points directly into **`.rodata`** (`r--`), so writing triggers a hardware MMU page fault (`SIGSEGV`).
4. Stack allocation is a single CPU instruction (`subq $100, %rsp`). Heap allocation requires searching allocator free-lists, handling alignment/headers, acquiring locks, and occasionally making OS syscalls (`brk`/`mmap`).
5. The Stack grows **downward** (new locals are at lower addresses than the saved `%rbp` and Return Address), while array indexing (`buf[0]`, `buf[1]`, ...) writes **upward** toward higher addresses—directly into the Return Address.
6. **Shared:** `.text`, `.rodata`, `.data`, `.bss`, Heap, and `mmap` libraries. **Private per thread:** Each thread has its own **Stack** (and CPU registers).
7. `malloc(64)` allocates from the main **Heap** (managed via `brk`/`sbrk`). `malloc(10 MB)` exceeds glibc's mmap threshold (~128 KB), so it calls **`mmap()`** directly in the Memory Mapping Segment, allowing `free()` to immediately return the pages to the OS via `munmap()`.
8. `res` lives in the function's **Stack frame**, which is popped/invalidated the instant the function returns. The returned pointer is a **dangling stack pointer** that will be silently overwritten by subsequent function calls.

---

#### 2. Anki Flashcards

* **Q:** Order the 5 main user-space memory segments from **lowest** virtual address to **highest**.
**A:** Text (`.text`) → Initialized Data (`.data`) → Uninitialized Data (`.bss`) → Heap (↑) → [mmap] → Stack (↓).
* **Q:** What are the MMU permissions (`rwx`) for `.text`, `.rodata`, and `.data`/`.bss`/Heap/Stack?
**A:** `.text` = `r-x`; `.rodata` = `r--`; `.data`, `.bss`, Heap, Stack = `rw-`.
* **Q:** Where does a `static int x = 0;` declared *inside* a function live?
**A:** `.bss` (uninitialized/zero-initialized static storage, never the Stack).
* **Q:** What is the difference in initial value between an uninitialized global variable and an uninitialized local variable?
**A:** Global (`.bss`) is guaranteed to be `0`; Local (Stack) contains indeterminate garbage (Undefined Behavior to read).
* **Q:** Which direction do the Heap and Stack grow in virtual memory?
**A:** Heap grows **upward** (Low → High via `brk`); Stack grows **downward** (High → Low via `%rsp`).
* **Q:** Why can multiple running instances of the same program share physical RAM for `.text`?
**A:** Because `.text` is read-only (`r-x`), so no process can modify it.
* **Q:** How does `free(ptr)` know how many bytes to free when you don't pass a size argument?
**A:** `malloc` stores a hidden metadata header (containing the chunk size) immediately before the returned `ptr` address.
* **Q:** What is the default Stack size limit on Linux, and what happens if you exceed it?
**A:** 8 MB (`ulimit -s`); exceeding it hits an unmapped guard page and crashes with `SIGSEGV` (Stack Overflow).
* **Q:** Where do dynamically linked shared libraries (`.so`) and large `malloc()` blocks live?
**A:** In the **Memory Mapping Segment (`mmap`)** located between the Heap and the Stack.
* **Q:** Which Linux command shows the live virtual memory map of a running process?
**A:** `cat /proc/<pid>/maps` (and `size ./binary` shows static `.text`, `.data`, `.bss` sizes before running).

---

#### 3. 60-Second Revision Block (Day 1 / 3 / 7 / 21)

* **Layout (Low → High):** `.text` (`r-x`) → `.rodata` (`r--`) → `.data` (`rw-`) → `.bss` (`rw-`) → **Heap ↑** (`rw-`) → `mmap` (`.so` libs) → **Stack ↓** (`rw-`) → Kernel.
* **Static Lifetime (`.data` vs `.bss`):** Both hold globals & `static` locals. Non-zero goes to `.data` (takes disk space); zero/uninitialized goes to `.bss` (0 data bytes on disk, OS zero-fills RAM).
* **Heap (↑):** Manual (`malloc`/`free`), flexible lifetime, grows upward via `brk` (or `mmap` for large chunks). Slower; prone to leaks, fragmentation, and dangling pointers.
* **Stack (↓):** Automatic LIFO frames (locals, saved `%rbp`, return addr), grows downward via `%rsp`. 1-instruction speed, ~8 MB limit, prone to Stack Overflow and returning dangling local pointers.
* **Security:** `r-x` vs `rw-` (NX bit) blocks executing Stack/Heap; ASLR randomizes segment base addresses every run.