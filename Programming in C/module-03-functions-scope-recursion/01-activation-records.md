### LAYER 1: THE SPEED LAYER

#### 1. Core Intuition (Feynman)

Every time you call a function in C, the CPU needs a private scratchpad to remember **who called it** (where to go back to), **what inputs it received** (parameters), and **its own temporary variables** (locals). That private scratchpad is an **Activation Record (Stack Frame)**. Because functions always finish in **Last-In, First-Out (LIFO)** order—the most recently called function must finish before its caller can resume—these frames are stacked on top of each other on the **Call Stack**, pushed on entry and popped on `return`.

#### 2. Master Mental Map (Anatomy of a Stack Frame — x86-64 Linux)

```text
High Address
┌────────────────────────────────────────────────────────┐
│              CALLER'S STACK FRAME (main)               │
├────────────────────────────────────────────────────────┤
│  Arguments > 6 (if any, pushed right-to-left)          │  +16(%rbp) ...
├────────────────────────────────────────────────────────┤
│  RETURN ADDRESS (Saved %rip pushed by `call` instr)    │   +8(%rbp)  ◄── Where `ret` jumps
├────────────────────────────────────────────────────────┤
│  SAVED BASE POINTER (Caller's %rbp / Dynamic Link)     │    0(%rbp)  ◄── Current %rbp
├────────────────────────────────────────────────────────┤
│  Callee-Saved Registers (if used: %rbx, %r12–%r15)     │   -8(%rbp)
├────────────────────────────────────────────────────────┤
│  LOCAL VARIABLES (int a, char buf[16], etc.)           │  -16(%rbp) ...
├────────────────────────────────────────────────────────┤
│  Temporary / Spilled Values & Alignment Padding        │             ◄── Current %rsp
└────────────────────────────────────────────────────────┘
                           │
                           ▼  (Next `call` pushes a new frame DOWNWARD)
Low Address

```

#### 3. 80/20 High-Yield Cheat Sheet

* **Two CPU Registers Run the Whole Show:**
* **`%rsp` (Stack Pointer):** Always points to the **top** (lowest address) of the active stack frame. Moves constantly as data is pushed/popped.
* **`%rbp` (Base / Frame Pointer):** Anchored at a **fixed spot** inside the current frame so local variables (`-4(%rbp)`) and stack arguments (`+16(%rbp)`) have constant, predictable offsets.


