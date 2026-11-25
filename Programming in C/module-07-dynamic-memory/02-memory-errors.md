# Memory Errors

- **Memory leak:** an allocated block becomes unreachable without being freed.
- **Use after free / dangling pointer:** accessing an object after its
  allocation has been released.
- **Double free:** releasing the same allocation more than once.
- **Out-of-bounds access:** reading or writing beyond the allocated object,
  potentially corrupting adjacent data.
- **Returning a local address:** returning a pointer to an automatic object
  whose lifetime ends when the function returns.

These are correctness and memory-safety failures. Do not rely on a particular
result after invalid access; behavior is undefined.
