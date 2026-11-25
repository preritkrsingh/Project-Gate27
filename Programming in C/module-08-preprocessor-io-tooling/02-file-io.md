# File I/O

`<stdio.h>` represents streams with `FILE *`. Common functions include
`fopen`/`fclose`, `fread`/`fwrite`, `fprintf`/`fscanf`, and `fseek`/`ftell`.
Check return values: opening can fail, reads may be short, and formatted input
may not match requested conversions. Close streams when finished.

Standard I/O is generally buffered. Use documented stream operations rather
than assuming when bytes reach the underlying file or device.
