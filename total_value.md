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

## Scoped expressions
In this section we divide all variables into two groups:
1. **Global variables** are module variables and static variables devined anywhere;
2. **Local variables** are function parameters and variables defined inside functions or code blocks.

Member variables are not considered since they depends by an object reference. Now given an expression `E` we says it's a *strongly indirected expression* if and only if its return type is indirected, we says it's instead *weakly indirected* if it's strongly indirected or it's an lvalue (see [documentation](https://dlang.org/spec/expression.html#.define-lvalue)) of any type. An expression not weakly indirected is said to be *directed*.

Given a statement `E` and a local scoped variable `a` we define the *set of indirections* `ind(a)` the set of all variables that may've received a reference to an object in the total value of `a`. If `E` is an expression we should define also `ret(E)` as the set of all (local and global) variables such that `E` may return a reference to one of them. In order to simplify the algorithm we use the keyword `global` to refer any global variable.

In order to test if the code inside a scope block is correct we must apply the following algorithm sequentially to each statement inside the block. If multiple scoped blocks are nested then the algorithm should be applied to *each* nested scoped block.

- If `E` is a variable `b` then `ret(E)={b}`;
- if `E` is equal to `*F`, `&F`, `++F`, `--F` then `ret(E)=ret(F)`;
- if `E` is any built-in binary operation (`+`, `-`, `*`, `/`, ..) since they don't hold indirection (pointer algebra is forbitten) then `ret(E)={}`;
- if `E` is equal to `F.member` where `member` is a member field then `ret(E)=ret(F)`;
- if `E` is equal to `F[G]` or to `F[G1 .. G2]` with `G, G1, G2` convertible to integers and `F` is a static or a dynamic array then `ret(E)=ret(F)`;
- if `E` is a built-in literal, a struct literal with an empty constructor or a `new` expression with a empty constructor then `ret(F)={}`;
- if `E` is equal to `F = G` where both `F` and `G` are expressions then
    - if `G` is strongly indirected and `ref(G)` contains a scoped variable then `ret(F)` can't contain any not scoped variable (or an error is issued). Also if `ret(G)` contains a not-scoped variable then all the variables in `ret(F)` are no more considered scoped (since the return type of `G` is strongly indirected) and should issue an error if found inside a `while`, `for` or `foreach` block. In every case we set `ret(E) = ret(F) ∪ ret(G)`;
    - if `G` is not strongly indirected then `ret(E) = ret(F)`.
- if `E` is the declaration `scope T f;` then `f` is added to the set of scoped variables in the current scope block. Expressions `T f = F;` are first translated to `T f; f = F;`;
- if `E` is in the form `ref T f = F` then we work as the preceding two points but we consider weakly indirection instead of strongly indirection.

This list is a draft and may change in successive releases. Any suggestion is welcome.
