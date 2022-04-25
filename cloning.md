## Cloning a class
Consider this very simple struct:
````d
struct A{
  int a;
  int b;
}
````
since `A` is a struct you can clone it with a simple assigment:
````d
A a;
....
A b = a;
````
now `b` refers to a different object of type `A` so any modification of `b` doesn't influence also `a`.

But if now we make `A` a class
````d
class A{
  int a;
  int b;
}
````
then the assigment operator will only copy the *reference* to your object, not the entire object. This can be both a benefit (no copying overhead if the copied object is `const`) and a disadvantage. So if we want to clone an object we should implement a `clone` method:
````d
class A{
  private:
    int a;
    int b;
  public:
    this(int aa, int bb) pure @safe nothrow @nogc{
      a = aa;
      b = bb;
    }
    A clone() const pure @safe nothrow{
      return new A(a, b);
    }
}
....
A a = new A(1, 2);
A b = a.clone();
````

So you think this is only a problem for classes? Not so fast:
````d
struct A{
  int a;
  int *b;
}
````
now the assigment operator copy both `a` and `b` member, but not the value referenced by `b`, so if the cloned object is not `const` we may inadvertently change a shared object and this ma be a problem. So any struct with indirected members should have a `clone` function too:
````d
struct A{
  int a;
  int *b;
  this(int aa, int *bb) pure @safe nothrow @nogc{
    a = aa;
    b = bb;
  }
  A clone() const pure @safe nothrow @nogc{
    int *nb = new int(*b);
    return A(a, nb);
}
````

But why we discuss it in these articles? Because when we want to clone an object we *don't want any shared reference* between the callee and the new object. This in `@safe` code can be enforced with `rcope` and `transfer` attributes in order to avoid bugs:
````d
struct A{
  int a;
  int *b;
  this(int aa, int *bb) pure @safe nothrow @nogc{
    a = aa;
    b = bb;
  }
  A clone() const pure @safe nothrow @nogc transfer{
    rscope int *nb = new int(*b);
    return A(a, nb);
}
````
or example if we wrote instead
````d
struct A{
  int a;
  int *b;
  this(int aa, int *bb) pure @safe nothrow @nogc{
    a = aa;
    b = bb;
  }
  A clone() const pure @safe nothrow @nogc transfer{
    return A(a, b);
}
````
then we would get a compile time error since return object `A(a, b)` was `rscope` (due to `transfer` attribute) but `b` is a reference that doesn't belong to `rscope` (since `clone` was not marked with `rscope`).

So you can use `rscope` in order to control your references.
