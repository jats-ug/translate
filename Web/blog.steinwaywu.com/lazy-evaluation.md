# 遅延評価

(元記事は http://blog.steinwaywu.com/lazy-evaluation/ です)

(これはボストン大学 CS320 コース用の記事です)

In most programming languages, expressions are evaluated right after it is bound to a variable.
This is **eager evaluation**.

However, you can't always evaluate an expression immediately.

```ats
// pseudo code
val natlist = 0 :: 1 :: 2 :: ....
```

An infinite list can't be expressed in a eager evaluation manner because that will cause a infinite loop and eventually corrupt your stack.

For such a reason, we need **lazy evaluation** which will defer the computation of an expression until it is actually needed. This is also known as **call-by-need**.

```ats
// pseudo code
val lazylist = $delay (0 :: 1 :: 2 :: ...)

// here we have to evaluate the first 10 elements
val _ = show (lazylist, 10)
```

Lazy evaluation has other advantages. Since lazy evaluation usually comes together with memorization, efficiency could be improved. Suppose we have a complex experssion consists of several subexpression, e.g. `(1+2)-(1+2)`

* for eager evaluation: every subexpression `(1+2)` will be evaluated right away no matter they are the same or not.
* for lazy evaluation with memorization: subexpression evaluation will be deferred until the evaluation of top-level expression `(...) - (...)`. Also, when a subexp `(1+2)` is evaluated, the result will be memorized and when the same subexp is evaluated for the second time, we just retrieve the result without actual computation.

## Lazy Evaluation in ATS

In ATS, we use `$delay (exp)` to delay the evaluation of some expression `exp`. And we use `lazy ()` to get a lazy type from an existing type.

If the type of `exp` is `T`, then `$delay (exp)` will have type `lazy (exp)`.

We use `!exp` to force the evaluation of some lazy expression exp.

## Example and Exercise

Please read the following code carefully, especially the definition of lazy list, those signatures of lazy functions, and an example of an infinite fibonacci number list.

Please implement

* `from (n:int)`: generate an infinite integer list from n
* `sieve (xs:lazy(list(int))): lazy (list (int))`: generate an infinite prime number list using this algorithm
* `intpairs(n:int): lazy (list (@(int, int)))`: generate integer pairs as shown here

```
(0,0),
(0,1),(1,0),
(0,2),(1,1),(2,0),
...
```

```ats
#include "share/atspre_staload.hats"
staload UN = "prelude/SATS/unsafe.sats"
//make patscc -o lazy lazy.dats -DATS_MEMALLOC_LIBC

datatype list (a:t@ype) = 
	| list_cons of (a, lazy (list (a)))
	| list_nil of ()


#define nil list_nil
#define cons list_cons
#define :: list_cons

exception EMPTY_EXN of ()

extern fun from (int): lazy (list (int))

extern fun {a:t@ype} head (lazy (list (a))): a
extern fun {a:t@ype} take (lazy (list (a)), int): lazy (list (a))
extern fun {a:t@ype} tail (lazy (list (a))): lazy (list (a))
extern fun {a:t@ype} get (lazy (list (a)), int): a
extern fun {a,b:t@ype} {r:t@ype} zip (lazy (list (a)), lazy (list (b)), (a, b) -<cloref1> r): lazy (list (r))
extern fun {a:t@ype} filter (lazy (list (a)), a -<cloref1> bool): lazy (list (a))
extern fun {a:t@ype} interleave (lazy (list (a)), lazy (list (a))): lazy (list (a))
extern fun {a:t@ype} merge (lazy (list (a)), lazy (list (a)), (a, a) -<cloref1> int): lazy (list (a))

symintr show
extern fun show_int (lazy (list (int)), int): void
extern fun show_pair (lazy (list (@(int, int))), int): void
overload show with show_int
overload show with show_pair

implement {a} interleave (xs, ys) = 
	$delay (
		case+ !xs of
		| cons (x, xs) => (x :: interleave (ys, xs))
		| nil () => !ys
	)


implement {a} merge (xs, ys, f) = 
	$delay (
		let
			val- cons (x, xs0) = !xs
			val- cons (y, ys0) = !ys
		in if f (x, y) < 0 
			then cons (x, merge (xs0, ys, f)) 
			else cons (y, merge (ys0, xs, f))
		end
	)


implement from (x) = $delay (x :: from (x+1))

implement {a} head (xs) = case+ !xs of
	| cons (x, xs) => x 
	| nil () => $raise EMPTY_EXN ()

implement {a} tail (xs) = case+ !xs of
	| cons (_, xs) => xs
	| nil () => $delay (nil ())

implement {a} take (xs, n) = 
	if n = 0 
	then $delay (nil ())
	else $delay (head (xs) :: take (xs, n-1))

implement {a} get (xs, n) = 
	if n = 0
	then head (xs)
	else get (tail (xs), n-1)


implement show_pair (xs, n) = 
	if n = 0
	then () where {
		val- @(r, c) = head<@(int, int)> (xs)
		val _ = println! ("(", r, ",", c, ")")
	} else show_pair (tail (xs), n-1) where {
		val- @(r, c) = head<@(int, int)> (xs)
		val _ = print! ("(", r, ",", c, ") : ")
	}


implement show_int (xs, n) = 
	if n = 0
	then () where {
		val _ = print (head (xs))
		val _ = print_newline ()
	} else show_int (tail (xs), n-1) where {
		val _ = print (head (xs))
		val _ = print_string (" : ")
	}

implement {a,b} {r} zip (xs, ys, f) = 
	$delay (f (head (xs), head (ys)) :: zip (tail (xs), tail (ys), f))

implement {a} filter (xs, f) = 
	if f (head (xs)) 
	then $delay (head (xs) :: filter (tail (xs), f))
	else filter (tail (xs), f)


extern fun sieve (lazy (list (int))): lazy (list (int))

implement sieve (xs) = 
	case+ !xs of 
	| nil () => $raise EMPTY_EXN ()
	| cons (x, xs) => $delay (x :: sieve (filter (xs, lam (e) => if e mod x = 0 then false else true)))

extern fun intpairs (int): lazy (list (@(int, int)))

implement intpairs (n) = let
	fun col (r: int, c: int): lazy (list (@(int, int))) =
		$delay (@(r, c) :: col (r, c + 1))

	fun row (r: int): lazy (list (@(int, int))) =
		$delay (@(r, 0) :: merge (row (r + 1), col (r, 1), lam (x, y) => let
					val- @(xr, xc) = x
					val- @(yr, yc) = y
				in 
					xr + xc - yr - yc
				end
			))
in 
	row (n)
end
	
implement main0 () = () where {
	val rec xs:lazy (list (int)) = $delay (1 :: $delay (1 :: zip<int,int><int> (xs, tail (xs), lam (x, y) => x + y)))
	val _ = show (xs, 10)
	val prime = sieve (from (2))
	val ip = intpairs (0)
	val _ = show (prime, 10)
	val _ = show (ip, 100)
}
```
