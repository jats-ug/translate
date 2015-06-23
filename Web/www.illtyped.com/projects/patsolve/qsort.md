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

連結リストを表わすストリームを使うことについて話してきましたが、それはプログラムにおける正しいポインタ演算の強制にどのように役立つのでしょうか？
ATS はプリミティブとしてポインタ型を持っていて、それはまたメモリ上のリソースを追跡する観です。
"観" はなんらかのデータ構造がメモリ上の位置に有ることを表わす単純な証明書です。
これは次のような例で示されます。

```ats
var counter: int

val pf = view@ counter (* pf is of type int @ counter *)
val p = addr@ p        (* p is of type ptr counter *)
```

全ての観は "線形" で、リソースを紛失しないようにプログラマは全ての観を消費しなければなりません。
逆に上記のように静的な変数から観を受け取ったら、プログラマはそれを元の場所に戻す義務があります。
リストを帰納的に定義したので、配列の観を定義して、それらを静的なストリームでインデックスすることができます。
これは次のような定義で与えられます。

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

ここでポインタ演算を使っていることに注意してください。
そのため配列における要素の観を得ることができます。
配列の個々の部分を処理するために、配列をサブの観に分割したくなるでしょう。
次のような証明関数を使うことでこれが可能になります。
非線形の部分と同様に、全ての観は証明の項で実行時表現を持たないことに注意してください。
それらは型検査に役立ちます。

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

ここで、`take` は `xs` の最初の `i` 個の要素から成るストリームを返し、`drop` はインデックス `i` 以降の全ての要素を含みます。
これらには次のような SMT-Lib 2 の解釈が与えられます。

```
(assert (forall ((A (Array Int Int)) (i Int) (j Int))
  (=> (<= 0 j)
    (= (select (take A i) j) (ite (< j i) (select A j) 0)))))


(assert (forall ((A (Array Int Int)) (i Int) (j Int))
  (=> (and (<= 0 i) (<= 0 j)) 
    (= (select (drop A i) j) (select A (+ i j))))))
```

以前言及しましたが、全ての線形観はそれらのスコープにおいて適切に使われなければなりません。
観がどこかのスコープで維持され、別の場所で消費 (解放) されることをプログラマは明記します。
維持しているスコープについては、配列を一緒に戻すことができます。
次の `unsplit` 関数はこれを実現しています。

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

このとき、`append` には次のような SMT-Lib 2 解釈が与えられています。

```
(assert (forall ((A (Array Int Int)) (B (Array Int Int)) (m Int) (n Int) (i Int))
  (=> (>= i 0)
    (= (select (append A m B n) i ) 
      (ite (< i m) (select A i) (select B (- i m)))))))
```

これらの関数を用いて、ポインタ演算を使って配列の `i` 番目の要素をデリファレンスする関数を実装してみましょう。

```ats
fun array_get_at
  {l:addr}{xs:stmsq}
  {n:int}{i:nat | i < n}
  (pf: !array_v(l, xs, n) | p: ptr(l), i: int i) : T(select(xs, i))
```

次のコードは、`(p+i)` の位置にある要素の観を得るために元の配列の観を分割して、そのポインタをデリファレンスし、その後 `unsplit` を使って配列の観を再構成します。
内部では、`take`, `drop`, `cons`, `append` の解釈から、返り値が実際に元の配列の `i` 番目の要素であることを、Z3 は決定できます。

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

静的な関数に解釈を提供することを除いて、プログラムを書くための手動の作業は少しです。
けれども Z3 はその正確さにとても強い保証を与えることができます。
もちろん、これは自明な例です。
しかし、ポインタ演算を含む Z3 の自動的な制約解決によって、このチュートリアルの最初に示した効率的なクイックソートを構築できることを示しています。

## クイックソートを構築する

ここで、ポインタ演算を用いた効率的で証明されたクイックソートを構築するために、配列の線形観と Z3 によって解釈されたシーケンスを使ってみましょう。
最初に、ユーザが選んだピボットで配列を分割する関数を表わす型が必要になります。
静的な型を使って次を保証できます。

* ポインタ `l` にある配列の長さは `n` である
* その配列はピボットによって分割されている
* 配列内のそのピボットは、ユーザがピボットとして与えた要素 `xs[piv]` である

これは ATS で次のようなシグニチャを作ります。

