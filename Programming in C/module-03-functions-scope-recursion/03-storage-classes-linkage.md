# Storage Classes and Linkage

**Scope** is where a name is visible, **storage duration** is an object's
lifetime, and **linkage** connects declarations across scopes or translation
units.

| Form | Main effect |
|---|---|
| Block-scope `auto` (default for ordinary local variables) | Automatic storage duration; indeterminate value if not initialized |
| Block-scope `static` | Static storage duration; zero-initialized by default; value persists between calls |
| File-scope `static` | Static storage duration and internal linkage; name is limited to that translation unit |
| `extern` declaration | Declares an object or function; a declaration alone need not define/allocate the object |
| `register` | Optional optimization hint for an automatic object; taking its address with `&` is prohibited |

Static storage duration does not require that an object physically reside in a
specific segment; data/BSS is a common implementation model.
