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
synchronized(i_mutex){
  ref int i_ref = cast(int)i;
  //do stuffs with i_ref
}
//Now i_ref must be unreachable
````
imagine if inside the `synchronize` block the address of `i_ref`/`i` is saved elsewhere, and someone uses it to access `i` without lock `i_mutex`. A very big problem.

## `scope` blocks

So in a `@safe` code is very important in certain cases to control who can access to the **total value** of a cariable (we'll explain it in some subsequent article, for the moment you can see as the union of the value referred by such a variable plus, if it's a pointer, the total value of the pointed object). We then use the `scope` attribute on a variable declaration in order to define such variable not in the "global scope" but in an "grouped scope" separated from the global one:
```` d
const A a = new A();
scope(group) A ma = cast(A)( scope_cast!"group"(a) )
ma.add_ref();
  //do other things with ma
  //if global_a is a global A variable the following row issues a compile time error 
  //since the total value of ma is bounded to this scope
  //global_a = ma;
````
In the preceding example we define a scoped variable `ma` inside the grouped scope with name `"group"`, the `scope_class` template function just temporally changes the scope of `a` from `global` to `"group"` in order to allow to be assigned to a `"group"` scoped variable (it may be defined as `@system` in order to implement safer interfaces). Now references of objects in the total value of `ma` can be retrived only by variables inside the `group` scope.

The `scope` attribute can be used also without specify a group name, in that case the predefined empty gropu name `""` is used. For example
```` d
//Global scope
shared(int) i;
shared(mutex) i_mutex;
...
//function scope
synchronized(i_mutex){
  scope ref int i_ref = cast(int) scope_cast!""(i);
  //do stuffs with i_ref
}
````
and every variable or function parameter declared as `scope` can access `i_ref` variable. When we use `synchronized` blocks it's better to define a unique grouped scope in order to manage the `shared` object. We could for example introduce a new construct like the following example
```` d
//Global scope
shared(int) i;
shared(mutex) i_mutex;
...
//function scope
synchronized(i_mutex : i){
    /*
    now i inside this block is automatically 
    casted to int, also i and every other variable declared
    inside this block is automatically scope(unique) where unique is an unique name 
    associated to this synchronized block
    */
}
````
but this is only sugar syntax.

We could also introduce *scope blocks* in order to automatically make any declared variables inside it `scope`:

````d
int a;//Not scoped

scope(A){

  int b;//also scope(A)
  float bb;//ditto
  
  scope(B){
    ulong c;//it's scope(B) not scope(A)
  }
  
  byte bbb;// scope(A)
  
  scope{
    char d;//it's scope but not scope(A)
  }
  
}
````

The real question now is, how we define where a variable escapes from its context? What is the total value of a variable? We'll discover it in the [next article](total_value.md)
