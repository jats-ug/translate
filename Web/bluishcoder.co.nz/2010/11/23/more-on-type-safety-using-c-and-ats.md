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

"json_array_get"関数は証明を返すように定義されています。
"minus"はATSスタンダードライブラリの証明関数です。
"json_array_get"の結果は"json_array_get"に渡された元のオブジェクトによって所有されていることを表わしています。
そこ結果が不要になった時点で、この証明は消費されます。

この証明とそれが消費されなければならないということが、
借用されたオブジェクトの生存期間を定義するよう、プログラマに要請します。
もし証明を消費できないとコンパイルエラーになります。

もし"json_decref"を所有しているオブジェクトに対して呼び出すと、
所有しているオブジェクト(この例では"a"です)は"json_decref"によって消費されてしまっているので、
それ以降で証明を消費することができません。
このため所有しているオブジェクトの生存期間で証明が消費されることが強制されます。

"json_array_get"の定義におけるもう一つの特徴は、"index"パラメータが自然数として定義されていることです。
マイナスの数をindexとして渡すとコンパイルエラーになります。

正しいATSコードは次のようになるでしょう。

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

"minus_addback"という証明関数の呼び出しに注意してください。
これは、証明システムの視点において借用されたオブジェクトを所有しているオブジェクトに返却しています。
証明関数はコンパイル時に使われるということ覚えておいてください。
実行時のオーバーヘッドはありません。

リファレンスカウントをインクリメントし、使い終わったらデクリメントすることで、
借用されたオブジェクトの生存期間を操作することはできます。
しかしこれは実行時のオーバーヘッドがあります。
"借用された"オブジェクトのポイントは速度効率にあります。
型システムを使うことで速度効率を維持して、なおかつ正確なコードに保つことができるのです。
もちろん"a2"を"a"より延命させたいのであれば、そのリファレンスカウントを操作することもできます。

## 内部メモリへのポインタ

借用されたオブジェクトに似た問題として、
JSONオブジェクト内部のメモリへのポインタを返すAPI呼び出しがあります。
そのような最初の例として"json_string_value"を見てみましょう。
文字列のメモリを誰が所有しているのあ、ドキュメントには書いてありません。
それを"解放"するためのAPIがないので、
JSONオブジェクトによって所有されているのではないかと仮定できます。
janssonのソースコードを調べてみると、
返却されている文字列は実際にはJSONオブジェクト内部の文字列値へのポインタであることが発見できるでしょう。
これは文字列の生存期間はJSONオブジェクトの生存期間に紐づいていることを意味しています。

また文字列の値を変更するための"json_string_set"呼び出しもあります。
この関数は古い内部ポインタを解放して、新しい文字列を確保します。
"json_string_value"より前に呼び出された関数の結果であるポインタを、
"json_string_value"呼び出しの後に参照すると、
解放されたメモリを指してしまっていることになります。
誤ったC言語プログラムの例を示します。

```c
int main() {
  json_t* a = json_string("this is a string");
  const char* s = json_string_value(a);
  json_string_set(a, "oops");
  printf("Old string value: %s\n", s);
  json_decref(a);
  return 0;
}
```

"json_string_set"の呼び出した後に"json_string_value"の結果を印字しようとすると、
解放されたメモリを印字してしまいます。

ATSでは、"json_string_value"の返したポインタが生きている場合に、
"json_set_string"を呼び出したり所有しているJSONオブジェクトのリファレンスカウントをデクリメントしたら
、コンパイルエラーになるように強制できます。
先のC言語コードと等価なATSコードを以下に示します。このコードはコンパイルに失敗するはずです。

```ocaml
implement main () = () where {
  val a = json_string("this is a string")
  val () = assert_errmsg(JSONptr_isnot_null a, "json_string failed")
  val (pf_s | s) = json_string_value(a)
  val _ = json_string_set(a, "oops")
  val () = print(s)
  val () = print_newline()
  prval () = JSONptr_minus_addback(pf_s, s | a)
  val () = json_decref(a)
}
```

"json_string_set"呼び出しを削除したり"JSONptr_minus_addback"呼び出しの後ろに移動させても、
コンパイルは通ります。

このしくみを作るためにjansson型のラッパーを修正しましょう。
これによって"json_string_value"の呼び出し元の数を記録してくれるようになります。
"json_t"ポインタの線形型は以下のようになります。

```ocaml
absviewtype JSONptr (l:addr,n:int)
```

