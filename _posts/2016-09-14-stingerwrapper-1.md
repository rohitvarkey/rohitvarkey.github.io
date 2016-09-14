---
layout: post
title: StingerWrapper.jl - Part 1
---

**This post will be the first among a series of blog posts describing my
experiences while developing [StingerWrapper.jl](https://github.com/rohitvarkey/StingerWrapper.jl),
a Julia interface to the [STINGER](https://github.com/stingergraph/stinger/)
C package for dynamic graph analysis. I have been working on StingerWrapper along
with Dr. James Fairbanks.**

In this blog post, I will describe our experiments with mapping C structures to
Julia types and the design decisions we made to provide a Julia API to the STINGER
C library.

### Julia and C

Julia provides a C foreign function interface (FFI) to call C functions exposed
through a shared library. Using `ccall`s as documented [here](http://docs.julialang.org/en/release-0.5/manual/calling-c-and-fortran-code/), let's us call C
functions while passing in the required parameters. Julia has a mapping of its
internal types to C which can be seen here and automatically casts the Julia
types to the C types. The `ccall` syntax is pretty long and easy to get
wrong for untrained users. Writing a Julia function to abstract away this ccall
from the users would be the first step to providing a Julia API to the C library.
For example, the `add` function is much easier for an end user to think about.

```julia
import Libdl: dlopen, dlsym
libtest = dlopen("shared_library_name") #Open a shared library in Julia
function add(a::Int32, b::Float32)
    ccall(dlsym(libtest, :add_nums), Float32, (Int32, Float32), (1, 2))
end
```

Julia also supports direct C structs mappings. Just defining a immutable
composite Julia type with the attributes and types of the C struct, let's Julia
decode these structures. This means you can create objects in Julia, pass them to
C, let C do its thing and get it back to Julia.

```c
struct foo {
    int a;
    float b;
}

float sum(struct foo bar){
    return bar.a + bar.b;
}
```

```julia
immutable Foo
    a::Int32
    b::Float32
end

bar = Foo(1, 2.0)
ccall(dlsym(libtest, :sum), Float32, (Foo,), bar) #Returns 3.0!
```

However, there are a few restrictions. For example,  Julia does not support arrays
of unknown sizes. And the Stinger C struct has a zero size array allocated to it.
This stops us from using this clean method to work with the Stinger C structure.
:sad:

As we can't make a Julia type with all the mappings to the Stinger C structure,
the approach we ended up using was to hand over the memory allocation parts to C
and keep a reference to that memory using a pointer in Julia as a handle to the
STINGER C object. To facilitate this, we created a wrapper type to just store the
handle to the pointer returned from C. We store this as a `Ptr{Void}`, because well,
"when in doubt, use void pointers!" - James.

```julia
type Stinger
    handle::Ptr{Void}
end
```

We defined the constructor of `Stinger` to call the C memory allocator for a STINGER C
structure on initialization of a `Stinger` type in Julia. This let's users just
say `Stinger()` and the C memory is allocated and the pointer is stored as the handle
attribute.

Now that we've allocated memory, we need to free it too! A Julia [`finalizer`](http://docs.julialang.org/en/release-0.5/stdlib/base/?highlight=finalizer#Base.finalizer)
is registered that takes care of this. It ensures that the calls the corresponding
free function in C when the Julia garbage collector gets around to it. All of this
occurs transparently to the user, who does not need to worry about memory
management. Our wrapper type now looks like this.

```julia
type Stinger
    handle::Ptr{Void}

    #Default constructor to create a Stinger data structure
    function Stinger()
        s = new(ccall(dlsym(stinger_core_lib, "stinger_new"), Ptr{Void}, ()))
        finalizer(s, stinger_free)
        s
    end
end

function stinger_free(x::Stinger)
    # To prevent segfaults
    if x.handle != C_NULL
        x.handle = ccall(dlsym(stinger_core_lib, "stinger_free"), Ptr{Void}, (Ptr{Void},), x)
    end
end
```

This basic definition will let us write convenience wrapper functions
that make `ccall`s to the STINGER C library. However, it is limited in terms of
being able to access or modify data directly from Julia and basically forces
us to call C to do anything useful. We will look at ways in which we can expose
the data to users in better way in part 2.
