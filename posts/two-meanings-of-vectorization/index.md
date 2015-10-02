<!--
.. title: Two Meanings of Vectorization
.. slug: two-meanings-of-vectorization
.. date: 2015-10-01 22:23:00 UTC-05:00
.. tags: vectorization
.. category:
.. link:
.. description:
.. type: text
-->


The term 'vectorize' as used by programmers has at least two separate uses.
Both uses can have implications for performance, which sometimes leads to confusion.

One meaning refers to a language syntax to express operations on multiple values - typically an entire array, or a slice of a array.
This can be a very convenient notation for expressing algorithms.

Here is a simple example (in Julia) using loop-oriented (devectorized) notation

```julia
a = ones(Float64, 10)
b = ones(Float64, 10)
# allocate space for result
c = zeros(Float64, 10)

for i in 1:10
    c[i] = a[i] + b[i]
end
```

Now compare with using vectorized (array) notation
```julia
a = ones(Float64, 10)
b = ones(Float64, 10)

# space for result automatically allocated

c = a + b
```

The vectorized notation is more compact.
Julia and Python/Numpy programmers usually mean this when referring to 'vectorization'.
See more in the Wikipedia entry on [Array programming](https://en.wikipedia.org/wiki/Array_programming)

John Myles White wrote a post discussing the performance implications of [vectorized and devectorized code](http://www.johnmyleswhite.com/notebook/2013/12/22/the-relationship-between-vectorized-and-devectorized-code/) in Julia and R.
Note that Python/Numpy operates similar to R as described in the post - good performance usually requires appropriately vectorized code, because that skips the interpreter and calls higher performing C routines underneath.

The other meaning of 'vectorization' refers to generating assembly code to make effective use of fine-grained parallelism in hardware SIMD units.
This is what Fortran or C/C++ programmers (and their compilers) mean by 'vectorization'.
In Julia, the `@simd` macro gives hints to the compiler that a given loop can be vectorized.

See more in the Wikipedia entry on [Automatic vectorization](https://en.wikipedia.org/wiki/Automatic_vectorization)


