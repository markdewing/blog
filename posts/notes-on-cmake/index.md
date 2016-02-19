<!-- 
.. title: Notes on CMake
.. slug: notes-on-cmake
.. date: 2016-02-19 5:05:00 UTC-06:00
.. tags: cmake, ctest
.. category: 
.. link: 
.. description: 
.. type: text
-->


Recently I started working on a project that uses CMake.  I've used CMake a little before, but never really
had to dive much into it.
In particular, I needed to understand the scripting parts of CMake for adding tests for CTest.

Below are some comments on aspects of CMake.

## Variables and variable substitution

Variables names are strings.  Substitution occurs when the variable is dereferenced with `${}`.

```cmake
SET(var, a)
MESSAGE("var = ${var}")
```
produces
```
var = a
```

Nested substitutions are possible
```cmake
SET(var, a)
SET(a, b)
MESSAGE("var = ${var}  ${${var}}")
```
will produce 
```var = a b```

Variable names can be composed during substitution
```cmake
SET(var, a)
SET(a_one, apple)
MESSAGE("var =  ${${var}_one}")
```
will produce ```var = apple```


## Variables and functions

Variable references act a little like pointers, but without a type system to enforce (and guide) how many indirections should be performed.

Example of using a variable inside a function:

```cmake
FUNCTION(MY_FUNC arg1)
    MESSAGE("arg1 = ${arg1}")
ENDFUNCTION()

MY_FUNC(hello)
SET(var, a)
MY_FUNC(var) # arg1 is set to 'var'
MY_FUNC(${var}) # arg1 is set to 'a' - this is usually what you want
```
produces
```
arg1 = var
arg1 = a
```


## Return values from functions
There is no built-in notion of a return value from a function.   To get values out of a function, write to one of the arguments.

A function creates a new scope - changes to a variable will only affect the variable's value 
inside the function.  To affect the value in the parent, the `PARENT_SCOPE` modifier should be given to the `SET` command.  (More on variable scopes [here](https://www.johnlamp.net/cmake-tutorial-5-functionally-improved-testing.html))

Another issue is the variable name for the output value needs to be dereferenced before being set.
Otherwise a variable with the name used in the function will be set in the parent, which can work by accident
if the variables have the same name.

Example:

```cmake
FUNCTION(MY_FUNC_WITH_RET ret)
    # The following line works by accident if the name of variable in the parent
    # is the same as in the function
    SET(ret "in function" PARENT_SCOPE)
    # This is the correct way to get the variable name passed to the function
    SET(${ret} "in function" PARENT_SCOPE)
ENDFUNCTION()

SET(ret "before function")
MY_FUNC_WITH_RET(ret)
MESSAGE("output from function = ${ret}")
```
 will produce `output from function = in function`



## Data structures
There is only the List type, with some functions for operations on lists.
The [documentation on lists](https://cmake.org/cmake/help/v3.3/manual/cmake-language.7.html#lists) says that "Lists ... should not be used for complex data processing tasks", but doesn't say what to use instead.

For associative arrays or maps there are some options:

* Two parallel lists - one of keys and one of values.  Search the key list and use the index to look up value.  More awkward for passing to functions.
* Single list with alternating keys and values.  Search the list for the key, and use index+1 to look up the value.  Only works if the range of possibilities for keys and values are distinct (e.g., keys are strings and values are always numbers).
* The environment (`ENV{key}`) is a built-in associative array.  It could be overloaded to store other values, at the risk of polluting the environment.

## Test timeout
The default timeout per test is 1500 seconds (25 minutes).

To increase this, adjust the value of `DART_TESTING_TIMEOUT`.
It needs to be set as a cache variable, and it needs to be set before the `enable_testing()` or `include(CTest)` is specified.
```cmake
SET( DART_TESTING_TIMEOUT 3600 CACHE STRING "Maximum time for one test")
```

See also this [Stack Overflow post](http://stackoverflow.com/questions/3545598/using-cmake-with-ctest-and-cdash)

 
