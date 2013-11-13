# Overview

The Options package provides macros that allow the simulation of optional
keyword arguments for Julia functions. Some of this functionality was
superseded by language support for keyword arguments ([see
\#485](https://github.com/JuliaLang/julia/issues/485)). However, the Options
package will likely remain relevant for cases where you need to pass optional
arguments to nested functions without the parent functions needing to be aware
of those arguments.

# Installation

Install via the Julia package manager, `Pkg.add("Options")`.

# Basic usage

You gain access to `Options` via

```
require("Options")
using OptionsMod
```

The `@defaults` macro is for writing functions that take optional parameters.
The typical syntax of such functions is:

      function myfunc(requiredarg1, requiredarg2, ..., opts::Options)
          @defaults opts a=11 b=2a+1 c=a*b d=100
          # The function body. Use a, b, c, and d just as you would
          # any other variable. For example,
          k = a + b
          # You can pass opts down to subfunctions, which might supply
          # additional defaults for other variables aa, bb, etc.
          y = subfun(k, opts)
          # Terminate your function with check_used, then return values
          @check_used opts
          return y
      end

Note the function calls `@check_used` at the end; this will be discussed below.

It is possible to have more than one `Options` parameter to a function, for
example:

      function twinopts(x, plotopts::Options, calcopts::Options)
          @defaults plotopts linewidth=1
          @defaults calcopts n_iter=100
          # Do stuff
          @check_used plotopts
          @check_used calcopts
      end

Within a given scope, you should have only one call to `@defaults` per options
variable.


The `@options` macro is typically the easiest way to _set_ the value of optional
parameters for a function that has been written to use them:

     opts = @options a=5 b=7

For a function that uses optional parameters `a` and `b`, this will override the
default settings for these parameters. You would likely call that function in
the following way:

     myfunc(requiredarg1, requiredarg2, ..., opts)

# Argument checking

For functions that take many optional arguments, one fairly common problem is
that the relevant options may change over time, but users may not notice the
change and continue providing parameters that are no longer used by the
function. A similar scenario arises when users simply misspell an optional
parameter. In such cases the user may be quite confused about why changing the
value of those parameters has no impact on the output of the function.

For this reason, the default expected behavior is for functions to check that
all optional parameters have been used by the function. This is achieved with
the `@check_used` macro. The test is performed at the end of the function body,
so that subfunctions which handle parameters not used by the parent function may
be "credited" for their usage. Each sub-function should also call `@check_used`,
for example:

      function complexfun(x, opts::Options)
          @defaults opts parent=3 both=7
          println(parent)
          println(both)
          subfun1(x, opts)
          subfun2(x, opts)
          @check_used opts
      end
      
      function subfun1(x, opts::Options)
          @defaults opts sub1="sub1 default" both=0
          println(sub1)
          println(both)
          @check_used opts
      end
      
      function subfun2(x, opts::Options)
          @defaults opts sub2="sub2 default" both=22
          println(sub2)
          println(both)
          @check_used opts
      end

Typically, unused parameters will result in an error. The user can control the
behavior this way:

     # throw an error if a or b is not used (the default)
     opts = @options CheckError a=5 b=2
     # issue a warning if a or b is not used
     opts = @options CheckWarn a=5 b=2
     # don't check whether a and b are used
     opts = @options CheckNone a=5 b=2

## Additional functionality

As an alternative to the macro syntax, you can also say:

     opts = Options(CheckWarn, :a, 5, :b, 2)

The check flag is optional.

The `@set_options` macro lets you add new parameters to an existing options
structure.  For example:

      @set_options opts d=99

would add `d` to the set of parameters in `opts`, or re-set its value if it was
already supplied.

### Implementation details

The fields of the `Options` type are

- `key2index`: a `Dict` that looks up an integer index, given the symbol for a
variable (e.g., `key2index[:a]` for the variable `a`)
- `vals`: `vals[key2index[:a]]` is the value to be assigned to the variable `a`
- `used`: A vector of booleans, one per variable, with `used[key2index[:a]]`
representing whether the value for variable `a` has been accessed by the
`@defaults` macro
- `check_lock`: A vector of booleans, one per variable. This is a "lock" that
prevents sub-functions from complaining that they did not access variables that
were intended for the parent function. `@defaults` sets the lock to true for any
options variables that have already been defined; new variables added through
`@set_options` will start with their `check_lock` set to `false`, to be handled
by a subfunction.

## Credits

The primary authors are Harlan Harris, Tim Holy, and Carlo Baldassi, with
contributions from Stefan Karpinski, Patrick O'Leary, John Myles White, and Jeff
Bezanson.
