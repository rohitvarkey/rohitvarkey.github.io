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
API for users to work with Stinger from Julia. We had a working Stinger
REPL in Julia at this point which enables a graph algorithm developer to
interactively construct and probe a graph with instant feedback. To test the
usability and performance of the wrapper, we use the Breadth First Search (BFS).
BFS is an algorithm that is used as a building block for more complex graph
processing algorithms like betweenness centrality.

### Julia BFS implementation

Our [implementation](https://github.com/rohitvarkey/StingerWrapper.jl/blob/7c64859cc2bfbd4210344b391068bd86ba9135cf/src/algorithms/bfs.jl#L13-L31) of BFS performs most of the steps within Julia. We
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

In the above 18 lines of Julia code, only the `getsuccessors` function makes a
C call to the Stinger C library. It returns a Julia `Vector{Int64}` of the vertex
ID's that are successors to the source. The rest of the algorithm is performed
entirely in Julia.

### The Graph 500 benchmark

The [Graph 500](http://www.graph500.org/) benchmark is a standard graph benchmark.
It consists of running BFS on a Kronecker generated graph.
We use this benchmark to test StingerWrapper as we feel it has sufficient
complexity, has a straightforward implementation and is a standard algorithm for
graph processing benchmarks.

### Kronecker graph generators

We use a [Kronecker generator](https://github.com/rohitvarkey/StingerWrapper.jl/commit/92ba860b94e4d0e1bbb96077f06ba9bfcb513314) based on the reference implementation
provided by the [Graph 500](http://www.graph500.org/specifications#alg:generator) organization.
The Kronecker generator generates a random set of `2^scale` vertices each with
`edgefactor` number of edges and according to the probabilities specified.

### The `get_nv` problem

In our earlier implementation of BFS, we were making one more call to the Stinger C
library. We were making a call to the `stinger_max_active_vertex` C function using the
`get_nv` wrapper Julia function during
the BFS to correctly allocate the memory for the parents array.

The call to `stinger_max_active_vertex` caused a large slowdown in machines with
large memory allocated to the Stinger graph. Controlled
experiments were used to determine and test this slowdown.
This counter intuitive behavior occurred due to the call to the
`stinger_max_active_vertex` function, which [iterates sequentially](https://github.com/stingergraph/stinger/issues/33)
through every single vertex for which memory has been allocated in the graph.

This bottleneck was removed by pulling out this call from the BFS and making the
user specify the maximum number of vertices in the `bfs` function. However, this
makes the API slightly less easy to use and brings the Julia interface closer to the underlying C interface. This is an instance of the design trade-offs inherent in productivity/performance.
Julia allows us to get better performance and productivity, but memory management
is still something that requires careful programming to get optimal performance.

### Benchmarks numbers

Here we present the benchmark numbers obtained for running BFS on the first 1000
vertices of Kronecker generated graphs with given scale and edgefactor.

|Scale|Edgefactor|median time|memory estimate| Traversed Edges Per Second (TEPS)|
|-----|----------|-----------|---------------|-------------------------------------|
|10|16|1.610 s (0.52% GC)|141.25 mb|10176.397|
|11|16|3.127 s (0.55% GC)|273.86 mb|10479.053|
|12|16|6.327 s (0.53% GC)|539.48 mb|10358.147|
|13|16|13.059 s (0.51% GC)|1018.62 mb|10036.909|
|14|16|28.129 s (0.43% GC)|1.99 gb|9319.35|
|15|16|55.993 s (0.43% GC)|3.82 gb|9363.45|
|16|16|110.894 s (0.47% GC)|7.43 gb|9455.66|
|17|16|221.210 s (0.40% GC)|14.39 gb|9480.367|
|18|16|442.722 s (0.39% GC)|27.33 gb|9473.9|
|19|16|868.489 s (0.94% GC)|53.65 gb|9658.85|
|20|16|1661.592 s (0.90% GC)|100.68 gb|10097.07|

We observe linear growth for both the times and well as the memory estimate.

### Profiling the BFS

On profiling the `bfs` call we notice that most of the time is spent in calling
to C using the `getsuccessors` function. Very less time is spent in the Julia parts
of the code. Explore the profile [here](/public/profile_results.svg).

<img src="/public/profile_results.svg" alt="Profile of bfs"/>

Also this shows that reducing the amount of memory that crosses the heap boundary is important.
Improving the interface to stinger to avoid having to copy the edgelist is an important step in improving performance. The key step here will be to implement julia [iterators](http://docs.julialang.org/en/release-0.5/manual/interfaces/) that
directly iterate over the stinger edgeblocks without making a copy.

We will be investigating whether it is crossing the memory boundary or just the
gathering up the edges into an array from the edge blocks that is costing more.
Optimistically for Julia, the cost is in stinger gathering things up into a packed array.
Since you aren't supposed to use those functions in stinger, they are not all that fast.
The iteration is implemented with C macros to avoid that gathering step as much as possible.   

### Future improvements

Having to working with 1 based indexing in Julia and 0 based indexing in Stinger makes writing
Julia code to interact with Stinger a bit troublesome. It also opens up possibilities of
off by one errors. We [plan](https://github.com/rohitvarkey/StingerWrapper.jl/issues/8) on using the shiny new Julia 0.5 feature of [custom abstract array types with non-1 indexing](http://docs.julialang.org/en/release-0.5/devdocs/offset-arrays/#writing-custom-array-types-with-non-1-indexing).
