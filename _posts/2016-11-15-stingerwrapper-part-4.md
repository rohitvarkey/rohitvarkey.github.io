---
layout: post
title: StingerWrapper.jl - Part 4
---

**This post is the fourth among a series of blog posts describing my
experiences while developing [StingerWrapper.jl](https://github.com/rohitvarkey/StingerWrapper.jl),
a Julia interface to the [STINGER](https://github.com/stingergraph/stinger/)
C package for dynamic graph analysis. I have been working on StingerWrapper along
with Dr. James Fairbanks.**

In parts 1, 2, and 3 we built up StingerWrapper to a state where we had a good
API for users to work with Stinger from Julia. We basically had a working Stinger
REPL in Julia at this point. To test the usability and performance of the wrapper,
we implemented a common algorithm used in graph processing - Breadth First Search (BFS).

### Julia BFS implementation

Our [implementation](https://github.com/rohitvarkey/StingerWrapper.jl/blob/7c64859cc2bfbd4210344b391068bd86ba9135cf/src/algorithms/bfs.jl#L13-L31) of BFS tries to perform most of the steps within Julia. We
interact with the C library only for the essential steps of getting the successors
of a node.

```julia
function bfs(s::Stinger, source::Int64, nv::Int64)
    parents = fill(-2, nv) #Initialize parents array with -2's.
    parents[source+1] = -1 #Set source to -1
    next = Vector{Int64}([source])
    sizehint!(next, nv)
    while !isempty(next)
        src = shift!(next) #Get first element
        vertexneighbors = getsuccessors(s, src) #Makes a call to C
        for vertex in vertexneighbors
            #If not already set, and is not found in the queue.
            if parents[vertex+1]==-2
                push!(next, vertex) #Push onto queue
                parents[vertex+1] = src
            end
        end
    end
    parents
end
```

In the above 20 lines of Julia code, only the `getsuccessors` function makes a
C call to the Stinger C library. It returns a Julia `Vector{Int64}` of the vertex
ID's that are successors to the source. The rest of the algorithm is completely
done in Julia.

### The Graph 500 benchmark

The [Graph 500](http://www.graph500.org/) benchmark is a standard graph benchmark.
It consists of running BFS on a Kronecker generated graph.
We chose to use this benchmark to test StingerWrapper as we felt it had sufficient
complexity, could be easily implemented and was a standard used in graph processing
benchmarks.

### Kronecker graph generators

We implemented a [Kronecker generator](https://github.com/rohitvarkey/StingerWrapper.jl/commit/92ba860b94e4d0e1bbb96077f06ba9bfcb513314) based on the reference implementation
provided by the [Graph 500](http://www.graph500.org/specifications#alg:generator) organization.
The Kronecker generator generates a random set of `2^scale` vertices each with
`edgefactor` number of edges and according to the probabilities specified.

### The `get_nv` problem

In our earlier implementation of BFS, we were making one more call to the Stinger C
library. We were making a call to the `stinger_max_active_vertex` C function using the
`get_nv` wrapper Julia function during
the BFS to correctly allocate the memory for the parents array.

We saw a huge
slowdown on running this on a server with a large memory allocated to a Stinger graph
compared to the same benchmark running on a laptop with much lower memory allocated.
This counter intuitive behavior was occurring due to the call to the `stinger_max_active_vertex`
function, which iterates sequentially through every single vertex for which memory
has been allocated in the graph.

Pulling out this call from the BFS and making the user specify the maximum number
of vertices in the `bfs` function removed this bottleneck and gave us similar times
on both machines.

### Benchmarks numbers

Here we present the benchmark numbers obtained for running BFS on the first 1000
vertices of Kronecker generated graphs with given scale and edgefactor.

|Scale|Edgefactor|median time|memory estimate|
|-----|----------|-----------|---------------|
|10|16|1.610 s (0.52% GC)|141.25 mb|
|11|16|3.127 s (0.55% GC)|273.86 mb|
|12|16|6.327 s (0.53% GC)|539.48 mb|
|13|16|13.059 s (0.51% GC)|1018.62 mb|
|14|16|28.129 s (0.43% GC)|1.99 gb|
|15|16|55.993 s (0.43% GC)|3.82 gb|
|16|16|110.894 s (0.47% GC)|7.43 gb|
|17|16|221.210 s (0.40% GC)|14.39 gb|
|18|16|442.722 s (0.39% GC)|27.33 gb|
|19|16|868.489 s (0.94% GC)|53.65 gb|
|20|16|1661.592 s (0.90% GC)|100.68 gb|

We observe linear scaling for both the times and well as the memory estimate.
On average, for a BFS on one vertex, it takes around `1.5ms` using StingerWrapper.jl.

### Profiling the BFS

On profiling the `bfs` call we notice that most of the time is spent in calling
to C using the `getsuccessors` function. Very less time is spent in the Julia parts
of the Code. Explore the profile [here](/public/profile_results.svg).

<img src="/public/profile_results.svg" alt="Profile of bfs"/>

### Future improvements

Having to working with 1 based indexing in Julia and 0 based indexing in Stinger makes writing
Julia code to interact with Stinger a bit troublesome. It also opens up possibilities of
off by one errors. We [plan](https://github.com/rohitvarkey/StingerWrapper.jl/issues/8) on using the shiny new Julia 0.5 feature of [custom abstract array types with non-1 indexing](http://docs.julialang.org/en/release-0.5/devdocs/offset-arrays/#writing-custom-array-types-with-non-1-indexing).
