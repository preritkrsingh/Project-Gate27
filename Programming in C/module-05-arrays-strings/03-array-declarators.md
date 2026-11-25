# Array Declarators

```c
int *a[N];    /* array of N pointers to int */
int (*a)[N];  /* pointer to an array of N int */
```

Parentheses change which operator binds to the identifier. Read declarations
from the identifier outward.

When an array is used as a function parameter, its declaration is adjusted to
a pointer parameter. The function therefore does not receive the original
array's size through `sizeof` on that parameter; pass the size separately.
