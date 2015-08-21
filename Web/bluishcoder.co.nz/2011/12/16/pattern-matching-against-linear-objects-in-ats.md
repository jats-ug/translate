# BLUISH CODER: ATSにおける線形オブジェクトのパターンマッチ

(元記事は http://bluishcoder.co.nz/2011/12/16/pattern-matching-against-linear-objects-in-ats.html です)

In a project I’m working on I’m using linear lists.
This is the `list_vt` type in the [ATS](http://www.ats-lang.org/) prelude.
`list_vt` is similar to the list types in Lisp and functional programming languages except it is linear.
The memory for the list is not managed by the garbage collector and the type system enforces the rule that only one reference to the linear object can exist.
This sometimes requires a bit of extra effort when using pattern matching against the `list_vt` instances.

## パターンマッチ

When pattern matching against linear objects you can do a destructive match or a non-destructive match.
The former will destroy and free the memory allocated for the object automatically.
The latter will not.
Destructive matches are done by having the pattern match clause prefixed with a `~`.
For example, the following will print an integer list and destroy the list while it does it:

```ats
fun print_list (l: List_vt (int)): void =
  case+ l of
  | ~list_vt_nil () => printf("nil\n", @())
  | ~list_vt_cons (x, xs) => (printf("cons %d\n", @(x)); print_list(xs))

fun test1 (): void = {
  val a = list_vt_cons {int} (1, list_vt_nil)
  val () = print_list (a)
}
```

Things get complicated when doing non-destructive matches.
The following won’t typecheck:

```ats
fun print_list2 (l: !List_vt (int)): void =
  case+ l of
  | list_vt_nil () => printf("nil\n", @())
  | list_vt_cons (x, xs) => (printf("cons %d\n", @(x)); print_list(xs))

fun test2 (): void = {
  val a = list_vt_cons {int} (1, list_vt_nil)
  val () = print_list2 (a)
  val () = list_vt_free (a)
}
```

The problem with this example is that when the match is made we are effectively taking the linear object out of the variable `l`.
This leaves `l` with a different type, but we’ve stated in the function signature for `print_list2` that the type is not modified or consumed.
We need a way of putting the linear object back into l once we’re done using the match.
This primitive to do this is `fold@` which I briefly introduced in my [linear datatypes post](http://bluishcoder.co.nz/2011/02/27/linear-datatypes-in-ats.html).
`fold@` will change the type of l back to the original and prevent access to the pattern match variables.
Usage looks like this:

```ats
fun print_list2 (l: !List_vt (int)): void =
  case+ l of
  | list_vt_nil () => (fold@ l; printf("nil\n", @()))
  | list_vt_cons (x, !xs) => (printf("cons %d\n", @(x)); print_list2(!xs); fold@ l)

fun test2 (): void = {
  val a = list_vt_cons {int} (1, list_vt_nil)
  val () = print_list2 (a)
  val () = list_vt_free (a)
}
```

You’ll notice with this version that the match for `list_vt_cons` has changed the `xs` parameter to be `!xs`.
The second argument in the cons constructor is a linear object.
If the object itself is matched against `xs` then it is another example of aliasing the linear object.
It is taken out of the l and needs to be put back.
The way ATS handles this is to require pattern matching with a `!` prefixed.
This makes `xs` be a pointer to the object rather than the object itself.
So in this example `xs` has the type `ptr addr` where `addr` is the address of the actual `List_vt` object.
This is why the `xs` is prefixed by `!` in the recursive call to `print_list2`.
The `!` means dereference the pointer, so the `List_vt` it is pointing to is passed as the argument to the recursive call.

In this way the linear object is never taken out, we only access it via its pointer.
The `fold@` call in this clause will change `xs` back to the `List_vt` object.
The `fold@` call is done after the usage of `!xs`.
If it was done before then we wouldn’t have access to the view for `xs` to be able to derefence it.
`print_list2` is still tail recursive as the `fold@` call is only used during typechecking and is erased afterwards.

## 線形リストをフィルタする

In my project I needed to filter a linear list.
Unfortunately ATS doesn’t have a filter implementation in the standard prelude for linear lists (it does for persistent lists).
My first attempt at writing a `list_vt_filter` looked like:

```ats
fun list_vt_filter (l: !List_vt (int), f: int -<> bool): List_vt (int) =
  case+ l of
  | list_vt_nil () => (fold@ l; list_vt_nil)
  | list_vt_cons (x, !xs) when f (x) => let
                                          val r = list_vt_cons (x, list_vt_filter (!xs, f))
                                        in
                                          fold@ l; r
                                        end
  | list_vt_cons (x, !xs) => let
                                val r = list_vt_filter (!xs, f)
                              in
                                fold@ l; r
                              end
```

This should look familiar since it’s very similar to the `print_list2` code shown previously in the way it uses non-destructive matching and `fold@`.
The function `list_vt_filter` takes a `list_vt` as an argument and a function to apply to each element in the list.
That function returns true if the element should be included in the result list.
Usage looks like:

```ats
val a  = list_vt_cons (1, list_vt_cons (2, list_vt_cons (3, list_vt_cons (4, list_vt_nil ()))))
val b  = list_vt_filter (a, lam (x) => x mod 2 = 0)
val () = list_vt_foreach_fun<int> (a, lam(x) =<> $effmask_all (printf("Value: %d\n", @(x))))
val () = list_vt_free (b)
val () = list_vt_free (a)
```

One issue with this implementation is it is not tail recursive.
It has stack growth proportional to the size of the result list.

## 末尾再帰フィルタ

In Lisp code I’d often build the result list tail recursively by passing an accumulator, with each new element in the result being prepended to the accumulator.
This builds a list in the reverse order so before returning it the list would be reversed.
The ATS code for this is:

```ats
fun list_vt_filter (l: !List_vt (int), f: int -<> bool): List_vt (int) = let
  fun loop (l: !List_vt (int), accum: List_vt (int)):<cloptr1> List_vt (int)  =
    case+ l of
    | list_vt_nil () => (fold@ l; accum)
    | list_vt_cons (x, !xs) when f (x) => let
                                            val r = loop (!xs, list_vt_cons (x, accum))
                                          in
                                            (fold@ l; r)
                                          end
    | list_vt_cons (x, !xs) => let
                                 val r = loop (!xs, accum)
                                in
                                  (fold@ l; r)
                                end
in
  list_vt_reverse (loop (l, list_vt_nil))
end
```

The `cloptr1` function annotation marks the inner function as being a closure where the memory for the closure’s environment is managed by the compiler using malloc and free instead of the garbage collector (which is what `cloref1` would signify).
See my post on [closures in ATS](http://bluishcoder.co.nz/2010/06/20/closures-in-ats.html) for more about the different closure and function types used by ATS.

Unfortunately the requirement to use `fold@` after we’ve finished with using the pattern matched variables makes the code slightly more verbose as we need to do the tail recursion, obtaining the result, then do the `fold@` and return the result.
Remember that the `fold@` is erased at type checking type which is how this code remains tail recursive even though the code structure makes it look like it isn’t.

One downside to this approach is we iterate over the list twice.
Once to build the result, and once over the result to reverse it.

## シングルパス末尾再帰フィルタ

The creation of the result list can be done in a single pass if we could create a cons with no second argument, and fill in that argument later when we have a result to store there that passes filtering.
ATS allows construction of datatypes with a ‘hole’ that can be filled in later. The ‘hole’ is an unintialized type and we get a pointer to it. An example of doing this is:

```ats
var x = list_vt_cons {int} {0} (1, ?)
```

This creates a `list_vt_cons` with the data set to `1` but no second parameter.
Instead of that parameter being of type `List_vt (int)` it is of type `List_vt (int)?`, the `?` signifying it is uninitialized.
For this example we have to pass the universal type parameters explicitly (the `{int} {0}`) as the ATS type inference algorithm can’t compute them.

To get a pointer to the ‘hole’ we have to pattern match:

```ats
val+ list_vt_cons (_, !xs) = x
val () = !xs := list_vt_nil
val () = fold@ x
```

In this example the `xs` is a pointer, pointing to the `List_vt (int)?`.
It assigns a `list_vt_nil` to this, making the tail of the cons a `list_vt_nil`.
Just like in our previous pattern matching examples using case, the code has to do a `fold@` to change the type of `x` back to that containing a linear object once we’ve finished using xs.

Now that we can get pointers to the tail of the list we can implement a single pass tail recursive filter function:

```ats
fun list_vt_filter (l: !List_vt (int), f: int -<> bool): List_vt (int) = let
  fun loop (l: !List_vt (int), res: &List_vt (int)? >> List_vt (int)):<cloptr1> void =
    case+ l of
    | list_vt_nil () => (fold@ l; (res := list_vt_nil))
    | list_vt_cons (x, !xs) when f (x) => let
                                            val () = res := list_vt_cons {int} {0} (x, ?)
                                            val+ list_vt_cons (_, !p_xs) = res
                                           in
                                             loop (!xs, !p_xs); fold@ l; fold@ res
                                           end
    | list_vt_cons  (x, !xs) => (loop (!xs, res); fold@ l)

  var res: List_vt (int)?
  val () = loop (l, res)
in
  res
end
```

The loop function here no longer turns a result.
Instead the result is passed via a reference (the & signifies ‘by reference’).
When there is something that needs to be stored in the list, a cons is created with a hole in the tail position.
This cons is stored in the result we are passing by reference and we tail recursively call with the hole as the new result.
ATS converts this to nice C code that is a simple loop rather than recursive function calls.

## 雑多なこと

The code examples in this post use `List_vt (a)`.
This is actually a typedef for `list_vt (a,n)` where a is the type and `n` is the length of the list.
The typedef allows shorter examples without needing to specify the sorts for the list length.
Using the full type though has the advantage of being able to specifiy a bit more type safety.
For example, the original filter function would be declared as:

```ats
fun list_vt_filter {n:nat} (l: !list_vt (int,n), f: int -<> bool): [r:nat | r <= n] list_vt (int, r)
```

This defines the type of the result as having a length equal to or less than that of the original list.
This helps prevent errors in the implementatin of the filter - it can’t accidentally leave extra items in the list.
I cover this type of thing in my post on [dependent types](http://bluishcoder.co.nz/2010/09/01/dependent-types-in-ats.html).

Another addition to safety that adding the extra sorts can provide is the ability to check that the function terminates.
This can be done by adding a termination metric to the function definition:

```ats
fun list_vt_filter {n:nat} .<n>. (l: !list_vt (int,n), f: int -<> bool): [r:nat | r <= n] list_vt (int, r)
```

The compiler checks that `n` is decreasing on each recursive call.
If this fails to happen the recursive calls may not terminate and it becomes a compile error.
This is discussed in the Termination-Checking for Recursive Functions section of the [Introduction to Programming in ATS book](http://jats-ug.metasepi.org/doc/ATS2/INT2PROGINATS/index.html).

A description of how `fold@` works is in the [ATS/Anairats User’s Guide PDF](http://ats-lang.sourceforge.net/htdocs-old/DOCUMENT/MISC/manual_main.pdf).
It’s in the ‘Dataviewtypes’ section of the ‘Programming with Linear Types’ chapter and is referred to as folding and unfolding a linear type.

It’s the usage of linear types and dealing with their restrictions that makes my examples a bit more complex.
If you use ATS mainly with non-linear types and link with the garbage collector then it becomes very much like using any other functional programming language, but with additional features in the type system.
My interest has been around avoiding using a garbage collector and having the compiler give errors when memory is not allocated or free’d correctly.
Don’t be put off from using ATS by these complex examples if you’re fine with using garbage collection and non-linear datatypes.
You might never need to deal with the cases that bring in the extra complexity.
