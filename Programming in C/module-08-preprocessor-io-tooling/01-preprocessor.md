# Preprocessor Directives and Macros

- `#include` inserts header contents during preprocessing.
- Object-like macros replace a name with replacement text.
- Function-like macros take arguments. Parenthesize both parameters and the
  whole replacement expression to avoid precedence surprises; macros may
  evaluate an argument more than once.
- Header guards (`#ifndef`, `#define`, `#endif`) prevent repeated inclusion
  within a translation unit.

Prefer functions or `static inline` functions when macro evaluation semantics
would be error-prone.
