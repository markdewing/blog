<!-- 
.. title: Integration Callbacks with Sympy and LLVM
.. slug: integration-callbacks
.. date: 2016-07-08 16:37:00 UTC-05:00
.. tags: sympy, llvm, integration, cubature
.. category: 
.. link: 
.. description: 
.. type: text
-->

This post explores various packages for multi-dimensional integration along with
generating callbacks for the integrands from Sympy using an LLVM JIT

## Problem to integrate

The particular problem is using the variational principle to find the ground state energy for atoms.
Some Jupyter notebooks with a description of the problem, along with various integration methods:

[Ground state energy of Hydrogen Atom](https://github.com/markdewing/next_steps_in_programming/blob/master/examples/integration/Hydogen%20Atom.ipynb)   (This yields a 3 dimensional integral.)

[Ground state energy of Helium Atom](https://github.com/markdewing/next_steps_in_programming/blob/master/examples/integration/Helium%20atom.ipynb)  (This yields a 6 dimensional integral.)

The standard solution to these integrals is to use Markov Chain Monte Carlo (the Quantum Monte Carlo method).  
However, I'm curious to see how far alternate integration schemes or existing integration packages would work.

## Integration libraries

The [scipy quadrature](http://docs.scipy.org/doc/scipy/reference/tutorial/integrate.html) routines accept a natively compiled callback for the integrand. 
(Noticing this in the documentation initiated the idea for using JIT compilation for callback functions.)

Next up is the [Cubature](http://ab-initio.mit.edu/wiki/index.php/Cubature) integration package, with the [Python wrapper for cubature](https://github.com/saullocastro/cubature)

Finally is the [Cuba](http://www.feynarts.de/cuba/) library, with the PyCuba interface (part of the [PyMultiNest](https://github.com/JohannesBuchner/PyMultiNest) package)

There are some other libraries such at [HIntLib](http://mint.sbg.ac.at/HIntLib/) that I would also like to try.  There doesn't seem to be a python interface for HIntLib.  Let me know if there is one somewhere. And if there are other multidimensional integration packages to try.


## Evaluating the integrand

One of my scientific programming goals is to generate efficient code from a symbolic expression.
To this end, I've been working on an LLVM JIT converter for Sympy expressions (using the [llvmlite](https://github.com/numba/llvmlite) wrapper).

For the Sympy code, see these pull requests: 

- [Create executable functions from Sympy expressions](https://github.com/sympy/sympy/pull/10451) 
- [Accelerated callbacks for integration routines](https://github.com/sympy/sympy/pull/10640)
- [JIT - handle multiple expressions (as returned from CSE)](https://github.com/sympy/sympy/pull/10683)
- [Add LLVM JIT callbacks for PyCuba integration](https://github.com/sympy/sympy/pull/11057)


As an aside, one can question if is this the right approach, compared with

1. Generate C++ or Fortran and compile using the existing autowrap functionality in Sympy.
2. Generate Python/Numpy and use Numba.
3. Use Julia

There is always a tradeoff between a narrow, specialized solution, which is faster to implement and
perhaps easier to understand, and a more general solution, which applies in more cases, but is
harder and slower to implement.

Using an LLVM JIT is a specialized solution, but it does have an advantage that there is a short path from the expressions to the compiled code.
One disadvantage is that it does not leverage existing compilers (Numba or C++), though LLVM compiler optimization passes are available.

Sometimes a solution just needs to be tried to gain experience with the advantages and drawbacks.

## Results

For the helium atom, the integration times are reported in the table below

| Integrator&nbsp;&nbsp; | Time (seconds) |
|:-------------------|------------:|
| Cubature           | 171  |
| Cubature w/CSE     | 141  |
| Cubature w/CSE and multiple evals  | 100  |
| Cuba (Vegas)  | 29  |
| Cuba (Cuhre)  | 22  |

<br/>

Note that `scipy.nquad` was not used for the 6D integral. It quickly runs out of steam because it consists of iterated one dimensional integrations, and the glue between the dimensions goes through Python, reducing the effectiveness of a compiled integrand.

The Cubature library does better.  Profiling shows that most of the time is spent internal to cubature and allocating memory, so faster integrand evaluation is not going to improve the time.
Some other approaches can help.  One is Common Subexpression Elimination (CSE), which Sympy can perform on the expression.  This extracts duplicate fragments so their value only needs to be computed once.

The library also allows multiple integrals to be performed at once.   This can amortize some of the overhead of the library.  In this case, the individual calls to integrator for the numerator and denominator can be reduced to a single call.


The Cuba library performs even better, as there is apparently less overhead inside the integration library.  The Cuhre integrator uses a deterministic grid-based algorithm similar to Cubature.  Vegas uses an adaptive Monte Carlo approach.

The results are not shown here, but I also used SIMD vectorization to make the function evaluation even faster, which succeeded for the bare function evaluation. (This was one of the original motivations for compiling straight to LLVM, as it would be easier to get vectorization working.)
 Unfortunately, it did not speed up the overall integration much (if at all), due to overhead in the libraries.

## Conclusions and future work
Using an LLVM JIT to create callbacks for integration works fairly well.

One important question is how to scale the creation of the callbacks to new libraries without explicitly programming them into Sympy.  
The [last pull request](https://github.com/sympy/sympy/pull/11057) has expanded the `CodeSignature` class, which seems like  a starting point for such a more general callback specification.

