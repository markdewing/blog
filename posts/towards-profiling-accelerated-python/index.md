<!-- 
.. title: Towards Profiling Accelerated Python
.. slug: towards-profiling-accelerated-python
.. date: 2015-10-07 20:58:00 UTC-05:00
.. tags: vmprof, PyPy, Numba, python
.. category: 
.. link: 
.. description: 
.. type: text
-->


One of the conclusions from last post is a need for better profiling tools to show where time is spent in the code.
Profiling Python + JIT'ed code requires dealing with a couple of issues.

The first issue is collecting stack information at different language levels.
A native profiler collects a stack for the JIT'ed (or compiled extension) code, but eventually the stack enters the implementation of the Python interpreter loop.
Unless we are trying to optimized the interpreter loop, this is not useful.
We would rather know what Python code is being executed.
Python profilers can collect the stack at the Python level, but can't collect native code stacks.

The PyPy developers created a solution in [vmprof](https://vmprof.readthedocs.org/en/latest/).
It walks the stack like a native profiler, but also hooks the Python interpreter
so that it can collect the Python code's file, function, and line number.
This solution is general to any type of compiled extension (C extensions, Cython, Numba, etc.)
Read the section in the vmprof docs on [Why a new profiler?](https://vmprof.readthedocs.org/en/latest/#why-a-new-profiler) for more information.

The second issue is particular to JIT'ed code - resolving symbol information after the run.
For low overhead, native profilers collect a minimum of information at runtime (usually the Instruction Pointer (IP) address at each stack level).
These IP addresses need to resolved to symbol information after collection.
Normally this information is kept in debug sections that are generated at compile time.
However, with JIT compilation, the functions and their address mappings are generated at runtime.

LLVM includes an interface to get symbol information at runtime.
The simplest way to keep it for use after the run is to follow the Linux perf standard (documented [here](https://github.com/torvalds/linux/blob/master/tools/perf/Documentation/jit-interface.txt)), which stores the address, size, and function name in a file `/tmp/perf-<pid>.map`.


To enable Numba with vmprof, I've created a version of llvmlite that is amenable to stack collection, at the *perf* branch [here](https://github.com/markdewing/llvmlite/tree/perf).

This does two things:

1. Keep the frame pointer in JIT'ed code, so a backtrace can be taken.[^1]
2. Output a perf-compatible JIT map file (not on by default - need to call `enable_jit_events` to turn it on)

[^1]: With x86_64, it is possible to use DWARF debug information to walk the stack.  I couldn't figure out how to output the appropriate debug information.  LLVM 3.6 has a promising target option named `JITEmitDebugInfo`.  However, `JITEmitDebugInfo` is a lie!  It's not hooked up to anything, and has been removed in LLVM 3.7.


To use this, modify Numba to enable JIT events and frame pointers:

1. In  `targets\codegen.py`, at the end of the `_init` method of `BaseCPUCodegen`, add `self._engine.enable_jit_events()`
2. And for good measure, turn on frame pointers for Numba code as well (set `CFLAGS=-fno-omit-frame-pointer` before building it)



The next piece is a modified version of vmprof ( in branch [*numba*](https://github.com/markdewing/vmprof-python/tree/numba) ).
So far all it does is read the perf compatible output and dump raw stacks.
Filtering and aggregating Numba stacks remains to be done (meaning neither the CLI nor the GUI display work yet).


How to use what works, so far:

1. Run vmprof, using perf-enabled Numba above:  `python -m vmprof -o vmprof.out <target python>`
2. Copy map file `/tmp/perf-<pid>.map` to some directory.   I usually copy `vmprof.out` to something like `vmprof-<pid>.out` to remember which files correlate.
3. View raw stacks with `vmprofdump vmprof-<pid>.out --perf perf-<pid>.map`.  






