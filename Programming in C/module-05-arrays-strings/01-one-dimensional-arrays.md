# One-Dimensional Arrays and Decay

In most expressions, an array expression converts to a pointer to its first
element. Important exceptions include when it is the operand of `sizeof` or
unary `&` (and certain initialization contexts).

For an array `a`, within its bounds:

```c
a[i] == *(a + i) == *(i + a) == i[a]
```

`a` is not itself a pointer variable. `&a[0]` points to the first element;
`&a` points to the entire array and has a different pointer type.
