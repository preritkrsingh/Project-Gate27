# `const` and Pointers

```c
const int *p;         /* pointer to const int: *p cannot be changed through p */
int *const p2 = ...;  /* const pointer: p2 cannot be reassigned */
const int *const p3 = ...; /* const pointer to const int */
```

Read from the identifier outward, paying attention to which `*` is qualified.
`const` restricts modification through that expression; it does not prove that
the underlying object is globally immutable.
