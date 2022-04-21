## Scoped expressions
In this section we divide all variables into two groups:
1. **Global variables** are module variables and static variables defined anywhere;
2. **Local variables** are function parameters and variables defined inside functions or code blocks;
3. **Depending variables** are member variables and outer variables accessed via a `this` pointer in general.

*Only local variables can be defined as* `rscope`, member variables may inherit it from the outer class reference.

Now given an expression `E` we says it's a *strongly indirected expression* if and only if its return type is `void` or indirected, we says it's instead *weakly indirected* if it's strongly indirected or it's an lvalue (see [documentation](https://dlang.org/spec/expression.html#.define-lvalue)) of any type. An expression not weakly indirected is said to be *directed*.

In order to scan a `@safe` function body in order to validate `rscope` correctness you can follow this algorithm. First to all for every local or global variable `a` we define `ind(a)` equal to string `"s"`, where `s` is an identifier, if and only if `a` is local and is declared as `rscope(s)` (if `a` is declared as `rscope` then `ind(a) = ""`) and `ind(a) = "0"` in other cases, since an identifier can't begin with a number. We don't need to define `ind` on depending variables.

For every expression `E` is an expression we should define also `ret(E)` as the minimal set of strings `s` such that the return type of `E` could hold a reference of a variable inside `sset(s)`. Let two sets `A`, `B` we write `A ~ B` if and only if `A == B` or at least one of `A` or `B` is empty.

The algorithm to be applied to each statement `E` is the following:
- if `E` is a (local or global) variable expression `b` then `ret(E)={ ind(b) }`, if `b` is a depending variable evaluate instead `this.b` or `outer.b`;
- if `E` is equal to `(F)`, `*F`, `&F`, `++F`, `--F`, `+F` or `cast(T) F` then `ret(E)=ret(F)`;
- if `E` is any built-in binary operation (`+`, `-`, `*`, `/`, ..) then `ret(E)={}` since they don't hold indirection (pointer algebra is forbitten);
- if `E` is equal to `F ? G : H` then `ret(E) = ret(G) ∪ ret(H)`;
- if `E` is equal to `F.member` where `member` is a member field then `ret(E)=ret(F)`;
- if `E` is equal to `F[G]` or to `F[G1 .. G2]` with `G, G1, G2` convertible to integers and `F` is a static or a dynamic array then `ret(E)=ret(F)`;
- if `E` is a built-in literal (numerical, characters, strings, ...) then `ret(F)={}`;
- if `E` is equal to `F = G` where both `F` and `G` are expressions then
    - if `G` is strongly indirected then the code is valid if and only if `ret(F) ~ ret(G)` and in such case `ret(E) = ret(F) ∪ ret(G)`;
    - if `G` is not strongly indirected then the code is automatically valid and `ret(E) = ret(F)`.
- if `E` is declaration `T f = F;` then it's first translated to `T f; f = F;`;
- if `E` is in the form `ref T f = F` then we work as the preceding two points but we consider weakly indirection instead of strongly indirection;
- if `E` is a `if`, `while`, `for` or a `switch` statement then recursively apply the algorithm to each internal statement;
- if `E` is a `foreach(v; F)` statement is an array then for each valid index `i` the expression `v = F[i]` must be valid.

Rules for functions, constructors and return statements are explained in [following articles](scope_fun.md).

The reason of requiring `ret(F) ~ ret(G)` is to avoid these results:
````d
int *global;
...
rscope{
    int i = 0; //i rscoped
    int **j = &global; //if assigment from global to scoped is valid then this is also valid
    *j = &i; //valid because j is scoped, but now global has i address
}
````
if we make global any variable that *may* hold a reference to a global reference then the following code transfer scoped references to global scope:
````d
int *global;
...
rscope{
    int i = 0;//i scoped
    int *j = true ? &i : global; //since j may hold a global reference we get ind(j) = "0"
    global = j; //valid but i escapes.
}
````

Scoped variables should not be used as argument in template alias parameters or in value template parameters with indirect type.

This list is a draft and may change in successive releases. Any suggestion is welcome.