以前はポインタのアドレスがパラメータでした。
これでJSONptrは整数のカウンタも含むようになりました。
このカウンタはオブジェクトが生成された時にゼロで初期化されます。

```ocaml
extern fun json_string
  (value: string)
  : [l:addr] JSONptr (l, 0) = "#json_string"
```

"json_string_value"を呼び出すと、カウンタがインクリメントされます。
そのため、この呼び出し以降はJSONオブジェクトの型は非ゼロの値を持つことになります。

```ocaml
extern fun json_string_value
  {l1:agz} {n:int} (
    json: !JSONptr (l1, n) >> JSONptr (l1, n+1)
  ) : [l2:addr] (JSONptr_minus (l1, strptr l2) | strptr l2)
  = "#json_string_value"
```

ATSの"X » Y"という構文は"現時点でのオブジェクトの型 » 呼び出し後のオブジェクトの型"という意味です。
返値は"JSONptr_minus"という証明を保持しています。
この証明はこれまで使ってきた証明"minus"の変種です。
この証明は、"n"の値を気にする必要がないように単純化されているという点を除いて、同じ目的で使われます。
よく似た"JSONptr_minus_addback"もあります。

```ocaml
absview JSONptr_minus (addr, view)
extern prfun JSONptr_minus_addback {l1:agz} {v:view} {n:int}
  (pf1: JSONptr_minus (l1, v), pf2: v |
   obj: !JSONptr (l1, n) >> JSONptr (l1, n-1)): void
```

オブジェクト内にあるパラメータ"n"の値を、
"JSONptr_minus_addback"呼び出しが減少させていることに注意してください。
"json_string_value"の結果が生きている間に"json_string_set"の呼び出しを許可しないようにできる秘密は、
有効な文字列のカウンタがゼロであるJSONオブジェクトのみを受け付けるように"json_string_set"を宣言することにあります。
"json_decref"も同じ方法で修正します。
つまり、内部の文字列が有効でないオブジェクトに対してのみ呼び出し可能にするのです。

```ocaml
extern fun json_string_set
  {l:agz} (
    json: !JSONptr (l, 0), value: string
  ) : int = "#json_string_set"
extern fun json_decref
  {l:agz} (json: JSONptr (l, 0))
  : void = "#json_decref"
```

この変更で型システムは、
確保していないメモリへのポインタをユーザが保持するミスを犯してしまうのを防止するようになります。

繰り返しますが、このラッパーコードは型検査でのみ使用されます。
ATSが生成する実行時のコードは生きている文字列のカウンタを持ちません。
生成されたコードは元のC言語コードとほとんど同じ見た目になるはです。

## 結論

型システムによる補助がない困難な手動によるリソース管理をしているC言語アプリケーションを、
ATSでラップすることができました。
APIの使用を安全するために型システムを使う方法を考え出し、
C言語を正確に使おうとするとき、「誰がこのオブジェクトやメモリを所有しているのだろうか？」と疑問を持つべきなのだと気づかされました。

この記事で紹介した手法のいくつかは、私の発言からはじまったATSメーリングリストでの
[議論](http://sourceforge.net/mailarchive/forum.php?thread_name=Pine.LNX.4.64.1011210923290.18045%40csa2.bu.edu&forum_name=ats-lang-users)
を元にしています。
この記事よりも多くのアイデアを得るために、このスレッドを読んでみてください。

ATSからjanssonの使用を安全にするためにすべきことは他にもあります。
jansson APIのいくつかは成功か失敗を表わす整数値を返します。
これらの値を確認することを強制できるでしょう。
その手法について、私は以前の記事で解説しました。
jansson APIの一部を使うと、JSONオブジェクトを走査(iteration)できます。
これらのAPIをラップして、
イテレータの生存期間や走査中のオブジェクトの変更などに対する安全を保証すべきです。

考えられるもう一つの改良点は、
間違った型のJSONオブジェクトに対して間違ったAPI呼び出しをできないよう保証することでしょう。
例えばarray型に対して"json_string_value"を呼び出すことを考えてみましょう。
このような時、型で特定された関数を呼び出す前に"json_type"をチェックするように強制すべきです。

私が作ったjanssonライブラリのラッパーはgithubの
[http://github.com/doublec/ats-jansson](http://github.com/doublec/ats-jansson)
に置いてあります。
このラッパーは未完成ですが、
この記事で説明したようなアイデアを試すのには有用だと思います。
