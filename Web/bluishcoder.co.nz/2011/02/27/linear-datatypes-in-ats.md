# BLUISH CODER: ATSにおける線形データ型

(元記事は http://bluishcoder.co.nz/2011/02/27/linear-datatypes-in-ats.html です)

A datatype in ATS is similar to types defined in languages like ML and Haskell. They are objects allocated in the heap controlled by the garbage collector. The programmer does not need to explicitly free the memory or manage the lifetime of the object.

A dataviewtype is similar to a datatype in that it is an object allocated on the heap. A dataviewtype is a linear type. The programmer must explicitly free the memory allocated and control the lifetime of the object. The type checker tracks usage of the linear type and ensures at the type level that it is consumed and that dangling references to it are not kept around.

To compare the datatype and dataviewtype I’ll start with a simple definition of a list type using datatype:

```ocaml
datatype list (a:t@ype) =
  | nil (a)
  | cons (a) of (a, list a)

fun {seed:t@ype}{a:t@ype} fold(f: (seed,a) -<> seed, s: seed, xs: list a): seed =
  case+ xs of
  | nil () => s
  | cons (x, xs) => fold(f, f(s, x), xs)

implement main() = let
  val a = cons(1, cons(2, cons(3, cons(4, nil))))
  val b = fold(add_int_int, 0, a)
  val c = fold(mul_int_int, 1, a)
  val () = printf("b: %d c: %d\n", @(b, c))
in
  ()
end
```

This program defines a list type that can hold instances of a particular type. A function fold is defined that performs a fold on a list. The main routine creates a list, a, and performs two folds. One to get a sum of the list and the other to get the product. The results of these are printed.

The fold function is similar to function templates in C++. The syntax fun {seed:t@type}{a:t@ype} defines a function template with two arguments. The type of the seed for the fold and the type of the list elements. When fold is called the function is instantiated with the actual types used.

add_int_int and mul_int_int are two ATS prelude functions that are defined to perform addition and multiplication respectively on two integers.

Notice there is no manual memory management. The list held in a is on the garbage collected heap. This ATS program must be built with -D_ATS_GCATS to link in the garbage collector to reclaim memory used by the list type:

```
atscc -D_ATS_GCATS test.dats
```

The above example shows that programming with datatypes is very similar to any other garbage collected functional programming language.

The following is the same program converted to use dataviewtype:

```ocaml
dataviewtype list (a:t@ype) =
  | nil (a)
  | cons (a) of (a, list a)

fun {a:t@ype} free(xs: list a): void =
  case+ xs of
  | ~nil () => ()
  | ~cons (x, xs) => free(xs)

fun {seed:t@ype}{a:t@ype} fold(f: (seed,a) -<> seed, s: seed, xs: !list a): seed =
  case+ xs of
  | nil () => (fold@ xs; s)
  | cons (x, !xs1) => let val s = fold(f, f(s, x), !xs1) in fold@ xs; s end

implement main() = let
  val a = cons(1, cons(2, cons(3, cons(4, nil))))
  val b = fold(add_int_int, 0, a)
  val c = fold(mul_int_int, 1, a)
  val () = printf("b: %d c: %d\n", @(b, c))
in
  free(a)
end
```

The definition of the list looks the same but uses the keyword dataviewtype. This makes list a linear type. Functions that accept linear types will show in the argument list whether they consume or leave unchanged the linear type. This is denoted by an exclamation mark prefixed to the type. For example:

```ocaml
// Consumes (free's) the linear type
fun consume_me(xs: list a) = ...

// Does not consume the linear type
fun use_me(xs: !list a) = ...

implement main() = let
  val a = nil
  val () = use_me(a)
  // Can use 'a' here because 'use_me' did not consume the linear type
  val () = consume_me(a)
  // The following line is a compile error because
  // 'consume_me' consumed the type
  val () = use_me(a)
in () end
```

Notice in the new definition of fold that the xs argument type is prefixed with an exclamation mark meaning it does not consume the linear type. This allows calling fold multiple times on the same list.

The implementation of fold had to be changed slightly to use the linear type without consuming it. The body of the function is:

```ocaml
case+ xs of
| nil () => (fold@ xs; s)
| cons (x, !xs1) => let val s = fold(f, f(s, x), !xs1) in fold@ xs; s end
```

Notice the fold@ call and the use of !. xs is a linear value (ie. our linear list). Because the linear value matches a pattern in a case statement that has a ! then the type of the linear value xs changes to be cons(int, an_address) where an_address is a memory address. xs1 is assigned the type ptr(an_address) where the value of xs is stored at address an_address. This is why the !xs1 is needed later. Since xs1 is now a pointer it must be dereferenced before calling fold again to pass the actual list. This pattern matching process is known as ‘unfolding a linear value’ in the documentation.

As the type of xs has changed during the pattern matching we need to change it back in the body of the pattern. The fold@ call returns the type of xs from cons(int,an_address) back to cons(int,list int). This is referred to as ‘folding a linear value’ in the documentation.

After using the list a in main we need to free the memory. This is done with the free function which iterates over the list consuming each element. Notice that free does not have an exclamation mark prefixed to the argument type.

Each line in the case statement in free has a ~ prefixed to the constructor to be pattern matched against. This is known as a destruction pattern. If a linear value matches the pattern then the pattern variables are bound and the linear value is free’d. The result of this is free walks the list freeing each element.

```ocaml
case+ xs of
  | ~nil () => ()
  | ~cons (x, xs) => free(xs)
```

There’s a lot of detail going on under the hood but the code between the two versions is very similar. ATS makes it relatively easy to deal with the complexities of manual memory management and fails at compile time if it’s not managed correctly. Much of my interest in ATS is how to write low level code that manages memory safely.
