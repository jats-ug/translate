# Scripting Patsolve

(元記事は http://www.illtyped.com/projects/patsolve/scripting.html です)

SMT ソルバの判定力の広さを知って、
私達は、設定や主張の環境をコントロールするため、もしくは ATS の静的な部分に適用するための扱いにくい SMT の公式を自動生成ために、基盤であるソルバにプログラマに直接アクセスさせたいと考えました。

ユーザが SMT ソルバを直接扱えるように、私達は patsolve に python インタープリタを組み込みました。
これによってユーザは静的な関数を解釈し、SMT-LIB2 での `define-fun` のよう独自のマクロを定義できます。
マクロを定義するのは、ATS の静的な関数をそれらの引数に関数を適用する表現を持つ SMT 公式に拡張するためです。
これらの拡張は python ファイルで記述することができ、さらに制約解決において patsolve に渡されます。

## マクロ

例として、32 ビットの符号無し整数が 2 の累乗であるかどうか判定する関数を書くことを考えてみましょう。
ビット演算を使うことで、これを簡単に判定することができます。
しかし、私達はその計算が実際全ての入力に対して数が 2 の累乗かどうか判定できるかどうか立証したいのです。
長さ 32 の静的なビットベクトルでインデックスされた符号無し整数型を考えてみましょう。

```ats
abst@ype uint (b:bv32) = uint
```

この型の符号無し整数は、ATS の静的なビットベクトル `b` の値を保持するよう制限されています。
ここで、ビットベクトルが 2 の累乗かどうか判定する述語が必要になります。

```ats
stacst power_of_two_bv32: (bv32) -> bool
stadef power_of_two = power_of_two_bv32 // overloading
```

デフォルトでは、patsolve はこの関数を SMT ソルバで解釈されない関数に変換します。
patsolve でのスクリプトを使うと、`power_of_two` を、与えられた静的なビットベクトルが 2 の累乗であるときに真になるマクロに変換できます。
Z3py ライブラリでは、この意味を捕捉する式は、次のような python 関数で構築されます。
コードに Z3py をインポートする必要がないことに注意してください。
patsolve が自動的に挿入します。

```python
def power_of_two_bv32 (b):
  """
  A 32 bitvector x is a power of two.
  """
  return reduce(Or, (x == (1 << i) for i in range(32)))
```

この関数を定義した python ファイルを patsolve に渡すと、静的な関数呼び出しをその python 関数が返す式で置き換えます。
いくらかの python コードを書くだけで制約ソルバを拡張可能になるという点で、これは有用です。

これらの要素をまとめて、符号無し整数が 2 の累乗であることを判定する次の関数を作ることができます。
制約ソルバは、私達が返した結果が python ファイルで与えられた素直な定義と実際に等しいことを検査します。

```ats
fun
power_of_two {x:bv32} (
  x: uint (x)
): bool (power_of_2 (x)) =
  if x = 0u then
    false
  else
    ((x land (x - 1u)) = 0u)
```

この検査は次のように実行できます。

```
patsopt --constraint-export -tc -d power.dats | patsolve -s power.py
```

ATS2 コンパイラは制約を抽出し、それらを patsolve に送ります。
この小さな関数でその制約は、その真偽値が静的な関数呼び出し `power_of_2 (x)` の真偽値と等しいことです。
パラメータ `x` に与えられた `動的な` 型 `uint(x)` がシングルトン型であることに注意してください。
これは `x` の値が `静的な` ビットベクトル `x` の値に制限されていることを意味しています。
この関数の返り値に与えられた型は、その値が静的な関数 `power_of_2(x)` に等しくなるようなシングルトン型です。

## 解釈された関数

時には、必要な関数をマクロだけでは簡単に表現できないことがあります。
その代わりに、SMT ソルバへの関数を表わす解釈 (interpretation) を提供します。
これは patsolve が使う内部の SMT ソルバに主張を追加することを要求します。
私達の制約ソルバには python を組み込んでいるので、ユーザがなじみのある Z3 ライブラリを用いて主張を追加できるように、patsolve から呼び出されるモジュールを python スクリプトに提供します。

識別子の無限ストリームを表現する ATS の静的な型を使いたいとしましょう。
これはスタンプと呼ばれます。
この新しい種 `stampseq` を次のように呼び出せます。

```ats
sortdef stamp = int

datasort stampseq = (* abstract *)
```

静的なシーケンスをコンストラクトするために、なんらかの方法が必要です。
次の演算子が欲しいとしましょう。

```ats
stacst stampseq_nil : () -> stampseq                 // empty sequence
stacst stampseq_sing : (stamp) -> stampseq           // single sequence
stacst stampseq_cons : (stamp, stampseq) -> stampseq // add to front
stacst stampseq_head : stampseq -> stamp             // get first stamp
stacst stampseq_tail : stampseq -> stampseq          // skip first stamp
```

これらのコンストラクタを使って、`stampseq` を用いた ATS の型を自由にインデックスできます。
しかし、これらの関数で生成された制約は、単純な式を除いて常に無効です。
その理由は、私達は SMT ソルバにこれらの関数のどのような解釈も与えなかったからです。
これを行なうには、次の python コードを使って、それらの挙動についていくつか主張を制約ソルバに追加します。
内在する SMT ソルバにおける整数に写像した整数の配列として、スタンプシーケンスを表現したいとします。

```python
import patsolve

# Get our solver
s = patsolve.solver

A = Array("A", IntSort(), IntSort())
i, j, x = Int("i"), Int("j"), Int("x")

StampSeqSort = lambda : ArraySort (IntSort(), IntSort())

# Undefined Section of an Array

s.add (
    ForAll ([A, i], Implies (i < 0, A[i] == 0))
)

# The "nil" sequence

nil = Function ('stampseq_nil', StampSeqSort())

s.add (
    ForAll (i, nil()[i] == 0)
)

# Sing

sing = Function ('stampseq_sing', IntSort(), StampSeqSort())

s.add (
    ForAll ([x, i], Select (sing(x), i) == Select(Store (nil(), 0, x), i))
)
```

## Alternative Ideas

The idea of exposing just a python interface doesn't entirely sit well with me. I like the Z3Py library because the Z3 authors obviously spent a lot of time trying to make it very easy to use. At the same time, it takes a lot of work to interact directly with the library through the Python interpreter in the constraint solver.

What if someone does not want to use Z3? Do they need python bindings for their own SMT solver? Instead, why can't I provide an interface for a user to specify programs (or equivalently scripts) that receive arguments as strings on the commandline and then produce SMT-LIB2 code that represents evaluating the given macro? Alternatively, there could be programs that generate interpretations for individual functions. This may seem odd, but it seems like a nice way to appeal to any SMT solver, and give the user the most flexibility for extending the constraint solver in the way he or she feels fit.
