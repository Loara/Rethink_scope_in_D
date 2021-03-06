## Scope function parameters
Function calls in scoped blocks are a delicate argument, so we discuss it here and not in the preceding article. Consider any expression `E` in the form `f(E1, E2, ..., En)` with `f` a classical function (not member, nested or delegate) with parameters `(p1, p2, ..., pn)`, we want to determine if expression `E` is valid inside a scope block.

Function parameter `pi` is said to *grab references* if and only if `pi` is `ref`/`out` (or also `in` see below) or `pi` has indirected type. Since function parameters can be declared `rscope` we can define `ind(pi)` as in the preceding article. We define also an equivalence relation `~` between such parameters in the following way: we say `pi ~ pj` if and only if at least one of these sentences is true:
- `i == j`;
- both `pi` and `pj` grab references and `ind(pi) == ind(pj)`.

So expression `E` equal to `f(E1, E2, ..., En)` is valid if and only if these conditions are satisfied:
1. for every `i, j` such that `pi ~ pj` we get `ret(Ei) ~ ret(Ej)`;
2. if `f` is not `pure` then for every `i` such that `pi` grabs references and `ind(pi) == "0"` we get `ret(Ei) ~ { "0" }`. 

Functions can have an additional attribute `transfer(s)`, with `s` identifier, in order to force returning only references inside the `s` scope group. Attribute `transfer` is equivalent to `transfer()` and if it's not present if function declaration then our function will only return references in the `global` scope. So if expression `E` equal to `f(E1, E2, ..., En)` is valid and `f` is defined as `retscope(s)` we define `ret(E)` in the following way: 
1. if `E` is *not* weakly indirected (it's not `ref` and its return type is directed) then `ret(E) = {}` (empty);
2. otherwise if `f` is declared as `transfer(s)` then we get

    ````
    ret(E) = ∪ { ret(Ei) | pi grabs references, ind(pi) == "s"}
    ````
    
3. otherwise if `f` is pure we get

    ````
    ret(E) = ∪ { ret(Ei) | pi grabs references, ind(pi) == "0"}
    ````
    
    and if `f` is not `pure`
    
    ````
    ret(E) = ∪ { ret(Ei) | pi grabs references, ind(pi) == "0"} ∪ { "0" }
    ````


Now we can explain how to check `return` statement:

- if statement `E` defined inside the body of function `f` is in the form `return F`, `f` has attribute `transfer(s)` (explicitly or implicitly) and its return type is

    - indirected if `f` is not `ref`;
    - any if `f` is `ref`,
 
  then `E` is valid if and only if `ret(F)` is empty or `ret(F) == { s }`.

Currently we don't make any assumption on `throw` statements.

Member functions can be treated as normal functions by adding hidden parameter `this`. Like `const` and `immutable` a member function can also be declared `rscope(s)` in order to make `this` `rscope`.

## Constructors
Constructors for type `T` declared as `this(p1, p2, ..., pn) scope(s) @safe` are validated as `void this_fun(scope(s) ref T this, p1, p2, ..., pn)` if `T` is a struct and `this_fun(scope(s) T this, p1, p2, ..., pn)` if `T` is a class.

Constrictor invocation as `new T(E1, E2, ..., En)` with `T` class is translated to `new_Tfun(E1, E2, ..., En)` where
````d
T allocate_T() transfer @trusted;

T new_Tfun(p1, p2, ..., pn) transfer(s) @safe{
    scope(s) T ret = allocate_T();
    this_fun(ret, p1, p2, ..., pn);
    return ret;
}
````

## Template functions and `auto ref`
When calling a template function it's considered only the specialization that it's effective called. So if a parameter `pi` with directed type and declared as `auto ref` grabs references if and only if the corresponding argument `Ei` is an lvalue.

## the `in` problem
Attribute `in` is defined as `const scope` in order to avoid saving its reference outside the function since it may be destroyed once the function terminates. But since addresses of a `in` parameter may transfer reference (if it's passed by reference) or may not (if passed by value) then a `in` parameter should grab references even if its type is directed.

If type of a `in` parameter is indirected we don't know if references held by that parameter are destroyed or can be safely stored, so for `in` parameters the current meaning of `scope` should be maintained.

## Nested functions and classes problem
Nested functions (and classes) can access their context, but if they're defined inside a function how they can access `rscope` local variables since member variables cannot be `rscope`?

The trivial solution is to completely forbid nested entities to access `rscope` variables. Another solution for nested function is the following: if `this` parameter is declared as `rscope(s)` then you can access only variables declared as `rscope(s)`.

[<<Prev](scope_algorithm) [Menu](README.md) [Next>>](cloning.md)
