## Scoped expressions
In this section we divide all variables into two groups:
1. **Global variables** are module variables and static variables defined anywhere;
2. **Local variables** are function parameters and variables defined inside functions or code blocks.

Member variables are not considered since they depends by an object reference. Now given an expression `E` we says it's a *strongly indirected expression* if and only if its return type is indirected, we says it's instead *weakly indirected* if it's strongly indirected or it's an lvalue (see [documentation](https://dlang.org/spec/expression.html#.define-lvalue)) of any type. An expression not weakly indirected is said to be *directed*.

In order to scan a `@safe` function body in order to validate `scope` correctness you can follow this algorithm. First to all for every identifier `group` we define `sset("group")` the set of all local variables `a` declared as `scope("group")`, conversely for every `a` local or global variable we define `ind(a)` equal to string `"group"` if and only if `a ∈ sset("group")` otherwise `ind(a) = "0"`. A valiable `a` is *scoped* if and only if `ind(a) != "0"`.

For every expression `E` is an expression we should define also `ret(E)` as the minimal set of strings `s` such that the return type of `E` could hold a reference of a variable inside `sset(s)`.

The algorithm to be applied to each statement `E` is the following:
- if `E` is a variable expression `b` then `ret(E)={ ind(b) }`;
- if `E` is equal to `(F)`, `*F`, `&F`, `++F`, `--F` or `cast(T) F` then `ret(E)=ret(F)`;
- if `E` is any built-in binary operation (`+`, `-`, `*`, `/`, ..) then `ret(E)={}` since they don't hold indirection (pointer algebra is forbitten);
- if `E` is equal to `F ? G : H` then `ret(E) = ret(G) ∪ ret(H)`;
- if `E` is equal to `F.member` where `member` is a member field then `ret(E)=ret(F)`;
- if `E` is equal to `F[G]` or to `F[G1 .. G2]` with `G, G1, G2` convertible to integers and `F` is a static or a dynamic array then `ret(E)=ret(F)`;
- if `E` is a built-in literal (numerical, characters, strings, ...) then `ret(F)={}`;
- if `E` is equal to `F = G` where both `F` and `G` are expressions then
    - if `G` is strongly indirected then the code is valid if and only if at least one of `ret(F)` or `ret(G)` is empty or `ret(F) == ret(G)`, otherwise an error should be issued. If the code is valid then set `ret(E) = ret(F) ∪ ret(G)`;
    - if `G` is not strongly indirected then the code is automatically valid and `ret(E) = ret(F)`.
- if `E` is declaration `T f = F;` then it's first translated to `T f; f = F;`;
- if `E` is in the form `ref T f = F` then we work as the preceding two points but we consider weakly indirection instead of strongly indirection;
- if `E` is a `if`, `while`, `for` or a `switch` statement then recursively apply the algorithm to each internal statement;
- if `E` is a `foreach(v; F)` statement is an array then for each valid index `i` the expression `v = F[i]` must be valid.

Rules for functions, constructors and return statements are explained in [following articles](scope_fun.md).

The reason of requiring `ret(F) == ret(G)` is to avoid these results:
````d
int *global;
...
scope{
    int i = 0; //i scoped
    int **j = &global; //if assigment from global to scoped is valid then this is also valid
    *j = &i; //valid because j is scoped, but now global has i address
}
````
if we make global any variable that *may* hold a reference to a global reference then the following code transfer scoped references to global scope:
````d
int *global;
...
scope(int i = 0){//i scoped
    int *j = true ? &i : global; //since j may hold a global reference it's set inside sset("0")
    global = j; //valid but i escapes.
}
````

This list is a draft and may change in successive releases. Any suggestion is welcome.
