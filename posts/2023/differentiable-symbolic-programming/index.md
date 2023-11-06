<!--
.. title: Differentiable Symbolic Programming
.. slug: differentiable-symbolic-programming
.. date: 2023-06-19 00:20:57 UTC-05:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->


Automatic diffentiation is useful, but runs into some limitations.

From an aesthetic point of program organization, I would like a format
the uses symbolic mathematics (or something very close to it) to represent
the statements.
Differentation is then performed on those statements, then it is code-generated
into the target language.


From a practical point of view, I want this because autodifferentiation tools
are not always available for the target platform and language.
And existing tools don't always work the way I want (or at least I can't figure
 it out)

One of the driving test cases involves second derivatives (just the Laplacian,
 don't need the full Hessian), and first derivative against a different
 variable.


Really, this is automatic differentiation applied between statements and
 symbolic differentiation applied within a statement.

A lot of the issues are bookkeeping.

Statements may contain function calls.
The derivative version of those function calls need to be modified to return additional derivatives.


The driving example is the variational calculation of the energy of a helium atom.
(Insert link)
It is an integral over 2 3D electron coordinates, resulting a 6 dimensional
 integral.   For this post I'm interested in generaing the integrand,
 and leave the integral evaluation elsewhere.

I make progress, and then stall out, drop the project, and then have to
pick it up when I come back to it.
I'm hoping being more public about it might force me to push development
further.

