## Scoped expressions
In this section we divide all variables into two groups:
1. **Global variables** are module variables and static variables devined anywhere;
2. **Local variables** are function parameters and variables defined inside functions or code blocks.

Member variables are not considered since they depends by an object reference. Now given an expression `E` we says it's a *strongly indirected expression* if and only if its return type is indirected, we says it's instead *weakly indirected* if it's strongly indirected or it's an lvalue (see [documentation](https://dlang.org/spec/expression.html#.define-lvalue)) of any type. An expression not weakly indirected is said to be *directed*.

In order to scan a `@safe` scoped block you can follow this algorithm. First to all for every integer `i` let `scoped(i)` the set of every variable with scope level `i` (we'll explain it later). Every global variable, unscoped function parameter or local variable outside any scope block is automatically inside `scoped(-1)`, scoped function parameters in `scoped(0)`. For every variable `a` we set `ind(a)` equal to `j` if and only if `a ∈ scoped(j)` (this value can change).

The algorithm is applied to each statement in a @safe function body. We initially set `i = 0`, each time we'll enter in a nested scoped block we'll increment `i`, each time we'll exit a scoped block we'll decrement it. 

A variable `b` is said to be `scoped` if and only if `b ∈ scoped(i)` for some `i >= 0`. Each time we enter in a scoped block (not normal blocks) we increment `i` and each time we leave a scoped block we decrement it.

For every expression `E` is an expression we should define also `ret(E)` as the set of all (local and global) variables such that `E` may return a reference to one of them. In order to simplify the algorithm we use the keyword `global` to refer any global variable.

The algorithm to be applied to each statement `E` is the following:
- If `E` is a simple variable declaration `T a;` without any initializer (or with a `void` initializer) then
    - if `i` is `0` then `a` is added to `scoped(-1)` (unscoped);
    - otherwise it's added to `scoped(i)` (scoped not function parameter);
- If `E` is a variable expression `b` then `ret(E)={b}`;
- if `E` is equal to `*F`, `&F`, `++F`, `--F` then `ret(E)=ret(F)`;
- if `E` is any built-in binary operation (`+`, `-`, `*`, `/`, ..) then `ret(E)={}` since they don't hold indirection (pointer algebra is forbitten);
- if `E` is equal to `F.member` where `member` is a member field then `ret(E)=ret(F)`;
- if `E` is equal to `F[G]` or to `F[G1 .. G2]` with `G, G1, G2` convertible to integers and `F` is a static or a dynamic array then `ret(E)=ret(F)`;
- if `E` is a built-in literal (numerical, characters, strings, ...) then `ret(F)={}`;
- if `E` is equal to `F = G` where both `F` and `G` are expressions then
    - if `G` is strongly indirected then for every `a ∈ ref(F)` and every `b ∈ ret(G)` we must have `ind(a) >= ind(b)` or an error is issued. In other words `min {ind(a) | a ∈ ret(F)} >= max{ind(b) | b ∈ ret(G)}`. We then set`ret(E) = ret(F) ∪ ret(G)`.  
    **Proposal 2  (avoid hostile takeover)**: If both `ret(F)` and `ret(G)` are not empty then every variable `a ∈ ret(F)` should be moved to `scoped(J)` where `J = min {ind(b) | b ∈ ret(G)}`since they're accepting a reference from an outer scope. Reallocation of scoped function parameters (`ind = 0`) mustn't happen and should issue an error. If we're in a `while`, `for` or `foreach` block we should then rescan the entire block with this modification. Must think a way to find and resolve possible infinite loops.  
    **Proposal 3 (weak proposal 2)**: issue a reallocation only when `J = -1`, so the hostile takehover can be undertaken only by scoped variables.
    - if `G` is not strongly indirected then `ret(E) = ret(F)` without perform any modification.
- if `E` is declaration `T f = F;` then it's first translated to `T f; f = F;`;
- if `E` is in the form `ref T f = F` then we work as the preceding two points but we consider weakly indirection instead of strongly indirection;
- if `E` is in the form `return F` then we must evaluate the declaration `ret_type _ret = F` with `ind(_ret)=-1`. 

Rules for functions and constructors are explained in the following section.

The need of proposal 2 is to avoid this result:
````d
int *global;
...
scope(int i = 0){//i scoped
    int **j = &global; //if proposal 2 is not active then j is also scoped
    *j = &i; //valid because j is scoped, but now global has i address
    //if proposal 2 or 3 are accepted then ind(j) = -1 and second row throws an error.
}
````

This list is a draft and may change in successive releases. Any suggestion is welcome.
