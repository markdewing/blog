<!-- 
.. title: First Performance Improvements
.. slug: first-performance-improvements
.. date: 2015-09-17 23:48:00 UTC-05:00
.. tags:  julia, python, Numba, PyPy, cython
.. category: 
.. link: 
.. description: 
.. type: text
-->

The [previous post](http://markdewing.github.io/blog/posts/comparing-languages-with-miniapps/) introduced the CoMD miniapp in Python and Julia.
This post will look a little deeper into the performance of Julia and various Python compilers.
I will use the nsquared version of the code for simplicity.

The `computeForce` function in `ljforce` takes almost all the time, and it is here we should focus.
In the nsquared version there are no further function calls inside this loop, which should make it
easier to analyze and improve the performance.

The summary performance table (all times are in microseconds/atom, the system is the smallest size at 256 atoms)

<!-- from laptop -->
| Language/compiler&nbsp;&nbsp; | version &nbsp;&nbsp;| Initial time | &nbsp;&nbsp;Final time|
|-------------------|--------------|--------------:|------------:|
| Python            | 2.7.10       | 560          |            |
| PyPy              | 2.6.1        |  98          |            |
| HOPE              | 0.4.0        |   8          |            |   
| Cython            | 0.23.1       | 335          |   8.3      |  
| Numba             | 0.21.0       | 450          |   8.3      |
| Julia             | 0.5.0-dev+50 |  44          |   7.1      |

<!-- From desktop 
| Python            | 2.7.10  | 438          |            |
| PyPy              |         |  84          |            |
| HOPE              |         |   7          |            |
| Cython            |         | 329          |  49        |
| Numba             |         | 405          |   7        |
| Julia             |         |  45          |  19        |
-->

<br/>

(This table uses a different code version and different hardware, so the numbers are not comparable to the previous post)
<br/>
(Hardware is i7-3720QM @ 2.6 Ghz and OS is Ubuntu 14.04)
<br/>
The 'Initial Time' column results from the minimal amount of code changes to get the compiler working.
<br/>
The 'Final Time' is the time after doing some tuning (for this post, anyway - there's still more work to be done).

<!--The level of tuning in this post is about adding type information and removing temporary memory allocations.
Further investigation of the generated intermediate and assembly code is yet to be done.
-->

## HOPE
The HOPE compiler will compile the nsquared version, provided the branch involving the cutoff is modified to avoid the `continue` statement.
The loop that zeros the force also needs a simple name change to the loop variable.
The backend C++ compiler is gcc 4.8.4.  No attempt was made to try different compilers or further optimize the generated code.


## Numba

To use Numba, we need to add `import numba` to `ljforce.py` and add the `@numba.jit` decorator to the `computeForce` method.
This gives the time in the initial column (450 &mu;s/atom) , which is about a 20% improvement over the plain Python version.

Code involving attribute lookups cannot currently be compiled to efficient code, and these lookups occur inside this inner loop.
And the compiler will not hoist attribute lookups outside the loop.
This can be done manually by assigning the attribute to a temporary variable before the loop, and replacing the values in the loop body. 
This transformation enables effective compilation of the loop.

(Internally Numba performs loop-lifting, where it extracts the loop to a separate function in order to compile the loop.)

<!--Loop-lifting is a way for Numba to extract a loop into a separate function in order to compile the loop.-->

The beginning of the function now looks like

```python
    @numba.jit
    def computeForce(self, atoms, sim):
        # hoist the attribute lookups of these values
        f = atoms.f
        r = atoms.r
        bounds = atoms.bounds
        nAtoms = atoms.nAtoms
        epsilon = self.epsilon
        
        # rest of original function
        # ...
```

Now Numba can compile the time consuming loop, and this gives about 9 &mu;s/atom.

The loop that zeros the force can be slightly improved by either looping over the last dimension
explicitly, or by zeroing the entire array at once.  This change yields the final number in the table (8.3 &mu;s/atom).

The whole modified file is [here](https://gist.github.com/markdewing/eb0bf52ea1b71995150a).


## Cython

The simplest change to enable Cython is to move `ljforce.py` to `ljforce.pyx`, and add `import pyximport; pyximport.install()` to the beginning of `simflat.py`.
This initial time (335 &mu;s/atom) gives a 40% improvement over regular python, but there is more performance available.

The first step is to add some type information.
In order to do this we need to hoist the attribute lookups and assign to temporary variables, as in the Numba version.
At this step, we add types for the same variables as the Numba version.
The beginning of the function looks like this:

```python
    def computeForce(self, atoms):
        cdef int nAtoms
        cdef double epsilon
        cdef np.ndarray[dtype=np.double_t, ndim=2] f
        cdef np.ndarray[dtype=np.double_t, ndim=2] r
        cdef np.ndarray[dtype=np.double_t] bounds

        f = atoms.f
        r = atoms.r
        bounds = atoms.bounds
        nAtoms = atoms.nAtoms
        epsilon = self.epsilon

        # rest of original function
        # ...
```

The time for this change about 54 &mu;s/atom.

Cython has a convenient feature that creates an annotated HTML file highlighting lines in the 
original file that may causing a performance issue.  Run `cython -a ljforce.pyx` to get the report.
This indicates some more type declarations need to be added.
Adding these types gives about 8.6 &mu;s/atom.   Finally a decorator can be added to remove bounds checks (`@cython.boundscheck(False)`) to get the final performance in the table (8.3 &mu;s/atom).

The whole `ljforce.pyx` file is [here](https://gist.github.com/markdewing/7017e23c883a2bd297cb)


## Julia
The biggest issue in this code seems to be allocations inside the inner loop.
The memory performance tools can indicate where unexpected allocations are occurring.
One tool is to use a command line option (`--track-allocation=user`) to the julia executable.

One problem is a temporary created inside the loop to hold the results of an array operation (the line that computes `dd`).
Moving this allocation out of the loop and setting each element separately improves performance 
to 19 &mu;s/atom.
Another allocation occurs when updating the force array using a slice.  Changing this to explicitly loop
over the elements improves the performance to the final numbers shown in the table (7.1 &mu;s/atom).

The final modified file is [here](https://gist.github.com/markdewing/f1c9a46a2fec8a3ade4f)

## Summary

The performance numbers have quite a bit of variance, and they are not a result of a rigorous benchmarking and statistics collection.  If you want to compare between compilers, the final results should probably be read something like: "The performance of Cython and Numba is roughly the same on this code, and Julia is a little bit faster for this code".
Also keep in mind we're not done yet digging into the performance of these different compilers.

Some simple changes to the code can give dramatic performance improvements, but the difficulty is
discovering what changes need to be made and where to make them.

Future topics to explore:
 
* Apply these lessons to the cell version of the code.
* With Julia and Numba, it's hard to connect intermediate code stages (internal IR, LLVM IR, assembly) to the original code, and to spot potential performance issues there.  The Cython annotation output is nice here.
* The difference between operating on dynamic objects versus the underlying value types.
* How well does the final assembly utilize the hardware. How to use hardware sampling for analysis.









