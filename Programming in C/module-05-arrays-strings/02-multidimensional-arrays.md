# Multidimensional Arrays

C stores multidimensional arrays in row-major order. For `int a[R][C]`, element
`a[i][j]` is at an offset of `i * C + j` `int` elements from the first element.

| Expression | Type / meaning |
|---|---|
| `a` in most expressions | `int (*)[C]`, pointer to a row |
| `*a` in most expressions | `int *`, pointer to the first element of row 0 |
| `&a` | `int (*)[R][C]`, pointer to the entire 2D array |

The column count is required in a pointer-to-row type because it determines
the row stride.
