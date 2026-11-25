# Pointer Arithmetic

For a pointer `p` to an element of an array, `p + k` advances by `k` elements,
equivalent to a byte offset of `k * sizeof *p`. Arithmetic is defined only
within the same array object, including a one-past-the-end pointer; that
one-past pointer may not be dereferenced.

Subtracting two pointers into the same array yields an element count
(`ptrdiff_t`), not a byte count. Both pointers must refer into the same array
object or one past it.
