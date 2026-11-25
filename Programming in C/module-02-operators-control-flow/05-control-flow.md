### LAYER 1: THE SPEED LAYER

#### 1. Core Intuition (Feynman)

At the CPU level, there are no `if`s, `switch`es, or `for` loops—only **conditional jumps** (`if zero, jump to line X`) and **unconditional jumps** (`jump back to line Y`). C's control-flow statements are thin, structured wrappers around those jumps. Because they are so thin, their quirks come straight from how the compiler translates them: `else` ignores your indentation and hugs the nearest `if`, `switch` is just a jump-table with labels (not isolated blocks), and `continue` behaves differently in `for` vs. `while` loops.

#### 2. Master Mental Map

```text
                                 C CONTROL FLOW
                                       │
          ┌────────────────────────────┴────────────────────────────┐
          ▼                                                         ▼
      BRANCHING                                                  LOOPING
          │                                                         │
   ┌──────┴──────────────┐                             ┌────────────┴────────────┐
   ▼                     ▼                             ▼                         ▼
if - else             switch                    Entry-Controlled          Exit-Controlled
• Dangling-else:      • Int/char/enum ONLY      (0 or more runs)          (1 or more runs)
  binds to NEAREST    • Cases = Labels          • while (cond)            • do { } while (cond);
  unmatched `if`      • Falls through w/o break • for (init; cond; upd)
                      • `default` anywhere
                                                       │
                      ┌────────────────────────────────┴────────────────────────┐
                      ▼                                                         ▼
                   `break`                                                 `continue`
     Exits INNERMOST loop OR `switch`                         Skips rest of INNERMOST loop body
                                                              • `for`: jumps to `update` → `cond`
                                                              • `while`: jumps straight to `cond`!

```

#### 3. 80/20 High-Yield Cheat Sheet

* **Dangling-Else Rule:** C ignores whitespace and indentation. An `else` **always** binds to the **nearest preceding unmatched `if**` in the same block. Always use `{}` to override.
* **`switch` Rules:**
* Controlling expression must be an **integer type** (`int`, `char`, `enum`—never `float`, `double`, or strings).
* `case` labels must be **compile-time integer constants** (no variables, no duplicate values).
* Execution **falls through** every subsequent `case` (and `default`) until it hits a `break`, `return`, or the closing `}`.


