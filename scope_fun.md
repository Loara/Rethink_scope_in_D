## Scope function parameters
Function calls in scoped blocks are a delicate argument, so we discuss it here and not in the preceding article. Consider any expression `E` in the form `f(E1, E2, ..., En)` with `f` a classical function (not member, nested or delegate) with parameters `(p1, p2, ..., pn)`, we want to determine if expression `E` is valid inside a scope block.

Function parameter `pi` is said to *grab references* if and only if `pi` is `ref` or `pi` has indirected type. Since function parameters can be declared `scope` we can define `ind(pi)` as in the preceding article. We define also an equivalence relation `~` between such parameters in the following way: we say `pi ~ pj` if and only if at least one of these sentences is true:
- `i == j`;
- both `pi` and `pj` grab references and `ind(pi) == ind(pj)`.

So expression `E` equal to `f(E1, E2, ..., En)` is valid if and only if these conditions are satisfied:
1. for every `i, j` such that `pi ~ pj` we get that or at least one of `ret(Ei)` and `ret(Ej)` is empty or `ret(Ei) == ret(Ej)`;
2. if `f` is not `pure` then for every `i` such that `pi` grabs references and `ind(pi) == "0"` we get `ret(Ei)` empty or equal to `{ "0" }`. 

Functions can have attribute `retscope(s)` (`scope` attribute  in order to force returning only references inside the `s` scope group, attribute `retscope` is equivalent to `retscope("")` and if it's not present then `retscope("0")` is used implicitly. So if expression `E` equal to `f(E1, E2, ..., En)` is valid and `f` is defined as `retscope(s)` we define `ret(E)` in the following way: 
1. if `E` is *not* weakly indirected (it's not `ref` and its return type is directed) then `ret(E) = {}` (empty);
2. otherwise we get

    ````
    ret(E) = ∪ { ret(Ei) | pi grabs references, ind(pi) == s}
    ````
    
    if `s` is not equal to `"0"` or `f` is `pure`, otherwise
    
    ````
    ret(E) = ∪ { ret(Ei) | pi grabs references, ind(pi) == s} ∪ { "0" }
    ````


Now we can explain how to check `return` statement:

- if statement `E` defined inside the body of function `f` is in the form `return F`, `f` has attribute `retscope(s)` (explicitly or implicitly) and its return type is

    - strongly indirected if `f` is not `ref`;
    - weakly indirected if `f` is `ref`,
 
  then `E` is valid if and only if `ret(F)` is empty or `ret(F) == { s }`.

Currently we don't make any assumption on `throw` statements.

Member functions, nested functions, lambdas and any other delegate can be treat as normal functions by adding hidden parameters.
