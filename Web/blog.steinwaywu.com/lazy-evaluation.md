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
