# CLI and Multi-File Projects

A common command-line entry point is:

```c
int main(int argc, char *argv[])
```

`argc` is the argument count; `argv` contains argument strings, with
`argv[argc]` a null pointer. Put shared declarations in headers and definitions
in source files. Header guards prevent duplicate inclusion; the linker combines
compiled object files and libraries.

A minimal Makefile expresses targets, prerequisites, and build commands so
only out-of-date outputs need rebuilding.
