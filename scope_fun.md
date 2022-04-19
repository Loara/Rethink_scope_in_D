## Scope function parameters
Function calls in scoped blocks are a delicate argument, so we discuss it in a separate document. Consider any expression `E` in the form `f(E1, E2, ..., En)` with `ret(Ei)` well defined, we want to determine if expression `E` is valid inside a scope block.

At this moment we don't use the `return scope` function attribute, so functions can't return any `scope` parameter. A function `f` is called *ref-free* if and only if at least one of these conditions are satisfied:

1. `f` doesn't return references, in other words `f` is not a `ref` function and its return type is directed;
2. `f` is `pure` and every parameter `p` of `f` is not `ref`, `in` or `out` and its type is directed.

If `f` is ref-free then `ret(E)={}` always, otherwise we set `ret(E)={global}`.

This page is just a draft. Updates will arrive soon.
