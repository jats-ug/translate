# BLUISH CODER: ATSにおける依存型

(元記事は http://bluishcoder.co.nz/2010/09/01/dependent-types-in-ats.html です)

依存型は式の値に依存した型のことです。[ATS](http://www.ats-lang.org/)
では依存型を使うことができます。この記事では私がドキュメントと例、そして様々な論文を通じて学んだ、いくかの基本的な使用法を紹介しようと思います。

ATS における依存型を学習するにあたって、私は以下の文書を参考にしました:

* [Tutorial on ATS datatypes](http://www.ats-lang.org/htdocs-old/TUTORIAL/contents/datatypes.html)
* [Tutorial on Parametric Polymorphism and Templates](http://www.ats-lang.org/htdocs-old/TUTORIAL/contents/templates.html)
* [Dependently Typed Datastructures](http://www.ats-lang.org/PAPER/DTDS-waaapl99.pdf)
* [Eliminating Array Bound Checking Through Dependent Types](http://www.cs.bu.edu/~hwxi/academic/papers/pldi98.ps)

これから紹介する例のほとんどは、これらの文書の例に基づいています。

## 種 (Sort) と型 (Type)

依存型の言語のいくつかは、その言語自身で表現された値に依存した型を許します。ATS
の依存型はその意味では限定的です。型が依存できる式は、ATS
言語全体ではなく制限された制約付きの言語によるものです。この制約付きの言語はそれ自身が型付けされていて、混乱を避けるためにこの言語の型は
'種 (sort)' と呼ばれます。
この記事で '種 (sort)' と呼んだ場合はこの制限された言語における型を指します。また
'型 (type)' と呼んだ場合には ATS 言語コアの型を指します。

その違いを例で見てみましょう。次のコードは2つの数を取り、それらを加え合わせた値を返す関数の例です。これは型がその言語の値自身に依存できる仮の依存型言語によるものです:

```ocaml
fun add(a: int, b: int): int (a+b) = a + b
```

上記では、結果の型は正確に2つの数を加え合わせた整数型になっています。なんらかのコードの本体の誤りがあったとしても、2つの数の和は型エラーになります。ATS ではこの関数は次のようになります:

```ocaml
fun add {m,n:int} (a: int m, b: int n): int (m+n) = a + b
```

ここで {m,n:int} が導入されていることに注意してください。これは ATS の型が依存できる値を表現する '制約された言語'
です。ここでは種 int であるような2つの値 m と n を宣言しています。加える2つの引数は a と b で、それらの型はそれぞれ
int m と int n です。それらの型は m と n
によって正確に表わされた整数です。その結果の型はそれら2つの値の合計した整数です。ATS
における依存型 (m, n, m+n) は制約された言語で表現された変数と計算であって、ATS
の変数ではないことに注意してください。a, b, a+b は ATS の変数です。

型の値に対して制約された言語を使うことで、型検査のプロセスを単純化できます。この言語において全ての計算は純粋で、作用を持ちません。この言語において種と関数は必ず終端します。無限ループや型検査中の例外を回避できるのです。

制限された依存型についてより詳細に知りたい場合には、
[Dependent Types in Practical Programming](http://www.ats-lang.org/PAPER/DML-popl99.pdf)
や
[ATS ウェブサイトにあるその他の論文](http://www.ats-lang.org/PAPER/)
を参照してください。

ATS のドキュメントではこの制約された言語は ATS の '静的な部分 (statics)'
と呼ばれ、[ATS user guide](http://www.ats-lang.org/htdocs-old/DOCUMENT/MISC/manual_main.pdf)
の5章で解説されています。

## 単純なリスト

依存型を用いたデータ型に入る前に、ATS
の構文に馴染みのない人のために依存型ではない型についておさらいしてみましょう。整数を含む基本的な
'リスト' 型は次のように ATS で定義できます:

```ocaml
datatype list =
  | nil
  | cons of (int, list)
```

この定義された型を用いて、整数のリストは次の構文を使って生成できます:

```ocaml
val a = cons(1, cons(2, cons(3, cons(4, nil))))
```

リストを操作する関数ではリストを分解するためにパターンマッチを使うことができます:

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

プログラムの完全な例は
[dt1.dats](http://bluishcoder.co.nz/ats/dt1.dats) ([html](http://bluishcoder.co.nz/ats/dt1.html)).
にあります。このプログラムは次のコマンドでビルドできます:

```
atscc -o dt1 -D_ATS_GCATS dt1.dats
```

-D_ATS_GCATS に注意してください。このオプションは ATS コンパイラにガベージコレクタをリンクさせます。このオプションは、データ型で定義した型をヒープに確保するために必要で、その解放にガーベッジコレクタを要求します。

## 多相的なリスト

整数のリストの代わりに、異なる型のリストを作りたいとしましょう。多相的なリストは次のように定義できます:

```ocaml
datatype list (a:t@ype) =
  | nil (a)
  | cons (a) of (a, list a)
```

このリストはどのようなサイズの要素も保持することができます。その要素は t@ype
によって参照されています。このような多相的なリストを操作する関数は多相的でないリストのバージョンと良く似ています。異なる点は、それらの関数が
'テンプレート' 関数であることと、テンプレート型によってパラメータ化されていることです:

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

fun キーワード直後の {a:t@ype} によってその関数はテンプレート関数とみなされます。これは C++
スタイルのテンプレートにとても良く似ています。より詳しくは
[ATS Parametric Polymorphism and Templates tutorial](http://www.ats-lang.org/htdocs-old/TUTORIAL/contents/templates.html)
を参照してください。

この例では整数以外の値のリストを生成でき、さらに list_zip
の定義を追加しています。この例の定義ではタプルのリストを返しています。それぞれのタプルは元のリストの要素を含んでいます。

完全なプログラム例は
[dt2.dats](http://bluishcoder.co.nz/ats/dt2.dats) ([html](http://bluishcoder.co.nz/ats/dt2.html))
にあります。プログラム例は次のようなコードを含んでいます:

```ocaml
val a = cons(1, cons(2, cons(3, cons(4, nil))))
val b = cons(5, cons(6, cons(7, cons(8, nil))))
val lena = list_length(a)
val lenb = list_length(b)
val c = list_append(a, b)
val d = list_zip(a, c) // <== different lengths!
val lend = list_length(d)
```

リスト c の長さが 8 であるのに、リスト a の長さが 4 であることに注意してください。2つの異なる長さのリストに list_zip
を呼び出すと、この場合、長さ 4 のリストが返ります。

リストの長さを型の一部にエンコードすることで、異なる長さの2つのリストを zip
しようとしたらコンパイルエラーにすることができます。型の一部となった長さが依存型です。

## 依存型のリスト

次のデータ型は、長さ n の多相的なリストを定義しています。このとき n は整数です:

```ocaml
datatype list (a:t@ype, int) =
  | nil (a, 0)
  | {n:nat} cons (a, n+1) of (a, list (a, n))
```

これは、int が追加されたことを除いて、前の多相的なリストの定義と良く似ています。nil
型コンストラクタはこの int を 0 に設定しています。リストが nil の時、リストの長さは 0 なのです。cons
型コンストラクタは長さ n のリストの cons が、長さ n+1 のリストになることを表わしています。{n:nat}
は (非負の整数である) 自然数を表わす n の型を制約しています。

この新しいリスト型に対する関数の実装は、前の例と同様に関数テンプレートです。またこれらの関数は長さの値を表わす追加のパラメータを取ります:

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

list_length の返値の型は、リストの長さである n で特殊化された整数値の型であることに注意してください。これはこの実装に異なった値を返すようなエラーがあっても、コンパイル時エラーになることを意味しています。例えば、以下のコードは型検査を通りません:

```ocaml
fun{a:t@ype} list_length {n:nat} (xs: list (a, n)): int n = 
  case+ xs of
  | nil () => 1
  | cons (_, xs) => 1 + list_length(xs)
```

同様に list_append の返値の型は、長さ n1+n2 のリストとして定義されます。つまり、入力された2つのリストの長さの合計です:

```ocaml
fun{a:t@ype} list_append {n1,n2:nat} (xs: list (a, n1), ys: list (a, n2)): list (a, n1+n2) =
  case+ xs of
  | nil () =>  nil
  | cons (x, xs) => cons(x, list_append(xs, ys))
```

次のコードはコンパイルエラーになります:

```ocaml
fun{a,b:t@ype} list_zip {n:nat} (xs: list (a, n), ys: list (b, n)): list ('(a,b), n) =
  case+ (xs, ys) of
  | (nil (), nil ()) => nil ()
  | (cons (x, xs), cons (y, ys)) => cons('(x,y), list_zip(xs, ys))

implement main() = let
  val a = cons(1, cons(2, cons(3, cons(4, nil))))
  val b = cons(5, cons(6, cons(7, cons(8, nil))))
  val c = list_append(a, b)
  val d = list_zip(a, c) // <== different lengths!
```

このコンパイルエラーは、異なる長さのリストを list_zip に渡していることに起因します。両方の引数が同じ長さ
n で定義されているためです。前の多相的なリストの例での list_zip
の使い方は、このコードではコンパイルエラーになるのです。

このプログラムの完全な例は
[dt3.dats](http://bluishcoder.co.nz/ats/dt3.dats) ([html](http://bluishcoder.co.nz/ats/dt3.html))
にあります。

## フィルタ

結果のリストの正確な長さをいつも知ることができるとは限りません。型にエンコードできるか問題になります。例として、1つのリスト取って、述語関数に渡すと真を返すような要素だけを含む結果のリストを返すような、フィルタ関数を挙げることができます。この場合、型検査器は述語関数を呼び出して、結果のリストの長さを決定できなければなりません。次のコードは型検査を通りません:

```ocaml
fun{a:t@ype} list_filter {m:nat} (
  xs: list (a, m),
  f: a -<> bool
  ): list (a, m) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs) => if f(x) then cons(x, list_filter(xs, f)) else list_filter(xs, f)
```

この誤った例では、その結果は長さ m のリストとして定義されています。しかし述語関数が要素をスキッップしてしまったら、結果は長さ m になりません。異なる値を持つ結果の長さを定義するために、存在量化型 ([n:nat]) を使ってみます:

```ocaml
fun{a:t@ype} list_filter {m:nat} (
  xs: list (a, m),
  f: a -<> bool
  ): [n:nat] list (a, n) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs) => if f(x) then cons(x, list_filter(xs, f)) else list_filter(xs, f)
```

不幸にも上記のコードは、型検査器が誤ったコードを検出できない場合があることを意味してしまっています。元のリストより長いリストを結果として返す以下のようなコードをコンパイルエラーにしたいのです:

```ocaml
fun{a:t@ype} list_filter {m:nat} (
  xs: list (a, m),
  f: a -<> bool
  ): [n:nat] list (a, n) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs) => if f(x) then cons(x, cons(x, list_filter(xs, f))) else list_filter(xs, f)
```

この問題に対する解決策は、結果の存在量化型を入力のリストの長さ以下の自然数に限定することです:

```ocaml
fun{a:t@ype} list_filter {m:nat} (
  xs: list (a, m),
  f: a -<> bool
  ): [n:nat | n <= m] list (a, n) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs) => if f(x) then cons(x, list_filter(xs, f)) else list_filter(xs, f)
```

[n:nat | n <= m] が境界を定義していることに注意してください。このおかげで誤ったコードはコンパイルに失敗し、正しいコードはコンパイルに成功するようになります。エラーの可能性全てを捕えることはできませんが、前のバージョンよりもエラーの捕捉という点ではより良いものでしょう。

このプログラムの完全な例は
[dt4.dats](http://bluishcoder.co.nz/ats/dt4.dats) ([html](http://bluishcoder.co.nz/ats/dt4.html))
にあります。

## ドロップ

リストから先頭の n 要素を取り除く、list_drop 関数の実装もまた list_filter と似た問題をかかえています。以下の定義は型検査を通りません:

```ocaml
fun{a:t@ype} list_drop {m:nat} (
  xs: list (a, m),
  count: int
  ): list (a, m) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs2) => if count > 0 then list_drop(xs2, count - 1) else xs
```

list_filter のように、このエラーは結果のリストで使っている長さが誤っているためです。list_filter
と同じ解決策を使って、存在量化型で長さを決めることもできますが、この場合には結果の長さは判明しています。引数として渡される
count を使います。結果のリストの長さは、入力のリストの長さから count を減じた長さと等しくなるべきです。新しい定義は次のようになります:

```ocaml
fun{a:t@ype} list_drop {m,n:nat | n <= m} (
  xs: list (a, m),
  count: int n
  ): list (a, m - n) =
  case+ xs of
  | nil () => nil ()
  | cons (x, xs2) => if count > 0 then list_drop(xs2, count - 1) else xs
```

上記のバージョンは型検査を正常に通ります。もし誤った実装が期待された長さと異なるリストを生成したら、コンパイル時エラーになります。ドロップする要素数 count がリストの長さより大きい場合にも、コンパイル時エラーになります。

この検査は count をシングルトン整数にすることで実現しています。その型は n という特別な値の整数です。n
はリストのサイズ m 以下になるように {m,n:nat | n <= m} で定義されています。結果のリストは長さ m-n
であることを要求されています。

このプログラムの完全な例は
[dt5.dats](http://bluishcoder.co.nz/ats/dt5.dats) ([html](http://bluishcoder.co.nz/ats/dt5.html))
にあります。

## 要素に依存したリスト

これまでのリストの例はリストの長さに依存したデータ型でした。リスト自身に保管されている要素に依存することも可能です。次の定義は偶数のみを含むことができるリストです:

```ocaml
datatype list =
  | nil of ()
  | {x:int | (x mod 2) == 0 } cons of (int x, list)
```

cons コンストラクタは整数とリストを取ります。リストは、整数の値が2で割り切れなければならないという制約付きで、その整数の値に依存しています。ここでリストに奇数を入れようとするとコンパイルエラーになります。奇数 3 を cons に渡そうとしているため、以下のコードは型検査を通りません:

```ocaml
val a = cons(2, cons(3, cons(10, cons(6, nil))))
```

このプログラムの完全な例は
[dt6.dats](http://bluishcoder.co.nz/ats/dt6.dats) ([html](http://bluishcoder.co.nz/ats/dt6.html))
にあります。

要素に依存したリストの別の例として、全ての要素が 100 未満で順番に格納されたリストがあるでしょう。整列されていないリストをコンストラクトしようとすると、このリストは型エラーになります。

```ocaml
datatype olist (int) =
  | nil (100) of ()
  | {x,y:int | x <= y} cons (x) of (int x, olist (y))
```

olist は int に依存しています。そのリストに cons された全ての要素は、この int で表わされた整数値より小さくなるべきです。nil コンストラクタは、全ての要素が 100 より小さくなることを示す値として 100 を使います。これまでの例のように、cons コンストラクタは1つ目の引数に依存しています。その引数は tail リストの依存型の値 y 以下になることを強制されています。

次のような順序が乱れたリストのコンストラクトは型エラーになります:

```ocaml
val a = cons(1, cons(12, cons(10, cons(12, nil))))
```

一方、次のコードは問題ありません:

```ocaml
val a = cons(1, cons(2, cons(10, cons(12, nil))))
```

このプログラムの完全な例は
[dt7.dats](http://bluishcoder.co.nz/ats/dt7.dats) ([html](http://bluishcoder.co.nz/ats/dt7.html))
にあります。

## 赤黒木

論文
[Dependently Typed Datastructures](http://www.ats-lang.org/PAPER/DTDS-waaapl99.pdf)
には依存型のデータ型を用いた
[赤黒木](http://en.wikipedia.org/wiki/Red-black_tree)
の例があります。この論文では [DML](http://www.cs.bu.edu/~hwxi/DML/DML.html) 言語を使っています。私はこの例を ATS に翻訳してみました。

赤黒木は次の条件を満たすような平衡二分木です:

* 全てのノードは赤もしくは黒の色をもつ。
* 全ての葉は黒で、その他のノードは赤もしくは黒。
* 任意のノードについて、そのノードから葉を結ぶ全てのパスには同じ個数の黒ノードがある。これはそのノードの黒高さと呼ばれる。
* 全ての赤ノード2つの子は黒でなければならない。

これらの制約は赤黒木のデータ型で定義でき、上記の条件を満たし常のバランスするような木を保証できます。正しい赤黒木でない木を生成する関数を実装することが、型検査によって不可能になるのです。これは次のように定義できます:

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

型 rbtree は (int, int, int) をインデックスとしています。それらはそれぞれ、ノードの色、木の黒高さ、色の違反回数です。最後のインデックスは赤ノードが連続する回数です。最初の条件から、正しい rbtree は必ず色の違反回数がゼロにならなければなりません。

コンストラクタ E は葉ノードです。このノードは黒で、高さはゼロ、色の違反はありません。この木は有効な rbtree です。

コンストラクタ B は黒ノードです。このコンストラクタは3つの引数を取ります。それらは、左の子ノード、ノードに保管される整数値、右の子ノードです。B でコンストラクトされたノードの型は、黒で、子ノードより1大きい高さを持ち、色の違反はありません。2つの子ノードの黒高さは等しくなければならないことに注意してください。

コンストラクタ R は赤ノードです。このコンストラクタは3つの引数を取り、それらは B コンストラクタと同じものです。
As this is a red node it doesn’t increase the black height so uses the same value of the child nodes.
The color violations value is calculated by adding the color values of the two child nodes.
If either of those are red then the color violations will be non-zero.

The type for a function that inserts a key into the tree can now be defined as:

```ocaml
fun insert {c:color} {bh:nat} ( 
  key: int, 
  rbt: rbtree(c ,bh, 0)
  ): [c:color] [bh:nat] rbtree(c, bh, 0)
```

This means our implementation of insert must return a correct rbtree.
It cannot have a color violations value that is not zero so it must conform to the conditions we outlined earlier.
If it doesn’t, it won’t compile.

When inserting a node into the tree we can end up with a tree where the red-black tree conditions are violated.
A function restore is defined below that pattern matches the combinations of invalid nodes and performs the required transformations to return an rbtree with no violations:

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

The type of the restore function states that it takes a left and right node, one of which may have a color violation, and returns a correctly formed red-black tree node.
It’s time consuming and error prone to look at the code and determine that it covers all the required cases to return a correctly formed tree.
However the type checker will do this for us thanks to the constraints that have defined on the function and the rbtree type.
It won’t compile if any of the pattern match lines are removed for example.

The use of =>> in the last pattern match line is explained in the
[tutorial on pattern matching](http://www.ats-lang.org/htdocs-old/TUTORIAL/contents/pattern-matching.html).
The ATS typechecker will typecheck each pattern line independently of the others.
This can cause a typecheck failure in the last match since it doesn’t take into account the previous patterns and can’t determine that the color violation value of a in the result will be zero.
By using =>> we tell ATS to typecheck the clause under the assumption that the other clauses have not been taken.
Since they all handle the non-zero color violation case this line will then typecheck.

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

The constraints generated by the dependent types must be ‘linear’ constraints.
An example of a linear constraint is the definition of ‘list_append’ earlier:

```ocaml
fun{a:t@ype} list_append {n1,n2:nat} (
  xs: list (a, n1), ys: list (a, n2)
  ): list (a, n1+n2)
```

The result type containing n1+n2 will typecheck fine.
However an example that won’t typecheck is the following definition of a list_concat:

```ocaml
fun{a:t@ype} list_concat {n1,n2:nat} (
  xss: list (list (a, n1), n2),
  ): list (a, n1*n2)
```

The n1*n2 results in a non-linear constraint being generated and the ATS typechecker cannot resolve this.
The solution for this is to use theoreom-proving.
See chapter 6 in the
[ATS user guide](http://www.ats-lang.org/htdocs-old/DOCUMENT/MISC/manual_main.pdf)
for details and an example using concat.

## Closing notes

Although I create my own list types in these examples,
ATS has lists, vectors and many other data structures in the standard library.

There’s a lot more that can be done with ATS and dependent types.
For more examples see the papers mentioned at the beginning at throughout this post.
The paper
[Why Dependent Types Matter](http://lambda-the-ultimate.org/node/693)
is also useful reading for more on the topic.
