## Scoped expressions
In this section we divide all variables into two groups:
1. **Global variables** are module variables and static variables devined anywhere;
2. **Local variables** are function parameters and variables defined inside functions or code blocks.

Member variables are not considered since they depends by an object reference. Now given an expression `E` we says it's a *strongly indirected expression* if and only if its return type is indirected, we says it's instead *weakly indirected* if it's strongly indirected or it's an lvalue (see [documentation](https://dlang.org/spec/expression.html#.define-lvalue)) of any type. An expression not weakly indirected is said to be *directed*.

In order to scan a `@safe` scoped block you can follow this algorithm. First to all for every integer `i` let `scoped(i)` the set of every variable with scope level `i` (we'll explain it later). Every global variable, unscoped function parameter or local variable outside any scope block is automatically inside `scoped(-1)`, scoped function parameters in `scoped(0)`.

A variable `b` is said to be `scoped` if and only if `b ∈ scoped(i)` for some `i >= 0`. Each time we enter in a scoped block (not normal blocks) we increment `i` and each time we leave a scoped block we decrement it.

For every expression `E` is an expression we should define also `ret(E)` as the set of all (local and global) variables such that `E` may return a reference to one of them. In order to simplify the algorithm we use the keyword `global` to refer any global variable.

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
