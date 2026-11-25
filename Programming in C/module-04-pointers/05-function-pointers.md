# Function Pointers

```c
int (*fp)(int, int);
```

This declares `fp` as a pointer to a function taking two `int` arguments and
returning `int`. Function pointers support callbacks and dispatch tables.
Function pointers and object pointers are distinct types and cannot be
portably converted to one another.
