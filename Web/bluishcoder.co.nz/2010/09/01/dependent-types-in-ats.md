# BLUISH CODER: ATSにおける依存型

(元記事は http://bluishcoder.co.nz/2010/09/01/dependent-types-in-ats.html です)

Dependent types are types that depend on the values of expressions.
[ATS](http://www.ats-lang.org/) uses dependent types and in this post I hope to go through some basic usage that I’ve learnt as I worked my way through the documentation, examples and various papers.

While learning about dependent types in ATS I used the following resources:

* [Tutorial on ATS datatypes](http://www.ats-lang.org/htdocs-old/TUTORIAL/contents/datatypes.html)
* [Tutorial on Parametric Polymorphism and Templates](http://www.ats-lang.org/htdocs-old/TUTORIAL/contents/templates.html)
* [Dependently Typed Datastructures](http://www.ats-lang.org/PAPER/DTDS-waaapl99.pdf)
* [Eliminating Array Bound Checking Through Dependent Types](http://www.cs.bu.edu/~hwxi/academic/papers/pldi98.ps)

Most of the examples that follow are based on examples in those resources.

## Sorts and Types

Some dependently typed languages allow types to depend on values expressed in the language itself. ATS provides a restricted form of dependent types. The expressions that types can depend on are in a restricted constraint language rather than the full language of ATS. This constraint language is itself typed and to prevent confusion they call the types in that language ‘sorts’. References to ‘sort’ in this post refer to types in that constraint language. References to ‘type’ refers to types in the core ATS language itself.

Here is an example of the difference. The following is an example of a function that takes two numbers and returns the result of them added together. This is in a hypothetical dependently typed language where types can depend on language values itself:

```ocaml
fun add(a: int, b: int): int (a+b) = a + b
```

Here the result type is the exact integer type of the two numbers added together. A mistake in the body of the code that resulted in anything but the sum of the two numbers would be a type error. In ATS this function would look like:

```ocaml
fun add {m,n:int} (a: int m, b: int n): int (m+n) = a + b
```

Notice here the introduction of {m,n:int}. This is the ‘constraint language’ used for values that the types in ATS can depend on. Here we declare two values, m and n, of sort int. The two arguments to add are a and b and they are of type int m and int n respectively. They are the type of the exact integer represented by the m and n. The result type is an integer which is the sum of these two values. Note that the dependent type in ATS (the m, n and m+n) are variables and computations expressed in the constraint language, not variables in ATS (the a, b, and a+b).

Having a restricted constraint language for type values simplifies the type checking process. All computations in this language are pure and have no effects. Sorts and functions in the language must terminate. This avoids infinite loops and exceptions during typechecking.

For more information on the reasoning behind restricted dependent types see
[Dependent Types in Practical Programming](http://www.ats-lang.org/PAPER/DML-popl99.pdf)
and
[other papers at the ATS website](http://www.ats-lang.org/PAPER/).

In ATS documentation the restricted constrainted language is called the ‘statics’ of ATS and is documented in chapter 5 of the
[ATS user guide](http://www.ats-lang.org/htdocs-old/DOCUMENT/MISC/manual_main.pdf).

## Simple Lists

Before I get into datatypes that use dependent types, I’ll do a quick overview of non-dependent types for those not familiar with ATS syntax. A basic ‘list’ type that can contain integers can be defined in ATS as:

```ocaml
datatype list =
  | nil
  | cons of (int, list)
```

With this type defined a list of integers can be created using the following syntax:

```ocaml
val a = cons(1, cons(2, cons(3, cons(4, nil))))
```

Functions that operate over lists can use pattern matching to deconstruct the list:

```ocaml
fun list_length(xs: list): int = 
  case+ xs of
  | nil () => 0
  | cons (_, xs) => 1 + list_length(xs)

fun list_append(xs: list, ys:list): list =
  case+ xs of
  | nil () => ys
  | cons (x, xs) => cons(x, list_append(xs, ys))
```

A complete example program is in
[dt1.dats](http://bluishcoder.co.nz/ats/dt1.dats) ([html](http://bluishcoder.co.nz/ats/dt1.html)).
This can be built with the command:

```
atscc -o dt1 -D_ATS_GCATS dt1.dats
```

Note the -D_ATS_GCATS. This tells the ATS compiler to link against the garbage collector. This is needed as types defined with datatype are allocated on the heap and require the garbage collector to be released.

## Polymorphic Lists

Instead of a list of integers we might want lists of different types. A list that is polymorphic can be defined using:

```ocaml
datatype list (a:t@ype) =
  | nil (a)
  | cons (a) of (a, list a)
```

Here the list can hold elements that are of any size. This is what the t@ype refers to. The functions that operate on these polymorphic lists are similar to the non-polymorphic list versions. The difference is they are ‘template’ functions and are parameterized by the template type:

```ocaml
fun{a:t@ype} list_length (xs: list a): int = 
  case+ xs of
  | nil () => 0
  | cons (_, xs) => 1 + list_length(xs)

fun{a:t@ype} list_append(xs: list a, ys: list a): list a =
  case+ xs of
  | nil () => ys
  | cons (x, xs) => cons(x, list_append(xs, ys))

fun{a,b:t@ype} list_zip(xs: list a, ys: list b): list ('(a,b)) =
  case+ (xs, ys) of
  | (nil (), nil ()) => nil ()
  | (cons (x, xs), cons (y, ys)) => cons('(x,y), list_zip(xs, ys))
  | (_, _) => nil ()
```

The {a:t@ype} immediately after the fun keyword identifies the function as a template function. These are very similar to C++ style templates. See the
[ATS Parametric Polymorphism and Templates tutorial](http://www.ats-lang.org/htdocs-old/TUTORIAL/contents/templates.html)
for more details.

This example adds a definition for list_zip now that lists of things other than integers can be created. In this example we return a list of tuples. Each tuple contains the elements from the original source lists.

The complete example program is in
[dt2.dats](http://bluishcoder.co.nz/ats/dt2.dats) ([html](http://bluishcoder.co.nz/ats/dt2.html)).
The example program has the following code:

```ocaml
val a = cons(1, cons(2, cons(3, cons(4, nil))))
val b = cons(5, cons(6, cons(7, cons(8, nil))))
val lena = list_length(a)
val lenb = list_length(b)
val c = list_append(a, b)
val d = list_zip(a, c) // <== different lengths!
val lend = list_length(d)
```

Note that the length of list a is 4, whereas the length of list d is 8. Calling list_zip with these two different lengthed lists results in a list of length 4 being returned.

We can encode the length of a list as part of the type to get a compile error if an attempt is made to zip two lists with different lengths. The length that is part of the type would be a dependent type (as the length is the value of an expression - the integer length of the list).

## Dependently Typed Lists

The following datatype definition defines a polymorphic list of length n, where n is an integer:

```ocaml
datatype list (a:t@ype, int) =
  | nil (a, 0)
  | {n:nat} cons (a, n+1) of (a, list (a, n))
```

It is very similar to the previous polymorphic list definition except for the additional int. The type constructor for nil has this set to 0. A nil list is a list of length 0. The cons type constructor shows that the cons of a list of length n will be a list of length n+1. The {n:nat} constrains the type of n to be natural numbers (non-negative integers).

The implementation of the functions shown previous for this new list type are function templates as they were in the previous example. They also have the additional parameter for the length value:

```ocaml
fun{a:t@ype} list_length {n:nat} (xs: list (a, n)): int n = 
  case+ xs of
  | nil () => 0
  | cons (_, xs) => 1 + list_length(xs)

fun{a:t@ype} list_append {n1,n2:nat} (xs: list (a, n1), ys: list (a, n2)): list (a, n1+n2) =
  case+ xs of
  | nil () => ys
  | cons (x, xs) => cons(x, list_append(xs, ys))

fun{a,b:t@ype} list_zip {n:nat} (xs: list (a, n), ys: list (b, n)): list ('(a,b), n) =
  case+ (xs, ys) of
  | (nil (), nil ()) => nil ()
  | (cons (x, xs), cons (y, ys)) => cons('(x,y), list_zip(xs, ys))
```

Notice the type of the return value of list_length is the type for the specific integer value of n - which is the length of the list. This means that any error in the implementation that would result in a different value being returned is a compile time error. For example, this won’t typecheck:

```ocaml
fun{a:t@ype} list_length {n:nat} (xs: list (a, n)): int n = 
  case+ xs of
  | nil () => 1
  | cons (_, xs) => 1 + list_length(xs)
```

Similarly the type of the return value of list_append is defined as being a list of length n1+n2. That is, the sum of the length of the two input lists. The following will fail with a compile error:

```ocaml
fun{a:t@ype} list_append {n1,n2:nat} (xs: list (a, n1), ys: list (a, n2)): list (a, n1+n2) =
  case+ xs of
  | nil () =>  nil
  | cons (x, xs) => cons(x, list_append(xs, ys))
```

It is now a compile error to pass lists of different lengths to list_zip. This is because both input arguments are defined to be of the same length n. This list_zip usage from the previous polymorphic list example is now a compile error.

The complete example program is in
[dt3.dats](http://bluishcoder.co.nz/ats/dt3.dats) ([html](http://bluishcoder.co.nz/ats/dt3.html)).

## Filter

It is not always possible to know the exact length of a result list which could make encoding the type problematic. A filter function that takes a list and returns a result list containing only those elements that return true when passed to a predicate function for example. In this case the typechecker would need to be able to call the predicate function to be able to determine the length of the result list. The following does not type check:

```ocaml
fun{a:t@ype} list_filter {m:nat} (
  xs: list (a, m),
  f: a -<> bool
  ): list (a, m) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs) => if f(x) then cons(x, list_filter(xs, f)) else list_filter(xs, f)
```

In this erroneous example the result is defined as a list of length m. But it won’t be of length m if the result of the predicate function means elements are skipped. A definition that typechecks uses an existential type definition (the [n:nat]) to define the result length as being a different value:

```ocaml
fun{a:t@ype} list_filter {m:nat} (
  xs: list (a, m),
  f: a -<> bool
  ): [n:nat] list (a, n) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs) => if f(x) then cons(x, list_filter(xs, f)) else list_filter(xs, f)
```

This unfortunately means that the typechecker won’t detect some examples of erroneous code. We’d like this to fail to compile if it results in a result list which is larger than the original list which should be impossible:

```ocaml
fun{a:t@ype} list_filter {m:nat} (
  xs: list (a, m),
  f: a -<> bool
  ): [n:nat] list (a, n) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs) => if f(x) then cons(x, cons(x, list_filter(xs, f))) else list_filter(xs, f)
```

The solution to this is to limit the existential type in the result to be all natural numbers less than or equal to the length of the input list:

```ocaml
fun{a:t@ype} list_filter {m:nat} (
  xs: list (a, m),
  f: a -<> bool
  ): [n:nat | n <= m] list (a, n) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs) => if f(x) then cons(x, list_filter(xs, f)) else list_filter(xs, f)
```

Note the [n:nat | n <= m] which defines the limit. This will now fail to compile the erroneous code but sucessfully compile the correct code. It won’t catch all possible errors but is better at catching some errors that the previous version.

The complete example program is in
[dt4.dats](http://bluishcoder.co.nz/ats/dt4.dats) ([html](http://bluishcoder.co.nz/ats/dt4.html)).

## Drop

The implementation of a list_drop function, which removes the first n items from a list, also has a similar problem to that of list_filter. This definition won’t typecheck:

```ocaml
fun{a:t@ype} list_drop {m:nat} (
  xs: list (a, m),
  count: int
  ): list (a, m) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs2) => if count > 0 then list_drop(xs2, count - 1) else xs
```

Like list_filter this is due to the wrong size being used in the result list. We could use the same solution as list_filter which is to set the size using an existential type definition but in this case we actually know the result size. It is based on the count that is passed as an argument. The result list size should be the same as the input list, less the count. Here’s the new definition:

```ocaml
fun{a:t@ype} list_drop {m,n:nat | n <= m} (
  xs: list (a, m),
  count: int n
  ): list (a, m - n) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs2) => if count > 0 then list_drop(xs2, count - 1) else xs
```

This version will typecheck correctly. It will give a compile time error if the implementation incorrectly produces a list that is not exactly the expected size. It will also be a compile time error if the given count of items to drop is greater than the size of the list.

This is done by making count a singleton integer. Its type is an integer of a specific value, called n. When n is declared we state that it must be less than the size of the list, m, as seen by the definition {m,n:nat | n <= m}. The result list is required to be of size m-n.

The complete example program is in
[dt5.dats](http://bluishcoder.co.nz/ats/dt5.dats) ([html](http://bluishcoder.co.nz/ats/dt5.html)).

## Lists that depend on their values

The previous list examples were datatypes that were dependant on the size of the list. It’s also possible to depend on the value stored in the list itself. The following is the definition for a list that can only hold even numbers:

```ocaml
datatype list =
  | nil of ()
  | {x:int | (x mod 2) == 0 } cons of (int x, list)
```

The cons constructor takes an int and a list. It is dependant on the value of the int with the constraint that the value of the int must be divisible by 2. It is now a compile error to put an odd number into the list. This won’t typecheck due to the attempt to pass the odd number 3 to cons:

```ocaml
val a = cons(2, cons(3, cons(10, cons(6, nil))))
```

The complete example program is in
[dt6.dats](http://bluishcoder.co.nz/ats/dt6.dats) ([html](http://bluishcoder.co.nz/ats/dt6.html)).

Another example of a list that depends on its values might be a list where all elements must be less than 100 and in order. It should be a type error to construct an unordered list.

```ocaml
datatype olist (int) =
  | nil (100) of ()
  | {x,y:int | x <= y} cons (x) of (int x, olist (y))
```

An olist is dependent on an int. This int is an integer value that all elements consed to the list must be less than. The nil constructor uses 100 for the value to show that all items must be less than 100. Like the previous example, the cons constructor depends on the first argument. It uses this in the constraint to ensure it is less than the dependant value of the tail list (the y).

Out of order construction like the following will be a type error:

```ocaml
val a = cons(1, cons(12, cons(10, cons(12, nil))))
```

Whereas this is fine:

```ocaml
val a = cons(1, cons(2, cons(10, cons(12, nil))))
```

The complete example program is in
[dt7.dats](http://bluishcoder.co.nz/ats/dt7.dats) ([html](http://bluishcoder.co.nz/ats/dt7.html)).

## Red-Black Trees

The paper
[Dependently Typed Datastructures](http://www.ats-lang.org/PAPER/DTDS-waaapl99.pdf)
has an example of using dependently typed datatypes to implement
[red-black trees](http://en.wikipedia.org/wiki/Red-black_tree). The paper uses the language
[DML](http://www.cs.bu.edu/~hwxi/DML/DML.html). I’ve translated this to ATS in the example that follows.

A red-black tree is a balanced binary tree which satisfies the following conditions:

* All nodes are marked with a colour, red or black.
* All leaves are marked black and all other nodes are either red or black.
* For every node there are the same number of black nodes on every path connecting the node to a leaf. This is called the black height of the node.
* The two sons of every red node must be black.

These constraints can be defined in the red-black tree datatype ensuring that the tree is always balanced and correct as per the conditions above. It becomes impossible to implement functions that produce a tree that is not a correct red-black tree since it won’t typecheck. This can be defined as:

```ocaml
sortdef color = {c:nat | 0 <= c; c <= 1}
#define red 1
#define black 0

datatype rbtree(int, int, int) =
  | E(black, 0, 0)
  | {cl,cr:color} {bh:nat}
     B(black, bh+1, 0)
       of (rbtree(cl, bh, 0), int, rbtree(cr, bh, 0))
  | {cl,cr:color} {bh:nat}
     R(red, bh, cl+cr)
       of (rbtree(cl, bh, 0), int, rbtree(cr, bh, 0))
```

The type rbtree is indexed by (int, int, int). These are the color of the node, the black height of the tree and the number of color violations respectively. The later is a count of the number of times a red node is followed by another red node. From the conditions given earlier it can be seen than a correct rbtree should always have a color violation number of zero.

The constructor, E, is a leaf node. This node is black, has a height of zero and no color violations. It is a valid rbtree.

The constructor B is a black node. It takes 3 arguments. The left child node, the integer value stored as the key, and the right child node. The type for a node constructed by B is black, has a height one greater than the child nodes and no color violations. Note that the black height of the two child nodes must be equal.

The constructor R is a red node. It takes 3 arguments, the same as the B constructor. As this is a red node it doesn’t increase the black height so uses the same value of the child nodes. The color violations value is calculated by adding the color values of the two child nodes. If either of those are red then the color violations will be non-zero.

The type for a function that inserts a key into the tree can now be defined as:

```ocaml
fun insert {c:color} {bh:nat} ( 
  key: int, 
  rbt: rbtree(c ,bh, 0)
  ): [c:color] [bh:nat] rbtree(c, bh, 0)
```

This means our implementation of insert must return a correct rbtree. It cannot have a color violations value that is not zero so it must conform to the conditions we outlined earlier. If it doesn’t, it won’t compile.

When inserting a node into the tree we can end up with a tree where the red-black tree conditions are violated. A function restore is defined below that pattern matches the combinations of invalid nodes and performs the required transformations to return an rbtree with no violations:

```ocaml
fun restore {cl,cr:color} {bh:nat} {vl,vr:nat | vl+vr <= 1} (
  left: rbtree(cl, bh, vl),
  key: int,
  right: rbtree(cr, bh, vr)
  ): [c:color] rbtree(c, bh + 1, 0) =
  case+ (left, key, right) of
    | (R(R(a,x,b),y,c),z,d) => R(B(a,x,b),y,B(c,z,d))
    | (R(a,x,R(b,y,c)),z,d) => R(B(a,x,b),y,B(c,z,d))
    | (a,x,R(R(b,y,c),z,d)) => R(B(a,x,b),y,B(c,z,d))
    | (a,x,R(b,y,R(c,z,d))) => R(B(a,x,b),y,B(c,z,d))
    | (a,x,b) =>> B(a,x,b)
```

The type of the restore function states that it takes a left and right node, one of which may have a color violation, and returns a correctly formed red-black tree node. It’s time consuming and error prone to look at the code and determine that it covers all the required cases to return a correctly formed tree. However the type checker will do this for us thanks to the constraints that have defined on the function and the rbtree type. It won’t compile if any of the pattern match lines are removed for example.

The use of =>> in the last pattern match line is explained in the
[tutorial on pattern matching](http://www.ats-lang.org/htdocs-old/TUTORIAL/contents/pattern-matching.html).
The ATS typechecker will typecheck each pattern line independently of the others. This can cause a typecheck failure in the last match since it doesn’t take into account the previous patterns and can’t determine that the color violation value of a in the result will be zero. By using =>> we tell ATS to typecheck the clause under the assumption that the other clauses have not been taken. Since they all handle the non-zero color violation case this line will then typecheck.

The insert function itself is implemented as follows:

```ocaml
fun insert {c:color} {bh:nat} (
  x: int,
  t: rbtree(c ,bh, 0)
  ): [c:color] [bh2:nat] rbtree(c, bh2, 0) = let
  fun ins {c:color} {bh:nat} (
    t2: rbtree(c, bh, 0)
  ):<cloptr1> [c2:color] [v:nat | v <= c] rbtree(c2, bh, v) =
    case+ t2 of
    | E () => R(E (), x, E ())
    | B(a, y, b) => if x < y then restore(ins(a), y, b)
                    else if y < x then restore (a, y, ins(b))
                    else B(a, y, b)
    | R(a, y, b) => if x < y then R(ins(a), y, b)
                    else if y < x then R(a, y, ins(b))
                    else R(a, y, b)
  val t = ins(t)
in
  case+ t of
  | R(a, y, b) => B(a, y, b)
  | _ =>> t
end
```

The complete example program is in
[dt8.dats](http://bluishcoder.co.nz/ats/dt8.dats) ([html](http://bluishcoder.co.nz/ats/dt8.html)).
For another example of red-black trees in ATS see the
[funrbtree example](http://www.ats-lang.org/htdocs-old/RESOURCE/contrib/funrbtree/funrbtree_dats.html)
from the ATS website.

## Linear constraints

The constraints generated by the dependent types must be ‘linear’ constraints. An example of a linear constraint is the definition of ‘list_append’ earlier:

```ocaml
fun{a:t@ype} list_append {n1,n2:nat} (
  xs: list (a, n1), ys: list (a, n2)
  ): list (a, n1+n2)
```

The result type containing n1+n2 will typecheck fine. However an example that won’t typecheck is the following definition of a list_concat:

```ocaml
fun{a:t@ype} list_concat {n1,n2:nat} (
  xss: list (list (a, n1), n2),
  ): list (a, n1*n2)
```

The n1*n2 results in a non-linear constraint being generated and the ATS typechecker cannot resolve this. The solution for this is to use theoreom-proving. See chapter 6 in the
[ATS user guide](http://www.ats-lang.org/htdocs-old/DOCUMENT/MISC/manual_main.pdf)
for details and an example using concat.

## Closing notes

Although I create my own list types in these examples, ATS has lists, vectors and many other data structures in the standard library.

There’s a lot more that can be done with ATS and dependent types. For more examples see the papers mentioned at the beginning at throughout this post. The paper
[Why Dependent Types Matter](http://lambda-the-ultimate.org/node/693)
is also useful reading for more on the topic.
