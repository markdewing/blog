<!-- 
.. title: Why Types Help Performance
.. slug: why-types-help-performance
.. date: 2015-09-24 10:48:00 UTC-05:00
.. tags: python, cython, Numba, julia
.. category: 
.. link: 
.. description: 
.. type: text
-->

In [previous](http://markdewing.github.io/blog/posts/first-performance-improvements) [posts](http://markdewing.github.io/blog/posts/comparing-languages-with-miniapps/), we've seen that adding type information can help the performance of the code generated by dynamic language compilers.
The documentation for Cython annotations talks of 'Python interaction', and Numba has 'object' mode and 'nopython' modes.
In post I will look more at what these mean, and how they affect performance.

To start, consider how values are represented in a computer, such as a simple integer ('int' type in C).
The bare value takes 4 bytes in memory, and no additional information about it is stored,
such as its type or how much space it occupies.
This information is all implicit at run time [^1].
That it takes 4-bytes and is interpreted as an integer is determined at compile time.

[^1]: The type can be accessible at run time via debug information.  See this Strange Loop 2014 talk: [Liberating the lurking Smalltalk in Unix](https://www.youtube.com/watch?v=LwicN2u6Dro)

In dynamic languages, this extra information can be queried at run-time.
For example:
```python
>>> a = 10
>>> type(a)
<type 'int'>
```

The value stored in `a` has type `int`, Python's integer type.
This extra information must be stored somewhere, and languages often solve this by wrapping
the bare value in an object.
This is usually called 'boxing'.
The value plus type information (and any other information) is called a 'boxed type'.
The bare value is called an 'unboxed type' or a 'primitive value'.


In Java, different types can be explicitly created (`int` vs. `Integer`), and the programmer
needs to know the differences and tradeoffs.
(See this [Stack Overflow question](http://stackoverflow.com/questions/13055/what-is-boxing-and-unboxing-and-what-are-the-trade-offs) for more about boxed and primitive types.)

Python only has boxed values ('everything is an object'). From the interpreter, this means we can
always determine the type of a value.
If we look a layer down, as would be needed to integrate with C, these values are accessed through the Python API.
The base type of any object is PyObject.  For our simple example, integers are stored as PyIntObject.

For example, consider the following Python code.
```python
  def add():
    a = 1
    b = 1
    return a + b
```

One way to see what calls the interpreter would make is to compile with Cython.
The following C is the result (simplified - reference counting pieces to the Python API are omitted.)

```c
PyObject *add()
{
  PyObject *pyx_int_1 = PyInt_FromLong(1);
  PyObject *pyx_int_2 = PyInt_FromLong(2);
  PyObject *pyx_tmp = PyNumber_Add(pyx_int_1, pyx_int_2);
  return pyx_tmp;
}
```

Without type information, Cython basically unrolls the interpreter loop, and makes
a sequence of Python API calls.
The HTML annotation output highlights lines with Python interaction, and can be expanded to show
the generated code.   This gives feedback on where and what types need to be added to avoid
the Python API calls.

Add some Cython annotations and the example becomes
```python
 cdef int add():
    cdef int a
    cdef int b
    a = 1
    b = 1
    return a + b
```

Now the following code is generated
```c
 int add()
 {
  int a = 1;
  int b = 2;
  return a+b;
 }
```

The add of the two integers is done directly (and runs much faster), rather than going through the Python API call.


##Numba
If its type information is insufficient, Numba will call the Python API for every operation.
Since all the operations occur on Python objects, this is called 'object' mode (slow).
With sufficient type information, code can be generated with no calls to the Python API, and hence
the name 'nopython' mode (fast).


##Julia
Julia has boxed object types, but is designed to try use the unboxed types as much as possible.
The most generic type is called 'Any', and it is possible to produce Julia code that runs this mode.  
See the section on [Embedding Julia](http://julia.readthedocs.org/en/latest/manual/embedding/) in the
documentation for more about Julia objects.

Julia's type inference only happens inside functions.
This is why composite types (structures) need type annotations for good performance.

This example demonstrates the issue
```julia
type Example
    val
end

function add(a, b)
    return a.val + b.val
end

v1 = Example(1)
v2 = Example(2)

@code_llvm add(v1, v2)  # The @code_llvm macro prints the LLVM IR.  
t = add(v1, v2)
println("res = ",t)
```

Since the type of the 'val' element is not known, the code operates on a generic object type `jl_value_t` and eventually calls `jl_apply_generic`, which looks up the right method and dispatches to it at execution time.
(The LLVM IR is not shown here - run the example to see it.)  Doing all this at execution time is slow.



Now add the type annotation
```julia
type Example
    val::Int
end
```
The resulting LLVM IR (also not shown here) is much shorter in that it adds two integers directly and
returns the result as an integer.
With type information, the lookup and dispatch decisions can be made at compile time.

Note that Julia uses a Just In Time (JIT) compiler, which means compilation occurs at run time.
The run time can be split into various phases, which include compilation and execution of
the resulting code.

##Summary

Hopefully this post sheds some light on how type information can affect the performance of dynamic languages.




