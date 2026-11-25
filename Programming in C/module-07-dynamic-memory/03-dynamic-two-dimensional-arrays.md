# Dynamic Two-Dimensional Arrays

Two common representations are:

1. **Array of row pointers:** allocate an `int *` table, then allocate each
   row. Rows may be noncontiguous, and every allocation needs cleanup.
2. **Contiguous block:** allocate `rows * columns` elements and compute
   `data[i * columns + j]`. This uses one allocation and gives contiguous
   elements; validate the multiplication before allocating.

The choice affects layout, cleanup, and compatibility with function parameter
types.
