---
layout: post
title: StingerWrapper.jl - Part 3
---

**This post is the third among a series of blog posts describing my
experiences while developing [StingerWrapper.jl](https://github.com/rohitvarkey/StingerWrapper.jl),
a Julia interface to the [STINGER](https://github.com/stingergraph/stinger/)
C package for dynamic graph analysis. I have been working on StingerWrapper along
with Dr. James Fairbanks.**

In part 1 and part 2, we looked at how we could interact between Julia and C,
including methods of loading and storing data between the 2 languages. As explained
in part 2, we decided to use a lazy approach to accessing data from C and only
load/store the required field instead of the whole object. In this post, we
will look at how we can abstract away the `unsafe_load` and `unsafe_store!`
functions from users and let them use Julia like syntax to access attributes.

Initially, we planned to overload the `getfield` and `setfield` functions to
make accessors on the `Stinger` wrapper type, make the calls to `unsafe_load`
and `unsafe_store!`. This would have let us use the normal Julia `a.b` syntax to
access fields. However, Julia does not allow overloading these core functions yet ([JuliaLang/julia#1974](https://github.com/JuliaLang/julia/issues/1974)). Had this
been possible, we could have allowed users to use a workflow like

```julia
s = Stinger()
s.max_nv = 100000 #setfield! - Should end up reflecting in C
s.max_nv #getfield! - Reads 100000 from C
```

Julia does let us overload the `getindex` and the `setindex!` functions though.
These functions are called when the syntax `a[b]` or `a[b] = 1` are used.
Overloading these functions, we can provide users with the ability to access
`STINGER` fields from the `Stinger` wrapper type using the indexing syntax.
Julia language interop packages such as [PyCall.jl](https://github.com/JuliaPy/PyCall.jl)
uses this method to provide syntactic sugar to their users. Following the PyCall
implementation, we can let users use `Symbol`s to access the fields.

```julia
s = Stinger()
s[:max_nv] = 100000 #setindex! - Should end up reflecting in C
s[:max_nv] #getindex! - Should read latest value from C
```

Unlike PyCall.jl which has to support generality, we know exactly what fields of
the `StingerGraph` type we need to expose to users. So we declared an
`Enum` using the `@enum` for all the fields that we need to expose. Leveraging
meta-programming made writing the `@enum` call much easier too :).
This allows dispatching on this `Enum` type (which we named `StingerFields`) for
`getindex` and `setindex!` with the following advantages:

- Invalid fields error out naturally. No explicit check is required to confirm if
the `Symbol` is part of the names of the fields in `StingerGraph`.
- We can encode the offsets from the base pointer required to load each field as
the value of the `Enum` instance. An example `getindex` implementation is given below.

```julia
function getindex(x::Stinger, field::StingerFields)
    idx = Int(field)
    basepointer = convert(Ptr{fieldtype(StingerGraph, idx)}, x.handle)
    unsafe_load(basepointer, idx)
end
```  

The type introspection at runtime is required as there are 2 fields in `StingerGraph`,
`batch_time` and `update_time`, that are `Float64`s while all the others fields
are `Int64`s. This introduces a type instability that can cause potential
performance bottlenecks. Rather than taking this performance hit to allow for
rarely used fields such as `batch_time` and `update_time`, we implemented special
getter functions for them. We [benchmarked](https://github.com/rohitvarkey/StingerWrapper.jl/commit/c78f3d7d54590e9e44ca4c9dc4a3e776c407bc5a) both these versions of `getindex`
with the following results for 10000000 samples.

|`getindex` Type|minimum time|median time|mean time|maximum time|memory estimate|
|----------|------------|-----------|---------|------------|---------------|
|Type unstable|366.00 ns (0.00% GC)|398.00 ns (0.00% GC)|433.29 ns (1.10% GC)|14.44 ms (99.79% GC)|32.00 bytes|
|Type stable|38.00 ns (0.00% GC)|41.00 ns (0.00% GC)|43.95 ns (0.00% GC)|219.11 μs (0.00% GC)|0.00 bytes|

These results show an order of magnitute reduction in latency for these core
operations and completely eliminate the need for memory allocation and garbage collection.
Finally, the syntax users can use to get or set fields on the `Stinger` wrapper
time is

```julia
s = Stinger()

#General field access
s[max_nv] = 100000
s[max_nv]

#Specialized methods for `batch_time` and `update_time`
s[batch_time] #Error
get_batchtime(s) #Loads the batch time.
```

We will be working on setting up benchmarks and implementing a BFS implementation
in the next week.
