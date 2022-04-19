## Scoped expressions
In this section we divide all variables into two groups:
1. **Global variables** are module variables and static variables devined anywhere;
2. **Local variables** are function parameters and variables defined inside functions or code blocks.

Member variables are not considered since they depends by an object reference. Now given an expression `E` we says it's a *strongly indirected expression* if and only if its return type is indirected, we says it's instead *weakly indirected* if it's strongly indirected or it's an lvalue (see [documentation](https://dlang.org/spec/expression.html#.define-lvalue)) of any type. An expression not weakly indirected is said to be *directed*.

In order to scan a `@safe` scoped blocks you can follow this algorithm. First to all for every integer `i` let `scoped(i)` the set of every variable with scope level `i` (we'll explain it later). Every global variable, unscoped function parameter or local variable outside any scope block is automatically inside `scoped(-1)`, scoped function parameters in `scoped(0)`. For every variable `a` we set `ind(a)` equal to `j` if and only if `a ∈ scoped(j)` (this value can change). Instead of putting every possible global variable inside `scoped(-1)` you should use the keyword `global` when referring any global reference.

The algorithm is applied to each statement in a @safe function body. We initially set `i = 0`, each time we'll enter in a nested scoped block we'll increment `i`, each time we'll exit a scoped block we'll decrement it. 

A variable `b` is said to be `scoped` if and only if `b ∈ scoped(i)` for some `i >= 0`. Each time we enter in a scoped block (not normal blocks) we increment `i` and each time we leave a scoped block we decrement it.

For every expression `E` is an expression we should define also `ret(E)` as the set of all (local and global) variables such that `E` may return a reference to one of them. In order to simplify the algorithm we use the keyword `global` to refer any global variable.

The algorithm to be applied to each statement `E` is the following:
- If `E` is a simple variable declaration `T a;` without any initializer (or with a `void` initializer) then
    - if `i` is `0` then `a` is added to `scoped(-1)` (unscoped);
    - otherwise it's added to `scoped(i)` (scoped not function parameter);

  In this way any variable declared inside a scoped block is automatically `scope` so we don't need anymore the `scope` attribute for local variables but only for function parameters;
- If `E` is a variable expression `b` then `ret(E)={b}`;
- if `E` is equal to `*F`, `&F`, `++F`, `--F` then `ret(E)=ret(F)`;
- if `E` is any built-in binary operation (`+`, `-`, `*`, `/`, ..) then `ret(E)={}` since they don't hold indirection (pointer algebra is forbitten);
- if `E` is equal to `F ? G : H` then `ret(E) = ret(G) ∪ ret(H)`;
- if `E` is equal to `F.member` where `member` is a member field then `ret(E)=ret(F)`;
- if `E` is equal to `F[G]` or to `F[G1 .. G2]` with `G, G1, G2` convertible to integers and `F` is a static or a dynamic array then `ret(E)=ret(F)`;
- if `E` is a built-in literal (numerical, characters, strings, ...) then `ret(F)={}`;
- if `E` is equal to `F = G` where both `F` and `G` are expressions then
    - if `G` is strongly indirected then set`ret(E) = ret(F) ∪ ret(G)` and apply **ref transfer** algorithm from `G` to `F` (see next section).
    - if `G` is not strongly indirected then `ret(E) = ret(F)` without perform any modification.
- if `E` is declaration `T f = F;` then it's first translated to `T f; f = F;`;
- if `E` is in the form `ref T f = F` then we work as the preceding two points but we consider weakly indirection instead of strongly indirection;
- if `E` is in the form `return F` then we must evaluate the declaration `ret_type _ret = F` with `ind(_ret)=-1`. 

Rules for functions and constructors are explained in following articles.

### Ref transfer
Given two expressions `E` and `F` with `ret(E)` and `ret(F)` well defined. The *ref transfer* from `F` to `E` is an algorithm used for example in assigments like `E = F` where we want to transfer references from `F` to `E`. You can apply it by following these steps:

- Test the statement `min {ind(a) | a ∈ ret(E)} >= max{ind(b) | b ∈ ret(F)}`, if it isn't satisfied an error must be issued;
-  **Proposal 2  (avoid hostile takeover)** if there exist `a ∈ ret(E)` and `b ∈ ret(F)` so that `ind(a) > ind(b)` then the proposal 2 adds the following rule: if `ind(b) == ind(b') == J` for every `b, b' ∈ ret(F)` then move every `a` in `ret(E)` to `scoped(J)`. If this reallocation happens inside a `while`, `for` or `foreach` block then the entire block should be rescanned with this new assigment. If instead `ind(b) != ind(b')` for some `b, b' ∈ ret(F)` then an error should be issued.
    - *Strict mode*: in some cases you want to apply strict mode: every time a reallocation of references should be performed an error is issued instead.
-  **Proposal 3 (weak proposal 2)** apply proposal 2 only when exists `b ∈ ret(F)` such that `ind(b) == -1` (not scoped) otherwise just ignore it.

The need of proposal 2/3 is to avoid this results:
````d
int *global;
...
scope(int i = 0){//i scoped
    int **j = &global; //if proposal 2 is not active then j is also scoped
    *j = &i; //valid because j is scoped, but now global has i address
}
````
the preceding proposal draft solves it by moving `j` into `scoped(-1)`, but this solution is wrong due to this other result:
````d
int *global;
...
scope(int i = 0){//i scoped
    int *j = true ? &i : global; //old proposal 2 makes ind(j)=-1
    global = j; //valid but i escapes.
}
````

This list is a draft and may change in successive releases. Any suggestion is welcome.
