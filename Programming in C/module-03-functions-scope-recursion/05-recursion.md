# Recursion

To trace recursion:

1. Write each call's arguments and automatic locals separately.
2. Follow calls until the base case.
3. Trace returns in reverse order, applying statements after each recursive
   call.
4. Track static local variables separately: all calls share the same object.

Distinguish changes made **before** a recursive call from changes made during
unwinding **after** it. Tail recursion performs no additional work after the
recursive call; C does not guarantee compiler optimization. Converting
recursion to iteration typically means storing changing state in explicit
loop variables.
