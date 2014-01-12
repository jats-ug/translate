# ATSのクロージャ

(元記事は http://bluishcoder.co.nz/2010/06/20/closures-in-ats.html です)

My
[last post on ATS](http://bluishcoder.co.nz/2010/06/13/functions-in-ats.html)
was about C style functions.
This post covers what I’ve learnt about closures in ATS.

As I mentioned in my last post, closures are functions combined with an environment mapping names to values.
These are like closures in dynamic languages that enable you to capture variables in the enclosing scope.
To create a closure you must explicitly mark it as being a closure.
A closure requires memory to be allocated to store the environment.
This means you will either need to allocate the closure on the stack (so it is freed automatically when the scope exits), free the memory for the closure manually, or link against the garbage collector.

## 永続クロージャ

A persistent closure is allocated on the heap and is freed by the garbage collector when it is no longer used.
To make this type of closure you use the tag ‘cloref’:

```ocaml
fun make_adder(n1:int): int -<cloref> int =
  lam (n2) =<cloref> n1 + n2;

implement main() = let
  val add5 = make_adder(5)
  val a = add5(10)
  val b = add5(20)
  val () = printf("a: %d\nb: %d\n", @(a,b))
in
  ()
end
```

As this example needs to use the garbage collector it must be built with the garbage collector included (using the -D_ATS_GCATS switch):

```
atscc -o eg1 eg1.dats -D_ATS_GCATS
```

In this example the ‘make_adder’ function takes an integer, n1, and returns a closure.
This closure takes an integer, ‘n2’ and returns the addition of ‘n1’ and ‘n2’.
The ‘main’ function creates an adder with the initial function of ‘5’ and uses the returned closure a couple of times.

## 線形クロージャ

xxx
