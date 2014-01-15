# BLUISH CODER: ATSのクロージャ

(元記事は http://bluishcoder.co.nz/2010/06/20/closures-in-ats.html です)

[以前のATSに関する記事](http://bluishcoder.co.nz/2010/06/13/functions-in-ats.html)
でC言語スタイルの関数について説明しました。
この記事ではATSのクロージャについて学んだことを紹介しようと思います。

以前の記事で述べた通り、クロージャは名前から値への写像である環境をともなう関数です。
動的型付け言語のクロージャのように、閉じられたスコープの中に値を保管することができます。
クロージャを生成するためには、明示的にクロージャであることを指示する必要があります。
クロージャは環境を保存するために確保されたメモリ領域を要求します。
つまり、スタックにクロージャを確保するか、手動でクロージャのメモリを解放するか、
GCと紐づける必要があります。
スタックにクロージャを確保した場合、そのスコープから抜けると自動的にクロージャは解放されてしまいます。

## 永続クロージャ

永続クロージャはヒープに確保されて、使われなくなった時にGCによって解放されます。
この型のクロージャを作るには "cloref" タグを使います。

```ocaml
fun make_adder(n1:int): int -<cloref> int =
  lam (n2) =<cloref> n1 + n2;

implement main() = let
  val add5 = make_adder(5)
  val a = add5(10)
  val b = add5(20)
  val () = printf("a: %d\nb: %d\n", @(a,b))
in
  ()
end
```

この例はGCを使っているので、GCを含んでコンパイルしなければなりません。
(-D_ATS_GCATS スイッチを使います)

```
atscc -o eg1 eg1.dats -D_ATS_GCATS
```

この例では、"make_adder" 関数は整数値であるn1を取り、クロージャを返します。
そこクロージャは整数値である "n2" を取り、"n1 と "n2" の和を返します。
"main"関数は初期化関数に "5" を与えて加算器を作り、返ってきたクロージャを数回使います。

## 線形クロージャ

線形クロージャはGCによって解放されません。
メモリを手動で解放する責任はプログラマにあります。
線形クロージャは型システムによって追跡され、
線形クロージャを解放する関数を呼び忘れるとコンパイル時エラーになります。
線形クロージャは "cloptr" タグを持っています。
先程と同じ例を線形クロージャを使って書いてみましょう。

```ocaml
fun make_adder(n1:int): int -<cloptr> int =
  lam (n2) =<cloptr> n1 + n2;

implement main() = let
  val add5 = make_adder(5)
  val a = add5(10)
  val b = add5(20)
  val () = printf("a: %d\nb: %d\n", @(a,b))
  val () = cloptr_free(add5)
in
  ()
end
```

関数のタグが "cloref" から "cloptr" に代わっていることと、
"main"関数の最後に "cloptr_free" 呼び出しが追加されていることに注意してください。
"cloptr_free" 呼び出しを削除すると、コンペイル時エラーになります。

```
$ atscc -o closure2 closure2.dats
...
error(3): the linear dynamic variable [add5] needs to be consumed but it is preserved...
...
```

線形クロージャが使用しているメモリを手動で管理するため、GCは不要です。

## スタックに確保されたクロージャ

ヒープではなくスタックに線形クロージャを確保することができます。
その場合には、スタックスコープを抜けると自動的に解放されるので、
手動で解放する必要はありません。

```ocaml
implement main() = let
  var !add5 = @lam(n2:int) => 5 + n2
  val a = !add5(10)
  val b = !add5(20)
  val () = printf("a: %d\nb: %d\n", @(a,b))
in
  ()
end
```

クロージャをスタックに確保するには "lam" の代わりに "@lam" を使います。
ここでは "add5" に割り当てています。
この変数はこれまでの例で使ってきた "val" ではなく "var" を使って宣言されています。
"val" は "値の宣言" で、 "var" は "変数の宣言" です。
後者は基本的にスタックに確保された変数で、評価の最中に更新される可能性があります。
詳しくは
[val and var tutorial](http://www.ats-lang.org/htdocs-old/TUTORIAL/contents/val-and-var.html)
を読んでください。

スタックに確保された変数は、
そのアドレスへの操作を許可するために使う暗黙の証明変数を持っています。
先の例での "!" はATSの略記のようで、ある状況で証明による管理を無効化するために使うようです。
私はなぜこれが必要で、どうやって機能するのかよく理解できていません。
助けになる情報がありましたら是非教えてください。
[pointer tutorial](http://www.ats-lang.org/htdocs-old/TUTORIAL/contents/pointers.html)
がこの話題について有用でしょう。

この例では "make_adder" 関数からクロージャを作りませんでした。
もしクロージャをスタックに確保して "make_adder" から抜けると、エラーになるでしょう。
スタックにオブジェクトを確保したにもかかわらずスタックから抜けることで破壊してしまうためです。
関数によってスタックフレームに確保されたクロージャを、
その関数から抜けてから使えないことがATSの型システムによって保証されています。
そのような場合にはコンパイル時に警告が起きます。

```
error(3): a flat closure is needed.
```

## Standard Prelude Higher Order Functions

リストや配列を扱う一般的な高階関数の実装をATSは持っています。
"map" や "filter" などの関数が使えます。
次の例では高階関数を使ってリストの要素の和を取っています。

```ocaml
staload "prelude/DATS/list.dats"

implement main() = () where {
  val a = '[1,2,3]
  val x = list_fold_left_cloref<int,int> (lam (a,b) =<cloref> a*b, 1, a)
  val () = printf("result: %d\n", @(x));
}
```

"list_fold_left_cloref" は "prelude/DATS/lists.dats" で定義されています。
これはC++のテンプレートとよく似た関数テンプレートで、
インスタンス化するためにテンプレートの型が必要になります。
この例ではその型は2つともintです。
このコードは"cloref"を使うために、GCが必要になります。

線形クロージャや関数に使うためのlist_fold_leftの変形があります。
例えば線形クロージャを使う場合には以下のようになります。

```ocaml
staload "prelude/DATS/list.dats"

implement main() = () where {
  val a = '[1,2,3]
  val clo = lam (pf: !unit_v | a:int,b:int):int =<cloptr> a*b;
  prval pfu = unit_v ()
  val x = list_fold_left_cloptr<int,int> (pfu | clo, 1, a)
  prval unit_v () = pfu
  val () = cloptr_free(clo)
  val () = printf("result: %d\n", @(x));
}
```

このコードでは "prval" を使っていることに気づくでしょう。
"list_fold_left_cloptr" の定義は以下のようになっています。

```ocaml
fun{init,a:t@ype} list_fold_left_cloptr {v:view} {f:eff}
  (pf: !v | f: !(!v | init, a) -<cloptr,f> init, ini: init, xs: List a):<f> init
```

この定義は "list_fold_left_cloptr" と引数 "f" が証明引数を必要としていることを示しています。
("|" というパイプ文字の左側全ては証明引数です。この例では "!v" です。)
この例では証明引数を使わないため、"unit_v ()" を作り、その後二回目の "prval" で解放します。
前にも書きましたが、なぜ証明引数を取るのか、なぜクロージャにそれを渡すのか、
私は理解できていません。
有識者のコメントを募集しています!

この例では手動でクロージャのメモリ管理をしているにもかかわらず、
リスト"a"の確保のためにGCが必要であることに注意してください。
GCが内蔵されていたとしても、効率のために必要であれば、
手動でメモリ管理を行なうことができるのです。

証明変数の使用し、クロージャーのメモリを手動管理し、より多くの証明をするのであれば、
ATSには別の関数が用意されています。
"smlbas" ルーチンは、"cloref1" 関数型が使えるMLの基本的な標準ライブラリを提供しています。
例を見てみましょう。

```ocaml
staload "libats/smlbas/SATS/list.sats"
staload "libats/smlbas/DATS/list.dats"
dynload "libats/smlbas/DATS/list.dats"

implement main() = () where {
  val a = list0_of_list1('[1,2,3])
  val x = foldl<int,int>(lam(a, b) => a + b, 1, a)
  val () = printf("result: %d\n", @(x));
}
```

ビルドするには "ats_smlbas" ライブラリをリンクする必要があります。

```
atscc -o closure8 closure8.dats -D_ATS_GCATS -lats_smlbas
```

コンパイル可能にするために "foldl" テンプレートに型を渡してやる必要があります。
型推論が他の関数型言語と同じようには働いてくれないのです。
整数の加算関数を使うことを指定してやるアプローチもあります。

```ocaml
  val x = foldl(lam(a, b) => add_int_int(a, b), 1, a)
```

不幸にも、クロージャの引数の型を定義するだけでは不十分です。

```ocaml
  val x = foldl(lam(a:int, b:int) => a + b, 1, a)
```

これは次のようなエラーになってしまいます。

```
  error(3): the symbol [+] cannot be resolved: there are too many matches
```

## 結論

This concludes my explorations of using closures and functions in ATS.
I find using higher order functions somewhat difficult due to dealing with proofs and having to declare types everywhere.

This would probably be a bit easier if there was reference documentation on the ATS standard library with usage examples.
Unfortunately I’m not aware of any apart from the library and example source code.

At least it’s possible to use the ‘smlbas’ library and the simplified ‘list0’ and ‘array0’ types to avoid needing to understand the low level details of proofs and types while learning.

xxx
