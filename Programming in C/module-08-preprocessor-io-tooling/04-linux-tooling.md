# Linux Build and Debugging Tools

Compile with useful diagnostics and debug symbols:

```sh
gcc -Wall -Wextra -Werror -g source.c -o program
```

Use `gdb` to set breakpoints, step through execution, inspect variables, and
examine call frames. For memory issues, use Valgrind where available or build
with AddressSanitizer:

```sh
gcc -Wall -Wextra -g -fsanitize=address source.c -o program
```

Tool availability and exact options depend on the installed compiler and
environment.
