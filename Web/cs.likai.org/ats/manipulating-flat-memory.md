# フラットなメモリを操作する

(元記事は http://cs.likai.org/ats/manipulating-flat-memory です)

このガイドはATSにおけるフラットなメモリデータ構造に対する操作のオーバービューです。
ATSがサポートするフラットなメモリのtypeはt@ypeという種を持ちます。

## フラットな型とサイズ

種t@ypeを持ち完全にフラットなビルトインのプリミテイィブ型は、
bool, char, double, int, uint, size_t, void です。
さらにATSはフラットなデータ構造型としてtuple, record, arrayをサポートしています。

| t@ype      | Type                                 |
|:-----------|:-------------------------------------|
| Tuple      | @(t1, t2, ... tn)                    |
| Record     | @{lab1 = t1, lab2= t2, ... labn= tn} |
| Array      | @[t][n]                              |

全てのt@ypeはサイズに関する情報をも持っています。
しかし、このサイズ情報はATSコンパイラから見えません。

tが t@ype もしくは viewt@ype であるとき、
静的な sizeof(t) 関数はint種である一意の静的変数を返します。
この変数は t のサイズを表わしています。
動的な "{t:viewt@ype} sizeof()" 関数は、
t@ype もくは viewt@ype であるtによってパラメーター化されたテンプレート関数で、
静的な sizeof(t) を示す静的な変数によってインデックスされた整数を返します。
このおかげでtのサイズを仮定して、
"if sizeof(t) == 4 then ... else ..." や "assert {t} sizeof() == 4"
のように書くことができます。
しかし一般に t@ype のサイズを私達は知ることができません。

## View of memory region

フラットなt@ypeそれ自体は有用ではありません。
それどころか、ただメモリのレイアウトについて表現しているだけで、他に何も表わしていません。
とりわけ次のことを表現していないのです。

* そのレイアウトのメモリ部分を所有しているかどうか
* メモリ領域のアドレス
* メモリ領域のアライメントが正しいかどうか
* メモリを確保した手段: GCされうるのか, malloc(), shmget()/shmat(), mmap(), スタックに確保されたのか

フラットなメモリを扱うために、2つの情報が必要になります。
1つ目はview "t @ l"の証明です。これはメモリ上の位置 l に type t を持つことを表現しています。
2つ目はtype "ptr l"の値です。これはメモリ上の位置 l を指すポインタを表わします。

一般に、
それぞれのヒープアロケータは、種"addr -> view"のabstract viewに対する線形論理の証明を同時に返します。
これはメモリ上の位置 l:addr がそのアロケータによって確保されたことを証明しています。
この方法で、あるヒープに確保されたメモリ領域を別のヒープに解放することを防止できます。

フラットなデータ構造をスタック上に確保するために、ATSは"var"束縛という新しい形を導入します。

| t@ype      | Expression                            | Type                                 |
|:-----------|:--------------------------------------|:-------------------------------------|
| Tuple      | @(e1, e2, ... en)                     | @(t1, t2, ... tn)                    |
| Record     | @{lab1 = e1, lab2 = e2, ... labn= en} | @{lab1 = t1, lab2= t2, ... labn= tn} |

スタック上のフラットなarrayは少し異なります。

<table>
<thead>
<tr>
<th align="left">t@ype</th>
<th align="left">Syntax</th>
<th align="left">Binding</th>
</tr>
</thead>
<tbody>
<tr>
<td align="left">Array<br />
(single initializer)<br />
</td>
<td align="left">var !p with pf = @[t][n](e)<br />
</td>
<td align="left">p:ptr l<br />pf:@[t][n] @ l<br />
</td>
</tr>
<tr>
<td align="left">Array<br />
(uninitialized)</td>
<td align="left">var !p with pf = @[vt][n]()</td>
<td align="left">p:ptr l<br />pf:@[vt?][n] @ l<br />
</td>
</tr>
</tbody>
</table>

### Terminology

ATSの動的な世界は3種類の項を持っています。

* type (もしくはt@ypeやviewt@ype) で特徴づけられた式
* viewで特徴づけられた線形論理による証明の項
* propで特徴づけられた古典論理による証明の項

コンパイル後には線形論理と古典論理の全ての項は消去されています。
プログラムにはプログラムが実行された時に評価される式が残ります。

## この文書のTODO

未完
