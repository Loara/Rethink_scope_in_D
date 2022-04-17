## Unsafe casts
Suppose now we want to implement a reference counting mechanism for some class `A`, so we provide it the `size_t` member variable `count` that counts the number of references on it.
A simple implementation could be
```` d
class A{
  private:
    int count;
  public:
    this(){
      count = 0;
    }
    
    void add_ref(){
      count++;
    }
    
    void rem_ref(){
      if(count == 0){
        finalize_this();
      }
      else{
        count--;
      }
    }
}
````

In order to call `add_ref` or `rem_ref` on an instance `a` of `A` we need `a` to be mutable, but if `a` is `const` we need first to all to cast away the `const` attribute:
```` d
const A a = new A();
{
  A ma = cast(A) a;
  ma.add_ref();
  //do other things with ma
}
//now ma must be inaccessible, only use a
````
in other words we've restricted the critical section inside a block in which we're allowed to do anything to `a` via `ma` reference. If such unrestricted reference escapes (and for example be assigned to a global variable, maybe via an opaque not `pure` `@trusted` function) the consequences could be very very unpleasant.

Another very important example is via `shared` variables:
```` d
//Global scope
shared(int) i;
shared(mutex) i_mutex;
...
//function scope
synchronize(i_mutex){
  ref int i_ref = cast(int)i;
  //do stuffs with i_ref
}
//Now i_ref must be unreachable
````
imagine if inside the `synchronize` block the address of `i_ref`/`i` is saved elsewhere, and someone uses it to access `i` without lock `i_mutex`. A very big problem.

## `scope` blocks

So in a `@safe` code is very important in certain cases to control who can access to the **total value** of a cariable (we'll explain it in some subsequent article, for the moment you can see as the union of the value referred by such a variable plus, if it's a pointer, the total value of the pointed object). Instead touse `scope` attributes for local variable (since we need to control also the scope) it's more interesting to introduce `scope` block as blocks in which the selected variables mustn't escape:
```` d
const A a = new A();
scope(A ma = cast(A) a){
  ma.add_ref();
  //do other things with ma
  //if global_a is a global A variable the following row issues a compile time error 
  //since the total value of ma is bounded to this scope
  //global_a = ma;
}
````
and
```` d
//Global scope
shared(int) i;
shared(mutex) i_mutex;
...
//function scope
synchronize(i_mutex){
  scope(ref int i_ref = cast(int)i){
    //do stuffs with i_ref
  }
}
````
or even better introduce the following equivalent notation:
```` d
//Global scope
shared(int) i;
shared(mutex) i_mutex;
...
//function scope
synchronize(i_mutex : i){
    //now i inside this block is automatically 
    //casted to int and cannot escape outside
}
````
but this is only sugar syntax.