```ats
fun
partition {l:addr} {xs:stmsq} {pivot,n:nat | pivot < n} (
  array_v (l, xs, n) | p: ptr l, int pivot, int n
): [p:nat | p < n]
   [ys: stmsq | partitioned (ys, p, n); 
    select(xs, pivot) == select (ys, p)
   ] (array_v (l, ys, n) | int p) =
```

結果が元の配列の置換であることを保証するために、より多くの命題を追加できますが、この例では十分です。
分割のために、はじめに所望のピボットと配列の最終地点を交換します。
その後、`i=0` からはじめて、分割インデックスを作ります。
このインデックスより左の要素はピボット `a[n-1]` 以下です。
このインデックスの右から要素 `i` までの要素は、ピボット以下です。
SMT ソルバでこれらの不変条件を表現する次の定義を使いましょう。

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

このループ中では、ピボットより小さい要素を見つけたら、それを現在の `pindex` と交換して `pindex` を1増やし、上記の不変条件を維持します。

すべて終わった時には `i = n - 1` で、ピボットに到達していて、そしてループの不変条件から、もし `n-1` のポインタと `pindex` を交換したら、所望のピボットで分割された配列が得られたことになります。
おまけに、停止性メトリクスはこのループが停止することを強制しています。
このメトリクスは、Z3 によって解決できる ATS における強制に変換されます。
この仕様に適合する実装は以下のようになります。
このコードでは、ポインタ演算を推論するために観を頻繁に使っています。

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

挿入ソートを用いると、Z3 に自動的に推論させることができない重要な公理があります。
クイックソートのこの実装ではその公理は、分割された配列の2つのサブ配列をソートした後、それらを結合して1つのソート済み配列を作るものです。
ここでも問題は、クイックソートの後でのみソートされた (元の配列の置換でへの参照のない) 2つのサブ配列の証明を持つということです。
これに対処するため、次の補題を導入します。

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

これは SMT ソルバに宣言できます。
もし `xs` が `p` で分割されたという証明があるなら、要素 `select(xs, p)` はその左側のサブ配列のどの要素より大きく、またそれはその右側の配列のそれぞれの要素以下です。
要するに、その配列が分割されていることを主張することができるのです。
これを使って、前に定義した `partition` 関数を用いて `quicksort` を素直に実装/証明できます。

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

## ジェネリックス

上記の例では、抽象型 `T` と動作する全ての関数はスタンプでインデックスされていました。
どうやって任意の型で動作できるようにこれらの関数を修正すればいいか、というのは良い質問です。
おおざっぱに言って ATS はC言語のフロントエンドなので、アンボックス化データ表現を共有できます。
したがって、私達のボックス化型 `T` を次のようなフラットなアンボックス化型で置換できます。

```ats
abst@ype T(a:t@ype, s: stamp) = a
```

この型のサイズは単純に `a` のサイズです。
けれども `T` のあらゆるインスタンスは順序を強制する対応するスタンプを持ちます。
これは型 `a` と整数の集合の間に準同型の定義を可能にします。
したがって、スタンプの静的な配列を用いた一般的な配列のインデックスはまた、これまで議論してきた不変条件を多相的な方法で捕捉します。
これを使って多相的なリストと配列を定義してみましょう。

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

配列の場合は興味深いものです。
静的に `sizeof()` 関数をモデリングすることで、正確なポインタ演算を強制できるのです。
上記の定義を用いて、ATS のテンプレートを使ってこのチュートリアルで示したソート関数の総称的なバージョンを作ることができます。

## 結論

これらのプログラムはメモリの使用において効率的であるにもかかわらず正確さが証明されているという点で、これは興味深いでしょう。
プログラムを構築するこの型は Z3 がなくても可能ですが、強力な自動的推論ツールを私達の制約ソルバに統合することで、このプロセスをより単純により直感的にできることを、ここでは示したかったのです。
この方向に進んで SMT-Lib 2 の新しい理論 (シーケンスや IEEE 浮動小数点数) をサポートしたいと私達は望んでいます。
またできれば Coq, Isabelle, ACL2 のような対話型の証明アシスタントも利用したいと考えています。

検査器と相互作用してプログラミング/デバッグをするこのスタイルでは最終的な例を示すのは難しいかもしれません。
それでもなお、プログラムの出力を観察するよりも、形式的にプログラムの推論するのは楽しい体験でしょう。
その上、プログラムの構築に形式的な推論を導入することで、実行時のテストで捕捉することが困難な正確さに対する自信につながります。
