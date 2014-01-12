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

Flat t@ype on its own isn't very useful. If anything, it describes just the layout of memory but doesn't tell you anything else. In particular, it doesn't tell you:

* If we do own a piece of memory with that layout.
* The address of the memory.
* Whether the memory is well-aligned.
* How the memory is allocated: from the garbage collector, malloc(), shmget()/shmat(), mmap(), or even on the stack.

In order to access flat memory, we need two pieces of information: a proof of the view "t @ l", which states that we have type t at memory location l, and a value of type "ptr l", which is a pointer pointing to memory location l.
Typically, each heap allocator will also give you a linear proof of an abstract view of the sort "addr -> view" which testifies that location l:addr is allocated by that allocator. This way, the memory allocated from one heap cannot be freed to another heap.

To allocate flat data structure on stack, ATS introduces a new form of "var" binding.

### Terminology

The dynamic world of ATS has three kind of terms.

* A type (also t@ype, viewt@ype) characterizes an expression.
* A view characterizes a linear proof term.
* A prop characterizes a propositional proof term.

After compilation, all linear and propositional proof terms are erased,
and the program is left with expressions to be evaluated at run-time when someone executes the program.

| t@ype      | Expression                            | Type                                 |
|:-----------|:--------------------------------------|:-------------------------------------|
| Tuple      | @(e1, e2, ... en)                     | @(t1, t2, ... tn)                    |
| Record     | @{lab1 = e1, lab2 = e2, ... labn= en} | @{lab1 = t1, lab2= t2, ... labn= tn} |

Flat array on stack is a bit different.

<table>
<thead>
<tr>
<th>t@ype</th>
<th>Syntax</th>
<th>Binding</th>
</tr>
</thead>
<tbody>
<tr>
<td>Array<br />
(single initializer)<br />
</td>
<td>var !p with pf = @[t][n](e)<br />
</td>
<td>p:ptr l<br />pf:@[t][n] @ l<br />
</td>
</tr>
<tr>
<td>Array<br />
(uninitialized)</td>
<td>var !p with pf = @[vt][n]()</td>
<td>p:ptr l<br />pf:@[vt?][n] @ l<br />
</td>
</tr>
</tbody>
</table>

xxx
