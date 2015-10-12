<!-- 
.. title: Prototype for Profiling Python
.. slug: prototype-for-profiling-python
.. date: 2015-10-12 12:20:00 UTC-05:00
.. tags: python, Numba, profiling, vmprof
.. category: 
.. link: 
.. description: 
.. type: text
-->


[Last post](../towards-profiling-accelerated-python/) covered some technical background using vmprof to profile Python with compiled or JIT'ed extensions.
Now I've created a prototype that can convert the output to callgrind format so it can be viewed with [KCachegrind](http://kcachegrind.sourceforge.net/html/Home.html).


To install the prototype using the Anaconda distribution:

1. Create a new environment (if you do not use a new environment, these packages may conflict with an existing Numba install): `conda create -n profiling python numpy`
2. Switch to the new environment: `source activate profiling`
3. Install prototype versions of Numba and llvmlite: `conda install -c https://conda.anaconda.org/mdewing numba-profiling`
4. Install prototype version of vmprof: `conda install -c https://conda.anaconda.org/mdewing vmprof-numba`
5. Make sure libunwind is installed.  (On Ubuntu `apt-get install libunwind8-dev`.)
(On Ubuntu, it must be the -dev version.  If not installed, the error message when trying to run vmprof is `ImportError: libunwind.so.8: cannot open shared object file: No such file or directory`)
6. Install KCachegrind (On Ubuntu, `apt-get install kcachegrind`)

There is a wrapper (`vmprofrun`) that automates the running and processing steps.
To use it, run `vmprofrun <target python script> [arguments to the python script]`. 
(No need to specify `python` - that gets added to the command line automatically.)
By default it will output `vmprof-<pid>.out`, which can be viewed in KCachegrind.

Underneath, the `vmprofrun` tool saves the vmprof output during the run to `out.vmprof`. After the run, it automatically copies the `/tmp/perf-<pid>.map` file to the current directory (if running under Numba).
It moves `out.vmprof` to `out-<pid>.vmprof`.
Finally it runs `vmproftocallgrind` using these files as input.

###Limitations

1. Only works on 64-bit Linux
2. Function-level profiles only - no line information (for either python or native code)
3. Sometimes the profiling hangs during the run - kill the process and try again.
4. Works with Python 2.7 or 3.4
5. Not well validated or tested yet
6. It does not work well yet with the existing vmprof web visualization and CLI tools.


Other notes:

* The stack dump tool will process the stacks to remove the Python interpreter frames.
* By default the Numba `Dispatcher_call` level is removed.  Otherwise the call graph in KCachegrind gets tangled by all the call paths running through this function.
* It should work with C extensions and Cython as well.

