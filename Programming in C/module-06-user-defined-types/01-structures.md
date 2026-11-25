# Structures and Member Access

Use `.` with a structure object and `->` with a pointer to a structure:

```c
node.value
node_ptr->value  /* equivalent to (*node_ptr).value */
```

Postfix operators such as `->` and `++` bind more tightly than unary `*`.
Therefore, parse `*p++` as `*(p++)`; use parentheses to distinguish pointer
movement from pointed-to-value modification. Structures passed as arguments
are passed by value; pass a pointer to operate on the caller's structure.
