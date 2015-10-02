<!--
.. title: Improvements in CoMD Cell Method Performance
.. slug: improvements-in-comd-cell-method-performance
.. date: 2015-10-02 13:56:00 UTC-05:00
.. tags: CoMD, python, Numba, PyPy, julia, cython
.. category: 
.. link: 
.. description: 
.. type: text
-->

A [previous post](http://markdewing.github.io/blog/posts/first-performance-improvements/) showed some performance improvements with the *nsquared* version of the code.
This post will tackle the *cell* version of the code.
In the *nsquared* version the time-consuming inner loop had no function calls.
The *cell* version does call other functions, which may complicate optimization.


| Language/compiler&nbsp;&nbsp; | Version &nbsp;&nbsp;| Initial time | &nbsp;&nbsp;Final time|
|-------------------|--------------|--------------:|------------:|
| C                 | 4.8.2        |  2.3        |   2.3          |
| Python            | 2.7.10       |  1014        |  1014          |
| PyPy              | 2.6.1        |   96         |   96         |
| Julia             | 0.4.0-rc3    |  87          | 6.1     |
| Cython            | 0.23.3       |  729         |   13     |
| Numba             | 0.21.0       |  867         |   47     |


<br/>
Times are all in &mu;s/atom. System size is 4000 atoms.
Hardware is Xeon E5-2630 v3 @ 2.4 Ghz, OS is Ubuntu 12.04.
<br/>
The 'Initial Time' column results from the minimal amount of code changes to get the compiler working.
<br/>
The 'Final Time' is the time after the tuning in this post.


## Julia

The *cell* version contains the same issue with array operations as the *nsquared* version - the computation of `dd` allocates a temporary to hold the results every time through the loop.

The [Devectorize](https://github.com/lindahua/Devectorize.jl) package can automatically convert array notation
to a loop.  If we add the `@devec` annotation, the time drops to 43 &mu;s/atom.
Unfortunately, the allocation to hold the result must still be performed, and it remains inside the inner particle loop.
If we manually create the loop and hoist the allocation out of the loop, time is 27 &mu;s/atom.



The code uses `dot` to compute the vector norm.  This calls a routine (`julia_dot`) to perform the
dot product.
For long vectors calling an optimized linear algebra routine is beneficial, but for a vector of length 3 this adds overhead.
Replacing `dot` with the equivalent loop reduces the time to 23 &mu;s/atom.


Looking through the memory allocation output (`--track-allocation=user`) shows some vector operations
when the force array is zeroed and accumulated.
Also in `putAtomInBox` in `linkcell.jl`.
 These spots are also visible in the profile output, but the profile output is less convenient because it is not shown with source.
The `@devec` macro does work here, and the performance is now 7.7 &mu;s/atom.   Explicit loops
give a slightly better time of 7.3 &mu;s/atom.

Profiling shows even more opportunities for devectorization in `advanceVelocity` and `advancePosition` in `simflat.jl`  Time is now 6.4 &mu;s/atom.


The Julia executable has a `-O` switch for more time-intensive optimizations (it adds more LLVM optimization passes).   This improves the time to 6.2 &mu;s/atom.

The `@fastmath` macro improves the time a little more, to 6.1 &mu;s/atom.
The `@inbounds` macro to skip the bounds checks did not seem to improve the time.

The final Julia time is now within a factor of 3 of the C time.  The code is [here](https://gist.github.com/markdewing/54709a0fd6a17348a7cb).  It's not clear where the remaining time overhead comes from. 


##PyPy
The PyPy approach to JIT compilation is very general, but that also makes it difficult to target what code
changes might improve performance.
The [Jitviewer](https://bitbucket.org/pypy/jitviewer) tool is nice, but not helpful at a cursory glance.
The [vmprof](https://vmprof.readthedocs.org/en/latest/) profiler solves an important problem by collecting the native code stack plus the python stack. 
In this particular case, it reports at the function level, and the bulk of the time was spent in `computeForce`.
I hope to write more about vmprof in a future post, as it could help with integrated profiling of Python + native code (either compiled or JIT-ed).


##Cython
The simplest step is to add an initialization line and move some `.py` files to `.pyx` files.  This gives 729 &mu;s/atom.
Adding types to the computeForce function and assigning a few attribute lookups to local variables so the types can be assigned (playing a game of 'remove the yellow' in the Cython annotation output) gives 30 &mu;s/atom.


Adding types and removing bounds checks more routines  (in  `halo.py`, `linkcell.py`, `simflat.py`) gives 13 &mu;s/atom.

Code is [here](https://gist.github.com/markdewing/3688c6eebc0a88081e07).
Further progress needs deeper investigation with profiling tools.

##Numba

Starting with adding `@numba.jit` decorators to `computeForce`, and the functions it calls gives the
initial time of 867 &mu;s/atom.
Extracting all the attribute lookups (including the function call to `getNeighborBoxes`) gives 722 &mu;s/atom.

We should ensure the call to `getNeighborBoxes` is properly JIT-ed.  Unfortunately, this requires more involved
code restructuring.  Functions need to be split into a wrapper that performs any needed attribute lookups, and
an inner function that gets JIT-ed.  Loop lifting automatically performs this transformation on functions
  with loops.  On functions without loops, however, it needs to be done manually.
Once this is done, the time improves dramatically to 47 &mu;s/atom.

Hopefully the upcoming "JIT Classes" feature will make this easier, and require less code restructuring. 

Code is [here](https://gist.github.com/markdewing/89cce577f5b8625cc776)

##Summary

Julia is leading in terms of getting the best performance on this example.  Many of these projects are rapidly improving, so this is just a snapshot at their current state.

All these projects need better profiling tools to show the user where code is slow and to give feedback on why the code is slow.
The Cython annotated output is probably the the best - it highlights which lines need attention. 
However it is not integrated with profiler output, so in a project of any size, it's not clear where a user should spend time adding types.  

Julia has some useful collection and feedback tools, but they would be much more helpful if combined.  The memory allocation output is bytes allocated.
It's useful for finding allocations where none were expected, or for allocations in known hot loops, but it's less clear which other allocations are impacting performance.
Ideally this could be integrated with profiler output and weighted by time spent to show which allocations are actually affecting execution time.


