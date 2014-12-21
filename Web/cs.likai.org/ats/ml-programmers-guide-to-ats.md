# MLプログラマ向けATS言語ガイド

(元記事は http://cs.likai.org/ats/ml-programmers-guide-to-ats です)

このガイドでは ATS (Applied Type System) でのプログラミング作法を解説はしません。
それでも、熟練のMLプログラマであればすぐにATSの用語を理解してATSのコードを読み始めることができると思います。

## ATSのすばらしき世界

ATSでは、3つの世界でプログラミングをすることを覚えておくと良いでしょう。

* Dynamics(動的な世界): プログラムを実行した時に評価される部分です。これは既存のプログラミング言語に馴染んだプログラマにとってもっとも親しみ深い部分でしょう。
* Proofs(証明の世界): 動的な特性を静的な特性に結びつけます。Proofs(証明)はプログラムの動的な部分と考えることができます。しかし証明はコンパイル後では消滅してしまい、実行時には存在しません。コンパイラはあなたが書いた証明を検査し、動作可能なコードを生成する前に削除するのです。
* Statics(静的な世界): 型検査時にコンパイラによって評価される部分です。静的な部分では静的な式の評価は常に終了しなければなりません。そのため型検査は決定可能です。

ATSの前身であるDependent MLでは、静的な世界からは依存型を通して動的な世界を見るしかありませんでした。
動的なそれぞれの式は静的な式によって指定された型を持っています。
静的な制約は静的な式の集合によって形成され、
型検査はそれらの制約が充足可能であるかどうか調べます。

ATSは依存型に加えて証明を使うことができ、
静的な世界をより強く動的な世界に関与させることができます。
プログラマが動的な式と対応する証明の項を混じり合って書くことができるのです。
証明はpropと呼ばれるclassical propositions(古典論理)や、
viewと呼ばれるlinear propositions(linear logic)(線形論理)
のどちらかを取ります。

Prologに馴染んだ人にとっては、propを定義するのはPrologの述語を定義することとよく似ています。
しかし、型検査は自動的にその述語の解決をしてくれません。
プログラムを記述するのと同じように、証明も手動で書き下す必要があります。
証明の記述をすることで、ATSを証明器として使うことができます。

最後に、動的な世界と静的な世界の分離については
「動的な項の種は静的な種によって特徴づけられる」
と考えることもできます。

* プログラムの項はtypeによって特徴づけられる
* 線形論理による証明の項はviewによって特徴づけられる
* 古典論理による証明の項はpropによって特徴づけられる

### ノート: 線形論理(Linear Logic)について

線形論理は、弱化規則と縮約規則という構造規則を許可しないという点で古典論理と異なります。
弱化規則は事実(fact)を与えて未使用のままに放置させません。
縮約規則は事実を複数回使用することを許しません。
つまり、事実はきっちり一度だけしか使えないのです。

viewはその意味では線形論理です。
一般にviewの証明は、リソースの所有の追跡やリソースの状態の更新に使われます。

## 拡張子

ATSで用いられるファイルはいくつかの拡張子を持っています。

* .datsファイルは動的と静的な宣言を格納します。これはSMLにおけるモジュールシステムのstructureやOCamlの.mlファイルに似ています。
* .satsファイルは静的な宣言を格納します。これはSMLにおけるモジュールシステムのsignatureやOCamlの.mliファイルに似ています。
* .catsファイルは.satsファイルから使われるC言語コードを格納します。
* .hatsファイルは動的もしくは静的なATSコードを格納します。これらは.satsもしくは.datsファイルからインクルードすることができます。

.datsファイルと.satsファイルの特筆すべき違いは、関数と値のシグニチャ宣言です。

```ocaml
(* .datsファイル *)
extern val c: int
extern fun foo (x:int): int
fun bar (x: int): int = ...
```

```ocaml
(* .satsファイル *)
val c: int
fun foo (x:int): int
(* foo関数の実体を宣言できません *)
```

けれども典型的には.satsファイルでは関数の型を定義して、.datsファイルに実装を書きます。
そのような場合、.datsファイルの実装には型注釈は不要です。

```ocaml
(* .datsファイル *)
implement baz (x) = ...
```

```ocaml
(* .satsファイル *)
fun baz (x:int): int
```

重要なことですが、MLと異なり、ATSの型検査器はとても限定された型推論しか行ないません。
あなたは明示的に十分な型注釈を与える必要があります。
そうすればATSは型の導出できるようになるでしょう。

## めくるめく静

ATSはMLに少し似ています。
型とデータ型を定義することができます。

```ocaml
(* ATS *)
typedef t = int
typedef pair (a:type, b:type) = '(a, b)
datatype option (a:type) =
  | Some(a) of a
  | None(a)
```

```ocaml
(* SML *)
type t = int
type ('a, 'b) pair = 'a * 'b
datatype 'a option =
    Some of 'a
  | None
```

上記のSMLコードでは、定義済みのint型があったとき、
新しいtという名前の型をint型の別名として定義しています。
また、いくつかの型変数をパラメータとして取るような型を別名として定義できます。
さらに型変数aをパラメータとして取るoptionというデータ型も定義しています。
これらの識別子はすべて静的な識別子です。
対して、動的な識別子というのは実行時の値になります。

一方ATSでは、データ型のコンストラクタに型変数を明示的に適用してやる必要があることに注意してください。
データ型に関する詳細な情報は
[Datatypes in the ATS tutorial](http://www.ats-lang.org/htdocs-old/TUTORIAL/contents/datatypes.html)
を参照してください。
しかし、まずはこの章の残りを読んでからでも良いでしょう。

型宣言は静的な識別子を"type"という種の静的な式に割り付けます。
つまりtypedefはstadefの特殊形です。
次に示す2つの静的な宣言はほぼ同じです。

```ocaml
(* ATS (1) *)
(* sta t:type *)
typedef t = int
typedef pair (a:type, b:type) =
  '(a, b)
```

```ocaml
(* ATS (2) *)
(* sta t:_ *)
stadef t = int
stadef pair (a:type, b:type) =
  '(a, b)
```

この2つが違うのはtypedefがtが種"type"であることを保証するのに対して、
stadefは何も保証しないという点です。

型は、静的な式や静的な識別子が取り得る数多くの種の内の一つです。
他の種としては以下が挙げられます。

* Ground種: int, bool, char, addrです。これらは依存型の土台になります。
* View種: 線形論理の命題です。
* Prop種: 古典論理の命題です。
* Viewtype種: 線形型です。これはType種とView種が結合したものに見えるかもしれません。

datasortを使うことで新しい種を定義できます。
[sllist.dats](http://www.cs.bu.edu/~hwxi/academic/courses/CS520/Fall08/assignments/05/sllst_dats.html)
の最初の数行はリストに似た種を定義する方法を示しています。
またsortdefを使えば種の別名を定義できます。
これはサブセット種と呼ばれる種にpiggy-backな制約をつけます。
[prelude/sortdef.sats](https://ats-lang.svn.sourceforge.net/svnroot/ats-lang/trunk/prelude/sortdef.sats)
はその例です。
制約はシンプルなboolean種の式です。

type, view, propの種について、別名定義、abstract quantityの宣言、代数的データのコンストラクタの宣言をすることができます。


|            | Static  | Type     | View     | Prop     | Viewtype     |
|:-----------|:--------|:---------|:---------|:---------|:-------------|
| Aliasing   | stadef  | typedef  | viewdef  | propdef  | viewtypedef  |
| Abstract   | sta     | abstype  | absview  | absprop  | absviewtype  |
| Algebraic  | (N/A)   | datatype | dataview | dataprop | dataviewtype |

type, view, prop, viewtypeはstatics(静的)の特殊形です。
つまり、typedef, viewdef, propdef, viewtypedefの代わりに単にstadefを使うことができます。
同様にabstractの宣言においてもabstype, absview, absprop, absviewtypeの代わりにstaを使うことができます。
けれどもdatasort, datatype, dataview, dataprop, dataviewtypeの間は交換することができません。
これらは新しい何かに対する代数的なコンストラクタを定義します。
(例: datatypeは新しい型のコンストラクタです。dataviewは新しいviewのコンストラクタです。などなど)

view, prop, viewtypeについて、ここでは取り上げません。
ここでは型とシンプルな静的な式について注目しましょう。

静的な式は種によって検査される第一級の言語です。
いくつかの例を見てみましょう。

* 静的な式として書かれた数値は、種"int"の静的な数値になります
* ビルトインされた数値比較の述語があります。<, <=, >, >=, ==, <>は静的なシグニチャ"(int, int) -> bool"を持っています
* ビルトインされた論理積と論理和である&&, ||は静的なシグニチャ"(bool, bool) -> bool"を持っています
* 0 <= 4 && 4 < 10 は種"bool"の静的な式です
* 一般的に、静的なnat_lt関数は次のように定義できます

```ocaml
(* ATS *)
stadef nat_lt (i: int, limit: int) =
  0 <= i && i < limit
```

静的な関数であるnat_ltは、種boolが要求されればいつでも使うことができます。
例えば、実行時に数値の引数の範囲に制約をつけたい場合を考えましょう。

* ビルトインの静的な識別子intはオーバーロードされています。 種"type"と種"int -> type"の2つの定義を持っています。二次的なintは依存型の数値で静的な式によって指示されています。
* int(3) は種"type"の静的な式です
* ガードを使った存在記号を用いて、数値型を指定した範囲に制限することができます。

```ocaml
(* ATS *)
typedef NatLt (n: int) =
  [i: int | nat_lt (i, n)] int(i)
```

これで静的な関数 NatLt: int -> type を定義できました。
静的な数値nを与えれば、NatLtは 0 から n - 1 までの範囲を持つ数値型のサブセットとしての型を返します。
NatLtの実装はこう読めるでしょう。
「種intであり、nat_lt(i, n)が真で、int(i)型として生産されるのようなiが存在する。」

NatLt(n)型を使った関数は以下のようになるでしょう。

```ocaml
(* ATS *)
fun digit_of_num (i: NatLt(10)): char = ...
```

ATS/Anariatsの中にビルトインされた多くの静的な演算子のシグニチャは
[prelude/basics_sta.sats](https://svn.code.sf.net/p/ats-lang/code/trunk/prelude/basics_sta.sats)
で見ることができます。
stadefを使ってそのような演算子をオーバーロードできることに注意してください。
中置の二項演算子の結合性は
[prelude/fixity.sats](https://ats-lang.svn.sourceforge.net/svnroot/ats-lang/trunk/prelude/fixity.ats)
で定義されています。

## 異型(Atypical)の複合型

最初の区別は、
固定サイズの型はシンプルにtypeで表わされ、
任意サイズの型はt@ypeで表わされるということです。
typeの値はポインタワードと同じサイズを持つという点でMLの値と似ています。
それはボックス化もしくはアンボックス化されています。
typeのボックス化された値はGCによって管理されたヒープの中に配置されます。
ボックス化された値のいくつかの例を見てみましょう。


<table>
<tbody>
<tr>
<td><font face="courier new,monospace"><b>type</b></font></td>
<td>Expression</td>
<td>Type </td>
</tr>
<tr>
<td>Tuple</td>
<td>'(e<sub>1</sub>, e<sub>2</sub>, ... e<sub>n</sub>)</td>
<td>'(t<sub>1</sub>, t<sub>2</sub>, ... t<sub>n</sub>)</td>
</tr>
<tr>
<td>Record</td>
<td>'{lab<sub>1</sub> = e<sub>1</sub>, lab<sub>2</sub> = e<sub>2</sub>, ... lab<sub>n</sub>= e<sub>n</sub>}</td>
<td>'{lab<sub>1</sub> = t<sub>1</sub>, lab<sub>2</sub>= t<sub>2</sub>, ... lab<sub>n</sub>= t<sub>n</sub>}</td>
</tr>
<tr>
<td>List</td>
<td>'[e<sub>1</sub>, e<sub>2</sub>, ... e<sub>n</sub>]</td>
<td>[n:int | n &gt;= 0] list(t, n)<br />
<br />
(* もしくは以下に相当 *)<br />
List t<br />
</td>
</tr>
</tbody>
</table>

けれどもMLの型はシステムプログラミングにとってあまりにも非力です。
メモリレイアウトをフラットな構造でしか表現できないのですから。
ここでt@ypeの値が役に立ちます。
しばしば
このようなメモリレイアウトはview(線形論理のリソース)とt@ypeを持つことがあります。
つまるところ、これがviewt@ypeと呼ばれます。
t@ypeはviewt@ypeの下位種であることに注意してください。
そのためviewt@ypeと書かれているところを単にt@ypeと書いても問題ありません。
それどころか、C言語と同じ意味を持つ静的な関数である sizeof(t) が存在し、
それは以下に示す静的なシグニチャを持っています。

```ocaml
(* ATS *)
sta sizeof: viewt@ype -> int
```

そしてこのsizeof静的関数はt@ypeにもviewt@ypeにも使えるのです。

ボックス化タプル, レコード, リストはフラットなtypeとよく似ています。

<table>
<tbody>
<tr>
<td>種<br />
</td>
<td><font face="courier new,monospace"><b>type</b></font></td>
<td><font face="courier new,monospace"><b>t@ype</b></font></td>
</tr>
<tr>
<td>Tuple</td>
<td>'(t<sub>1</sub>, t<sub>2</sub>, ... t<sub>n</sub>)</td>
<td>@(t<sub>1</sub>, t<sub>2</sub>, ... t<sub>n</sub>)</td>
</tr>
<tr>
<td>Record</td>
<td>'{lab<sub>1</sub> = t<sub>1</sub>, lab<sub>2</sub> = t<sub>2</sub>, ... lab<sub>n</sub>= t<sub>n</sub>}</td>
<td>@{lab<sub>1</sub> = t<sub>1</sub>, lab<sub>2</sub>= t<sub>2</sub>, ... lab<sub>n</sub>= t<sub>n</sub>}</td>
</tr>
<tr>
<td>Sequence<br />
(List/Array)<br />
</td>
<td>(* list *)<br />
list(t, n)<br />
</td>
<td>(* array *)<br />
@[t][n]<br />
</td>
</tr>
</tbody>
</table>

しかしフラットな値を構築することは少し異なります。
フラットな値はヒープ(ボックス化値と同じ)もしくはスタック(MLはスタックにデータ構造を確保しませんが)のどちらかに確保されます。
これについては別の記事
[Manipulating flat memory](http://cs.likai.org/ats/manipulating-flat-memory)
でお話しします。

フラットなメモリを初期化しないこともできます。
"t?"という注釈は未初期化のメモリの型を表わしています。
そしてそのサイズはt型と同じです。
例えば、未初期化のintは"int?"と書きます。(章末のノートを見てください)
しかし、型t1とt2について sizeof(t1) == sizeof(t2) であったとしても、
コンパイラはt1?とt2?を異なる型として扱います。

最後に、ときどきtype(およびviewtype)やt@ype(およびviewt@ype)の後ろに
'+'と'-'という注釈を目にすることがあるでしょう。
'+'はそのパラメータが共変性であることを意味し、'-'はそのパラメータが反変性であることを意味しています。
注釈がない場合には、パラメータは不変性です。
これは静的な式の型に対して下位種を作るのに役に立ちます。
共変性は良く使われますが、反変性を使うことはまれです。

### ノート: intの型はフラット

読者をおどろかせるかもしれませんが、ATSにおいてintはt@ypeです。
64ビットマシンのC言語型であるintとlongを考えてみると、
intは32ビット、longは64ビット、そしてポインタは64ビットです。
intはポインタと同じサイズではありません。
同様に、ATSではlongもまたt@ypeです。
16ビットマシンの場合、intは16ビット、longは32ビット、ポインタは16ビットであるからです。

## 落ち着かない関数たち

ATSでの関数の型の詳細は注意深く調査するべきでしょう。
その簡単な構成はMLと似ていて、シンプルなものです。

```ocaml
(* ATS (.sats) *)
fun function_name (x1: t1, ..., xn: tn): t' =
  "c_name"
```

```ocaml
(* SML (signature) *)
val function_name: (t1 * ... * tn) -> t' 
```

C言語に変換すると、function_nameはc_nameという名前になることに注意してください。
もっともこれは任意ですが。
これは以下のように使われます。

1. 既存のC言語関数をATSにインポートして、ATSの型をつける
2. ATSで実装された関数をC言語側へエクスポートする

これはいくつかの重要な帰結を持ちます。

* ATSにおける関数の型はC言語の関数に直接写像されうる
* ATSにおける関数は常にアンカリー化されている
* タグを指定しないかぎり、デフォルトでは関数はクロージャとして使えず、部分適用も使えない (タグについては後に説明します)

依存型にガードとアサートを追加すれば、以下のようになります。

```ocaml
(* ATS (.sats) *)
fun function_name
    {forall quantifiers | guards} (x1: t1, ..., xn: tn):
    [exists quantifiers | asserts] t' =
  "c_name"
```

ガードはforall量化子の項で書くことができる種boolの静的な式です。
アサートはforallとexists量化子の項で書くことができる種boolの静的な式です。
なおかつ、t1 ... tn はforall量化子の項で書くことができる種typeの静的な式です。
そしてt'はforallとexists量化子の項で書くことができる種typeの静的な式です。

証明の項をさらに追加してみましょう。

```ocaml
(* ATS (.sats) *)
fun function_name
    {forall quantifiers | guards} (pf1: prop1, ..., pfp: propp | x1: t1, ..., xn: tn):
    [exists quantifiers | asserts] (prop'1, ..., prop'q | t') =
  "c_name"
```

アンカリー化された引数は、証明の変数とプログラムの変数の2つの部分に分断されたことに注意してください。
証明の変数はコンパイルが完了すると削除されます。
プログラムの変数はコンパイルされてもそのまま残ります。

最後になりますが、ATSはタグとして副作用を追跡するためにアロー型を使います。

```ocaml
(* ATS (.sats) *)
fun function_name
    {forall quantifiers | guards} (pf1: prop1, ..., pfp: propp | x1: t1, ..., xn: tn)
  :<tag1, ..., tagk>
    [exists quantifiers | asserts] (prop'1, ..., prop'q | t') =
  "c_name"
```

もし関数の型を静的な式として書きたければ、
(これまで見てきた関数のシグニチャに反していますが)
型の表現は以下のようになるでしょう。

```ocaml
(* ATS (static expression) *)
{forall...} (prop1, ..., propp | t1, ..., tn) -<tag1, ..., tagk> [exists...] (prop'1, ..., prop'q | t')
```

## タグ付きのアロー型

ATSにおけるアロー型は"-<>"のような見た目をしています。
アロー型は -\<tag1, ..., tagk\> のような形で装飾されます。
関数の様々な種を区別するために、現時点でATSは次のようなタグを解釈します。

* prf: 証明関数 (型検査が終わると削除れます)
* lin: 確実に一度だけしか呼び出されない関数
* fun: 通常の関数 (デフォルト)
* cloptr: 明示的に解放されるべきクロージャ
* cloref: GCされるクロージャ

同様に次のタグは副作用の追跡に用いられます。

* exn: 例外を起こす
* ntm: 終了しない可能性がある
* ref: グローバルメモリへの参照を共有している
* 0: 副作用なし (純粋)
* 1: 副作用あり (判断できない場合も含まれる)

副作用の有無を示すための表現である0や1の接尾辞を、関数の種を区別するタグに加えて付けることができます。
例えば cloref1 は、MLのようにGC対象となるクロージャであり、
副作用(例外発生など)を持っているというタグになります。

## stdio.h から例を

ATSにおける典型的な関数の宣言はどのような見た目になるのでしょうか？
ここでは、次のプロトタイプ宣言を持つ標準Cライブラリを例に取りましょう。

``` {.c}
char *fgets(char *s, int size, FILE *stream);
```

manページには
「fgets()はstreamから最大で size - 1 個の文字を読み込み、sが指すバッファに格納する。
'\\0'が一つバッファの中の最後の文字の後に書き込まれる。」
とあり、さらに
「fgets()は、成功するとsを返し、エラーや1文字も読み込んでいないのにファイルの終わりになった場合に NULL を返す。」
とあります。

ATSでの宣言を見てみましょう。コメントをたくさん入れてみました。

```ocaml
(* fgets_vはfgetsの結果である成功か失敗を表わします。
 *)
dataview fgets_v (sz:int, addr, addr) =
  | (* 成功した場合、lアドレスは長さnの文字列を格納している
     * サイズszの文字列バッファでしょう。
     *)
    {n:nat | n < sz} {l:addr | l <> null}
      fgets_v_succ(sz, l, l) of strbuf(sz, n) @ l

  | (* 失敗した場合、lアドレスは単にサイズszの空箱のはずです *)
    {l:addr}
      fgets_v_fail(sz, l, null) of bytes(sz) @ l

fun fgets
    (* countは読み込むバイト数です。
     * それはバッファのサイズであるsz以内でなければなりません。
     *)
    {count, sz:int | 0 < count; count <= sz}

    (* fmはファイルモードのための静的な種です。 *)
    {m:fm}

    (* lはバッファのアドレスを示す依存型です。 *)
    {l:addr}

    ( (* ファイルモードが読み出しアクセスを含んでいることの証明。 *)
      pf_mod: file_mode_lte(m, r),
      (* szバイトの列がlに存在することに対する線形論理の証明。 *)
      pf_buf: bytes(sz) @ l
    | (* lアドレスによって指示されるポインタを引数に取る。 *)
      s: ptr(l),
      (* 読み込む最大のバイト数を引数に取る。  *)
      size: int(count),
      (* モードmによってオープンされたFILE構造体への参照を引数に取る。 *)
      stream: &FILE(m)
    )
  :<>
    (* l'は返値のポインタを指示します。 *)
    [l':addr]

    ( (* pf_bufは消費され、ここでl'に一致する変化を形成します。 *)
      (* pf_buf is consumed, and this determines what it turns into
       * according to l'.
       *)
      fgets_v(sz, l, l')
    | (* l'で示すポインタを返します。 *)
      ptr(l')
    ) =
  "fgets"
```

この例はATS/Anariatsの
[libc/SATS/stdio.sats](https://svn.code.sf.net/p/ats-lang/code/trunk/libc/SATS/stdio.sats)
を元にしています。
わずかな修正と多くのコメントによる注釈が加えてあります。
はじめて見る人には明確には思えないかもしれません。
いくつかポイントがあります。

* 実際のバッファサイズ(静的な変数sz)がfgets()が使用するサイズ(静的な変数count)よりも大きいことを許可しています。これはlibc/SATS/stdio.satsにおけるfgetsで定式化されています。しかしfgets()にバッファを全て使わないようには強制しません。
* fgets()がNULLではない値を返す場合、以前と同じポインタを返さなければなりません。しかもそのポインタはstrbufによって解釈を与えれていて、文字列バッファはゼロ終端されています。そうでない場合、同じく bytes(sz) @ l が返ります。fgets()の結果の種別を見分ける唯一の方法は、if式でそのポインタがNULLであるのかチェックすることです。もしあなたがチェックを見逃したら、型エラーになります。しかしエラーのチェックに飽き飽きすることもあるでしょう。そんな時はlibc/SATS/stdio.satsで定義されている例外を使うバージョンであるfgets_exn()を使うこともできます。

参考のために、上記のコードからコメントを削除したものを載せておきます。

```ocaml
dataview fgets_v (sz:int, addr, addr) =
  | {n:nat | n < sz} {l:addr | l <> null}
      fgets_v_succ(sz, l, l) of strbuf(sz, n) @ l
  | {l:addr}
      fgets_v_fail(sz, l, null) of bytes(sz) @ l

fun fgets
    {n, sz:int | 0 < n; n <= sz} {m:fm} {l:addr} (
      pf_mod: file_mode_lte(m, r), pf_buf: bytes(sz) @ l
    | s: ptr(l), size: int(n), stream: &FILE(m))
  :<>
    [l':addr] (fgets_v(sz, l, l') | ptr(l')) =
  "fgets"
```

## この文書のTODO

* fn (再帰なし) と fun (再帰あり) の比較との終端の計測
