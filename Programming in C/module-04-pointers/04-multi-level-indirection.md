# Multi-Level Indirection

A double pointer such as `int **pp` points to an `int *`. It is useful when a
function must update a caller's pointer. For example, a linked-list insertion
function can receive `struct Node **head` and replace `*head`.

The pointer levels must match the object being addressed. Passing `int *` lets
a function modify an `int`; passing `int **` can let it modify an `int *`
owned by its caller.
