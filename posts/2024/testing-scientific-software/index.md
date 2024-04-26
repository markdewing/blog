<!-- 
.. title: Testing Scientific Software
.. slug: testing-scientific-software
.. date: 2024-04-25 11:20:00 UTC-05:00
.. tags: testing, sympy, python
.. category: 
.. link: 
.. description: 
.. type: text
-->


A perennial question when writing scientific code is how do you know the code is producing the right answer?

This post describes the [QMC algorithms](https://github.com/QMCPACK/qmc_algorithms) repository and testing in QMCPACK development to provide some answers this question.
QMCPACK implements Quantum Monte Carlo (QMC) methods for solving the Sch&ouml;dinger equation for atoms, molecules, and solids.

The repository focuses on a few areas to contribute to testing and understanding scientific software:

1. Derived formulas using symbolic math
2. Code generation from symbolic math
3. Small problems with simple algorithms
4. Reproduce and explain papers


Another driving force in this work is that I have a strong conviction that symbolic mathematics is the most appropriate semantic level to describe scientific algorithms.
In this repository, Sympy is used for symbolic mathematics.



## Deriving formulas

Some parts of scientific code are the final result of a series of derivation steps that are then
translated to code.  How do we know these steps are correct?  Especially the final step of math to code.
Usually these steps are all carried out by hand, with opportunities for errors to creep in.
We can use the computer to perform and check some of these steps.

### Computing derivatives for comparisons

A Pad&eacute; (rational function) form is the simplest functional form for a Jastrow factor, which describes electron-electron or electron-nucleus correlation in a wavefunction.
The [Pade_Jastrow.ipynb](https://github.com/QMCPACK/qmc_algorithms/blob/master/Wavefunctions/Pade_Jastrow.ipynb) notebook simply expresses the form, computes some derivatives, and evaluates them at a particular value.

The simplest way to use the results is to cut and paste the values from the notebook to the test case.
This gets tedious very quickly and is not easy to regenerate, especially when there are more
than a few values to copy.
It's straightforward to make a little template to output the values in a form
that is more directly usable in the test.

For example, the following Python code will output a line suitable for use in a C++ unit test (with [Catch2](https://github.com/catchorg/Catch2) assertions).

```python
A,B,r = symbols('A B r')
sym_u = A*r/(1.0 + B*r) - A/B
val_u = u.subs({A:-0.25, B:0.1, r:1.0})
print(f'CHECK(u == Approx({val_u}));')
```

### Other Derivations
There are a few types of splines used in QMCPACK.

For [cubic splines](https://github.com/QMCPACK/qmc_algorithms/blob/master/Wavefunctions/Cubic%20Splines%20Basic.ipynb), the notebook sets up the equations based on constraints and boundary conditions.
It computes an example and solve the equations using a dense linear solver. (An example of using a simpler algorithm for validation.  The real code uses a more efficient tridiagonal solver.)
This gets incorporated into QMCPACK tests with the script [gen_cubic_spline.py](https://github.com/QMCPACK/qmcpack/blob/develop/src/Numerics/tests/gen_cubic_spline.py) and the output of that script is in [test_one_dim_cubic_spline.cpp](https://github.com/QMCPACK/qmcpack/blob/develop/src/Numerics/tests/test_one_dim_cubic_spline.cpp).
    

B-splines are used in a one-dimensional form for Jastrow electron-electron and electron-ion correlation factors.
And also in a three-dimensional form for representing single-particle periodic orbitals.
They are much faster to evaluate than a plane wave representation.
The [Explain_Bspline.ipnyb](https://github.com/QMCPACK/qmc_algorithms/blob/master/Wavefunctions/Explain_Bspline.ipynb) notebook starts from B-splines defined in Sympy and works through how they get used in QMCPACK.

## Code generation
Code generation is used for the solver for the coefficients of cubic splines.
It starts from the tridiagonal matrix equations from Wikipedia, derives the cubic spline equations (same as previously), combines the two and outputs the resulting algorithm to C++ code.

The notebook is [CubicSplineSolver.ipynb](https://github.com/QMCPACK/qmc_algorithms/blob/master/Wavefunctions/CubicSplineSolver.ipynb).
The corresponding script in QMCPACK is [gen_cubic_spline_solver.py](https://github.com/QMCPACK/qmcpack/blob/develop/src/Numerics/codegen/gen_cubic_spline_solver.py).
The generated spline solver is located in [SplineSolvers.h](https://github.com/QMCPACK/qmcpack/blob/develop/src/Numerics/SplineSolvers.h).


Some simpler code generation in the QMCPACK repository involves the angular part of atomic orbitals.  
There is a notebook about [Gaussian orbitals](https://github.com/QMCPACK/qmc_algorithms/blob/master/Wavefunctions/GaussianOrbitals.ipynb).
In this case, the starting expression is simple and computing the various derivatives is tedious
and the script in QMCPACK, [gen_cartesian_tensor.py](https://github.com/QMCPACK/qmcpack/blob/develop/src/Numerics/codegen/gen_cartesian_tensor.py), takes care of that part.



## Simple algorithms

Some ways to obtain reference values for test cases include special cases that have analytic solutions
and small problems that can be solved using an alternative method that is easier to understand and verify than the implemented algorithm.

Computing long-range sums of periodic Coulomb potentials requires some tricks to ensure convergence.
The Ewald method splits the sum in two pieces and uses the Fourier transform of the long-range part for faster convergence. It is implemented in a simple Python script, [ewald_sum.py](https://github.com/QMCPACK/qmc_algorithms/blob/master/LongRange/ewald_sum.py), which can be used to verify more sophisticated splits of the sum used in the code.


For the Schr&ouml;dinger equation, an exact analytic solution is known for the hydrogen atom.
A deliberately incorrect wavefunction is used as a trial wavefunction in [Variational_Hydrogen.ipynb](https://github.com/QMCPACK/qmc_algorithms/blob/master/Variational/Variational_Hydrogen.ipynb) to demonstrate the variational principle.

A slightly larger problem is that of a helium atom.
The ground state wavefunction for a helium atom does not have an analytic solution and is the simplest example involving electron-electron correlation.
The notebook [Variational_Helium.ipynb](https://github.com/QMCPACK/qmc_algorithms/blob/master/Variational/Variational_Helium.ipynb) uses symbolic means to find the derivatives of the local
energy, and grid-based integration instead of Monte Carlo.
(The details of the integration are discussed [here](https://markdewing.github.io/blog/posts/integration-callbacks/).)

Another approach to the derivatives is autodifferentiation.
There is a minimal QMC code used for creating reference values, particular for derivatives of variational parameters and orbital rotation.
It uses the [autograd](https://github.com/HIPS/autograd) package for differentiation.
The QMC loop is in [run_qmc.py](https://github.com/QMCPACK/qmcpack/blob/develop/src/QMCWaveFunctions/tests/run_qmc.py).

The target systems are:
* Helium atom in [gen_rotated_lcao_wf.py](https://github.com/QMCPACK/qmcpack/blob/develop/src/QMCWaveFunctions/tests/gen_rotated_lcao_wf.py)
* Beryllium atom in [rot_be_sto_wf.py](https://github.com/QMCPACK/qmcpack/blob/develop/src/QMCWaveFunctions/tests/rot_be_sto_wf.py)
* Beryllium atom with two determinants in [rot_multi_be_sto_wf.py](https://github.com/QMCPACK/qmcpack/blob/develop/src/QMCWaveFunctions/tests/rot_multi_be_sto_wf.py)
* Solid Beryllium with a psuedopotential in [eval_bspline_spo.py](https://github.com/QMCPACK/qmcpack/blob/develop/src/QMCWaveFunctions/tests/eval_bspline_spo.py)

(To improve on the performance of run_qmc.py and related scripts, I wrote a version using the multiprocessing package in Python. For even more performance, there is a port of these scripts to Julia.)


## Reproduce and explain papers

Published papers contain derivations that leave intermediate steps as an exercise for the reader 
should they want to reproduce or implement the described algorithm.

There is a nice writeup, contributed by Andrew Baczewski, to explain and reproduce a paper ["Observations on the statistical iteration of matrices"](https://journals.aps.org/pra/abstract/10.1103/PhysRevA.30.2713)
in this notebook
[Reproduce Hetherington PRA1984.ipynb](https://github.com/QMCPACK/qmc_algorithms/blob/master/StochasticReconfiguration/Reproduce_Hetherington_PRA1984.ipynb)
The paper describes a stochastic reconfiguration method for controlling variance and bias in a Monte Carlo simulation.

Another example involves a cusp correction scheme, which adds some modifications to Gaussian-type orbitals
commonly used in quantum chemistry to work better for a Quantum Monte Carlo wavefunction.
It comes from the paper 
[Scheme for adding electron-nucleus cusps to Gaussian orbitals](https://doi.org/10.1063/1.1940588).
The notebook
[CuspCorrection.ipynb](https://github.com/QMCPACK/qmc_algorithms/blob/master/Wavefunctions/CuspCorrection.ipynb)
follows through solving some polynomial equations and presents some examples.
The script used to generate reference values for tests in QMCPACK is [gen_cusp_corr.py](https://github.com/QMCPACK/qmcpack/blob/develop/src/QMCWaveFunctions/tests/gen_cusp_corr.py).


The process for using notebooks is
1. Jupyter notebook provides exposition of an algorithm or concept.
2. The code gets copied to a script in the appropriate test directory in the
QMCPACK repository, and that location is the code that produces the results for the QMCPACK tests.

This does result in duplication of code, but at least the scripts and tests in the QMCPACK repository are self-contained.

## Final thoughts

This post described several techniques for testing scientific software.

One of the important aspects of learning is working with the material and wrestling with it.
At some point, you have to do the derivation yourself in order to learn it.
These notebooks are a record of my approach to exploring these topics.
Are these useful to anyone else? Could they be made more useful?

The descriptions in many of the notebooks are very short and could be expanded.
What would make them more useful?  For someone trying to understand
the code?  Or for someone trying to reproduce and extend the work?
