# ATSのクロージャ

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
The variable is introduced using ‘var’ instead of ‘val’ which the examples have been using so far.
‘val’ is for ‘value declarations’ and ‘var’ is for ‘variable declarations’.
The latter are basically variables allocated on the stack that can be changed during the course of evaluation.
See the
[val and var tutorial](http://www.ats-lang.org/TUTORIAL/contents/val-and-var.html)
for details.

Variables allocated on the stack have an implicit proof variable that is used to allow access to its address.
The use of ’!’ in the example appears to be a shorthand provided by ATS that can be used to avoid explicitly managing the proof’s in some circumstances.
I don’t entirely understand why this is needed or how it works.
If anyone has any helpful tips please let me know.
The
[pointer tutorial](http://www.ats-lang.org/TUTORIAL/contents/pointers.html)
seems to have some coverage of this.

For this example I don’t create the closure in a ‘make_adder’ function.
If the closure was allocated on the stack and returned from ‘make_adder’ it would be an error as we’d be returning a stack allocated object from a stack that just got destroyed.
In ATS it is guaranteed by the type system that a closure allocated on the stack frame of a function cannot be used once the call to the function returns.
A compile time warning occurs if this happens:

```
error(3): a flat closure is needed.
```

## Standard Prelude Higher Order Functions

xxx
