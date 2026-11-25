# Strings and the String Library

A C string is a sequence of `char` ending with a null character `'\0'`.

```c
char s[] = "abc";  /* modifiable array of 4 chars including '\0' */
char *p = "abc";   /* points to a string literal; modifying it is undefined */
```

`sizeof s` includes the array's full size, while `strlen(s)` counts characters
before the first null character. `strlen` requires a valid null-terminated
string.

Core `<string.h>` functions:

- `strcpy(dst, src)` copies through the terminating null character; destination
  must have enough space and source/destination regions must not overlap.
- `strcmp(a, b)` compares strings lexicographically and returns less than,
  equal to, or greater than zero; do not rely on exact nonzero values.
- `strcat(dst, src)` appends through the terminating null character; `dst`
  needs adequate capacity and the regions must not overlap.
