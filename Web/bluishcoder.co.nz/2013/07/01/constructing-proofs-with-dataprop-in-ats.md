# BLUISH CODER: ATS でデータ命題を使って証明を構築する

(元記事は http://bluishcoder.co.nz//2013/07/01/constructing-proofs-with-dataprop-in-ats.html です。)

私の以前の記事では [ATS での証明を使って](http://bluishcoder.co.nz//2012/10/04/implementing-a-stack-with-proofs-in-ats.html)
スタックを実装しました。
この記事では少し戻って、`prfun` を使った証明とデータ命題 (dataprop) を使ったより小さな例を見てみます。
この例は、ソフトウェアの基礎 ([Software Foundations](http://www.cis.upenn.edu/~bcpierce/sf/current/index.html)) の章 [Propositions and Evidence](http://www.cis.upenn.edu/~bcpierce/sf/current/Prop.html)
に基づいています。
この本は [Coq Proof Assistant](https://coq.inria.fr/what-is-coq) をカバーしていて、私は ATS でその例のいくつかを解いてみました。

章 Propositions and Evidence は自然数の特性を導入することから始まります。
この特性を持つ数は "素晴しい" と考えられます。
その数が 0, 3, 5 もしくは2つの素晴しい数の和であるなら、その自然数は素晴しいとします。
これは次の4つのルールで定義されます:

* ルール b_0: 数 0 は素晴しい。
* ルール b_3: 数 3 は素晴しい。
* ルール b_5: 数 5 は素晴しい。
* ルール b_sum: もし n と m が素晴しいならば、それらの和も素晴しい。

これは [データ命題](http://jats-ug.metasepi.org/doc/ATS2/INT2PROGINATS/c2842.html) を使って ATS で表現することができます:

```ats
dataprop BEAUTIFUL (n:int) =
| B_0 (0)
| B_3 (3)
| B_5 (5)
| {m,n:nat} B_SUM (n+m) of (BEAUTIFUL m, BEAUTIFUL n)
```

データ命題はデータ型に似ていますが、それは証明システムのために存在しています。
他の証明要素 (`prfun` など) のように、コンパイル時にそれらは消去され実行時のオーバーヘッドはありません。
データ命題は関係や命題をエンコードするために用いられます。
データ命題 `BEAUTIFUL` は依存型を使っています - それは型のインデックスとして自然数を持ち、`BEAUTIFUL` に数の特性を成します。
それぞれの行ではこれまで説明したルールを実装しています。

`B_SUM` 行における波括弧中のセクション `{m,n:nat}` は2つの普遍的な値を宣言しています。
この行は "全ての自然数で素晴らしい数 m と n において、それらの数の和もまた素晴しい" と読めます。

命題が得られたので、証明を構築することができます。
注目する最初の証明は二番目のルールの証明です - つまり "数 3 は素晴しい" です。
次の証明関数を実装する必要があります:

```ats
prfun three_is_beautiful ():<> BEAUTIFUL 3
```

証明関数は `prfun` や `prfn` で導入され、証明を実装するのに使われます。
これらの証明は、プログラムが正しいことを保証するために ATS によって検査されます。
`prfn` は非再帰関数を表わすのに使われ、`prfun` は再帰できます。
もし再帰するのであれば、それが停止することを証明できなければなりません。
これをどうやって行なうかは後で説明します。
この関数は `BEAUTIFUL 3` のインスタンスを返します。
定義における `:<>` は、この関数が純粋な関数であることを意味しています。
つまり副作用なく実行できます。
全ての証明関数は純粋で停止しなくてはなりません。

この証明に対する実装はシンプルです:

```ats
prfn three_is_beautiful ():<> BEAUTIFUL 3 = B_3
```

本体は単に `B_3` ルールを返すだけです。
このコードは次のような型検査に成功します:

```
$ atscc -tc proof.dats
The file [proof.dats] is successfully typechecked!
```

証明関数を宣言してそれらを実装しないことができます。
これは "その証明は自明だ" とコンパイラに言っていることになります。
証明の実装に時間がかかるわりに明らかであるような場合に、この方法が使えます。
この場合、`three_is_beautiful` はルールから明らかなので、その実装を除いて宣言することもできます:

```ats
praxi three_is_beautiful ():<> BEAUTIFUL 3
```

`prfun` の代わりに `praxi` を使ったことに注意してください。
後者は証明する必要のない基本的な公理を表わすのにしばしば使われます。
それらは実装されることを期待されていません。
現在の ATS コンパイラは `prfun` が未実装でも警告を出しませんが、将来は警告が出るかもしれません。
実装を意図しない証明を表わすのに `praxi` を使うのは良い習慣でしょう。

次のステップとして、`B_SUM` を使っていくつか証明をしてみます。
8 が素晴しい数であることを証明してみましょう:

```ats
prfn eight_is_beautiful ():<> BEAUTIFUL 8 = B_SUM (B_3, B_5)
```

この証明の実装は、他の `BEAUTIFUL` ルール `B_3` と `B_5` を使って `B_SUM` を構築しています。
これを型検査するとこのコードがルールを証明するのに十分であることを確認できます。

証明関数は主張を証明 (もしくは反証) するために使うことができます。
例えば、どのような素晴しい数に 3 を足すと素晴しい数になると主張するとします。
これは次のような証明関数で表わせます:

```ats
prfn b_plus3 {n:nat} (b: BEAUTIFUL n):<> BEAUTIFUL (3+n)
```

実装は自明で、ATS コンパイラの型検査に成功し、どのような素晴しい数に 3 を加えると新しい素晴しい数を作ることを証明しています。

```ats
prfn b_plus3 {n:nat} (b: BEAUTIFUL n):<> BEAUTIFUL (3+n) =
  B_SUM (B_3, b)
```

別の主張は、どのような素晴しい数も2倍すると素晴しい数になるというものです。
この主張は証明として次のように実装できます:

```ats
prfn b_times2 {n:nat} (b: BEAUTIFUL n):<> BEAUTIFUL (2*n) =
  B_SUM (b, b)
```

より複雑な証明としては、どのような素晴しい数もなんらかの自然数倍すれば素晴しい数になる、というのがあります:

```ats
prfun b_timesm {m,n:nat} .<m>. (b: BEAUTIFUL n):<>
       [p:nat] (MUL (m,n,p), BEAUTIFUL p) =
```

これは、再帰が必要な最初の証明です。
再帰証明関数はそれらが終了することを示さなければなりません。
これは (`.<` と `.>` で囲まれた) [停止節](http://jats-ug.metasepi.org/doc/ATS2/INT2PROGINATS/x2478.html) を追加することで指定します。
停止節は静的な型の値を含まなければなりません。
この値はそれぞれの再帰呼び出しで減少していることを証明するために使われます。
この場合それは `m` です。

この証明関数が返す結果は存在量化の値である `p` を含み、`MUL` と素晴しい数のタプルを返します。
`MUL` は ATS prelude の一部です。
これはどのような数 `m` と `n` についても数 `p` は `m*n` であるという関係をエンコードしています。
ここでの `p` はまた返される素晴しい数の型インデックスとしても含まれます。
この証明は "全ての自然数 `m` と全ての素晴しい数 `n` について、`m*n` となるような素晴しい数 `p` が存在します" のように読めます。

この証明の実装は次のようになります:

```ats
sif m == 0 then
  (MULbas, B_0)
else let
  prval (pf, b') = b_timesm {m-1,n} (b)
in
  (MULind pf, B_SUM(b, b'))
end
```

最初に注目できることは `if` の代わりに `sif` を使っていることです。
後者は値 `m` と `n` が属する静的な型システムのためのものです。
`0` の場合 (素晴しい数をゼロ倍する場合)、ルール `B_0` と `MUL` 命題の基底ケースを返します。

非ゼロで乗じた場合、再帰的に `b_timesm` を呼び出して `m` を減少させます。
`m` と `n` は静的な型システムにおける普遍的な値として、`{...}` 構文を使って渡されています。
その結果は和を取ります。
基本的には再帰的な加算を使って乗算を実装しているのです。
より多くの `MUL` の使用例はATS ドキュメントの章
[データ命題として関係をエンコードする](http://jats-ug.metasepi.org/doc/ATS2/INT2PROGINATS/c2842.html#encoding_relations_as_dataprops)
に見つかります。

素晴しい数に関する証明をしたとして、どうやってそれらを実際のプログラムで使うのでしょうか？
素晴しい数の和を取って、その結果を表示するプログラムを想定してみましょう。
型検査を除くとそれは次のようになるでしょう:

```ats
fun add_beautiful (n1: int, n2: int): int =
  n1 + n2

fun do_beautiful () = let
  val n1 = 3
  val n2 = 8
  val n3 = add_beautiful (n1, n2)
in
  printf("%d\n", @(n3))
end
```

これは数が実際に素晴しい数であると保証する検査を信頼しています。
また、その作用のために実行時のアサートを使うかもしれません。
同じプログラムを証明を使うと次のように定義できます:

```ats
fun add_beautiful {n1,n2:nat}
         (pf1: BEAUTIFUL n1, pf2: BEAUTIFUL n2
          | n1: int, n2: int): [n3:nat] (BEAUTIFUL n3 | int) =
  (B_SUM (pf1,pf2) | n1 + n2)

fun do_beautiful () = let
  prval pf1 = three_is_beautiful ()
  prval pf2 = eight_is_beautiful ()
  val n1 = 3
  val n2 = 8
  val (pf3 | n3) = add_beautiful (pf1, pf2 | n1, n2)
in
  printf("%d\n", @(n3))
end
```

このプログラムは `add_beautiful` に素晴しい数でない数を渡すとコンパイルできません。
ここで構築した証明では、使用している2つの数は素晴しく、それらの証明を `add_beautiful` に渡していました。
この関数は、その2つの和と返される値もまた素晴しい数であるとの証明を返します。

実装が正しいことを立証するために証明を使って与えられた整数が素晴しい数であるかどうか検査する `assert_beautiful` 関数を実装は読者への練習問題とします。
実装された関数を使って次のようなコードを書けます:

```ats
extern fun assert_beautiful (n: int): [n:nat] (BEAUTIFUL n | int n)

fun do_beautiful2 (a:int, b:int): void = let
  val (pf1 | n1) = assert_beautiful (a)
  val (pf2 | n2) = assert_beautiful (b)
  val (pf3 | n3) = add_beautiful (pf1, pf2 | n1, n2)
in
  printf("%d\n", @(n3))
end
```

[Software Foundations](http://www.cis.upenn.edu/~bcpierce/sf/current/index.html)
に書かれている通り、数が素晴しいという根拠を構築することで、証明システムを使ってそれらを立証することもできます。
数が素晴しいという根拠を構築する唯一の方法は、データ命題のコンスラクタを使うことです。
次の例では、華麗な数という数の新しい性質を作り、素晴しい数との関係を証明します。

```ats
dataprop GORGEOUS (n:int) =
| G_0 (0)
| {n:nat} G_plus3 (3+n) of GORGEOUS (n)
| {n:nat} G_plus5 (5+n) of GORGEOUS (n)
```

素晴しい数に対して行なったのと同じ方法で、華麗な数に対する証明を読者に練習問題として残します。
2つの華麗な数の和が華麗な数になることを証明してください。

ここで立証したい証明は、全ての華麗な数はまた素晴しい数であるというものです。
この場合における直感的なアイデアを与える定義を見てみましょう。
それを証明する証明関数は次のようになります:

```ats
prfun gorgeous__beautiful {n:nat} (g: GORGEOUS n): BEAUTIFUL n
```

この実装では、`GORGEOUS` のコンストラクタをディスパッチして相当する `BEAUTIFUL` を生成するのに `case+` を使っています:

```ats
prfun gorgeous__beautiful {n:nat} .<n>. (g: GORGEOUS n): BEAUTIFUL n = 
  case+ g of 
  | G_0 () => B_0
  | G_plus3 g => B_SUM (B_3, gorgeous__beautiful (g))
  | G_plus5 g => B_SUM (B_5, gorgeous__beautiful (g))
```

もし華麗な数が素晴しい数でないのであれば、このコードは型検査に失敗するはずです。
これは型検査を通るので、この場合成立することがわかります。
