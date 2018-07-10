<!--
.. title: Bash for C programmers
.. slug: bash_for_c_programmers
.. date: 2018-07-10 16:41:01 UTC-05:00
.. tags: bash, scripting
.. category:
.. link:
.. description:
.. type: text
-->

# Bash scripting

Why use Bash?  It is not my preferred language, but it has two big advantages:

* Availability - we can safely assume bash is present everywhere.  No need to install additional software.
* Ease of starting and incremental building - a script can start as a small list of commands and grow incrementally.


The rest of this post not a tutorial, but some ways to think about bash scripting in order to avoid common traps when coming from C/C++ or Python.

## Quick Intro


### Whitespace is significant

Almost everything in bash is a command with arguments. After all the substitutions and quoting, the basic structure of each line is a command with all its arguments are separated by whitespace.  This makes whitespace significant in ways that can be confusing when coming from other languages.

One place this shows up is in setting variables - there should be no whitespace on either side of the equal sign.
The statement `a = 1` will fail with an error about 'a' not found - the shell interprets it as a command.
Similarly `a= 1` will fail with an error about '1' not found.

Another place significant whitespace appears is in `if` statements.
The opening bracket for the test (`[`) looks like syntax, but it is a command.  (By symmetry you might expect `]` to be a command, but it's not - it gets passed as an argument to `[`).
```bash
a=0
if [ $a -eq 0 ]; then
  echo "a is 0"
fi
```

Also note the semicolon after the comparison to terminate the command.  As an alternative, the `then` could be put on the next line
```bash
a=0
if [ $a -eq 0 ]
then
  echo "a is 0"
fi
```

Bash also has a double bracket form of testing that is a built-in, and not an external command.

### Exit values

Exit values from commands are 'success' or 'failure'.  Tests of the exit value, like the 'if' statement or the control operators (`&&` and `||`) operate on these exit values (and only exit values).
Numerically, success maps to 0, whereas failure maps to a non-zero value.
This numerical mapping is opposite of many other languages, like C/C++ and Python, where 0 is false and non-zero values are true.
Using multiple values for failure can give some indication of what went wrong with the command.

The best way to think about this is not to mentally reverse the values, but to think in terms of 'success' and 'failure', and only think about the numerical mapping if necessary.

Sequencing and control operator idioms:

* `A;B`  Run A and then B, regardless of success of A
* `A && B` Run B if A succeeded
* `A || B` Run B if A failed
* `A &` Run A in the background

( from <https://unix.stackexchange.com/questions/24684/confusing-use-of-and-operators> )


### Functions

Functions start with "`function` *name* `{`" or "*name*  `() {`".
The body is a series of commands, and the function ends with `}`.
Once again, whitespace is significant.  The function declaration should be on a line by itself,
and the closing brace should also be on a line by itself.

A function call looks like a normal command statement, with the usual space-separated arguments.

Inside a function, arguments are referenced using `$1`, `$2`, etc.  Unset variables are empty.

There is a syntax for optional arguments:
```bash
var=${1:-default_string_here}
```

### Variable replacement

Variable replacement occurs *before* bash breaks the line into a command and arguments based on whitespace.
Unexpected behavior can happen if a variable contains whitespace and the developer was not expecting it.
This can happen when the variable holds a directory name, and the script is run using a directory containing a space.

```bash
a="hi there"
cmd $a
```
expands to calling `cmd` with two arguments 'hi' and 'there'.
Enclose the variable in quotes to keep spaces confined.

This will call `cmd` with one argument: 'hi there'.
```bash
a="hi there"
cmd "$a"
```



For a gory step-by-step look at how bash processes input, see <https://mywiki.wooledge.org/BashParser>

### Robust bash scripts

See <https://www.davidpashley.com/articles/writing-robust-shell-scripts/> for advice on writing robust scripts.

Part of the advice is to use these settings

* `set -o nounset`  to detect the use of uninitialized variables
* `set -o errexit`  to exit the script if any statement fails


