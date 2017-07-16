Setup.shl: A simple Bash library to replace Makefiles
=================================================

`make` (and similar dependency-chasing tools, such as SCons, Rake,
Waf, Ant, Maven, Gradle et al, which I will refer to as `make`-like
tools from now on) offer very useful primitives for building complex
file hierarchies from a small set of source files, while minimizing
the amount of work needed to rebuild after a small change.

Setup.shl tries to offer the same basic set of features, as a (mostly
pure) Bash library. It supports parallel compilation, and nested
builds, as well as a new kind of dependency, all in 20kB of Bash
code. Other than that, it tries to be as easy to use as possible.

Installing and using Setup.shl
--------------------------

Since it's basically just a Bash script, you can start using Setup.shl
from your shell by simply setting the `SETUP_INSTALL_DIR` variable to
the root of this project, and sourcing the
[lib/setup.shl](lib/setup.shl) file (if you are using Bash, that is).

This file defines two functions, `prepare` and `setup` (and a third,
`teardown`, if you want to perform continuous builds), whose jobs are
to respectively prepare computations and run them.

This repository also provides a [bin/setup](bin/setup) executable that
does something similar to the `make` tool : it searches a file named
`Setup` in the current directory or its parents, and runs that file in
an environment where the setup library was already sourced. It can
also optionally, using the `--watch` option,  all source files
and trigger a new build every time they are written to.

This project provides a Setup file to illustrate its own
usage. Installing Setup.shl can be done by running
[bin/setup](bin/setup) and installing the resulting archive (called
`.pkg.tar.gz` because I like my generated files to be hidden) to the
root of your filesystem :

    bin/setup package
    sudo tar -xvzf .pkg.tar.gz -C /

### The `prepare` function

Usage: **prepare FILE... = COMMAND (-OPT|@FILE|FILE)...**

This function declares that the FILEs on the left of the `=` are the
result of the application of the COMMAND to its arguments. It does
*not* itself run the command.

