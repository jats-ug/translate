# Scripting Patsolve

(元記事は http://www.illtyped.com/projects/patsolve/scripting.html です)

Given the breadth of SMT solvers' decision power, we naturally want to give the programmer direct access to the underlying solver in order to control its settings, assertion environment, or automatically produce SMT formulas that would be cumbersome to do just in the ATS statics.

We embedded a python interpreter into patsolve so that users can interact with the SMT solver directly. This allows users to provide interpretations of static functions or define their own macros like define-fun provides in SMT-LIB2. The purpose of defining macros would be to expand ATS static functions into SMT formulas that represent applying functions to their arguments.These extensions can be written in a stand alone python file and then given to patsolve during constraint solving.

## Macros

As an example, let's say we want to write a function that determines whether a 32 bit unsigned integer is a power of two. Using bitwise arithmetic, we can determine this fairly easily, but we want to verify that the computation does, in fact, determine if a number is a power of two for all possible inputs. Let's say we have an unsigned integer type indexed by a static bitvector of length 32.

```ats
abst@ype uint (b:bv32) = uint
```

Any unsigned integer of this type is restricted to hold the value of b, a bit vector in the ATS statics. Now, we need a predicate to determine if a bitvector is a power of two.

```ats
stacst power_of_two_bv32: (bv32) -> bool
stadef power_of_two = power_of_two_bv32 // overloading
```

By default, patsolve will turn this function into an uninterpreted function in the SMT solver. Using the scripting capability in patsolve, we can turn power_of_two into a macro that is true iff the static bit vector given to it is a power of two. In the Z3py library, an expression that captures this meaning is build by the following python function. Note, there is no need to import Z3py into the code, patsolve will do that automatically.

```ats
def power_of_two_bv32 (b):
  """
  A 32 bitvector x is a power of two.
  """
  return reduce(Or, (x == (1 << i) for i in range(32)))
```

When we pass a python file that defines this function to patsolve, it will replace the static function call with the expression returned by the python function. This is useful because it allows us to extend constraint solver's capabilities by simply writing some python code.

We can put these pieces together into the following function that determines whether an unsigned int is a power of two. Our constraint solver checks that the result we return is in fact equal to the straightforward definition we gave in the python file.

```ats
fun
power_of_two {x:bv32} (
  x: uint (x)
): bool (power_of_2 (x)) =
  if x = 0u then
    false
  else
    ((x land (x - 1u)) = 0u)
```

To check this, you would do the following.

```
patsopt --constraint-export -tc -d power.dats | patsolve -s power.py
```

The ATS2 compiler will extract the constraints and send them to patsolve. In this small function, the constraint is that the boolean value is equal to the boolean value of the static function call power_of_2 (x). Note, the dynamic type uint(x) that isgiven to the parameter x is a singleton type. This means the value of x is restricted to the value of the static bit vector x. The type we give to this function's return value is a singleton type as well whose value must is equal to the static function power_of_2(x).

## Interpreted Functions

Sometimes the functions we want in a domain cannot easily be expressed by using just a macro. Instead, we would like to provide an interpretation for a function to the SMT solver. This requires adding assertions to the internal SMT solver that patsolve uses. Since we embed python into our constraint solver, we provide a module called patsolve to python scripts so that users can add assertions with the familiar Z3 library.

Suppose we want to use a new static type in the ATS statics that represents an infinite stream of identifiers, which we call stamps. We could call this new sort stampseq with the following.

```ats
sortdef stamp = int

datasort stampseq = (* abstract *)
```

We will need some way to construct static sequences. Suppose we want the following operators.

```ats
stacst stampseq_nil : () -> stampseq                 // empty sequence
stacst stampseq_sing : (stamp) -> stampseq           // single sequence
stacst stampseq_cons : (stamp, stampseq) -> stampseq // add to front
stacst stampseq_head : stampseq -> stamp             // get first stamp
stacst stampseq_tail : stampseq -> stampseq          // skip first stamp
```

Using these constructors, we are free to index ATS types with stampseq, but the constraints generated with these functions will always be invalid for anything but the simplest expression. The reason is because we have not given the SMT solver any interpretation of these functions. To do this, we could add some assertions about their behavior to the constraint solver using the following python code. Suppose we represent a stamp sequence as an array of integers mapping to integers in the underlying SMT solver.

```python
import patsolve

# Get our solver
s = patsolve.solver

A = Array("A", IntSort(), IntSort())
i, j, x = Int("i"), Int("j"), Int("x")

StampSeqSort = lambda : ArraySort (IntSort(), IntSort())

# Undefined Section of an Array

s.add (
    ForAll ([A, i], Implies (i < 0, A[i] == 0))
)

# The "nil" sequence

nil = Function ('stampseq_nil', StampSeqSort())

s.add (
    ForAll (i, nil()[i] == 0)
)

# Sing

sing = Function ('stampseq_sing', IntSort(), StampSeqSort())

s.add (
    ForAll ([x, i], Select (sing(x), i) == Select(Store (nil(), 0, x), i))
)
```

## Alternative Ideas

The idea of exposing just a python interface doesn't entirely sit well with me. I like the Z3Py library because the Z3 authors obviously spent a lot of time trying to make it very easy to use. At the same time, it takes a lot of work to interact directly with the library through the Python interpreter in the constraint solver.

What if someone does not want to use Z3? Do they need python bindings for their own SMT solver? Instead, why can't I provide an interface for a user to specify programs (or equivalently scripts) that receive arguments as strings on the commandline and then produce SMT-LIB2 code that represents evaluating the given macro? Alternatively, there could be programs that generate interpretations for individual functions. This may seem odd, but it seems like a nice way to appeal to any SMT solver, and give the user the most flexibility for extending the constraint solver in the way he or she feels fit.
