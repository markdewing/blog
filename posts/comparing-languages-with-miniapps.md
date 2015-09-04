.. title: Comparing languages with miniapps
.. slug: comparing-languages-with-miniapps
.. date: 2015-09-02 11:18:50 UTC-05:00
.. tags: Mantevo, python, julia, CoMD, Numba, PyPy
.. category: 
.. link: 
.. description: 
.. type: text


The [Mantevo project](https://mantevo.org/) provides a collection of miniapps, which are simplified versions
of real scientific applications that make it easier to explore performance, scaling, languages, programming models, etc.
I want to focus on the language aspect and port some apps to new languages to see how they compare. 


The first miniapp I started with is [CoMD](http://www.exmatex.org/comd.html), a molecular dynamics code in C.


For the language ports, I made multiple variants of increasing complexity.

nsquared
:   This uses the naive algorithm in computing inter-particle interactions. The central loop computes the
   interaction of every particle with every other particle.  The scaling of run time vs number of particles is N<sup>2</sup>.

cell
:   The cell list method divides space into cells and tracks the particles in each cell.  When computing interactions, only the particles in neighboring cells need to be considered.  The scaling of run time vs. the number of particles is N.

mpi  
:   Parallel version of the cell method.


The C version corresponds to the 'cell' and 'mpi' variants (plus the C version has OpenMP and several other programming model variants)

Currently there are Python and Julia ports for the nsquared and cell variants, and a Python version of the mpi variant.
They are available in my 'multitevo' github repository: [https://github.com/markdewing/multitevo](https://github.com/markdewing/multitevo)

The Julia version is a pretty straightforward port of the Python version, so it is probably not very idiomatic Julia code.
(I would be happy to take suggestions from the Julia community on how to improve the style and organization)

<!--
C does not have multidimensional arrays so the 3D cell indices need to be mapped to a linear index of atom coordinates (and
the reverse), and this involves some complicated expressions.
The Julia and Python version preserve this approach, but since they have multidimensional arrays,
it might make the code much simpler to store the atom information directly in a multidimensional array.
-->

## Scaling with system size

First let's verify the scaling of the nsquared version vs. the cell list version (using the Julia versions).

![Graph of scaling nsquared and cell list method](../../2015/scaling.png)

As expected, the cell list variant has better scaling at larger system sizes.

<!--
Data for nsquared variant
natoms time(s)   time_per_atom (us/atom)
256   0.011350  44.335089
500   0.034140  68.280823
864   0.099966 115.700927
1372   0.239239 174.372460
2048   0.509494 248.776288
2916   1.050071 360.106739
4000   1.942931 485.732711
5324   3.327456 624.991804
6912   5.769225 834.667932
-->

<!--
Data for cell method variant
natoms  time(s)   time_per_atom (us/atom)
256   0.045821 178.987427
500   0.049810  99.619442
864   0.142849 165.333940
1372   0.151120 110.145543
2048   0.310324 151.525439
2916   0.344807 118.246625
4000   0.344878  86.219497
5324   0.621829 116.797365
6912   0.626944  90.703648
-->

##  Initial performance

For a purely computational code such as this, performance matters.
The ultimate goal is near C/Fortran speeds using a higher-level language to express the algorithm.

Some initial timings (for a system size of 4000 atoms, using the cell variant)

| Language/Compiler&nbsp;&nbsp;     | Version   |  Time/atom (microseconds) |
|--------------|:---------|-------------------------:|
| C - gcc      | 4.8.2         | 2.3 | 
| Julia        | 0.3.11            | 153.0 |
| Julia        | 0.4.0-dev+6990    | 88.8 |
| Python       | 2.7.10 (from Anaconda) &nbsp;  | 941.0 |
| Numba        | 0.20.0         | 789.0 |  
| Pypy         | 2.6.1          | 98.4 | 


<br/>

(Hardware is Xeon E5-2630 v3 @ 2.4 Ghz, OS is Ubuntu 12.04)

**Disclaimer**: These numbers indicate how a particular version of the language and compiler perform on a particular version of this code. The main purpose for these numbers is a baseline to measure future performance improvements.

I tried [HOPE](http://www.cosmology.ethz.ch/research/software-lab/HOPE.html), a Python to C++ JIT compiler.
It require some modifications to the python code, but then failed in compiling the resulting C++ code.
I also tried [Parakeet](http://www.parakeetpython.com/).  It failed to translate the Python code, and I did not investigate further.

It is clear when comparing to C there is quite a bit of room for improvement in the code using the high-level language compilers (Julia, Numba, PyPy).
Whether that needs to come from the modifications to the code, or improvements in the compilers, I don't know yet.

The only real performance optimization so far has been adding type declarations to the composite types in Julia.
This boosted performance by about 3x. Without the type declarations, the Julia 0.4.0 speed is about 275 us/atom.
First performance lesson: Add type declarations to composite types in Julia.

Julia and Numba have a number of similarities and so I want to focus on improving the performance of the code
under these two systems in the next few posts.

<!--
Julia and Numba have similar stages (infer types, convert to LLVM IR, convert to native code), and so I hope to look
at them in parallel going forward.  
-->

