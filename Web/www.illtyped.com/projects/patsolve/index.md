# patsolve

The patsolve constraint solver is the product of an effort to put constraint solving outside of the ATS compiler so that we can leverage off the shelf reasoning tools. By doing so, we hope to make constraint solving more reliable and more productive. Using actively developed tools like SMT solvers gives us more confidence in a constraint solvers' answers, and their wide decision power allows us to automate the verification of more properties in ATS programs. In its current implementation, patsolve uses the Z3 SMT solver in order to prove the validity of constraints generated from ATS programs.

For details on how to use patsolve, please follow the installation instructions below and check out the examples given in the case studies.

## Advantages

If you haven't worked with an SMT solver before, you can still effectively use patsolve as a drop in replacement for the built-in constraint solver. Some reasons why you would want to do this include:

### Stability

The internal ATS2 constraint solver uses Fourier-Motzkin Elimination to solve constraints. While this works well, the code is all custom made and difficult to modify. In contrast, SMT solvers are actively developed tools that are rigorously tested for correct implementation. Trusting an SMT solver means we do not need to worry about implementing and, more importantly, debugging our own decision procedures.

### Decision Power

Even if you opt to not use more advanced domains such as bitvectors or arrays in the statics, SMT solvers can sometimes lighten the amount of work you must put in to prove invariants. This is an anecdotal claim. Nonetheless, SMT solvers' continual improvement directly empower the ATS statics.

## Installation

If you'd like to run this work locally, install the latest version of ATS and the related contrib package. You will need to install the following other dependencies:

* [Z3](http://z3.codeplex.com/) (4.3.2)
* python 2.7.x (my apologies, python 3 is not supported by Z3py)

Next, go to the following folder in the contrib packages.

```
projects/MEDIUM/ATS-extsolve
```

and run

```
make
```

As long as you set up the PATSHOME environment variable, the patsolve binary will be placed in your path. If not, you will need to issue the following command (possibly as root dependong on where ATS is installed) in order to install patsolve.

```
cp patsolve $PATSHOME/bin/
```

Now, you should be all set to type check the examples. We provide a python scripting interface that lets you interact with Z3 directly. This allows you to provide interpretations to functions and define SMT macros using python. Using this tool, you will need to pass the python script as a command line argument.

The following command shows how to do this with the quicksort example we present in a case study.

```
cd $PATSHOME/contrib/libats-/wdblair/prelude/
patsopt --constraint-export -tc -d TEST/quicksort.dats | patsolve -s SMT/stampseq.py
```

If all goes well, you will see a message saying the file has typechecked. If you want to see an SMT-LIB 2 trace of constraint solving, add a -v switch to patsolve and you will see the entire transcript between the constraint solver and Z3 in something resembling SMT-LIB 2.

## Usage

In order to use patsolve as a replacement for the built-in constraint solver, you would normally typecheck a program by doing the following.

patsopt --constraint-export -tc -d foo.dats | patsolve

Once all the unsolved constraints have been found, you can produce C code by calling patsopt again, but skipping constraint solving.

patsopt --constraint-ignore -d foo.dats

## Case Studies and Tutorials

### [Scripting patsolve](http://www.illtyped.com/projects/patsolve/scripting.html)

A short tutorial that goes over how to extend the functionality of patsolve using the Z3Py library.

### [Verified Efficient Programs in ATS: qsort](http://www.illtyped.com/projects/patsolve/qsort.html)

A method of implementing the qsort function from the C standard library using pointer arithmetic while also verifying its memory safety and functional correctness.

## Future Work

There is no special reason why Z3 must be the underlying SMT solver for patsolve. Given that we use it essentially as an automated theorem prover, we would like to explore using interactive theorem provers like Coq or Isabelle for constraint solving. We imagine that these provers would be useful for complicated domains that SMT solvers are not well suited for, such as proving properties of recursive datastructures. One can imagine a constraint solver consisting of layers to handle constraints of varying difficulty. At the top layer we could have Z3 which can prove the constraints we currently find in ATS programs. At the next layer a system like Coq could be used to prove constraints involving induction where datatypes such as lists or trees are involved.

In the meantime, we would like to add support for using CVC4, and possibly replacing the python interface with a generic file based interface instead.
