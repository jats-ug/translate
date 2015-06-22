# ATSにおける証明された効率的なプログラム: qsort

(元記事は http://www.illtyped.com/projects/patsolve/qsort.html です)

By Will Blair

## 説明

このチュートリアルは、ATS のために Z3 SMT ソルバを使って私達が設計した、新しい制約ソルバの外観です。
静的な部分が ATS においてどのように動作するのか、新しい種と述語を用いてそれらをどうやって拡張できるか、そして述語の意味を制約ソルバに与えられるか、を議論します。
最後に、以前は証明できなかった、もしくは大幅は手動の努力が必要だったプログラムの関数的な正確さを証明するために、この手法をどのように使えるのか示します。
同時に、この成果のゴールは ATS コードを書く際の手動の定理証明を取り除くことではないことを強調しておきます。

代わりに、私達は自動と手動の定理証明の組み合わせを支持しています。
このアプローチに対する私達の理論的根拠は、型エラーメッセージの有用性を維持する要求を解決することです。
実際、自動化のためにあまりに多くのプログラムの証明を作ると、ツールは失敗するかもしれません。
そしてそれはなぜ失敗したのか、明確ではないかもしれないのです。
手動で証明すると、エラーメッセージがあなたの証明が失敗した正確な位置を指摘します。
そしてその理由は明確に与えられるでしょう。
同時に、手動の証明は時間がかかり、退屈になりえます。
このチュートリアルでは、2つのアプローチを一緒に使う方法を探訪します。
特に、ポインタ演算と線形リソースを必要とする低レベルなプログラムに注意をはらいます。
プログラムが意図した通りに動くこと、つまり関数的な正しさについて自動証明を使い、メモリ安全性の検証について手動証明を使います。

この成果で開発された制約ソルバは ATS の外部で動作するよう作られました。
通常は、型検査のソルバは内部で制約をかけますが、より強力な検査を可能にするために、ATS2 コンパイラにオプションを渡してなじみのある JSON フォーマットで制約をエクスポートできます。
制約ソルバは単純なツールで、この情報を構文解析し、プログラムのその形式的な仕様の間の矛盾を見つけようとします。
論理的な主張に対する反例を見つけることは SMT ソルバにとって基本的な仕事です。
そのため Z3 はこの役目に適合できます。
Z3 はまた、伸長配列、実装、固定長ビットベクタなどソフトウェア開発の共通要素のような私達の静的言語の範囲外の仮定を理解します。

これは高位ではとても好ましいですが、どのように Z3 の動作の上に制約ソルバを正確に構築すれば良いでしょうか？
また、どうすればその広範囲な判定力を効果的に利用できるでしょうか？
このチュートリアルでは、プログラムの不変条件における ATS での静的な推論の例を示すことで、これらの疑問に答えようと思います。
私達は動的な値に適用された静的な型のない ATS コードからスタートします。
それからプログラムの型をより形式的な仕様に徐々に洗練していきます。
その過程で、私達はプログラムの正確さの保証が結果としてより強くなることを主張します。

## ゴール

このチュートリアルのゴールは、私達が Z3 の上に構築した制約ソルバが、次の C コードで与えられたクイックソートに対応する効率的で証明された ATS プログラムを構築することに使えることを示すことです。

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

私達はこのコードを C 言語で設計しませんでした。
実際、

私達は、ATS のクイックソート実装を構築して証明するのに制約ソルバを使い、それからその正確さを証明してから対応する C 言語実装を書きました。
ATS プログラムは C 言語にコンパイルされて直接実行されるので、C 言語における上記の実装はまったく重複しています。
これはソフトウェア検証における私達の哲学を示すためです。
私達はプログラマ中心の方法でプログラムを証明することに興味があります。
プログラマはプログラムの構築中に証明器とやりとりするのです。
そうすることで、プログラマは彼の推論の早い段階で論理的なエラーをとらえることができるのです。
コードがまったくコンパイルされる前に。
このチュートリアルの残りでは、この制約ソルバがこのプログラミングのスタイルをどのように促進するのか、いくつかのプログラム例を構築/証明するのにどのように使えるのかを述べます。

## ATS の基礎

より進んだ機能を除いて ATS を議論しましょう。
この点では、ATS は ML のような標準的な関数型プログラミング言語に似ています。
なんらかの一般的な型 `T` の連結リストを表現する単純なデータ型があると仮定します。

```ats
abstype T

datatype list =
    | list_nil of ()
    | list_cons of (T, list)
```

ここで、リストの n 番目の要素を得る単純なアクセサ関数を定義しましょう。
パターンマッチを使えばこれは簡単です。

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

この関数を見るかぎり、少しアドホックに思えます。
もし `list_nth` の各呼び出しにおいてエラーが起きないことを保証できるなら、この関数をよりきれいにできます。
つまり、`list_nth` に渡される値 `i` が常にそのリストの長さより小さいのであれば。
これはデータ型に静的な要素を追加すれば簡単です。

## 静的な型を用いて洗練する

`list_nth` のエラー無し版を得るために、リストのそれぞれの値を、そのリストの長さを表わす静的な整数でインデックスしましょう。
このインデックスは型検査時でのみ存在し、実行時のプログラムでは具体的な表現やオーバーヘッドを持たないことに注意してください。
私達がそれを導入するのは、`list_nth` に与えられたリストの長さよりも `list_nth` に与えられたインデックスが常に小さいことを推論するためです。
この不変条件は次のリスト定義を使って強制することができます。

```ats
datatype list (n:int) =
    | list_nil (0) of ()
    | {n:nat}
       list_cons (n+1) of (T, list(n))
```

初心者にとってこの構文は混乱しやすいかもしれません。
文字列 `{n:nat}` は `list_cons` コンストラクタで使う全称量化子です。
これは全ての自然数 `n` について、長さ `n` のリストと型 `T` のコンスが、長さ `n+1` のリストを生じることを意味しています。
これは単純で、私達が強制したい長さの不変条件を正確に捕捉しています。
ここで、全てのリスト `xs` について、全てのインデックスが `xs` の長さより小さくなることを保証するために `list_nth` を再定義してみましょう。
結局のところ `xs` の長さは正であると主張することに注意してください。

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

これはより良いもので、プログラマが `list_nth` を使うときは、インデックスがリストの長さより常に小さいことを示す証明を提供しなければならないことが、型で定義されています。
これは少し大胆な主張です。
制約ソルバは、この実装が範囲外エラーを発生しないことをどうやって知ったのでしょうか？
一般の SMT-Lib2 構文を使って、`list_nth` 関数を型検査するための、制約ソルバと Z3 の自動的なやりとりを見てみましょう。
内部的には、Z3 の C API に私達が書いた ATS バインディングを通じてこのやりとりは行なわれます。

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

上記の全てが満たされたとしたら、もし有効な入力 (すなわち `i` が `n` より小さい) を与えると、リストの範囲外アクセスがないことになります。
これはもちろん素晴しいことです。
しかしこれは私達が返す要素が実際にそのリストの `i` 番目の要素なのか保証していません。
実際、返り値がリスト中の要素であるかどうかさえなんの保証もないのです!
このようなより強い保証を得るために、さらに関数の型で強制を捕捉するために、Z3 の配列理論 (array theory) を使うことができます。
これはオリジナルの制約ソルバでは扱いにくかった不変条件を捕捉できます。

## ストリームとしての Z3 配列

前の例では、インデックスとリストの長さを含む不変条件が私達のコードで強制されているかどうか自動的に検査するために、整数に対する Z3 の解釈を使いました。
配列の知見を使うことで、より多くの性質を静的に保証できるようになります。
リストの型が Z3 で解釈される "静的な" ストリームでインデックスされるように、これを使ってみましょう。
ATS で次のような静的な種 (sort) があるとします。

```ats
datasort stampseq = (* abstract *)
```

ATS の型システムはその認識がないので、この種は "抽象的" であると言っています。
代わりに、私達のソルバで強制を構文解析して、種 `stampseq` の全ての定数に Z3 の `(Array Int Int)` 種を与えます。
これで Z3 の境界のない配列としてスタンプの無限ストリームをモデルできます。
シーケンスにおける2つの一般的なコンストラクタと SMT-Lib2 でのそれらの解釈を定義してみましょう。

```ats
stacst stampseq_nil  : () -> stampseq
stacst stampseq_cons : (stamp, stampseq) -> stampseq
```

Z3 の観点からは、解釈されない関数があるだけです。
SMT-Lib2 ファイルで次の主張をすることで、それらに意味を与えることができます。
このファイルは制約ソルバによって構文解析され、Z3 のコンテキストに渡されます。

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

リストの定義と `T` にインデックスを追加して、スタンプの静的なストリームとしてリストの中身を表わしましょう。

```ats
abstype T(x:stamp)

datatype list (stmsq, int) = 
    | list_nil (nil, 0) of ()
    | {xs:stmsq} {n:nat} {x:stamp}
       list_cons (cons(x, xs), n+1) of (T(x), list(n))
```

この定義によると、`list_nth` の挙動を正確に捕捉できます。
つまり、それはリストの `i` 番目の要素を返すのです。
ここで、この動作を表現するために、配列理論における静的な "select" 関数を使います。

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

さて、もし `i = 0` のとき `x` の他の値を返そうとすると、`x` のみが `select (xs, i)` 表わすので、型検査器はエラーを発生するでしょう。
これもまた、ストリーム操作の解釈を与えれば、完全に自動的にこの関数の正確さを Z3 が決定できる点で興味深いものです。
次の例ではこのアイデアを進めて、正しくソートされたリストを返すことが保証された挿入ソートの実装を示します。

## 連結リストのソート

ソートアルゴリズムの証明に最初に必要なことは SMT ソルバでシーケンスのソートを表現する方法です。
SMT-Lib 2 で、次の定義を使うことができます。

```
(define-fun sorted ((A (Array Int Int)) (n Int)) Bool
  (forall ((i Int) (j Int))
    (=> (<= 0 i j (- n 1))
        (<= (select a i) (select a j)))))
```

ソート関数の実装は自明でしょう。
不幸なことですが、連結リストのソートにおける最も重要な公理の一つは Z3 で自動的に解決できません。
その公理は、どのようなソート済みリストの tail もまたソート済みリストである、というものです。
これは次のような SMT 解釈によって示され、この公理の正当性が疑われた場合にはこの解釈は "unknown" を返します。

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

幸運にも、ATS には定理証明システムがあります。
公理としてのこの特性を宣言でき、それをプログラム中で使うことができるのです。
次のコードはシーケンスのコンス操作のために、(ここでは抽象命題として表現された) シーケンスのソートに関連した公理のセットです。
証明が終われば、`xs` がソートされていることを主張するために `SORTED_elim` を呼ぶことができます。

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

最初に言及したこの問題を解決する主な公理は `SORTED_uncons` です。
なぜなら、それがリストの先頭が常にソートされた tail の先頭以下であるという主張を追加するからです。
次の関数を実装するためにこれらの公理を使ってみましょう。

```ats
extern
fun sort
  {xs:stmsq}{n:int}
  (xs: list (xs, n)): [ys:stmsq] (SORTED (ys, n) | list (ys, n))
```

すなわち、どのような長さ `n` のリスト `xs` が与えられても、私達が返すソート済みの長さ `n` のリスト `ys` が存在するのです。
挿入ソートとしてこれを実装するためには、ソート済みリストの正しい位置に要素 `x` を挿入する関数が必要です。

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

これは強い描写になっています。
どのようなソート済みリスト `xs` が与えられても、元のリスト `xs` に `x0` を挿入したソート済みリストを必ず返します。
これは `insord` の結果が、インデックス `i` に挿入された新しい要素と元のリストを生成することを保証しています。
次のコードは SMT-Lib 2 における挿入関数の解釈です。

```
(assert (forall ((A (Array Int Int))(x Int) (i Int) (j Int))
  (=> (and (<= 0 j) (<= 0 i))
    (= (select (insert A i x) j)
     (ite (< j i) (select A j)
       (ite (= j i) x (select A (- j 1))))))))
```

この解釈を Z3 は使って、ソート済みであるという証明と共に `insert(xs, i, x0)` を生成する `insord` の最終的な実装を型検査できます。

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

この実装で使われている全ての命題は、定理証明の目的に使われることに注意してください。
つまり、それらは追加のオーバーヘッドにならず、実際に制約解決の後では完全に消えさります。
この関数を用いると、リストのソートを書くのは簡単で、次のようになります。

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

## 線形観 (Linear Views)

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
