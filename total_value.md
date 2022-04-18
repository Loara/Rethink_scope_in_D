## Total value
A type `T` is said to be *indirected* if and only if it's at least one of
- a pointer (it can be dereferenced);
- a class type;
- a dynamic array or an associative array;
- a struct that contains at least one member with indirected type;
- a static array of indirected type;
- a function pointer or a delegate pointer.
Type `T` is instead *directed* if it's instead
- a primitive type (numeric, characters, not-dereferencable addresses);
- an empty struct or a struct with only directed members;
- a static array of directed type.

**Proposal 1:** make any `void` pointer (`void *`, `const(void *)`, etc.) directed since should not be possible to dereference it without using unsafe casts, and so it's equivalent to a raw address.

We remember that an *object* in memory can be uniquely identified by its *address* (unless the compiler places it inside one or more registers), its *type* its *size* in bytes and its *value*. The *total value* of an object of type `T` is an union of all the objects that can be safely accessed from it, more precisely:
1. if `T` is a primitive type then its equal to its value;
2. if `T` is a struct then it's the union of total values of all its members;
3. if `T` is a static array then it's the union of the total values of each element;
4. if `T` is a pointer then it's its value (containing the address of the referenced object) plus the total value of the referenced object;
5. if `T` is a class then it's first converted to a pointer to a struct before to compute its total value;
6. if `T` is a dynamic array then it's total value is equal to that of an equivalent struct composed by a `size_t` member and a pointer to a "static array", in a similar way we compute the total value of an associative array, functions and delegate pointers.

So *avoid an object* `a` *to escape from block* `B` means that *inside block* `B` *any operation that stores outside* `B` *any reference to any object inside the total value of* `a`. Think about a `shared` object inside a `synchronized` block with its `shared` attribute casted away, any reference to some object inside its total vamue should not escape outside the `synchronized` block. 

At some point someone could ask us "but why not making `scope` a type qualifier or transitive?", and the answer is that the `scope` attribute is too permissive and too limitating at this time. Consider for example this function
```` d
void fun (ref scope int *a, ref scope int *b, ref scope int *c) @safe{
  *a = *b + *c;
}
````
and this module:
```` d
int *global;
...
void bar(int *par) @safe{
  scope(int *i = new int(3)){
    fun(par, i, global);
  }
}
````
then if we don't investigate the implementation of `fun` we *must* have a compile time because if we substitute `fun` with
```` d
void fun (ref scope int *a, ref scope int *b, ref scope int *c) @safe{
  c = b;
}
````
then this is also a valid implementation of `fun` but in this way we let escape `i` outside its scope.

Clearly this issue was born from a bad design of `fun` since their parameters should be `ref int` instead of `ref int *`, so there're two possible way to handle this scenario:
1. Hold the `scope` function parameters attribute with the meaning "a reference to any of its total value can be stored only inside other `scope` function parameters or return type if it is `return scope`";
2. Remove `scope` and build at compile time for every function parameter a "dependency set" of each variable that *could* store a reference to each object in its total value (and optionally adding special keywords `return` if it could be returned and `global` if assigned to a module scope or static variable).

The first approach is simpler to implement, can be used in not-`final` functions and speeds up compilation, but a lot of legacy code without `scope` attributes won't work with `scope` variables. The second approach allows to call legacy functions but can't be used on virtual functions.

Since virtual functions are a key feature of D we'll consider only the first approach in these articles.

Actually there exists a third approach compatible with 1. but useful for fine adjustments:
3. Adding the `scope(label)` attribute where *label* is any label. Parameters with `scope(A)` attribute can share any of their references only with local variables (unless they share them to forbidden variables) or other `scope(A)` parameters.

The algorithm to validate scoped blocks is developped in [this page](scope_algorithm.md)
