---
layout: post
title: StingerWrapper.jl - Part 2
---

**This post is the second among a series of blog posts describing my
experiences while developing [StingerWrapper.jl](https://github.com/rohitvarkey/StingerWrapper.jl),
a Julia interface to the [STINGER](https://github.com/stingergraph/stinger/)
C package for dynamic graph analysis. I have been working on StingerWrapper along
with Dr. James Fairbanks.**

In this blog post, I will describe our experiments with loading and storing
C variables in Julia and the design decisions we made regarding this in StingerWrapper.jl.

As I mentioned in Part 1, we had to use a `Ptr{Void}` pointer  as a handle to
refer to the C STINGER object. However, we wanted to offer a Julia like syntax for
users to examine values and set values of the STINGER object rather than working
directly with the pointer.

The first step was to figure out how we could load values pointed to by
C pointers into Julia. We found the `unsafe_load` and `unsafe_store!` functions ([docs](http://docs.julialang.org/en/release-0.4/manual/calling-c-and-fortran-code/#accessing-data-through-a-pointer)) which lets you access data from pointers.
`unsafe_load` lets you pass in a pointer and Julia will attempt to load an
object from that depending on the type parameter of the `Ptr` passed into it. It
also lets you specify an index to act as an offset. `unsafe_store!` attempts to
store the value passed in into the memory pointed to by the `Ptr`. The use of
both of these functions is considered unsafe as undefined behavior occurs if the
memory is not valid when accessed. (Julia will probably segfault).

At this point, we had 2 options.

1. Write a Julia type mapping the `Stinger` C structure and attempt to load that.
2. Hand code `unsafe_load` and `unsafe_store!` for every field.

We weren't sure if option 1 would work because of the unknown sized arrays at the
end of the `STINGER` C structure definition. It wasn't clear if Julia could automatically
decode C structures which had complete mappings between Julia and C [either]([http://docs.julialang.org/en/release-0.5/manual/calling-c-and-fortran-code/#mapping-c-types-to-julia]).
In order to figure out these questions, we decided to write a few tests and check
how Julia performs in these situations. We tried to see if `unsafe_load` and
`unsafe_store!` would work in the following cases:

1. A C structure with a complete Julia type mapping.
2. A C structure with an unknown array at the end and a Julia mapping leaving
out the unknown array.

Both of these implementations work, as shown by this [experiment](https://github.com/rohitvarkey/Julia-C-Experiments). So, we implemented a Julia mapping
for the `STINGER` C structure leaving out the unknown array. Now, we
could do a `unsafe_load` on the pointer handle to the `STINGER` C object, to
load a representation in Julia. We could then modify it and use `unsafe_store!`
to write it back to C. A workflow such as the following was possible at this
point.

```julia
s = Stinger() #Creating a `Stinger` object with a handle to the `STINGER` C pointer
#The convert is required to tell Julia to decode as a `StingerGraph`
sgraph = unsafe_load(convert(Ptr{StingerGraph}, s.handle))
#Attributes available using normal Julia syntax to perform gets and sets on.
sgraph.max_nv = 100000
unsafe_store!((convert(Ptr{StingerGraph}, s.handle), s)
```

Integrating this functionality with the wrapper type we
created in part 1 was our next goal. Here, we had to deal with a major restriction
that is brought about when working with a Julia type that has an incomplete
mapping to the C type like `STINGER`, i.e, **Memory MUST be allocated in C.**
This restriction is enforced because `STINGER`'s' C internal functions make use
of the unknown size arrays, which will not be allocated if we let Julia handle
the memory allocation. A handle to the C allocated memory **MUST** be maintained
to be able to address this memory from Julia.

A major design decision at this point was:

- Should we always maintain a Julia representation of the `Stinger` C object
using an eager approach?
- Or should we only load the representation on demand using a lazy approach?

When using the eager approach, even though we can decode the C structure using a
pointer to a type in Julia, this does not update when the C memory updates. This puts
the onus on the package to maintain some sort of consistency with the C memory.
This could be done by making sure to do an `unsafe_load` every time a `ccall`
occurs. On using a lazy approach, we have the additional overhead of having to
load the memory when the user is trying to access it, rather than having an already
loaded object in Julia memory that can be accessed.

We explored the trade-offs that were occurring here in terms of complexity
with respect to `ccalls` and `loads` and summarized them in the following
table.

|Operation|Eager|Lazy|
|---------|-----|----|
|getfields|Already cached|Has to do a load|
|setfields|Store to pointer|Store to pointer|
|ccalls|Has to be updated after every `ccall`. Loads occurring for every `ccall`| No op|

Using this table, it was obvious that had to choose between a design depending on
the number of  `ccalls` and `getfields` expected.
We modeled the time taken for set, load and ccalls in the following equation.

`T = αS + βL + γC`, where S - number of sets, L - number of loads, C - number of ccalls

For the eager approach:

- S = SF, number of setfields
- L = C, number of ccalls
- C = C

`Te = αSF + βC + γC`

For the lazy approach:

- S = SF, number of setfields
- L = GF, number of getfields
- C = C

`Tl = αSF + βCF + γC`

`Te - Tl = β(C - GF)`

The workflow we expect from the user will
generally contain more `ccalls` than `getfields`.

```
G > CF
=> Tl < Te
```

Hence, we decided to use the lazy approach.

We had one final design decision to make regarding the mapping.

- On a `getfield` or a `setfield`, should we load/store the entire Julia representation to C?
- Or just load/store only the attribute that was demanded?

The `STINGER` C structure
is not expected to change and we know the offsets for every field. Using the
entire Julia representation to store/load requires more memory transfer than just
storing/loading one field. We debated on whether loading the entire structure
would help with the caching, but figured that due to the temporal and spatial nature
of caches, loading just a field would also give us this favorable behavior.

In the next part, we will look at how we can provide users with a clean interface
to access the fields of a `Stinger` object and perform get and set operations.
