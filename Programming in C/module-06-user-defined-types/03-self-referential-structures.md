# Self-Referential Structures

A structure can contain pointers to its own type, enabling linked lists, trees,
and graph representations:

```c
struct Node {
    int value;
    struct Node *next;
};
```

It cannot contain a complete object of its own type directly as a member,
because that would require infinite size. A pointer to the type is a
fixed-size member.
