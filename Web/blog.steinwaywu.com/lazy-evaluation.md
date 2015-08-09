# 遅延評価

(元記事は http://blog.steinwaywu.com/lazy-evaluation/ です)

(これはボストン大学 CS320 コース用の記事です)

ほとんどのプログラミング言語では、式は変数を束縛した後、右に向かって評価されます。
これは**先行評価**です。

けれども、常に式を即時に評価できるわけではありません。

```ats
// pseudo code
val natlist = 0 :: 1 :: 2 :: ....
```

無限ループに陥いって最終的にスタックを食い潰してしまうために、無限リストは先行評価の作法で評価することができません。

そのような場合、式の計算を必要になるまで延期する**遅延評価**が必要になります。
これは **call-by-need** としても知られています。

```ats
// pseudo code
val lazylist = $delay (0 :: 1 :: 2 :: ...)

// here we have to evaluate the first 10 elements
val _ = show (lazylist, 10)
```

遅延評価は他にも利点があります。
通常、遅延評価はメモ化も行なうので、効率が良くなります。
`(1+2)-(1+2)` のような、いくつかのサブ式から成る複合の式を想像してください。

* 先行評価では: 全てのサブ式 `(1+2)` は右方向に評価され、それらが同一かどうか構いません。
* メモ化を共なう遅延評価では: サブ式の評価はトップの式 `(...) - (...)` の評価まで延期されます。またサブ式 `(1+2)` が評価されるとき、その結果はメモ化され、同じサブ式が2度目に評価されるときには実際の計算が行なわれず単にその結果を回収します。

## ATS における遅延評価

ATS では、`$delay (exp)` を使って式 `exp` の評価を延期させます。
また `lazy ()` を使うと現存する型から遅延された型を得ることができます。

もし `exp` の型が `T` でならば、`$delay (exp)` は型 `lazy (exp)` を持ちます。

`!exp` を使うと、遅延された式 `exp` の評価を強制します。

## 例と練習問題

次のコード、特に遅延リストの定義、遅延関数のシグニチャ、そして無限フィボナッチ数のリストの例を注意深く読んでください。

以下を実装してください。

* `from (n:int)`: n から無限整数リストを生成する
* `sieve (xs:lazy(list(int))): lazy (list (int))`: このアルゴズムを使って無限素数リストを生成する
* `intpairs(n:int): lazy (list (@(int, int)))`: 次に示す整数のペアを生成する

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
