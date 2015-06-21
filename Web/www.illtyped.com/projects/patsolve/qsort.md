# Verified Efficient Programs in ATS: qsort

(元記事は http://www.illtyped.com/projects/patsolve/qsort.html です)

## 説明

このチュートリアルは、ATS のために Z3 SMT ソルバを使って私達が設計した、新しい制約ソルバの外観です。
We'll discuss how the statics work in ATS, how we can extend them with new sorts and predicates, and how to give predicate's meaning to the constraint solver.
Finally, we'll show how we can use this technique to verify the functional correctness of programs that we either could not verify before, or would take significant manual effort.
At the same time, we stress that the goal of this work is not to remove manual theorem proving when writing ATS code.

Instead, we advocate a combination of automated and manual theorem proving.
Our rationale for this approach revolves around a desire to maintain the usefulness of type error messages to the user.
Indeed, if you discharge too much of a program's proof to automation, the tool may fail, but it may not be entirely clear why it fails.
When proving manually, error messages point out the exact location where your proof fails, and the reason can be given explicitely.
At the same time, manual proofs take time and can be quite tedious.
In this tutorial, we will explore how we can use both approaches effectively together.
In particular, we'll pay attention to low-level programs involving pointer arithmetic and linear resources.
We will use automated proving to reason about functional correctness, that our program does what we intend it to do, and manual proving to verify memory safety.

The constraint solver developed in this work was made to work outside ATS.
Typically, the typechecker solvers constraints internally, but in the interest of enabling more powerful verification, the ATS2 compiler gives an option to export constraints in the familiar JSON format.
A constraint solver is simply a tool that can parse this information, and try to find any contradiction between the program and its formal specification.
Finding counter examples to logical assertions is a basic task for an SMT solver, so naturally one as powerful as Z3 can fit this role.
Z3 also comprehends theories that were previously outside the reach of our static language such as extensional arrays, real numbers, and fix width bit vectors, all of which are common domains in software development.

This is all very nice at a high level, but how exactly does a constraint solver built with Z3 work, and how can we effectively utilize its broad decision power?
This tutorial aims to answer these questions with illustrative examples that show static reasoning in ATS verifying program invariants.
We'll start with a discussion of ATS without any static types applied dynamic values, and gradually refine the types of our programs into a more formal specification.
As we go along we'll argue that the correctness guarantees of our programs will become stronger as a result.

## Goal

The end goal of this tutorial is to show how we can use the constraint solver we built on top of Z3 to construct an efficient and verified ATS program corresponding exactly to this version of quicksort, given in C.

```c
void swap (int *a, int *b) {
  int tmp = *a; *a = *b; *b = tmp;
}

int partition (int *ar, int p, int n) {
  int *pn, *pi, *pindex, xn;
  pn = ar + (n - 1);
  pi = ar + p;

  swap (pi, pn);

  xn = *pn;

  for(pi = ar, pindex = ar; pi != pn; pi++)
    if (*pi < xn)
      swap (pi, pindex++);

  swap (pindex, pn);

  return pindex - ar;
}

void quicksort (int *ar, int n) {
  if (n <= 1) {
    return;
  }
  int p = (int) rand () % n;
  int piv = partition (ar, p, n);

  quicksort (ar, piv);
  quicksort (ar+piv+1, n-piv-1);
}
```

We didn't develop this code in C. In fact, we used our constraint solver to construct and simultaneously verify an implementation of quicksort in ATS, and then wrote its C counter part after we had proven its correctness. Writing the above in C is completely redundant, as ATS programs are compiled to C and run directly. We did this to demonstrate our philosophy behind software verification. We are interested in verifying programs in a programmer centric way, where the programmer interacts with a verifier during a program's construction. In doing so, he can catch logical errors in his reasoning early, before any code is ever compiled. The rest of this tutorial will describe how the constraint solver helps faciliate this style of programming, and how we can use it to construct and verify several example programs.

## Basic ATS

With this in mind, let's talk about ATS without its more advanced features. In this way, it will appear more like your standard functional programming language, such as ML. Suppose we have a simple datatype representing a linked list of some generic type T.

```ats
abstype T

datatype list =
    | list_nil of ()
    | list_cons of (T, list)
```

Now, let's define a simple accessor function to get the nth element in a list. Using pattern matching this is completely trivial.

```ats
fun list_nth (xs: list, i:int): T =
case+ xs of 
| list_nil () => 
  abort ()
| list_cons (x, xss) =>
  if i = 0 then 
     x
  else 
     list_nth (xss, i-1)
```

Looking over this function, it seems a little ad-hoc. It would be cleaner if we could guarantee for every call of listnth, no errors would occur. That is, the value i passed to listnth is always less than the length of the list. This can be easily done by adding statics to our data types.

## Refining with Static Types

In order to get an always error free version of listnth, we index every value of list with a static integer representing its length. Note, this index occurs only in type checking, and has no concrete representation or overhead in the final program at runtime. We introduce it because it allows us to reason about whether any indice given to listnth is always less then the length of the list given to list_nth. This is an invariant we can enforce using the following definition of list.

```ats
datatype list (n:int) =
    | list_nil (0) of ()
    | {n:nat}
       list_cons (n+1) of (T, list(n))
```

The syntax will likely be confusing for new comers. The text {n:nat} is a universal quantifier we use for the list_cons constructor. It means that for all natural numbers n, the cons of a list of length n and a type T yields a list of length n+1. This is simple enough, and precisely captures the length invariant we wish to enforce.

Now, let's redefine listnth to ensure that for all lists xs and all indices less than the length of xs, listnth is well defined. Note, we also end up asserting that the length of xs must be positive.

```ats
fun list_nth {n,i:nat | i < n} (
    xs: list (n), i: int i
): T = let
   val+ list_cons (x, xss) = xs
in
    if i = 0 then
       x
    else
       list_nth (xss, i - 1)
end
```

This is much better, and by the definition of its type, whenever the programmer uses listnth, they must provide proof that the index is always less then the length of the list. This is a pretty bold claim. How exactly does the constraint solver know this implementation never produces an out of bound error? Using the common SMT-Lib2 syntax, let's see the automated interaction our constraint solver has with Z3 to type check the listnth function. Internally, this interaction is achieved through ATS bindings we wrote for Z3's C API.

```
;; first, state our assertions
(declare-const n Int)
(declare-const i Int)

;; n and i are natural numbers
(assert (<= 0 n))
(assert (<= 0 i))
(assert (< i n))

;; check that the length of the list can never be 0
;; or list_cons is the only pattern that can occur
(declare-const m Int)
(declare-const x Int)

(assert (<= 0 m))
(assert (= n (+ m 1)))

(push)
(assert (not n > 0))
(check-sat)
;; an assignment that satisfies this assertion amounts to a counter 
;; example to the guard we have placed on list_nth.

;; unsatisfiable means our assertion is valid under all inputs given 
;; our assumptions so far.
(pop)


;; no constraints to check if i = 0

;; check the else case
(push)
(assert (not (= i 0)))

(push)
;; check our invariant for the tail of xs
(assert (not (< (- i 1) m)))
(check-sat)
(pop 2)
```

If all of the above return unsatisfiable, we know that if we provide valid input (i.e. i is less than n), then it will never go outside the bound of the list. This is certainly nice, but it provides no guarantee that the element we return is actually the ith element within the list. Indeed, there is no guarantee what we return is even an element of the list! In order to get this much stronger guarantee, and capture what we want precisely in the function's type, we can leverage Z3's array theory. This allows us to capture invariants that would have been quite cumbersome in our original constraint solver.

## Z3 Arrays as Streams

In the last example, we used Z3's understanding of integers to automatically check whether invariants involving indexes and lengths of lists were enforced in our code. Using its knowledge of arrays, we can take this a step further to statically guarantee more properties. Let us use this to refine our type of list to be indexed by a "static" stream understood by Z3. Suppose we have the following static sort in ATS.

```ats
datasort stampseq = (* abstract *)
```

We say this sort is "abstract" because the ATS type system has no knowledge about it. Instead, we parse constraints in our solver and give all constants of sort "stampseq" an (Array Int Int) sort in Z3. This allows us to model an infinite stream of stamps as an unbounded array in Z3. Let us define a couple common constructors on sequences and their interpretations in SMT-Lib2.

```ats
stacst stampseq_nil  : () -> stampseq
stacst stampseq_cons : (stamp, stampseq) -> stampseq
```

From Z3's perspective, these are just uninterpreted functions. We can give them meaning by putting the following assertions in an SMT-Lib 2 file which our constraint solver parses and passes to a Z3 context.

```
(declare-const undef Int)

(assert (forall ((A (Array Int Int)) (i Int))
(=> (< i 0) (= (select A i) undef))))

(assert (forall ((i Int))
  (= (select stampseq_nil i) undef)))

(assert (forall ((A (Array Int Int))(x Int))
  (= (select (stampseq_cons x A) 0) x)))

(assert (forall ((A (Array Int Int))(x Int) (i Int))
  (=> (> i 0) (= (select (stampseq_cons x A) i) (select A (- i 1))))))
```

With these tools in place, let's add an index to our definition of list and T that represents the contents of the list as a static stream of stamps.

```ats
abstype T(x:stamp)

datatype list (stmsq, int) = 
    | list_nil (nil, 0) of ()
    | {xs:stmsq} {n:nat} {x:stamp}
       list_cons (cons(x, xs), n+1) of (T(x), list(n))
```

With this definition, we can capture precisely the behavior of list_nth. That is, it returns the ith element in the list. Here, we use the static "select" function in the Array theory to represent this action.

```ats
fun list_nth {xs:stmsq} {n,i:nat | i < n} .<i>. (
    xs: list (xs, n), i: int i
): T(select(xs, i)) = let
   val+ list_cons (x, xss) = xs
in
    if i = 0 then
       x
    else
       list_nth (xss, i - 1)
end
```

Now, if we were to return anything but x when i = 0, the type checker will catch the error, as only x represents select (xs, i). This is also very interesting as Z3 is able to determine the correctness of this function completely automatically given our interpretations of the stream operations. In the next example, we'll take this idea a step further to provide an implementation of insertion sort that is guaranteed to return a well-sorted list.

## Sorting Linked Lists

The first thing we need to verify a sorting algorithm is a way to express the sortedness of a sequence to the SMT solver. Using SMT-Lib 2, we can use the following definition.

```
(define-fun sorted ((A (Array Int Int)) (n Int)) Bool
  (forall ((i Int) (j Int))
    (=> (<= 0 i j (- n 1))
        (<= (select a i) (select a j)))))
```

So implementing a sort function should be trivial. Unfortunately, one of the most important axioms we have when sorting a linked list cannot be solved automatically by Z3. This axiom is that the tail of any sorted list is also a sorted list. This is demonstrated with the following SMT interaction, which responds "unknown" when queried to the axioms validity.

```
 (declare-const xs (Array Int Int))

 (declare-const x Int)
 (declare-const xss (Array Int Int))

 (declare-const n Int)

 (assert (sorted xs n))
 (assert (= x (cons x xss)))

 (push)
 (assert (not (sorted xss (- n 1))))
 (check-sat)
 (pop)
```

Luckily, in ATS we have a system for theorem proving in place that allows us to state this property as an axiom and then use it in our programs. The following are our set of axioms that relate the sortedness of a sequence (represented here as a an abstract prop) to the sequence cons operation. When we are done proving, we can just call SORTED_elim to obtain the assertion that xs is sorted.

```ats
absprop SORTED (xs:stmsq, n:int)

extern
praxi
SORTED_elim
  {xs:stmsq}{n:int}
  (pf: SORTED(xs, n)): [sorted(xs, n)] void

extern
praxi
SORTED_nil(): SORTED (nil, 0)

extern
praxi
SORTED_sing{x:stamp}(): SORTED (sing(x), 1)

extern
praxi
SORTED_cons
  {x:stamp}
  {xs:stmsq}{n:pos}
  {x <= select(xs,0)}
  (pf: SORTED (xs, n)): SORTED (cons(x, xs), n+1)

extern
praxi
SORTED_uncons
  {x:stamp}
  {xs:stmsq}{n:pos}
  (pf: SORTED (cons(x, xs), n)): [x <= select(xs,0)] SORTED (xs, n-1)
```

The main axiom that solves the problem we first mentioned is SORTED_uncons, because it adds the assertion that the head of a list is always less then or equal to the head of a sorted tail. let us use these axioms to implement the following function.

```ats
extern
fun sort
  {xs:stmsq}{n:int}
  (xs: list (xs, n)): [ys:stmsq] (SORTED (ys, n) | list (ys, n))
```

That is, given any list xs of length n, there exists a sorted list ys of length n, which we return. In order to implement this as insertion sort, we need a function that inserts an element x into the correct position in a sorted list.

```ats
extern
fun insord
  {x0:stamp}
  {xs:stmsq}{n:nat}
(
  pf: SORTED(xs, n) | x0: T(x0), xs: list (xs, n)
) : [i:nat]
(
  SORTED (insert(xs, i, x0), n+1) | list (insert(xs, i, x0), n+1)
)
```

This gives us a strong description of what we want. Given any sorted list xs, we must return a sorted list where x0 is inserted into the original list xs. This guarantees that our result from insord will yield the original list with one new element inserted at index i. The following is an interpretation of the insert function in SMT-Lib 2.

```
(assert (forall ((A (Array Int Int))(x Int) (i Int) (j Int))
  (=> (and (<= 0 j) (<= 0 i))
    (= (select (insert A i x) j)
     (ite (< j i) (select A j)
       (ite (= j i) x (select A (- j 1))))))))
```

With Z3 using this interpretation, we can type check the final implementation of insord that produces "insert(xs, i, x0)" along with a proof that it is sorted.

```ats
implement
insord {x0} (pf | x0, xs) =
(
case+ xs of
| list_nil () =>
    #[0 | (SORTED_sing{x0}() | list_cons (x0, list_nil))]
| list_cons {xs1}{x} (x, xs1) =>
  (
    if x0 <= x
      then
        #[0 | (SORTED_cons{x0} (pf) | list_cons (x0, xs))]
      else let
        prval (pfs) = SORTED_uncons {x}{xs1} (pf)
        val [i:int] (pfres | ys1) = insord {x0} (pfs | x0, xs1)
      in
        #[i+1 | (SORTED_cons{x} (pfres) | list_cons (x, ys1))]
      end // end of [if]
    // end of [if]
  )
) (* end of [insord] *)
```

Note that all the props used in this implementation are simply used for the purpose of theorem proving. That is, they add no additional overhead and indeed are completely erased after constraint solving. With this function, writing the sort routine on any list is fairly trivial, and is given below.

```ats
implement
sort (xs) =
(
case+ xs of
| list_nil () =>
    (SORTED_nil() | list_nil())
| list_cons (x, xs1) => let
    val (pf1 | ys1) = sort (xs1) in insord (pf1 | x, ys1)
  end // end of [list_cons]
) (* end of [sort] *)
```

## Linear Views

So far we've only spoke of using streams for linked lists, but how can they aid us in enforcing correct pointer arithmetic in programs? While ATS has a primitive pointer type, it also has a view system of tracking resources in memory. A "view" is simply a certificate that some data structure lies at a point in memory. This is illustrated in the following example.

```ats
var counter: int

val pf = view@ counter (* pf is of type int @ counter *)
val p = addr@ p        (* p is of type ptr counter *)
```

All views are "linear" in that the programmer must consume all views so that resources are not lost. Conversely, if a programmer takes a view from a static variable, as above, she is obligated to put it back. Just as we defined lists inductively, so can we define array views and index them with a static stream. This gives us the following definition

```ats
dataview
array_v
  (addr, stmsq, int) =
  | {l:addr}
    array_v_nil (l, nil, 0) of ()
  | {l:addr}{xs:stmsq}{x:stamp}{n:int}
    array_v_cons (l, cons (x, xs), n+1) of (T(x) @ l, array_v (l+1, xs, n))
// end of [array_v]
```

Notice we use pointer arithmetic here so we may get more views of elements in the array. Naturally, we make want to split an array into sub-views to process individual sections of an array. We can do this with the following proof function. Note, just like their non-linear counterparts, all views are proof terms and have no runtime representation. They simply aid the process of type checking.

```ats
prfun
array_v_split
  {l:addr}{xs:stmsq}
  {n:int}{i:nat | i <= n}
(
  pf: array_v(l, xs, n), i: int (i)
) : (
  array_v (l, take(xs, i), i)
, array_v (l+i, drop(xs, i), n-i)
) (* end of [array_v_split] *)
```

Where take returns a stream consisting of the first i elements in xs, and drop contains all elements from index i and on. They are given the following interpretation in SMT-Lib 2.

```
(assert (forall ((A (Array Int Int)) (i Int) (j Int))
  (=> (<= 0 j)
    (= (select (take A i) j) (ite (< j i) (select A j) 0)))))


(assert (forall ((A (Array Int Int)) (i Int) (j Int))
  (=> (and (<= 0 i) (<= 0 j)) 
    (= (select (drop A i) j) (select A (+ i j))))))
```

As we mentioned before, all linear views must be used properly in the scope they are used. A programmer could specify that a view is preserved in some scope, and consumed (freed) in another. In the case of a preserving scope, we need some way to put arrays back together. The following unsplit function accomplishes just that.

```ats
prfun
array_v_unsplit
  {l:addr}
  {xs,ys:stmsq}
  {n,m:int}
(
  pf1: array_v(l, xs, n)
, pf2: array_v(l+n, ys, m)
) :
(
  array_v (l, append (xs, n, ys, m), n+m)
) (* end of [array_v_unsplit] *)
```

Where append is given the following SMT-Lib 2 interpretation.

```
(assert (forall ((A (Array Int Int)) (B (Array Int Int)) (m Int) (n Int) (i Int))
  (=> (>= i 0)
    (= (select (append A m B n) i ) 
      (ite (< i m) (select A i) (select B (- i m)))))))
```

With these tools in place, let's implement a function that dereferences the ith element in an array using pointer arithmetic.

```ats
fun array_get_at
  {l:addr}{xs:stmsq}
  {n:int}{i:nat | i < n}
  (pf: !array_v(l, xs, n) | p: ptr(l), i: int i) : T(select(xs, i))
```

In the following, we first split the original array view to obtain a view of the element at position (p+i), dereference the pointer, and then reconstruct the array view using unsplit. Internally, Z3 is able to determine from our interpretations of take, drop, cons, and append that our return value is in fact the ith element in the original array.

```ats
implement
array_get_at
  (pf | p, i) = x where
{
//
prval (pf1, pf2) = array_v_split (pf, i)
prval array_v_cons (pf21, pf22) = pf2
//
val x = !(p+i)
//
prval ((*void*)) =
  pf := array_v_unsplit (pf1, array_v_cons (pf21, pf22))
//
} (* end of [array_get_at] *)
```

Besides providing interpretations for static functions, the manual effort to write the program is fairly slight, yet Z3 is able to provide a very strong guarantee for its correctness. Of course, this is a trivial example, but showing Z3 automatically solving constraints involving pointer arithmetic enables us to construct the efficient version of quicksort we presented at the beginning of the tutorial.

## Constructing Quicksort

Now, let's use what we know about linear views of arrays and sequences interpreted by Z3 to build an efficient and verified version of quicksort using pointer arithmetic. The first thing we need is a type for a function that partitions an array by a user chosen pivot. We can use static types to guarantee the following

* The resulting array at pointer l is of length n
* That the array is partitioned by a pivot
* That the pivot in the resulting array is the element xs[piv] which the user gave as the desired pivot

This yields the following signature in ATS.

```ats
fun
partition {l:addr} {xs:stmsq} {pivot,n:nat | pivot < n} (
  array_v (l, xs, n) | p: ptr l, int pivot, int n
): [p:nat | p < n]
   [ys: stmsq | partitioned (ys, p, n); 
    select(xs, pivot) == select (ys, p)
   ] (array_v (l, ys, n) | int p) =
```

More props may be added to assure that the result is a permutation of the original array, but this example is already verbose enough :D. To partition, we first swap the desired pivot to the last spot in the array. Then we start at i=0 and maintain a partition index. Any element to the left of this index is less then or equal to a[n-1] (the pivot). Any element to the right of this index, up until the current element i, must be greater than or equal to the pivot. Let us use the following definitions to describe these invariants to the SMT solver.

```
 (define-fun part-left ((a (Array Int Int)) (pindex Int) (last Int))
  (forall ((i Int))
    (=> ( and (<= 0 i) (< i pindex) )
      (<= (select a i) (select a last)))))

 (define-fun part-left ((a (Array Int Int)) (i Int) (pindex Int) (last Int))
  (forall ((j Int))
    (=> ( and (<= pindex j) (j < i))
      ((select a last) <= (select a j)))))
```

During this loop, if we encounter an element less than the pivot, we swap it with the current pindex and increment pindex by one and maintain the above invariants.

When we're all done, and i = n - 1, we've reached the pivot, and so from our loop invariants, if we swap the pointer at n-1 with pindex, we'll have an array partitioned by the desired pivot. As an added bonus, we have a termination metric to enforce the loop to terminate. This metric is translated into constraints by ATS that Z3 will solve as well. An implementation that fits this specification is given below, where we make heavy use of views to reason about our pointer arithmetic.

```ats
implement
partition {l}{xs}{pivot,n} (pf | p, pivot, n) = let
  val pi = p + pivot
  val pn = p + (n - 1)
  val () = array_ptrswap {l}{..}{..}{pivot, n-1} (pf | pi, pn)
  val xn = array_ptrget {l}{..}{..}{n-1} (pf | pn)
  //
  fun loop {ps:stmsq} {i, pind: nat | pind <= i; i <= n-1 |
    part_left (ps, pind, n-1);
    part_right (ps, i, pind, n-1);
    select (ps, n-1) == select (xs, pivot)
  } .<n-i>. (
    pf: array_v (l, ps, n) | pi: ptr (l+i), pind: ptr (l+pind)
  ): [ys:stmsq] 
     [p:nat | p < n;
      partitioned (ys, p, n); select (ys, p) == select (xs, pivot)] (
    array_v (l, ys, n) | int p
  ) =
    if pi = pn then let
      val () = array_ptrswap {l}{..}{..}{pind,n-1}(pf | pind, pn)
    in 
      (pf | ptr_offset{l}{pind}(pind))
    end
    else let
      val xi = array_ptrget {l}{..}{..}{i} (pf | pi)
    in
      if xi < xn then let
          val () = array_ptrswap {l}{..}{..}{i, pind}(pf | pi, pind)
        in
          loop {swap_at(ps,i,pind)}{i+1, pind+1} (pf | pi+1, pind+1)
        end
      else
        loop {ps} {i+1,pind} (pf | pi+1, pind)
    end
  //
in loop {swap_at(xs,pivot,n-1)} {0,0} (pf | p, p) end
```

Just as with insertion sort, there is an important axiom we cannot completely leave Z3 to infer automatically. In our implementation of quicksort, this axiom is that after we sort two sub arrays of a partitioned array, combining them forms a sorted array. What we lose here is that after quicksort we only have a proof that we have two sub arrays that are sorted (with no reference to them being a permutation of the original array). To address this, we put forth the following lemma.

```ats
absprop PARTED (l:addr, xs: stmsq, p:int, n:int)

extern
praxi 
PARTED_make 
  {l:addr}
  {n,p:nat | p < n}
  {xs:stmsq | partitioned (xs, p, n)}
(!array_v (l, xs, n), int p): PARTED(l, xs, p, n)

extern
praxi partitioned_lemma
  {l:addr}
  {xs:stmsq} {p,n:nat | p < n}
  {ls,rs:stmsq} (
  PARTED(l, xs, p, n),
  !array_v (l, ls, p),
  !T(select(xs, p)) @ l+p,
  !array_v (l+p+1, rs, n - p - 1)
): [
  lte (ls, p, select(xs, p)); lte (select(xs, p), rs, n - p - 1)
] void
```

This allows us to state to the SMT solver, if I have a proof that xs is partitioned by p, then the element select(xs, p) is greater than every element in the left sub array, and it is less then or equal every element in the right array. In short, it allows us to assert the array is still partitioned. Using this, implementing and verifying quicksort with the previously defined partition function is fairly straightforward.

```ats
fun quicksort {l:addr} {xs:stmsq} {n:nat} .<n>. (
  pf: array_v (l, xs, n) | p: ptr l, n: int n
): [ys:stmsq | sorted (ys, n)] (
  array_v (l, ys, n) | void
) =
  if n <= 1 then
    (pf | ())
  else let
    val pivot = random_int (n)
    val (pf | pi) = partition (pf | p, pivot, n)
    val parted = PARTED_make (pf, pi)
    //
    prval (left, right) = array_v_split (pf, pi)
    prval array_v_cons (pfpiv, right) = right
    //
    val (left  | ()) = quicksort (left | p, pi)
    val (right | ()) = quicksort (right | (p+pi+1), n - pi - 1)
    //
    prval () = partitioned_lemma (parted, left, pfpiv, right)
    //
    prval (pf) = array_v_unsplit (left, array_v_cons (pfpiv, right))
  in
    (pf | ())
  end
```

## Generics

In our examples above, all functions worked with an abstract type T that was indexed with a stamp. An excellent question is how could we modify these functions so that they work on any type? Recall tha ATS is largely a front end for C, and so it shares its unboxed representation for data. Therefore, we can replace our boxed type T with a flat unboxed type as follows

```ats
abst@ype T(a:t@ype, s: stamp) = a
```

The size of this type will simply be that of a, yet every instance of T will have a corresponding stamp that allows us to enforce orderedness constraints. This allows us to define a homomorphism between a type "a" and the set of integers. Therefore, indexing a generic array with a static array of stamps still captures the invariants we've discussed so far, but in a polymorphic way. Let us use this to define polymorphic versions of list and array.

```ats
datatype
list (a:t@ype, stmsq, int) =
  | list_nil (a, nil, 0) of ()
  | {n:nat} {x:stamp} {xs:stmsq}
    list_cons (a, cons(x, xs), n+1) of (T(x), list(a, xs, n))

dataview
array_v (a:t@ype, addr, stmsq, int) = 
  | {l:addr}
    array_v_nil (a, l, nil, 0) of ()
  | {l:addr} {xs:stmsq} {x:stamp}{n:int}
    array_v_cons (a, l, cons(x, xs), n+1) of (T(x) @ l, array_v (a, l+sizeof(a), xs, n))
```

The case of array is an interesting one, as it allows us to enforce correct pointer arithmetic by modelling the sizeof() function in the statics. Using the above definitions, we can use templates in ATS to generate completely generic versions of the sorting functions we have presented in this tutorial.

## Conclusion

This is certainly exciting, as these programs are efficient in their use of memory but also verified as correct. This type of program construction may be possible without Z3, but we hope to demonstrate here that integrating a powerful automated reasoning tool into our constraint solver can make the process much simpler and intuitive. We hope to continue on this path by adding support for new theories being purposed for SMT-Lib 2 (Sequences, IEEE Floats) and possibly utilizing full fledged interactive proof assistants such as Coq, Isabelle, and ACL2.

This style of programming/debugging by interacting with a verifier is difficult to illustrate with finished examples. Nonetheless, it's quite a more enjoyable experience reasoning about programs formally rather than simply observing their outputs. Moreover, introducing formal reasoning into a program's construction leads to some assurance to its correctness in a way that is difficult to capture using only run-time testing.
