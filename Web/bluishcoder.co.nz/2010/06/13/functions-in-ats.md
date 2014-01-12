# BLUISH CODER: ATSの関数

(元記事は http://bluishcoder.co.nz/2010/06/13/functions-in-ats.html です)

[ATS](http://www.ats-lang.org/) において高階関数を使うことは、
はじめは私にとって少し努力が必要でした。
正しい型が得られるように取り組まねばならなかったためです。
私は動的型付けの言語に馴染んでいて、それらの言語では無名関数を生成したり渡したりすることは簡単でした。
型推論があるため、
[Haskell](http://haskell.org/)
のような他の静的型付け言語でさえ理解するのは容易です。
ATSにおける関数の型の違いに慣れるために、私は詳細に調査しました。
私が大きなATSプログラムを作るのにそれらを正しく使いはじめるために重要だったことをこの記事では解説しようと思います。

ATSは2つの型の関数を区別しています。

* "関数": これはまさにC言語の関数に似ています。それは関数本体を表わすマシンコードを指すメモリアドレスのことです。関数はそれが生成された場所での環境を保存できません。そのため他の言語における"クロージャ"のような振舞いはしません。デフォルトでは、トップレベルで定義されたATS関数は"関数"です。関数ポインタ使って外部のC言語ルーチンに直接渡すことができます。
* "クロージャ": これは名前から値への写像である環境をともなう関数です。まさに他の動的型付け言語のクロージャと同じで、閉じられたスコープの中に値を保管することができます。クロージャを生成するためには、明示的にクロージャであることを指示する必要があります。クロージャは環境を保存するために確保されたメモリ領域を要求します。つまり、スタックにクロージャを確保するか、手動でクロージャのメモリを解放するか、GCと紐づける必要があります。スタックにクロージャを確保した場合、そのスコープから抜けると自動的にクロージャは解放されてしまいます。

関数の2つの型だけではなく、ATSは関数呼び出しによって起きるエフェクトを追跡することができます。
追跡できるエフェクトの型を次に挙げます。

* exn - その関数は例外を発生させることができます
* ntm - その関数は終了しない可能性があります
* ref - その関数は共有メモリの内容を更新する可能性があります

これは多くの組み合わせを作り、異なる型を持つ関数をもたらします。

これらのエフェクトと"関数"か"クロージャー"の分類は、
関数の定義での返値の型の前にある "<" と ">" の間に配置された "タグ" で注釈されます。
タグのない関数定義は次のようになります。

```ocaml
fun add5 (n: int): int = n+5
```

タグを追加してみると次のようになるでしょう。

```ocaml
fun add5 (n: int):<tag1,tag2,tag...> int = n+5
```

タグとして有効な値は次のものです。

* !exn - 例外を発生させる可能性がある関数
* !ntm - 終了しない可能性がある関数
* !ref - 共有メモリを更新する可能性がある関数
* 0 - 純粋な関数(エフェクトを全く持たない)
* 1 - 全てのエフェクトを持つ関数
* fun - 通常の関数、クロージャではない
* cloptr - 線形クロージャ、明示的な解放が必要
* cloref - 永続クロージャ、GCによる解放を要求

単純化のために "0" と "1" のタグは "fun", "cloptr", "cloref"
の接尾辞として使えることに注意してください。

* fun0 - 純粋な関数
* fun1 - 全てのエフェクトを持つ関数
* cloptr0 - 純粋な線形クロージャ
* cloptr1 - 全てのエフェクトを持つ線形クロージャ
* cloref0 - 純粋な永続クロージャ
* cloref1 - 全てのエフェクトを持つ永続クロージャ

タグの使用例をいくつか見てみましょう。
1つ以上使用例が与えられている場合はそれらの定義は等価です。

次の関数は、intを取り、intを返し、純粋で、副作用を起こさず、例外を発生させず、必ず終了します。

```ocaml
fun add5 (n: int):<fun0> int = n+5
fun add5 (n: int):<fun> int = n+5
fun add5 (n: int):<> int = n+5
```

次の関数は、intを取り、intを返し、副作用を起こす可能性があり、例外を発生させる可能性もあり、終了しない可能性もあります。

```ocaml
fun add5 (n: int):<fun1> int = n+5
fun add5 (n: int):> int = n+5
```

次の関数は、intを取り、intを返し、例外を発生させる可能性があることを除いては純粋です。

```ocaml
fun add5 (n: int):<!exn> int = n+5
```

エフェクトが一致しないためにコンパイル時エラーになる例を1つ見てみましょう。
"outer"関数は純粋な関数として定義されていますが、純粋でない関数を呼び出しています。

```ocaml
fn inner ():<fun1> int = 5
fn outer ():<fun0> int = inner()

implement main() = print_int(inner())
```

ここでは"fun"ではなく"fn"を使っていることに注意してください。
"fn"は再帰できない関数を明示的に定義するのに使われます。
"fun"は自分自身を呼び出し再帰を作ることを関数に許します。
後者は終了しない関数であることを注釈していて、
もし"fun"を使ったら別のコンパイルエラーになってしまいます。
終了する関数であること(termination metric)を自動的に示すために、ここでは"fn"を使いました。

エフェクトが一致しないために起きるもう一つのコンパイル時エラーとしては、
"outer"が例外を発生させないにもかかわらず、例外を発生させる関数を呼び出してしまう例があります。

```ocaml
fn inner ():<!exn> int = 5
fn outer ():<fun0> int = inner()

implement main() = print_int(inner())
```

"outer"が例外を発生できるように修正すれば、前の例のコンパイルは通ります。

```ocaml
fn inner ():<!exn> int = 5
fn outer ():<fun0,!exn> int = inner()

implement main() = print_int(inner())
```

## 高階関数

関数は他の関数を引数として取ったり、関数を返すことができます。
これらの関数の型の定義は、トップレベル関数の定義の構文と次の点が違うだけです。

* "fun" や "fn" を取り除く
* 関数名をつけない
* 返値とタグの前にある ":" を "-" に変える

例えば次に示すように関数の引数と返値の型を定義できます。

```ocaml
typedef foo_t = int -<fun1> int
fun bar (f: foo_t): int = f(42);

fun add5 (n: int): int = n+5
fun baz (): foo_t = add5;

fun bar2 (f: int -<fun1> int) = f(42);
fun baz2 (): int -<fun1> int = add5;
```

関数のエフェクトと型は一致していなければならないことに注意してください。
次の例は、純粋な関数と純粋でない関数を使うことによるエラーです。

```ocaml
fun add5 (n: int):<fun1> int = n+5
fun bar (f: int -<fun0> int) = f(42)

implement main() = print_int(bar(add5));
```

"bar"関数は、純粋な関数が引数として渡されることを期待しています。
しかし"add5"は純粋な関数ではありません。

ATSは、"lam"キーワードを使った無名関数の生成をサポートしています。

"lam"を使った無名関数は、トップレベル関数の構文と次の点だけが違います。

* "fun" や "fn" を "lam" に変える
* 関数名をつけない

例を挙げます。

```ocaml
fun bar (f: int -<fun1> int) = f(42)
implement main() = print_int(bar(lam (n) => n+10));
```

"lam"と一緒にタグを使うこともできます。

```ocaml
fun bar (f: int -<fun1> int) = f(42)
implement main() = print_int(bar(lam (n) =<fun1> n+10));
```

"fun"タグが付いた関数は、"関数へのポインタ" を引数に取ることを期待するC言語の関数に、
直接渡すことができます。
例を次に挙げます。

```ocaml
%{^
typedef int (*func)(int);

void adder(func f) {
  int n = f(10);
  printf("from C: %d\n", n);
}
%}

extern fun adder(f: (int) -<fun1> int):<fun1> void = "#adder"
fn add5 (n: int):<fun0> int = n+5
implement main() = adder(add5);
```

今後の記事では、永続クロージャと線形クロージャについて、
そしてATSライブラリでの高階関数の使い方を紹介しようと思います。

次のドキュメントがこの記事を書く助けになりました。

* ATS tutorialの[Function or Closure](http://www.ats-lang.org/TUTORIAL/contents/function-or-closure.html)
* [ML Programmers Guide to ATS](http://cs.likai.org/ats/ml-programmers-guide-to-ats)
* [ATS/Anairatsユーザーズガイド](http://www.ats-lang.org/RESOURCE/resource.html)
* [メーリングリストへの投稿: Strange compiler error message](http://article.gmane.org/gmane.comp.lang.ats.user/182)
* [メーリングリストへの投稿: cloptr1 vs cloref1](http://sourceforge.net/mailarchive/message.php?msg_id=496E2053.3000902%40ece.ucsb.edu)
