# Allocation Functions

Declared in `<stdlib.h>`:

- `malloc(size)` allocates a block whose contents are indeterminate.
- `calloc(count, size)` allocates an array and sets every byte to zero.
  All-bits-zero is not guaranteed to represent every pointer or floating-point
  zero on every implementation.
- `realloc(ptr, size)` resizes a block and may move it. On success, use the
  returned pointer. For a nonzero requested size, failure leaves the original
  block allocated; do not overwrite the only pointer before checking.
- `free(ptr)` releases a live allocation; `free(NULL)` has no effect.

Check allocation results before use. `sizeof *p` expresses the element size
without repeating the pointed-to type:

```c
int *p = malloc(count * sizeof *p);
if (p == NULL) {
    /* handle allocation failure */
}
```

Ensure a count/size multiplication cannot overflow before allocating.
