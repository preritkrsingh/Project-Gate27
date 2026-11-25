# Parameter Passing

C passes arguments **by value**: a function receives its own parameter object.
To let it modify a caller's object, pass that object's address:

```c
void set_value(int *p) {
    *p = 10;
}
```

Passing a pointer is still passing the pointer value by value. To update the
caller's pointer itself, pass a pointer to that pointer.
