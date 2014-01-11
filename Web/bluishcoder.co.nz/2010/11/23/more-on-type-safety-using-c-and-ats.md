# BLUISH CODER: より安全なATSによるC言語の使い方

(元記事は http://bluishcoder.co.nz/2010/11/23/more-on-type-safety-using-c-and-ats.html です)

以前 [ATSプログラミング言語](http://www.ats-lang.org/) を使った
[安全なC言語の使い方](http://bluishcoder.co.nz/2010/06/02/safer-c-code-using-ats.html) を紹介しました。
私は最近いくつかのプロジェクトでATSを使っていて、C言語ライブラリをATSから扱わねばなりませんでした。
この記事ではある種類のエラーを型検査のエラーにすることで、
C言語ライブラリの不正な使用を防止するより多くの方法を紹介します。

私のプロジェクトでは [JSON](http://json.org/) リクエストを構文解析する必要がありました。
この用途に適したC言語ライブラリは [jansson](http://digip.org/jansson) です。
このライブラリは良い実装と
[良いドキュメント](http://www.digip.org/jansson/doc/1.3) を合わせ持っています。

janssonのAPIで使われるJSONオブジェクトはリファレンスカウントによって管理されていて、
ドキュメントにはどのAPIが参照の所有権を持ち、どのAPIが持たないかが明記されています。
それでも一見正しいコードが問題を引き起こすことはあります。
そのようなコーナーケースは、ATSのラッパーがコンパイル時に検出することが望まれます。

## リファレンスカウント

リファレンスカウントの操作ミスは明らかにエラーの発生源です。
次のC言語プログラムはJSONオブジェクトのデリファレンスに失敗しています。
結果としてメモリリークを引き起こしてしまいます。

```c
#include <stdio.h>
#include <jansson.h>
    
int main() {
  json_t* a = json_string("this is a string");
  const char* s = json_string_value(a);
  printf("String value: %s\n", s);
  return 0;
}
```

[valgrind](http://valgrind.org/) を使ってこのプログラムを走らせるとリークしていることを確かめらます。
JSONオブジェクトのリファレンスカウントをデクリメントさせれば解決します。

```c
json_t* a = json_string("this is a string");
const char* s = json_string_value(a);
printf("String value: %s\n", s);
json_decref(a);
```

JSON型をATSの線形型として設計することで、
そのオブジェクトが確実に解放されることをコンパイル時に保証することができます。
これは、"json_t"に対応する"absviewtype"を宣言し、
関数の宣言内で必要に応じてその線形型を保持したり消費したりすることで実現できます。
janssonコードをATSでラップすると次のようになります。

```ocaml
%{^
#include <jansson.h>
%}

absviewtype JSONptr (l:addr)

extern fun json_string
  (value: string)
  : [l:addr] JSONptr l = "#json_string"
extern fun json_string_value
  {l1:agz} (json: !JSONptr l1)
  : string = "#json_string_value"
extern fun json_decref
  {l:agz} (json: JSONptr l)
  : void = "#json_decref"
extern fun JSONptr_is_null
  {l:addr} (x: !JSONptr l)
  :<> bool (l == null) = "#atspre_ptr_is_null"
extern fun JSONptr_isnot_null
  {l:addr} (x: !JSONptr l)
  :<> bool (l > null) = "#atspre_ptr_isnot_null"
```

これらのラッパーを定義すると、
"main"関数の中身はC言語のバージョンと良く似た見た目になります。

```ocaml
implement main () = () where {
  val a = json_string("this is a string")
  val () = assert_errmsg(JSONptr_isnot_null a, "json_string failed")
  val s = json_string_value(a)
  val () = print(s)
  val () = print_newline()
  val () = json_decref(a)
}
```

このATSコードはさらに別の安全性を保証もしています。
janssonのドキュメントによると、"json_string"はNULLを返す可能性があります。
"json_string"を型としてATSで定義しました。
"json_string_value"を非NULLポインタを要求するように宣言しました。
"a"がNULLかどうかチェックしていないプログラムをATSはコンパイルしません。

"json_decref"は線形型を消費するように宣言しました。
これによって元のC言語バージョンコードのように"json_decref"を呼び出しを削除すると、
プログラムのコンパイルに失敗するようになります。

## 借用されたオブジェクト

janssonではいくつかのAPIが"借用された"オブジェクトを返します。
インクリメントしなかったリファレンスカウントを持つJSONオブジェクトがあるのです。
呼び出し元は当該のオブジェクトを所有しません。
もし呼び出し元が当該オブジェクトを所有し続けたいなら、
リファレンスカウントを自分自身でインクリメントする必要があります。

これはおそらく速度効率のためでしょう。
もしもただオブジェクト回収して捨てるだけれであれば、
呼び出し元はリファレンスカウントを最適化のために削除できます。

どの関数が借用されたオブジェクトを返し、
どれが呼び出し元が所有するオブジェクトを返すのかについて、
janssonのドキュメントにはしっかり書いてあります。

オブジェクトが破棄されてしまった後に、
借用されたオブジェクトを不正に使うC言語プログラムの例をここでは見てみましょう。

```c
int main() {
  json_error_t e;
  json_t* a = json_loads("[\"a\", \"b\", \"c\"]", 0, &e);
  assert(a);
  json_t* a2 = json_array_get(a, 1);
  assert(a2);
  json_decref(a);
  const char* s = json_string_value(a2);
  printf("String value: %s\n", s);
  return 0;
}
```

この例では"a"が破棄された後に"a2"オブジェクトが使用されています。
しかし"a2"は借用されたオブジェクトで"a"によって所有されています。
"a"が破棄されてましまうと、同時に"a2"も破棄されてしまいます。
これはコンパイルが通り実行できますが、結果は不正なものになります。
(場合によっては実行に失敗する可能性もあります。)
valgrindで実行すればメモリエラーが見つかるはずです。

同じことをするATSプログラムを見てみましょう。
(このコードはまだコンパイルできないません。)

```ocaml
implement main () = () where {
  var e: json_error_t?
  val a = json_loads("[\"a\", \"b\", \"c\"]", 0, e)
  val () = assert_errmsg(JSONptr_isnot_null a, "json_loads failed")
  val (pf_a2 | a2) = json_array_get(a, 1)
  val () = assert_errmsg(JSONptr_isnot_null a2, "json_array_get failed")
  val () = json_decref(a)
  val s = json_string_value(a2)
  val () = print(s)
  val () = print_newline()
}
```

このコードは追加のjansson呼び出しのためにATSの定義が必要です。

```ocaml
abst@ype json_error_t = $extype "json_error_t"
extern fun json_array_get
  {n:nat} {l1:agz} (json: !JSONptr l1, index: int n)
  : [l2:addr] (minus(JSONptr l1, JSONptr l2) | JSONptr l2)
  = "#json_array_get"
extern fun json_loads
  (input: string, flags: int, error: &json_error_t?)
  : [l:addr] JSONptr l = "#json_loads"
```

The ‘json_array_get’ function is defined to return a proof.
The ‘minus’ is an ATS standard library proof function.
It basically states that the result of ‘json_array_get’ is owned by the object originally passed to it.
This proof must be consumed when the result is no longer needed.

By requiring this proof, and the need for it to be consumed, we ask the programmer to define the lifetime of the borrowed object.
If we don’t consume the proof we get a compile error.
If we call ‘json_decref’ on the owning object then we can’t consume the proof later as the owning object (’a’ in this example) is consumed by the ‘json_decref’.
This forces the proof to be consumed within the lifetime of the owning object.

Another feature of the ‘json_array_get’ definition is that the ‘index’ parameter is defined as being a natural number.
It is a compile error to pass a negative number as the index.

The correct ATS code becomes:

```ocaml
implement main () = () where {
  var e: json_error_t?
  val a = json_loads("[\"a\", \"b\", \"c\"]", 0, e)
  val () = assert_errmsg(JSONptr_isnot_null a, "json_loads failed")
  val (pf_a2 | a2) = json_array_get(a, 1)
  val () = assert_errmsg(JSONptr_isnot_null a2, "json_array_get failed")
  val s = json_string_value(a2)
  val () = print(s)
  val () = print_newline()
  prval () = minus_addback(pf_a2, a2 | a)
  val () = json_decref(a)
}
```

xxx
