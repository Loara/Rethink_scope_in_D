## Scope function parameters
Function calls in scoped blocks are a delicate argument, so we discuss it in a separate document. Consider any expression `E` in the form `f(E1, E2, ..., En)` with `ret(Ei)` well defined, we want to determine if expression `E` is valid inside a scope block.

Let `f` a classical function with parameters `(p1, p2, ..., pn)` we say `pi` *grabs references* if and only if `pi` is `ref` or `pi` has indirected type. Take now an expression `E` in the form `f(E1, E2, ..., En)` we define these *references sets*:
- the *global set* defined as

    ````
    grset(E) = ∪ { i | pi grabs references, pi not scoped }
    ````
    
- the *scoped set* defined as

    ````
    srset(E) = ∪ { i | pi grabs references, pi scoped }
    ````
    
and the following *homogeneity rules* should be satisfied:
1. for every `i, j ∈ srset(E)` we get either `pi` is `const` or we can ref transfer from `Ej` to `Ei` strictly (without reallocations);
2. for every `i, j ∈ grset(E)` we get either `pi` is `const` or we can ref transfer from `Ej` to `Ei`;
3. if `f` is not `pure` then for every `i ∈ grset(E)` and every `a ∈ ret(Ei)` we should get `ind(a) = -1` in order to avoid reference escape.

If `f` hasn't any `return scope` parameter then we get
````
ret(E) = ∪ { ret(E_i) | i ∈ grset(E) }
````
if `f` is `pure` otherwise
````
ret(E) = ∪ { ret(E_i) | i ∈ grset(E) } ∪ { global }
````
