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

xxx