An argument can take three shapes :

  - starting with a `-` indicates a *flag argument*, which doesn't
    need to describe a file
  - starting with a `@` indicates a *splice dependency*, which is
    described in more detail [here](#splice-dependencies)
  - otherwise, it is a simple dependency that triggers the command
    when it becomes newer than any of the FILEs

### The `setup` function

Usage: **setup TARGET...**

This function computes all the files that have been prepared before
it, in addition to the TARGETs passed as  arguments. 

Like `make`, `setup` uses timestamps to avoid wastefully recompiling
when a file is already more recent than its dependencies.

### The `teardown` function

Usage: **teardown FILE...**

This function makes the build system treat the FILEs as though they
had just been modified. Subsequent calls to `setup` will rebuild every
intermediate target that depends on those FILEs (although the FILEs
themselves are considered up-to-date and won't be rebuilt).

### Other useful functions and variables

Alongside those two main functions, Setup.shl also allows minor
configurations to take place, by using the following functions and
variables :

  - the `SETUP_JOBS` variable can be set at any point in the script to
    the desired number of parallel jobs to run (default: 8) on the
    next call to `setup`

  - the `Setup.params PARAM...` function prints the value of the first
    PARAM that was specified on the command-line, if it exists. If
    that PARAM starts with `-`, it is not printed out. The function
    exits non-zero if none of the PARAMs were specified on the
    command-line. As such it can be used to define build flags, like
    so :
    
        if Setup.params -install; then
           ...
        fi
        
    Coupled with an assignment, it can also retrieve a parameter's
    value in a variable :

        if target="$(Setup.params target)"; then
            # The 'target' param was specified, and is now in the 'target' variable
            ...
        fi

    These flag can then be triggered by running the commands `setup
    install` or `setup target=TARGET` (or `setup install
    target=TARGET` for both).

  - the `Setup.use MODULE...` function loads modules from the
    `$SETUP_INSTALL_DIR/lib/setup.d/MODULE.shl` files. It's basically
    a wrapper around `source`.

    After loading a module, it also tries to sources a local module
    file from `.setup/lib/MODULE.shl` if it exists, allowing you to
    override some module-specific functions on a per-project basis.

  - the `Setup.load SETUP_FILE PARAM...` function loads another Setup
    file in the current context, in order to use its preparations as
    dependencies further down the line.

    Each PARAM can be what you would specify on the command-line for a
    standalone invocation, such as `install` or `target=X` (see the
    `Setup.params` function for more information), and is made
    available alongside the PARAMs of the current context, overriding
    them when they already exist.
    
  - the `Setup.hook FUNCTION...` can be used to define new automatic
    dependency generators.

    A generator is a function which take a single file name as
    argument, and should prepare this file if it knows how. Otherwise,
    it should return non-zero to signal to try the next generator.

  - the `Setup.state-file NAME` function specifies an optional state
    file, which is used by Setup.shl to remember information from one
    build to the next and speed up dependency resolution upon
    successive invocations of `setup`.

    This file is created automatically by Setup.shl and can safely be
    deleted if it causes problems (it shouldn't but there are always
    exceptions).

    Warning: the `prepare` function doesn't do anything if a file is
    already known to the build system. When using a state file, that
    means that once a file is specified, you can no longer change the
    command it is associated with, or its arguments (splice arguments
    are still recomputed correctly, don't worry), even in subsequent
    builds. If that happens, simply delete the state file, and restart
    the build to acknowledge the new dependency graph.

### Automatic targets via dependency hooks

Other build tools usually provide a sort of "wildcard target" to avoid
repetition, as in :

    %.o: %.c
         ...

Setup.shl is no exception. You can declare your own callbacks to
prepare files whenever they are needed, by using the `Setup.hook`
function. For example, the following code would be equivalent to the
above rule :

    Setup.hook C.auto_o
    function C.auto_o() {
        case "$1" in
            *.o) prepare "$1" = CC "${1/%.o/.c}";;
            *) return 1;;
        esac
    }

It doesn't look as pretty, but it is a much more powerful way to
describe automatic dependencies, as it allows the full power of Bash
to be brought forth to take advantage of contextual information.

Although, for simple cases like the above, there is a `prepare-match`
function that can be used like so :

    prepare-match '^(.*)\.o$' = CC '$1.c' @'$1.includes'

The first parameter is a regex, which is used to match the file name,
and every parameter after the equal sign can use the regex matches as
positional parameters (quoted, because they are not in scope when we
define the rule).

For more example of automatic dependencies, visit the
[lib/setup.d](lib/setup.d) directory.

What Setup can do
-----------------

<a name="splice-dependencies"></a>
### Splice dependencies for a more expressive build process

The complexities of some build processes are not accurately captured
by the dependency model of Make-like tools, specifically
automatically-generated dependencies. As an example, consider the
following example :

file: test.c

    #include "test.h"

    int main() { printf("%d\n",f()); }

file: test.h

    #include "A.h"

    A_type f(void);

file: A.h

    typedef int A_type;

file: Makefile

    %.o: %.c ???
        gcc -c $< -o $@

In this example, Make offers no simple way for `test.o` to know that
it depends on `A.h`, or even `test.h` if `test.c` doesn't exist a
priori. We would like to be able to express the dependency as "`%.o`
depends on `%.c` and _all the includes of `%.c`_', but our tools don't
allow us to express that last part because the includes of %.c are
generated, and not known when the Makefile is read.

This inability to use the content of generated files within the
dependency graph leads to a lot of silliness down the line. For
instance, many compilers/interpreters now offer a `-MM` option, used
to generate a complete dependency graph in a Makefile format, which
can then be included by the main Makefile.

Using Setup.shl, these same dependencies can be simply expressed as : 

    prepare-match '^(.*)\.includes$' = Includes '$1.c'
    prepare-match '^(.*)\.o$' = CC '$1.c' @'$1.includes'

The `@` in front of the last dependency denotes a *splice dependency*,
which indicates that, in order to compute `"$file.o"`, we first need
to compute a list of include names in `"$file.includes"`, then splice
that list and use each element as a dependency. 

### Nested builds

Suppose you have two projects, A and B, such that A needs a part of B
to do its job (imagine that B can produce a certain library, which A
needs). B doesn't need A to function, so it provides its own
independent build script as a way to define its setup. Ideally, A
would be able to load B's build script, and add its own rules before
running a full-blown build operation with all the information it
needs, without any changes to B whatsoever.

Setup.shl allows this workflow by providing a function, `Setup.load`,
which -- as the name implies -- loads a nested Setup script from a
parent context in order to use its productions later on.

As a motivating example, consider a two-phase build of a C program :
one phase builds the binaries and libraries, and the second phase
builds a distribution package from those artifacts. 

file: C_proj/Setup

    #!/usr/bin/setup -f
    Setup.use C
    prepare main = C.ld main.o

file: Setup

    #!/usr/bin/setup -f
    Setup.use Pkg
    Setup.load C_proj/Setup
    
    Pkg.package Pkg.files usr/bin/main=C_proj/main

From this setup, any update to a C source file in `C_proj` will be
detected by the packaging script, and cause the necesary compilation
steps to be taken before rebuilding the affected packages.

Note the shebang lines at the beginning of both Setups. In cases like
the above, where the Setup script is called from its project
directory, the shebangs are not necesary. In fact, they are never
necesary, but they can be useful if you want to achieve maximum
comfort when building your project. For example, if you find yourself
repeatedly needing to run `setup -f PROJDIR/Setup`, you can easily
make your Setup file executable and create a symbolic link to it
somewhere in your PATH (in `$HOME/.bin` for instance), like so :

    chmod +x PROJDIR/Setup
    ln -s PROJDIR/Setup "$HOME/.bin/my-setup" 

Now, you can run a simple `my-setup` (`my-setup install` or `my-setup
--watch` or anything you would have run with `setup`) to build your
project. Setup.shl will follow the symlink and anchor the build to its
rightful directory, as if you had been there all along.