* **`continue` Trap (`for` vs. `while`):** In a `for` loop, `continue` executes the **`update` expression** (`i++`) before re-checking `cond`. In a `while` loop, `continue` jumps straight to `cond`, **skipping** any `i++` at the bottom of the loop (classic infinite loop bug!).
* **`break` Scope Rule:** `break` exits **only one level**—the innermost enclosing loop or `switch`.
* **Loop Invariant Rule (GATE/DSA):** A property that is true (1) **before** the loop starts, (2) **before and after** every iteration, and (3) **at termination** (combined with `!cond` to prove the algorithm's result).

---

### LAYER 2: EXHAUSTIVE DEEP-DIVE

#### Module 1: How does C resolve the "dangling-else" ambiguity?

* **Plain-English Intuition (Feynman):**
The C parser does not see indentation—it strips all spaces and newlines during Lexing (Stage 2). When you nest two `if`s without braces followed by a single `else`, the parser resolves the tie using one rigid rule: **pair the `else` with the closest open `if` above it.**
* **Exact Mechanics (Elaborative Interrogation):**
1. **Lexer Strips Whitespace:** Indentation is strictly for human eyes; the compiler sees a flat stream of tokens.
2. **Nearest-Unmatched-`if` Binding:** In `if (A) if (B) stmt1; else stmt2;`, the parser attaches `else stmt2;` to **`if (B)`**, never `if (A)`.
3. **Forcing Outer Binding:** To attach `else` to `if (A)`, you **must** wrap the inner `if (B)` in curly braces `{ ... }`, closing its scope before the `else` appears.


* **Visual / Structure (Dual Coding):**

```text
MISLEADING INDENTATION (Bug!)           ACTUAL C PARSE                    FIXED WITH BRACES
─────────────────────────────           ────────────────────              ─────────────────
if (a > 0)                              if (a > 0) {                      if (a > 0) {
    if (b > 0)                              if (b > 0)                        if (b > 0)
        printf("1");                            printf("1");                      printf("1");
else                                        else                          } else {
    printf("2");  // Looks outer!               printf("2"); // Inner!        printf("2");
                                        }                                 }

```

* **Concrete Examples:**
1. **Trace on Misleading Code:** With `a = -5, b = 10` in the left snippet above, **`if (a > 0)` is False**, so the entire inner `if-else` is skipped—**nothing prints!** (A reader fooled by indentation expects `"2"`.)
2. **`else if` Ladder:** `if (A) ... else if (B) ... else ...` is not a special keyword in C—it is literally an `if` nested inside the `else` of the previous `if`.


* **Traps & Edge Cases (Warning):**
* **The Stray Semicolon Bug:**
```c
if (x > 0);          // Semicolon ends the `if` with an EMPTY statement!
{
    printf("Hi\n");  // ALWAYS executes, even if x <= 0!
}

```


And if you put an `else` after that block, Stage 2 throws a **`'else' without a previous 'if'`** syntax error.



---

#### Module 2: Why does `switch` fall through, and how does `default` placement work?

* **Plain-English Intuition (Feynman):**
A `switch` statement is **not** a chain of independent `if-else` blocks—it is a single room with multiple entry doors (`case` labels). The CPU jumps through *one* door and then keeps walking straight through the rest of the room until it hits an exit sign (`break`) or the back wall (`}`).
* **Exact Mechanics (Elaborative Interrogation):**
1. **Jump Table Implementation:** For dense integer cases (e.g., `0, 1, 2, 3, 4`), the compiler builds an $O(1)$ **jump table** in `.rodata` (array of code addresses) and jumps directly to the matching `case` label in 1 step—faster than an $O(n)$ `if-else` chain.
2. **Strict Type & Constant Rules:**
* `switch (expr)`: `expr` undergoes **Integer Promotion** and must be an integer type (`int`, `char`, `short`, `long`, `enum`). Floats and strings are compile errors.
* `case const_expr:`: Must be a **compile-time integer constant** (`1`, `'A'`, `ENUM_VAL`, `2 + 3`). Variables (`case x:`) are illegal. Duplicate case values are compile errors.


3. **Fall-Through:** `case` labels are purely **jump targets** (like `goto` labels). They do *not* terminate the previous case. Without `break`, execution falls straight into the next case's statements—ignoring its `case` condition.
4. **`default` Placement:** `default:` can appear **anywhere** inside the `switch` (top, middle, or bottom).
* The `switch` checks **all `case` labels first**.
* Only if **no `case` matches** does it jump to `default:`.
* Once inside `default:`, if there is no `break`, it **falls through** into whatever `case` labels sit physically below it!




* **Visual / Structure (Dual Coding — `default` at the Top):**

```c
int x = 99;
switch (x) {
    default:  printf("D");  // 1. No case matches 99 -> jumps here, prints "D"
                            //    (NO break -> FALLS THROUGH!)
    case 1:   printf("1");  // 2. Prints "1"
              break;        // 3. Exits switch
    case 2:   printf("2");
}
// Output: "D1"

```

* **Concrete Examples:**
1. **Intentional Fall-Through (Grouping Cases):**
```c
switch (ch) {
    case 'a': case 'e': case 'i': case 'o': case 'u':
        is_vowel = 1;
        break;
    default:
        is_vowel = 0;
}

```


2. **Unreachable Code in `switch`:** Any statement placed *inside* `switch (x) {` **before** the first `case:` or `default:` (e.g., `int y = 5; case 1: ...`) is **never executed** because `switch` jumps directly to a label!


* **Traps & Edge Cases (Warning):**
* **Variable Declaration Directly After `case:` (C99/C11):**
```c
switch (x) {
    case 1:
        int a = 10; // COMPILE ERROR in C99/C11! (A label must precede a statement)
        break;
}

```


*Fix:* Wrap the case body in braces: `case 1: { int a = 10; break; }`.



---

#### Module 3: How do `for`, `while`, and `do-while` differ in execution order and Loop Invariants?

* **Plain-English Intuition (Feynman):**
`while` and `for` check the condition **at the front door** (so the body might run **0 times**). `do-while` lets you inside first and checks the condition **at the exit door** (guaranteeing the body runs **at least 1 time**). A **Loop Invariant** is the one mathematical truth that stays unbroken on every single lap of the loop, proving your loop actually solves the problem.
* **Exact Mechanics (Elaborative Interrogation):**
1. **`for (init; cond; update) body` Execution Order:**
* Step 1: `init` runs **once** at the very start (C99 allows `for (int i = 0; ...)` scoped to the loop).
* Step 2: `cond` is evaluated. If `0` (False), exit loop immediately. (An omitted `cond` in `for (;;)` is treated as **`1` (True)**.)
* Step 3: `body` executes.
* Step 4: `update` executes, then jump back to Step 2.


2. **`while (cond) body`:** Evaluates `cond` first; if True, runs `body` and repeats. Runs $0$ or more times.
3. **`do { body } while (cond);`:** Runs `body` first, then evaluates `cond`. Runs $1$ or more times. **Requires a trailing semicolon `;`.**
4. **Loop Invariants (3-Step Correctness Proof — GATE Staple):**
* **Initialization:** Invariant $I$ is true immediately *before* the first iteration.
* **Maintenance:** If $I$ is true before an iteration and `cond` is True, then $I$ remains true *after* the iteration.
* **Termination:** When the loop ends (`cond` is False), $I \land (\neg\text{cond})$ mathematically implies the desired post-condition.




* **Visual / Structure (Dual Coding — `for` Execution Flow):**

```text
   ┌──────────┐
   │  1. init │  (Runs ONCE)
   └────┬─────┘
        ▼
   ┌──────────┐  False
┌─►│  2. cond ├──────────► [EXIT LOOP]
│  └────┬─────┘
│       │ True
│       ▼
│  ┌──────────┐
│  │  3. body │
│  └────┬─────┘
│       ▼
│  ┌──────────┐
└──┤ 4. update│
   └──────────┘

```

* **Concrete Examples:**
1. **Loop Invariant Example (Array Sum):**
```c
int sum = 0;
for (int i = 0; i < n; i++) {
    sum += a[i];
}

```


* **Invariant:** At the start of each iteration, `sum` equals $\sum_{k=0}^{i-1} a[k]$.
* **Init (`i = 0`):** Empty sum = `0` (True).
* **Maintenance:** Adding `a[i]` and incrementing `i` preserves the sum for $0 \dots i$.
* **Termination (`i == n`):** `sum` equals $\sum_{k=0}^{n-1} a[k]$ (Proven!).


2. **The `do { ... } while (0)` Macro Idiom:** Wrapping a multi-statement `#define` macro in `do { stmt1; stmt2; } while (0)` lets callers safely write `if (cond) MY_MACRO(); else ...` without breaking the `if-else` structure or leaving a stray semicolon.


* **Traps & Edge Cases (Warning):**
* **Off-By-One Termination Value:** After `for (i = 0; i < 10; i++);`, the final value of `i` after the loop exits is **`10`**, not `9`. (If `i` is `unsigned` and the condition is `i >= 0`, it never terminates—`size_t` underflow!).



---

#### Module 4: How do `break` and `continue` differ—especially in `for` vs. `while` and nested `switch`?

* **Plain-English Intuition (Feynman):**
`break` is the **emergency exit**—it kills the current loop (or `switch`) completely. `continue` is the **skip-this-lap button**—it aborts the rest of the current iteration and jumps to the next lap. The catch: in a `for` loop, "next lap" still runs the `i++` update; in a `while` loop, "next lap" jumps straight to the condition, skipping any `i++` sitting below `continue`.
* **Exact Mechanics (Elaborative Interrogation):**
1. **`break`:** Immediately terminates the **innermost** enclosing `for`, `while`, `do-while`, or `switch`. Control resumes at the first statement *after* that block.
2. **`continue`:** Valid **only** inside loops (`for`, `while`, `do-while`)—**never** applies to `switch`.
* Inside `for (init; cond; update)`: jumps directly to **`update`**, then `cond`.
* Inside `while (cond)` / `do-while (cond)`: jumps directly to **`cond`**.
* Inside a `switch` nested inside a `for`/`while`: `continue` skips the rest of the **enclosing loop's** iteration!




* **Visual / Structure (Dual Coding — The `for` vs `while` `continue` Trap):**

```text
FOR LOOP (Terminates cleanly!)             WHILE LOOP (INFINITE LOOP BUG!)
──────────────────────────────             ───────────────────────────────
for (int i = 0; i < 5; i++) {              int i = 0;
    if (i == 2) continue; ──► runs i++!    while (i < 5) {
    printf("%d ", i);                          if (i == 2) continue; ──► skips i++!
}                                              printf("%d ", i);
// Output: 0 1 3 4                             i++;
                                           }
                                           // Output: 0 1 (hangs forever at i == 2!)

```

* **Concrete Examples:**
1. **Exiting Nested Loops:** `break` inside an inner `for` loop only exits the inner loop. To exit *both* outer and inner loops at once in C, either set a flag variable, `return` from a helper function, or use a clean forward `goto cleanup;`.
2. **`continue` inside `switch` inside `for`:**
```c
for (int i = 0; i < 3; i++) {
    switch (i) {
        case 1: continue; // Jumps to i++ in the `for` loop!
    }
    printf("%d ", i);
}
// Output: 0 2

```




* **Traps & Edge Cases (Warning):**
* **Thinking `for` and `while` are 100% identical:** `for (init; cond; upd) { body }` is only equivalent to `init; while (cond) { body; upd; }` **when `body` contains NO `continue` statements**. The moment `continue` is present, `for` executes `upd`, while `while` skips it.



---

### LAYER 3: SYNTHESIS & INTERLEAVING

#### Compare & Contrast Table

| Confused Pair | Option A | Option B | Decisive Rule / Diagnostic |
| --- | --- | --- | --- |
| **`if-else` vs. `switch**` | **`if-else`:** Any boolean/range/float/string condition. Sequential $O(n)$ checks. | **`switch`:** Exact integer/char/enum constants only. $O(1)$ jump table possible; falls through. | Use `switch` for discrete integer/enum dispatch; use `if-else` for ranges (`x > 10`) or floats. |
| **`while` vs. `do-while**` | **`while`:** Entry-controlled ($0+$ runs). No `;` after `while(cond)`. | **`do-while`:** Exit-controlled ($1+$ runs). **Must** end with `;`. | Use `do-while` when the body must execute at least once (menus, input validation, macro wrappers). |
| **`break` vs. `continue**` | **`break`:** Exits innermost **loop OR `switch**`. | **`continue`:** Skips to next iteration of innermost **loop ONLY** (ignores `switch`). | Remember: `continue` in `for` runs `update`; `continue` in `while` skips body's `i++`. |

#### Cross-Connections

* **Sequence Points → Control Flow:** The controlling expression of `if (expr)`, `while (expr)`, `switch (expr)`, and each of the 3 clauses in `for (init; cond; upd)` ends with a **Sequence Point**. So `while (i++ < 5)` commits `i`'s increment completely before entering the loop body.
* **Short-Circuiting → Loop Conditions:** `while (i < n && arr[i] != key)` relies on `&&` short-circuiting to prevent out-of-bounds reads on the final iteration when `i == n`.
* **Build Pipeline (Stage 2) → `switch`:** During Compilation (Stage 2), dense `switch` statements are lowered into a `.rodata` jump table and an indirect `jmp` instruction, explaining why `case` labels *must* be compile-time constants.

---

### LAYER 4: LOCK-IN LAYER

#### 1. Blank-Page Self-Test

Trace or answer all 7 before opening the key.

1. **Dangling-Else Trace:** What does this print?
```c
int a = 0, b = 1;
if (a)
    if (b)
        printf("A");
else
    printf("B");

```


2. **`switch` Fall-Through + `default` Placement:** What does this print?
```c
int x = 5;
switch (x) {
    default: printf("X");
    case 1:  printf("1");
    case 2:  printf("2"); break;
    case 3:  printf("3");
}

```


3. **`continue` in `for` vs `while`:** How many times does `"Hi"` print in each loop?
* Loop A: `for (int i = 0; i < 4; i++) { if (i == 2) continue; printf("Hi "); }`
* Loop B: `int i = 0; while (i < 4) { if (i == 2) continue; printf("Hi "); i++; }`


4. **`switch` Inside a Loop:** What does this print?
```c
for (int i = 0; i < 3; i++) {
    switch (i) {
        case 0: printf("A"); break;
        case 1: printf("B"); continue;
        case 2: printf("C");
    }
    printf("!");
}

```


5. **Stray Semicolon:** What does this print?
```c
int i;
for (i = 0; i < 5; i++);
printf("%d\n", i);

```


6. **`switch` Legality:** Why does `float f = 1.0f; switch (f) { case 1.0f: break; }` fail to compile?
7. **Loop Invariant:** For `int p = 1; for (int i = 0; i < n; i++) p *= 2;`, state the loop invariant at the start of each iteration.

1. **Prints nothing.** `else` binds to the nearest `if (b)`. Because `if (a)` is `0` (False), the entire inner `if-else` is skipped.
2. **Prints `X12`.** `x = 5` matches no `case`, so it jumps to `default` (prints `X`), falls through to `case 1` (prints `1`), falls through to `case 2` (prints `2`), and exits at `break`.
3. * Loop A: Prints `"Hi "` **3 times** (`i = 0, 1, 3`; `continue` still runs `i++`).
* Loop B: Prints `"Hi "` **2 times** (`i = 0, 1`), then enters an **infinite loop** at `i == 2` because `continue` skips `i++`.


4. **Prints `A!BC!`.**
* `i = 0`: prints `A`, `break` exits `switch`, prints `!`.
* `i = 1`: prints `B`, `continue` skips `printf("!")` and jumps directly to `i++` in the `for` loop!
* `i = 2`: prints `C`, exits `switch`, prints `!`.


5. **Prints `5`.** The `;` makes the `for` loop body empty; `i` increments `0→1→2→3→4→5` until `i < 5` is False, then `printf` runs once.
6. `switch` controlling expressions and `case` labels must be **integer types** (and `case` labels must be compile-time integer constants); floating-point types are forbidden by the C standard.
7. At the start of each iteration, **`p == 2^i`** (Init: `i=0, p=1=2^0`; Maintenance: `p *= 2` and `i++` preserves `2^i`; Termination: `i == n` gives `p == 2^n`).

---

#### 2. Anki Flashcards

* **Q:** How does C resolve a dangling `else` when braces `{}` are omitted?
**A:** It always binds `else` to the **nearest preceding unmatched `if**`, completely ignoring indentation.
* **Q:** What types are legal for a `switch` controlling expression and its `case` labels?
**A:** Controlling expression: **integer types only** (`int`, `char`, `enum`). `case` labels: **unique compile-time integer constant expressions**.
* **Q:** What happens if `default:` is placed at the *top* of a `switch` statement without a `break`?
**A:** `switch` still checks all `case`s first; if none match, it jumps to `default:` and **falls through** into the `case`s below it.
* **Q:** What is the exact difference in how `continue` behaves in a `for` loop vs. a `while` loop?
**A:** In `for`, `continue` jumps to the **`update` expression** before `cond`; in `while`, `continue` jumps straight to **`cond`**, skipping any increment at the bottom of the body.
* **Q:** What happens when `continue` is executed inside a `switch` that is nested inside a `for` loop?
**A:** `continue` ignores the `switch` and jumps to the **`update` expression of the enclosing `for` loop**.
* **Q:** What is the only loop in C guaranteed to execute its body at least once, and what syntax detail is required?
**A:** `do { ... } while (cond);` — requires a trailing **semicolon `;**`.
* **Q:** What are the 3 parts of a Loop Invariant correctness proof?
**A:** (1) **Initialization** (true before loop), (2) **Maintenance** (preserved by each iteration), (3) **Termination** (invariant + `!cond` proves the final result).
* **Q:** What happens to code placed inside `switch (x) {` *before* the first `case:` or `default:`?
**A:** It is **unreachable / never executed** because `switch` jumps directly to a matching label.

---

#### 3. 60-Second Revision Block (Day 1 / 3 / 7 / 21)

* **Dangling-Else:** Indentation is invisible to the compiler; `else` always binds to the **nearest unmatched `if**`. Watch for stray `;` after `if (cond);`.
* **`switch`:** Integer types only, compile-time constant `case`s, $O(1)$ jump table. Cases are **labels**, so execution **falls through** without `break`. `default` can sit anywhere (checked last, still falls through without `break`).
* **Loops:** `for` & `while` = entry-controlled ($0+$ runs); `do-while` = exit-controlled ($1+$ runs, ends with `;`). Prove loops via **Loop Invariants** (Init → Maintenance → Termination).
* **`break` vs. `continue`:** `break` exits innermost loop OR `switch`. `continue` applies to **loops only** (in `for` it runs `update`; in `while` it jumps straight to `cond`, risking infinite loops if `i++` is skipped).