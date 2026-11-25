# Lexical and Dynamic Scoping

C uses **lexical (static) scoping**: name lookup follows source-code nesting
and declaration rules. The name selected by a reference is determined from
where that reference appears in the program.

Under **dynamic scoping**, name lookup instead depends on the active call
chain. It is a useful theoretical contrast, but it is not C's name-resolution
model.
