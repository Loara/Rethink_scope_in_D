## Total value
A type `T` is said to be *indirected* if and only if it's at least one of
- a pointer (it can be dereferenced);
- a class type;
- a dynamic array or an associative array;
- a struct that contains at least one member with indirected type;
- a static array of indirected type;
- a function pointer or a delegate pointer.
Type `T` is instead *directed* if it's instead
- a primitive type (numeric, characters, not-dereferencable addresses);
- an empty struct or a struct with only directed members;
- a static array of directed type.

**Proposal 1:** make any `void` pointer (`void *`, `const(void *)`, etc.) directed since should not be possible to dereference it without using unsafe casts, and so it's equivalent to a raw address.

We remember that an *object* in memory can be uniquely identified by its *address* (unless the compiler places it inside one or more registers), its *type* its *size* in bytes and its *value*. The *total value* of an object of type `T` is an union of all the objects that can be safely accessed from it, more precisely:
1. if `T` is a primitive type then its equal to its value;
2. if `T` is a struct then it's the union of total values of all its members;
3. if `T` is a static array then it's the union of the total values of each element;
4. if `T` is a pointer then it's its value (containing the address of the referenced object) plus the total value of the referenced object;
5. if `T` is a class then it's first converted to a pointer to a struct before to compute its total value;
6. if `T` is a dynamic array then it's total value is equal to that of an equivalent struct composed by a `size_t` member and a pointer to a "static array", in a similar way we compute the total value of an associative array, functions and delegate pointers.

So *avoid an object* `a` *to escape from block* `B` means that *inside block* `B` *any operation that stores outside* `B` *any reference to any object inside the total value of* `a`. Think about a `shared` object inside a `synchronized` block with its `shared` attribute casted away, any reference to some object inside its total vamue should not escape outside the `synchronized` block. 
