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

xxx
