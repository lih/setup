Setup.shl: A simple Bash library to replace Makefiles
=================================================

`make` (and similar dependency-chasing tools) offer very useful
primitives for building complex file hierarchies from a small set of
source files, while minimizing the amount of work needed to rebuild
after a small change.

Setup.shl tries to offer the same basic set of features, in a Bash
environment. It supports parallel compilation, and transitive
dependencies. Other than that, it tries to be as easy to learn as
possible.

Installing and using Setup.shl
--------------------------

Since it's basically just a Bash script, you can start using Setup.shl
from your shell by simply setting the `SETUP_INSTALL_DIR` variable to
the root of this project, and sourcing the
[lib/setup.shl](lib/setup.shl) file (if you are using Bash, that is).

This file defines two functions, `prepare` and `setup`, whose job it
is to respectively prepare computations and run them.

There is also a [bin/setup](bin/setup) executable that performs a job
similar to the `make` tool : it searches a file named `Setup` in the
current directory or its parents, and runs that file in an environment
where the setup library was already sourced.

This project provides a Setup file to illustrate its own
usage. Installing Setup.shl can be done by running
[bin/setup](bin/setup) and installing the resulting archive (called
`.dist.tar.gz`) to the root of your filesystem :

    SETUP_INSTALL_DIR="$PWD" bin/setup package
    sudo tar -xzf .dist.tar.gz -C /

### The `prepare` function

Usage: **prepare FILE... = COMMAND (-OPT|@FILE|FILE)...**

This function declares that the FILEs on the left of the `=` are the
result of the application of the COMMAND to its arguments. It does
*not* itself run the command.

An argument can take three shapes :

  - starting with a `-` indicates a *flag argument*, which doesn't need to describe a file
  - starting with a `@` indicates a *splice dependency*, which is described in more detail [here](#why-not-use-make)
  - otherwise, it is a simple dependency

### The `setup` function

Usage: **setup FILE...**

This function computes all the files that have been prepared before
it, in addition to the FILEs passed as  arguments. 

### Automatic targets

Other build tools usually provide a sort of "wildcard target" to avoid
repetition, as in :

    %.o: %.c
    	 ...

Setup.shl is no exception, even if it doesn't provide nice syntactic sugar
for those rules. Instead, you can declare your own callbacks to prepare
files at the last minute. For example, the following code would
be equivalent to the above rule :

    function C.auto_o() {
        case "$1" in
            *.o) prepare "$1" = GCC-c "${1/%.o/.c}";;
            *) return 1;;
        esac
    }
    Setup.hook C.auto_o

It doesn't look as pretty, but it is a much more powerful way to
describe automatic dependencies, as it allows the full power of Bash
to be brought forth to take advantage of contextual information.

For more example of automatic dependencies, visit the
[lib/setup.d](lib/setup.d) directory.

Why not use Make ?
------------------

### Splice dependencies for a more expressive build process

The complexities of some build processes are not accurately captured
by the dependency model of Make-like tools. As an example, consider
the following example :

file: test.c

    #include "test.h"

    int main() { printf("%d\n",f()); }

file: test.h

    #include "A.h"

    A_type f(void);

file: A.h

    typedef int A_type;

file: Makefile

    test.o: test.c ???
    	gcc -c $< -o $@

In this example, Make offers no simple way for `test.o` to know that
it depends on `A.h`, or even `test.h` if `test.c` doesn't exist a
priori. We would like to be able to express the dependency as
"`test.o` depends on `test.c` and _all the includes of `test.c`_', but
our tools don't allow us to express that last part.

Using Setup.shl, these dependencies can be simply expressed as : 

    prepare "$file.includes" = Includes "$file.c"
    prepare "$file.o" = GCC-c "$file.c" @"$file.includes"

The `@` in front of the last dependency denotes a *splice dependency*,
which indicates that, in order to compute `"$file.o"`, we first need
to compute a list of include names in `"$file.includes"`, then splice
that list and use each element as a dependency. 

### Composability

Suppose you have two projects, A and B, such that A needs a part of B
to do its job (imagine that B can produce a certain library, which A
needs). B doesn't need A to function, so it provides its own
independent build script as a way to define its setup. Ideally, A
would be able to load B's build script, and add its own rules before
running a full-blown build operation with all the information it
needs, without any changes to B whatsoever.

Traditional build tools don't usually integrate well with such nested
projects, instead resorting to separate sub-processes running their
own dependency resolution, and coarse-grained parallelization. 

To solve this problem, Setup.shl provides a function, `Setup.load`,
which -- as the name implies -- loads a nested Setup script inside a
containing context in order to use its productions later on.

