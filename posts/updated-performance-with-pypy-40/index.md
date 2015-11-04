<!-- 
.. title: Performance Updates with PyPy 4.0
.. slug: updated-performance-with-pypy-40
.. date: 2015-11-04 15:39:00 UTC-06:00
.. tags:  PyPy, CoMD
.. category: 
.. link: 
.. description: 
.. type: text
-->

The PyPy team recently released version [4.0](http://morepypy.blogspot.com/2015/10/pypy-400-released-jit-with-simd.html)
(The jump in version number is to reduce confusion with the Python version supported.)
One of the features is improved performance.

But first, an issue with reporting accurate timings with this version of CoMD should be addressed. The initial iteration contains overhead from tracing and JIT compilation (Cython and Numba have the same issue).
For this example we are concerned with the steady-state timing, so the time for the first iteration should be excluded.
I've added a '--skip' parameter to the CoMD code (default: 1) that skips the first `printRate` steps (default: 10) in computing the overall average update rate at the end.

Now the table with the most recent performance numbers  (from [this post](http://markdewing.github.io/blog/posts/improvements-in-comd-cell-method-performance/) ), updated with PyPy 4.0 results:


| Language/compiler&nbsp;&nbsp; | Version &nbsp;&nbsp;| Time|
|-------------------|--------------|--------------:|------------:|
| Python            | 2.7.10       |  1014          | |
| PyPy              | 2.6.1        |    96         | |
| Numba             | 0.21.0       |     47     | |
| PyPy              | 4.0          |    30         |  &nbsp; &nbsp; New result |
| Cython            | 0.23.3       |     13     | |
| Julia             | 0.4.0-rc3    |    6.1     | |
| C                 | 4.8.2        |    2.3          | |


<br/>
Times are all in &mu;s/atom. System size is 4000 atoms.
Hardware is Xeon E5-2630 v3 @ 2.4 Ghz, OS is Ubuntu 12.04.
<br/>

The new release of PyPy also includes some SIMD vectorization support (`--jit vec=1` or `--jit vec_all=1`).  Neither of these provided any improvement in performance on this code.   Not too surprising given the vectorization support is new, and the code contains
conditionals in the inner loop.

PyPy 4.0 gives a very good 34x performance improvement over bare Python, and 3x improvement over the previous release (2.6.1).
PyPy is attractive here in that no modifications made to the source.  (Both Cython and Numba required source changes)


