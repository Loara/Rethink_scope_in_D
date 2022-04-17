# Rethink_scope_in_D
## Bad documentation
Since DIP1000 the life of `scope` attribute hasn't been easy. It was introduced firstly in order to implement RAII (Resource Acquisition Is Initialization) and stack allocation of classes (and maybe to please a lot of C++ nostalgic). The idea of a `scope`d variable is simple: they *cannot escape the current scope in which they're defined*, at least as explained in [this page](https://dlang.org/spec/function.html#scope-parameters) of dlang documentation. 

But in [this other page](https://dlang.org/spec/attribute.html#scope) `scope` associated to local variables simply states that its destructor should be called and memory released when its scope ends. This in particular implies that scope variables can't escape their scope but it's still possible to write code in the following way:
```` d
scope int *a;
{
  scope int *b = stack_allocate(1);
  a = b;
}
...
writeln(a);
````
the assignment is tecnically correct since both `a` and `b` are `scope` variables, whereas they belongs to different scopes. A more destructive example and source of bugs in manual memory managment is the following example
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
if this code compiles then we may get many runtime errors and segmentation faults since variable `a` is destroyed before `b` but the reference in `b` is not `null`. A simple solution could be force `scope` variables to be used only as `scope` function parameters:
```` d
  void set(scope A p) scope{//this is scope too
    str = p;//Error: p escapes from function set since this.str is not scope
  }
````
(notice that `scope` function, not `scope` function parameters, are not documented in [member function attributes](https://dlang.org/spec/class.html#member-functions) page but they can be used anyway) but in this way we've made `set` completely useless, why we have to define a `set` function if we can't use it to set a member?

Notice also that this we exchange `b` with `a` the resulting code in now safe:
```` d
{
  scope A a = new A(3);
  scope B b = new B();
  ...
  b.set(a)
}
````
so:
- the problem lies into the calling context (since `a` and `b` are in the wrong order);
- but we can't find the problem unless we investigate inside the called context (`set`'s body).

If we try to compile the following code with `dmd -dip1000`:
```` d
import std.stdio;

class A{
    public int a =3;
}

class B{
    public A str;
    void set(scope A p) @safe scope{
        str = p;
    }

    ~this() @safe{
        if(str !is null){
            writeln(str.a);
        }
    }
}

int main() @safe{
    scope A a = new A();
    scope B b = new B();

    b.set(a);

    return 0;
}
````
we clearly get the following error
````
main.d(10): Error: scope variable `p` assigned to non-scope `this.str`
````

So the `scope` attribute is not what the manual-memory managment fans are waiting, currently the simplest and safest way to manage it is via, drum roll, the *Garbage Collector* (and yes, also reference counting counts as garbage collector, more precisely the dumb brother of the big GC family due to their [poor global performances compared to other solutions](https://en.wikipedia.org/wiki/Reference_counting)).

So the scope attribute as explained in [this page](https://dlang.org/spec/attribute.html#scope) is not so useful, but the idea behind the [scoped function parameters](https://dlang.org/spec/function.html#scope-parameters) is not so bad, rather it can be used in order to introduce a *safe cast mechanism* to perform **shared data synchronization**, but we'll explain it in a subsequent article.
