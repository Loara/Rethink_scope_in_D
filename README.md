# Rethink_scope_in_D
## Introduction
Since DIP1000 the life of `scope` attribute hasn't been easy. It was introduced firstly in order to implement RAII (Resource Acquisition Is Initialization) and stack allocation of classes (and maybe to please a lot of C++ nostalgic). The idea of a `scope`d variable is simple: they *cannot escape the current scope in which they're defined*, at least as explained in [this page](https://dlang.org/spec/function.html#scope-parameters) of dlang documentation. But in [this other page](https://dlang.org/spec/attribute.html#scope) `scope` associated to local variables simply states that its destructor should be called and memory released when its scope ends. 