* **The 3-Step Lifecycle of Every Function Call:**
1. **`call func`:** Pushes the **Return Address** (next instruction's `%rip`) onto the stack and jumps to `func`.
2. **Prologue (Push Frame):** `push %rbp` → `mov %rsp, %rbp` → `sub $N, %rsp` (saves caller's anchor, sets new anchor, carves out `N` bytes for locals).
3. **Epilogue (Pop Frame):** `mov %rbp, %rsp` → `pop %rbp` → `ret` (discards locals, restores caller's anchor, pops Return Address back into `%rip`).


* **Parameter Passing Reality (x86-64 System V ABI vs. Textbook):**
* **Textbook / 32-bit x86 (`cdecl`):** All parameters are pushed onto the stack.
* **Modern 64-bit Linux (x86-64):** The **first 6 integer/pointer arguments** are passed in CPU registers (`%rdi`, `%rsi`, `%rdx`, `%rcx`, `%r8`, `%r9`) for speed; only the **7th+ arguments** go onto the stack. Return value goes in **`%rax`**.


* **Recursion Rule:** Every recursive call pushes a **brand-new, independent Stack Frame**—which is why `fact(3)` and `fact(2)` have separate copies of parameter `n` that never overwrite each other.

---

### LAYER 2: EXHAUSTIVE DEEP-DIVE

#### Module 1: What exact fields live inside an Activation Record, and why is each one needed?

* **Plain-English Intuition (Feynman):**
An activation record is a "save state" + "workspace" bundle. Without the **Return Address**, the CPU would get lost after `return`. Without the **Saved Frame Pointer (Dynamic Link)**, the caller wouldn't know where its own variables are. Without dedicated **Local Variable slots**, nested or recursive calls would trample each other's data.
* **Exact Mechanics (Elaborative Interrogation):**
An Activation Record (in compiler theory / GATE and x86-64 hardware) contains 5 core components:
1. **Return Address:** The memory address (in `.text`) of the exact instruction immediately following the `call` in the caller. Automatically pushed onto the stack by the CPU's `call` instruction.
2. **Control Link / Dynamic Link (Saved `%rbp`):** A pointer to the **caller's** stack frame base. Forms a linked list of active frames all the way back to `main()` (which is literally how `gdb`'s `backtrace` command prints your call stack!).
3. **Actual Parameters (Arguments):** Values passed from caller to callee (in registers `%rdi`–`%r9` on x86-64, spilling to `+16(%rbp)`+ on the stack if $>6$ args, or if their address `&param` is taken).
4. **Local Variables:** Automatic (non-`static`) variables declared inside the function, allocated at negative offsets from `%rbp` (e.g., `-4(%rbp)`, `-8(%rbp)`).
5. **Saved Machine Status (Registers) & Return Value Slot:** Callee-saved registers (`%rbx`, `%r12`–`%r15`) backed up before the callee overwrites them, and restored before returning. Return values are passed back in `%rax` (or via a hidden pointer for large structs).


* **Visual / Structure (Dual Coding — The Dynamic Link Chain):**

```text
  [main's frame]   ◄───────────┐
   saved %rbp = 0              │
                               │ (Dynamic Link)
  [foo's frame]    ◄───────┐   │
   saved %rbp ─────────────┼───┘
                           │ (Dynamic Link)
  [bar's frame]            │
   saved %rbp ─────────────┘   ◄── Current %rbp
   local vars                  ◄── Current %rsp

```

* **Concrete Examples:**
1. **C Pass-by-Value Proof:** Because `foo(int x)` receives a *copy* of the argument inside `foo`'s own stack frame/register, modifying `x = 99;` inside `foo` only mutates `foo`'s frame—never the caller's variable.
2. **Static Locals Exception:** `static int count = 0;` inside `foo()` is **NOT** stored in `foo`'s activation record—it lives in `.bss`, so it survives when `foo`'s frame is popped.


* **Traps & Edge Cases (Warning):**
* **Static Link (Access Link) vs. Dynamic Link (GATE Compiler Design):** Standard C does **not** support nested functions (functions defined inside functions), so C activation records only need a **Dynamic Link** (caller's frame). Languages with nested lexical scopes (Pascal, Python, GCC's non-standard nested functions) also store a **Static Link (Access Link)** pointing to the lexically enclosing function's frame.



---

#### Module 2: Step-by-step, how do the Function Prologue and Epilogue push and pop a frame?

* **Plain-English Intuition (Feynman):**
Entering a function is like setting up a desk: save the previous person's bookmark (`push %rbp`), place your own bookmark (`mov %rsp, %rbp`), and clear desk space for your papers (`sub $N, %rsp`). Leaving is the exact reverse: sweep the desk clean (`mov %rbp, %rsp`), restore the previous bookmark (`pop %rbp`), and teleport back to where you were called (`ret`).
* **Exact Mechanics (Elaborative Interrogation):**
When `main()` executes `call add`:
1. **Caller (`call add`):**
* Decrements `%rsp` by 8 and writes the address of the next instruction in `main` (**Return Address**) at `(%rsp)`.
* Sets `%rip` (Instruction Pointer) to `add`'s first instruction in `.text`.


2. **Callee Prologue (Setup):**
* `pushq %rbp` → Decrements `%rsp` by 8 and saves `main`'s `%rbp` on the stack.
* `movq %rsp, %rbp` → Anchors `add`'s `%rbp` to this exact spot.
* `subq $32, %rsp` → Reserves 32 bytes for `add`'s local variables (16-byte aligned per x86-64 ABI).


3. **Callee Body Executes:** Reads/writes locals at `-4(%rbp)`, `-8(%rbp)`, puts result into `%eax`/`%rax`.
4. **Callee Epilogue (Teardown):**
* `movq %rbp, %rsp` → Instantly frees all 32 bytes of local variables in 1 instruction (bytes are *not* zeroed—just abandoned!).
* `popq %rbp` → Restores `main`'s `%rbp` and increments `%rsp` by 8. *(Note: `leave` is a single x86 instruction that does both `mov %rbp, %rsp` and `pop %rbp`.)*
* `ret` → Pops the Return Address off the stack into `%rip` and increments `%rsp` by 8, resuming `main` right after `call add`.




* **Visual / Structure (Dual Coding — Prologue & Epilogue Assembly):**

```text
add:
    pushq %rbp            # 1. Save caller's frame pointer
    movq  %rsp, %rbp      # 2. Establish my frame pointer
    subq  $16, %rsp       # 3. Allocate 16 bytes for local vars
    ...                   #    (Function body runs; result placed in %eax)
    leave                 # 4. movq %rbp, %rsp  +  popq %rbp  (Teardown)
    ret                   # 5. Pop return address into %rip   (Jump back)

```

* **Concrete Examples:**
1. **Why Stack Allocation is $O(1)$:** Whether a function declares 1 `int` or 50 local variables, allocating all of them is still a **single** `subq $N, %rsp` instruction—and freeing all of them is a single `movq %rbp, %rsp`.
2. **Why Uninitialized Locals Have Garbage:** Because `movq %rbp, %rsp` never zeroes out old stack frames, the next function that calls `subq $N, %rsp` inherits whatever leftover bytes the previous function left behind!


* **Traps & Edge Cases (Warning):**
* **`-fomit-frame-pointer` Optimization (`-O2`):** At `-O2`, GCC often skips `push %rbp` / `mov %rsp, %rbp` entirely, addressing locals directly via `%rsp` and freeing `%rbp` as an extra general-purpose register. (Debugging/profiling tools use `-fno-omit-frame-pointer` to keep clean stack traces.)



---

#### Module 3: How do frames push and pop during Nested Calls and Recursion?

* **Plain-English Intuition (Feynman):**
Think of a stack of trays. Calling `main()` → `A()` → `B()` stacks 3 trays. `B()` must finish and pop its tray before `A()` can continue; `A()` must pop before `main()` can continue. Recursion is **no different**—calling `fact(3)` → `fact(2)` → `fact(1)` just stacks 3 trays with the same label (`fact`) but completely different local numbers inside each tray.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Nested Call Trace (`main` calls `A`, `A` calls `B`):**
* Max stack depth = 3 active frames simultaneously (`main`, `A`, `B`).
* If `main` calls `A()`, `A()` returns, and *then* `main` calls `C()`: `C`'s frame **reuses the exact same memory addresses** that `A`'s frame just vacated!


2. **Recursive Call Trace (`fact(3)`):**
* **Winding Phase (Pushing):** `fact(3)` pauses at `3 * fact(2)` → pushes `fact(2)` → pauses at `2 * fact(1)` → pushes `fact(1)` → hits base case (`return 1`).
* **Unwinding Phase (Popping):** `fact(1)` pops and returns `1` in `%eax` → `fact(2)` resumes, computes `2 * 1 = 2`, pops → `fact(3)` resumes, computes `3 * 2 = 6`, pops.


3. **Space Complexity of Recursion:** Even if a recursive function uses $O(1)$ local variables, its **auxiliary space complexity** is **$O(\text{max recursion depth})$** because of the active stack frames!


* **Visual / Structure (Dual Coding — `fact(3)` Winding & Unwinding):**

```text
WINDING (Push Frames Downward)             UNWINDING (Pop Frames Upward)
──────────────────────────────             ─────────────────────────────
┌──────────────────┐                       ┌──────────────────┐
│ main()           │                       │ main()           │ ◄── gets 6
├──────────────────┤                       └──────────────────┘
│ fact(n=3)        │  (waiting: 3 * ?)           ▲ returns 6
├──────────────────┤                       ┌─────┴────────────┐
│ fact(n=2)        │  (waiting: 2 * ?)     │ fact(n=3)        │ (3 * 2 = 6)
├──────────────────┤                       └──────────────────┘
│ fact(n=1) → ret 1│                             ▲ returns 2
└──────────────────┘                       ┌─────┴────────────┐
  Max Depth = 4 frames                     │ fact(n=2)        │ (2 * 1 = 2)
                                           └──────────────────┘
                                                 ▲ returns 1

```

* **Concrete Examples:**
1. **DFS vs. Tree Height:** Recursive DFS on a balanced binary tree of $N$ nodes uses $O(\log N)$ stack space; on a skewed (linked-list-like) tree of $N$ nodes, it pushes $N$ frames ($O(N)$ space) and can cause **Stack Overflow** for $N = 10^6$.
2. **Tail Call Optimization (TCO):** If the recursive call is the **very last action** in the function (`return fact_tail(n - 1, acc * n);`), GCC at `-O2` replaces `call` with a simple **`jmp`**—reusing the **exact same stack frame** in $O(1)$ stack space!


* **Traps & Edge Cases (Warning):**
* **Why `return n * fact(n - 1);` is NOT Tail Recursive:** The multiplication `n * ...` happens **after** `fact(n - 1)` returns, so the CPU *must* keep `fact(n)`'s frame alive on the stack to remember `n`.



---

#### Module 4: What are the 3 classic Stack Frame failure modes?

* **Plain-English Intuition (Feynman):**
Because the Stack has a fixed size limit and stores both your **data** (local buffers) and your **navigation map** (Return Address) right next to each other, overflowing it or holding onto a popped frame causes immediate crashes or security compromises.
* **Exact Mechanics (Elaborative Interrogation):**
1. **Stack Overflow:** Infinite recursion (missing base case) or huge local arrays (`int a[5000000];`) push `%rsp` past the OS stack limit (default **8 MB** on Linux) into an unmapped guard page → hardware fault **`SIGSEGV`**.
2. **Dangling Stack Pointer (Returning `&local`):** Returning the address of a local variable hands the caller a pointer into a **popped frame**. The next function call pushes its own frame onto those exact addresses and silently corrupts the data.
3. **Stack Buffer Overflow (Return Address Hijack):** Writing past the end of a local array (`gets(buf)` or unchecked `strcpy`) writes **upward** (low → high addresses) straight over **Saved `%rbp**` and the **Return Address**. When `ret` executes, the CPU jumps to the attacker's overwritten address!


* **Visual / Structure (Dual Coding — Stack Smashing):**

```text
High Addr │ Return Address  │ ◄── OVERWRITTEN by overflow! (`ret` jumps to attacker!)
          │ Saved %rbp      │ ◄── OVERWRITTEN!
          │ buf[15] ... [0] │ ───► strcpy() writes UPWARD past buf[15]
Low Addr  └─────────────────┘

```

* **Concrete Examples:**
1. **Modern Mitigations:** GCC compiles with **Stack Canaries** (`-fstack-protector`: places a random secret value between `buf` and Saved `%rbp`, checking it before `ret`), **NX** (non-executable stack), and **ASLR**.
2. **Fixing Dangling Local Return:** Instead of returning `&local`, either return the struct/value **by value**, pass an output buffer pointer from the caller, or allocate on the **Heap** (`malloc`).


* **Traps & Edge Cases (Warning):**
* **Why a Dangling Stack Pointer "Works" in Simple Tests:** If you call `int *p = bad(); printf("%d", *p);`, it might occasionally print the old value because `mov %rbp, %rsp` didn't wipe the bytes—until `printf` itself pushes its own stack frame and overwrites `*p`! Never trust a test that "happened to work" on UB.



---

### LAYER 3: SYNTHESIS & INTERLEAVING

#### Compare & Contrast Table

| Confused Pair | Option A | Option B | Decisive Rule / Diagnostic |
| --- | --- | --- | --- |
| **`%rsp` vs. `%rbp**` | **`%rsp` (Stack Pointer):** Top (lowest addr) of stack; moves on every push/pop. | **`%rbp` (Base Pointer):** Fixed anchor for current frame; gives constant offsets to locals (`-N(%rbp)`). | `%rbp` makes debugging & variable addressing stable; `%rsp` tracks the live edge. |
| **Dynamic Link vs. Static Link** | **Dynamic Link (Saved `%rbp`):** Points to the **caller's** frame (runtime call chain). | **Static Link (Access Link):** Points to the **lexically enclosing** function's frame. | Standard C only uses **Dynamic Links** (no nested functions). |
| **Normal Recursion vs. Tail Recursion** | **Normal (`n * f(n-1)`):** Must keep frame alive for work *after* return → **$O(d)$ stack space**. | **Tail (`return f(n-1, acc)`):** No work after return → compiler converts `call` to `jmp` (**$O(1)$ stack space** at `-O2`). | If any operation happens *after* the recursive call returns, it is **not** tail recursive. |
| **Stack Frame vs. Heap Block** | **Stack Frame:** LIFO, 1-instruction alloc/free (`sub`/`mov`), ~8 MB limit. | **Heap Block:** Arbitrary lifetime (`malloc`/`free`), slower allocator, GBs available. | Use Stack for small function-scoped locals; use Heap for large or escaping data. |

#### Cross-Connections

* **Process Memory Layout → Activation Records:** The **Stack** segment (growing downward from high to low addresses) is literally a contiguous array of **Activation Records**.
* **Build Pipeline (Stage 2) → Prologue/Epilogue:** The compiler's Back-End (Stage 2) automatically injects the Prologue (`push %rbp; mov %rsp, %rbp; sub $N, %rsp`) and Epilogue (`leave; ret`) into every function in the `.s` assembly file.
* **Uninitialized Locals → Frame Teardown:** Because the Epilogue pops a frame by simply moving `%rsp` back up (without zeroing memory), uninitialized local variables in future calls inherit leftover garbage bytes from popped frames.

---

### LAYER 4: LOCK-IN LAYER

#### 1. Blank-Page Self-Test

Answer all 7 before opening the key.

1. **Register Roles:** What are the exact roles of **`%rsp`**, **`%rbp`**, and **`%rip`** during a function call?
2. **Prologue/Epilogue:** Why does `call foo` push the Return Address onto the stack *before* `foo`'s prologue pushes `%rbp`?
3. **x86-64 ABI:** If a C function takes 8 `int` parameters on 64-bit Linux, where are the first 6 passed, and where are the 7th and 8th passed?
4. **Recursion Space:** Consider `int fib(int n) { if (n <= 1) return n; return fib(n-1) + fib(n-2); }`. What is the **maximum number of activation records** on the stack at any single moment for `fib(5)`? (Hint: $O(2^n)$ or $O(n)$?)
5. **Garbage Values:** Why do uninitialized local variables contain random garbage instead of `0`, whereas uninitialized globals (`.bss`) are always `0`?
6. **Stack Smashing:** In a downward-growing stack, why does overflowing `char buf[16]` overwrite the **Return Address** of the current function rather than the next function's frame?
7. **Tail Recursion:** Why can't GCC apply Tail Call Optimization to `int sum(int n) { return n == 0 ? 0 : n + sum(n - 1); }`?

1. **`%rsp`:** Points to the top (lowest address) of the active stack frame. **`%rbp`:** Fixed base anchor of the current frame (holds saved caller `%rbp`). **`%rip`:** Instruction Pointer (holds address of next instruction in `.text`; pushed as Return Address by `call`).
2. `call foo` is executed by the **caller** (it must save where to return before jumping into `foo`). Once inside `foo`, the **callee's** prologue saves the caller's `%rbp` right below the Return Address.
3. First 6 parameters are passed in CPU registers (**`%rdi`, `%rsi`, `%rdx`, `%rcx`, `%r8`, `%r9**`); the 7th and 8th are pushed onto the **Stack** (accessed at `+16(%rbp)` and `+24(%rbp)`).
4. **$O(n)$ — specifically 5 frames** (`fib(5) → fib(4) → fib(3) → fib(2) → fib(1)`). Although there are $O(2^n)$ *total* calls, the left branch finishes and **pops** its frames before the right branch pushes new ones!
5. Popping a stack frame only moves `%rsp` upward without zeroing the abandoned bytes, so new frames reuse dirty memory. `.bss` is explicitly zero-filled by the OS loader at program startup.
6. Local arrays grow **upward** (from low addresses to high addresses: `buf[0]` → `buf[15]`), while the Saved `%rbp` and Return Address were pushed *earlier* at **higher** addresses right above `buf`.
7. Because the addition **`n + ...`** must execute **after** `sum(n - 1)` returns, so the current frame cannot be reused/discarded before the recursive call.

---

#### 2. Anki Flashcards

* **Q:** List the 4 main items stored in a C Activation Record (Stack Frame).
**A:** (1) Return Address, (2) Saved `%rbp` (Dynamic Link), (3) Parameters (spilled/stack args), (4) Local variables & callee-saved registers.
* **Q:** What does the x86-64 `call func` instruction do in hardware?
**A:** Pushes the **Return Address** (next `%rip`) onto the stack and jumps `%rip` to `func`.
* **Q:** Write the 3 standard x86-64 **Function Prologue** instructions.
**A:** `pushq %rbp` → `movq %rsp, %rbp` → `subq $N, %rsp`.
* **Q:** Write the standard x86-64 **Function Epilogue** instructions.
**A:** `movq %rbp, %rsp` → `popq %rbp` (or `leave`) → `ret`.
* **Q:** What does the `ret` instruction do in hardware?
**A:** Pops the Return Address from the top of the stack directly into `%rip`, resuming execution in the caller.
* **Q:** On x86-64 Linux (System V ABI), how are function arguments and return values passed?
**A:** First 6 integer/pointer args in `%rdi, %rsi, %rdx, %rcx, %r8, %r9`; 7th+ on the **Stack**; return value in **`%rax`**.
* **Q:** What is the difference between a **Dynamic Link** and a **Static Link** in an activation record?
**A:** Dynamic Link points to the **caller's** frame (used in C); Static Link points to the **lexically enclosing** function's frame (used in nested-function languages).
* **Q:** What is the maximum stack space complexity of recursive `fib(n)`?
**A:** **$O(n)$** (max call depth), NOT $O(2^n)$—because left-branch frames pop and are reused by right-branch calls.
* **Q:** How does a Stack Canary defend against Stack Buffer Overflows?
**A:** Places a random secret value between local buffers and Saved `%rbp`/Return Address, and aborts the program if that value is altered before `ret`.

---

#### 3. 60-Second Revision Block (Day 1 / 3 / 7 / 21)

* **Frame Anatomy (High → Low):** Stack args (`+16(%rbp)`) → **Return Address** (`+8(%rbp)`) → **Saved `%rbp**` (`0(%rbp)`, Dynamic Link) → Callee-saved regs → **Local variables** (`-N(%rbp)`) ← `%rsp`.
* **Lifecycle:** `call` pushes Return Address → **Prologue** (`push %rbp; mov %rsp, %rbp; sub $N, %rsp`) → Body → **Epilogue** (`leave; ret` pops locals, restores `%rbp`, jumps back to Return Address).
* **Nested & Recursive Calls:** Push downward (winding), pop upward (unwinding). Popped frames are **reused** by later calls (so `fib(n)` uses $O(n)$ stack space, and uninitialized locals inherit garbage). Tail recursion (`-O2`) reuses 1 frame via `jmp` ($O(1)$ space).
* **Failure Modes:** Exceeding 8 MB → **Stack Overflow (`SIGSEGV`)**; returning `&local` → **Dangling Stack Pointer**; overflowing `buf` upward → overwrites **Return Address** (mitigated by Canaries, NX, ASLR).