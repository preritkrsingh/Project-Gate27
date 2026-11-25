# Unions and Endianness

Union members share storage. Its size is sufficient for its largest member
and appropriate alignment; padding may affect the final size. Reading a
different member from the one most recently written has limited,
implementation-dependent aspects; it is not a universally portable type-
punning technique.

**Endianness** describes byte order for multibyte values. A common way to
inspect it on a specified platform is to examine an integer's bytes through an
`unsigned char *`. C does not mandate little-endian byte order.
