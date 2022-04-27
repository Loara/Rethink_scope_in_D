# Rethink_scope_in_D
## Index
- [Why at this moment scope is not a clear concept](README.md)
- [Scope blocks](scope_block.md)
- [Total value](total_value.md)
- [Validation of scoped code](scope_algorithm.md)
- [Validation of scoped functions](scope_fun.md)
- [Clone methods](cloning.md)
## Why at this moment scope is not a clear concept
Since DIP1000 the life of `scope` attribute hasn't been easy. It was introduced firstly in order to implement RAII (Resource Acquisition Is Initialization) and stack allocation of classes (and maybe to please a lot of C++ nostalgic). The idea of a `scope`d variable is simple: they *cannot escape the current scope in which they're defined*, at least as explained in [this page](https://dlang.org/spec/function.html#scope-parameters) of dlang documentation. 

But in [this other page](https://dlang.org/spec/attribute.html#scope) `scope` associated to local variables simply states that its destructor should be called and memory released when its scope ends. This in particular implies that scope variables can't escape their scope but it's still possible to write code in the following way:
```` d
scope int *a;
{
  scope int *b = stack_allocate!int(1);
  a = b;
}
...
writeln(a);
````
if we follow only the [documentation page](https://dlang.org/spec/attribute.html#scope) the assignment is correct since both `a` and `b` are `scope` variables, but the compiler correctly triggers an error since we're trying to "escape `b`'s reference" outside its scope to `a`, ~~but this code~~
````d
scope int *a;
{
  scope int **b = stack_allocate!(int *)();
  *b = stack_allocate!int(1);
  a = *b;
}
...
writeln(a);
````
~~compiles since `scope` is not transitive, so it only protects the "first reference order" of a variable~~.

A more interesting example and source of bugs in manual memory managment is the following example
```` d
class A{
  int t;
  this(int i){
    t = i;
  }
}

class B{
  A str;
  void set(A p){
    str = p;
  }
  
  ~this(){
    if(str !is null ){
      writeln(str.t);
    }
  }
}

...

{
  scope B b = new B();
  scope A a = new A(3);//stack allocated
  ...
  b.set(a)
}
````
if this code compiled then we would get many runtime errors and segmentation faults since variable `a` is destroyed before `b` but the reference in `b` is not `null`. But luckly this code doesn't compile since `set` cannot accept `scope` references. Even if we make `set` a `scope` member function
```` d
  void set(scope A p) scope{//this is scope too
    str = p;//Error: p escapes from function set since this.str is not scope
  }
````
we get a compile time error since `this.str` is not `scope`.  In some cases but `scope` is "almost transitive", for example this code
````d
class A{
  public this() pure @safe{}
  public int i = 1;
}

int *j;

void main() @safe{
  scope A a = new A();
  j = &(a.i);
}
````
doesn't compile and on `dmd` this error is issued:
````
 Error: scope variable `a` assigned to non-scope `j`
````

~~So the scope attribute as explained in [this page](https://dlang.org/spec/attribute.html#scope) is not so useful,~~ but the idea behind the [scoped function parameters](https://dlang.org/spec/function.html#scope-parameters) is not so bad, rather it can be used in order to introduce a *safe cast mechanism* to perform **shared data synchronization**, but we'll explain it in a [subsequent article](scope_block.md).

[Next>>](scope_block.md)
