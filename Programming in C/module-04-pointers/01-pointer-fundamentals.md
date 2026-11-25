# Pointer Fundamentals and Categories

- `&x` produces the address of an object (when permitted); `*p` accesses the
  object designated by a valid pointer.
- Pointer sizes are implementation-dependent. They are commonly 4 bytes on
  32-bit systems and 8 bytes on 64-bit systems, independent of the pointed-to
  type on those systems.
- A **null pointer** points to no object. A **wild pointer** is uninitialized;
  a **dangling pointer** no longer designates a live object. Do not dereference
  any of them.
- `void *` can hold a pointer to an object of any type and converts back to the
  original object-pointer type. Standard C does not allow arithmetic on
  `void *` or dereferencing it without first converting to an appropriate
  object pointer type.
