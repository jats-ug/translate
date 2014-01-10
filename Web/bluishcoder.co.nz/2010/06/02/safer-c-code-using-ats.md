# BLUISH CODER: ATSによる安全なC言語の使い方

(元記事は http://bluishcoder.co.nz/2010/06/02/safer-c-code-using-ats.html です)

[Firefox向け独自Oggバックエンド](http://bluishcoder.co.nz/2008/07/theora-video-backend-for-firefox-landed.html)
を開発する時、
C言語ライブラリと統合する必要がありました。
公開して数ヶ月後バックエンドに様々は不具合が見つかりました。
これらの多くは共通のコーディングミスの結果によるものだったのです。
それは、C言語のAPI呼び出しからの返値をチェックし忘れたり、
リソースを正しく解放していなかったり、というようなミスです。
これは、私達のコード中だけではなく、私達が使用している第三者のライブラリの中でも起きていました。
この問題のおかげで、
私はこのような種類の不具合を発見しやすいプログラミング言語を探すことに興味を持つようになりました。

[以前紹介した](http://bluishcoder.co.nz/tags/ats/) [ATS](http://ats-lang.org/)
は依存型と線形型を使ったプログラミング言語です。
ATSではC言語コードを直接使うことができます。
さらにそのコードに型注釈を付けることである種類のエラーをコンパイル時に防止することができます。
ATSの使い方に関する論文がいくつかあります。

* [Operating System Development with ATS](http://portal.acm.org/citation.cfm?id=1707793&dl=ACM)
* [Implementing Reliable Linux Device Drivers with ATS](http://portal.acm.org/citation.cfm?id=1292597.1292605)
* [Safe Programming with Pointers through Stateful Views](http://www.ats-lang.org/PAPER/SPPSV-padl05.pdf)

この記事では、返値とリソース解放の静的なチェックを説明し、
[ATS](http://ats-lang.org/) からC言語ライブラリを使う方法を考察します。
まずはじめにシンプルなラッパーからはじめて、
ATSを使って一般的なプログラミングエラーを防止するようなAPIの利用方法を探ります。
この題材として使うライブラリは [libcurl](http://curl.haxx.se/libcurl/) です。

URLからHTMLを読み出して標準出力に印字するlibcurlのシンプルな使い方は以下のようになります。
([simple.c](http://bluishcoder.co.nz/ats/curl/simple.c))

```c
#include <curl/curl.h>

int main(void)
{
  CURL *curl;
  CURLcode res;

  curl = curl_easy_init();
  curl_easy_setopt(curl, CURLOPT_URL, "bluishcoder.co.nz");
  res = curl_easy_perform(curl);
  curl_easy_cleanup(curl);
  return 0;
}
```

以下のようにコンパイルして実行できるでしょう。

```
$ gcc -o simple simple.c `curl-config --libs`
$ ./simple
...
```

## 確実なリソース解放

先のコードをATSに翻訳してみると
[simple1.dats](http://bluishcoder.co.nz/ats/curl/simple1.dats)
([pretty-printed version](http://bluishcoder.co.nz/ats/curl/simple1.html))
のようになるでしょう。

```ocaml
%{^
#include <curl/curl.h>
%}

absviewtype CURLptr (l:addr) // CURL*

abst@ype CURLoption = $extype "CURLoption"
macdef CURLOPT_URL = $extval(CURLoption, "CURLOPT_URL")

extern fun curl_easy_init {l:addr}
  ()
  : CURLptr l = "#curl_easy_init"
extern fun curl_easy_setopt {l:addr} {p:type}
  (handle: !CURLptr l, option: CURLoption, parameter: p)
  : int = "#curl_easy_setopt"
extern fun curl_easy_perform {l:addr}
  (handle: !CURLptr l)
  : int = "#curl_easy_perform"
extern fun curl_easy_cleanup {l:addr}
  (handle: CURLptr l)
  : void = "#curl_easy_cleanup"

implement main() = let
  val curl  = curl_easy_init();
  val res = curl_easy_setopt(curl, CURLOPT_URL, "www.bluishcoder.co.nz");
  val res = curl_easy_perform(curl);
  val () = curl_easy_cleanup(curl);
in
  ()
end;
```

"main"関数はC言語のものと、とてもよく似ています。
それより前に列挙された定義によって、libcurlのC言語関数,型,列挙子はATSから使えるようになります。
これをコンパイルして実行するには以下のようにします。

```
$ atscc -o simple1 simple1.dats `curl-config --libs`
$ ./simple1
..
```

libcurlの型CURLoptionはC言語の列挙子です。
ATSでこれをラップするために $extype を使ってC言語の名前と直接結びつけています。

```ocaml
abst@ype CURLoption = $extype "CURLoption"
```

この型の値を一つだけ参照することにしましょう。
それは CURLOPT_URL で、これはまたC言語の値を直接参照させます。
これは型ではなく値なので $extval を使います。

```ocaml
macdef CURLOPT_URL = $extval(CURLoption, "CURLOPT_URL")
```

CURL型はC言語における抽象型です。
ここではそれが実際は何で構成されているのかは気にしないことにします。
CURLオブジェクトはいつもポインタを経由して参照されます。
これはATSで以下のようにモデル化できます。

```ocaml
absviewtype CURLptr (l:addr) // CURL*
```

"CURLptr" が "CURLオブジェクトへの参照" として振る舞うと考えてください。
これはC言語では基本的に"CURL*"で表わされます。

上記の定義によって、
CURLptrはメモリアドレス"l"に配置された"CURLptr"型のオブジェクトであることを宣言していることになります。

curl_easy_initのC言語定義は以下です。

```
CURL *curl_easy_init(void);
```

ATSにおける定義は以下のようになります。

```ocaml
extern fun curl_easy_init {l:addr} () : CURLptr l = "#curl_easy_init"
```

行頭の"extern"と行末の"#curl_easy_init"は、
この関数の実装は外部ライブラリの中にある"curl_easy_init"と呼ばれるC言語関数であることを意味しています。
関数名の後に続く"{l:addr}"は返値の型において使う"l"の型を与えています。
つまりそれはポインタです。
返値の型である"CURLptr l"は既に定義されています。
この関数は"l"というアドレスに配置されたCURLptrオブジェクトを返します。

curl_easy_setoptのC言語側の定義は次のようになっています。

```c
CURLcode curl_easy_setopt(CURL *curl, CURLoption option, ...);
```

このC言語の関数は可変長引数を取るようです。
この例では追加の引数を1つだけ取れるようにしましょう。
確かにごまかしかもしれませんが、ATSが解釈できる宣言です。
可変長引数を扱う方法は後で説明します。
ATSでの定義は以下のようになります。

```ocaml
extern fun curl_easy_setopt {l:addr} {p:type}
  (handle: !CURLptr l, option: CURLoption, parameter: p)
  : int = "#curl_easy_setopt"
```

この関数は3つの引数を取り、1つの返値を返します。
その引数というのは以下になります。

* "!CURLptr l"。先に宣言されている通り、これは"l"アドレスに配置されたCURLptrオブジェクトです。"!"はcurl_easy_setoptがそのオブジェクトを消費しないことを表わしています。つまり、後の処理でこのオブジェクトを継続使用できることになります。{l:addr} の使用に先立って、"l"は"addr"型であると宣言されています。
* "CURLoption"
* 型"p"のパラメータ値。この値は"type"という型です。{p:type} と定義されているためです。マシンワード(ポインタ、整数、などなど)のサイズに一致する全てのオブジェクトは、ATSではこの型になります。

この例で使っている残った2つのCURL関数はこれまでの変形です。
主な差異は"curl_easy_cleanup"のATSにおける定義でしょう。

```ocaml
extern fun curl_easy_cleanup {l:addr}
  (handle: CURLptr l)
  : void = "#curl_easy_cleanup"
```

ここでの"handle"のパラメータは"CURLptr l"です。
またその前に"!"を付けていません。
これはこの関数が引数を消費してしまい、
この呼び出し以降その引数を使用することができないことを意味しています。
"CURLptr l"型は線形型です。
この型の値はプログラムのどこかで破棄されねばらず、
そして一度破棄されたら二度と使えないのです。

このように"curl_easy_cleanup"を定義することで、
プログラマがまだ解放されていない"CURLptr l"とともにこの関数を呼び出すことを強制できます。
練習のためにですが、
もしcleanup呼び出しを削除してコンパイルしてみるとどうなるでしょうか？
コンパイルエラーが起きることになるでしょう。
またcleanup呼び出しの後にcurlハンドラを使ってみてもコンパイルエラーになります。

線形型を使用しているため、
プログラマが起こしがちなエラーの一部については既に安全です。
確保されたオブジェクトのcleanup関数呼び忘れなようなことを防止できるのです。

## NULLポインタの扱い

しかし、このバージョンでもまだ潜んでいるバグがあります。
そのバグは同時にC言語プログラムにも潜んでいます。
["curl_easy_init"](http://curl.haxx.se/libcurl/c/curl_easy_init.html)
はNULLを返すことがあります。
NULLが返った場合には他のいかなるcurl関数も呼び出してはなりません。
ATSを使うことでNULLをチェックする要件を定義でき、コンパイルを通すことができます。

そのような変更をほどこした先の例の修正バージョンが
[simple2.dats](http://bluishcoder.co.nz/ats/curl/simple2.dats)
([pretty-printed version](http://bluishcoder.co.nz/ats/curl/simple2.html))
です。

このバージョンではCURLptr型に2つのtypedefを追加します。
1つ目であるCURLptr0はNULLを取ることができます。
2つ目のCURLptr1はNULLを取ることができません。

```ocaml
absviewtype CURLptr (l:addr) // CURL*
viewtypedef CURLptr0 = [l:addr | l >= null] CURLptr l
viewtypedef CURLptr1 = [l:addr | l >  null] CURLptr l
```

使用しているcurlの関数の定義はこれらの型を使うように変更されています。

```ocaml
extern fun curl_easy_init
  ()
  : CURLptr0 = "#curl_easy_init"
extern fun curl_easy_setopt {p:type}
  (handle: !CURLptr1, option: CURLoption, parameter: p)
  : int = "#curl_easy_setopt"
extern fun curl_easy_perform
  ( handle: !CURLptr1)
  : int = "#curl_easy_perform"
extern fun curl_easy_cleanup
  (handle: CURLptr1)
  : void = "#curl_easy_cleanup"
```

"curl_easy_init"はNULLを返すので、CURLptr0を返すように変更します。
他の関数はCURLptr1を使い、NULLを返してはならないことを表わします。
この段階で以前のバージョンの"main"関数をそのまま使い"curl_easy_init"の結果をチェックしない場合、
コンパイル時エラーになります。
このエラーは"curl"変数の型の型が"curl_easy_opt"による要求と一致しないと言っているはずです。

これを修正するにはNULLのチェックを追加します。

```ocaml
implement main() = let
  val curl = curl_easy_init();
  val () = assert_errmsg(CURLptr_isnot_null curl, "curl_easy_init failed");
  val res = curl_easy_setopt(curl, CURLOPT_URL, "www.bluishcoder.co.nz");
  val res = curl_easy_perform(curl);
  val ()  = curl_easy_cleanup(curl);
in
  ()
end;
```

コンパイラは、assertチェックをした後には"curl"がNULLを取らないことを知っています。
このおかげでプログラムの残り部分の型検査をすることができるようになります。

# 返値チェックの強制

もう一つの一般的なエラーに、エラー時のC言語APIの返値のチェック漏れが挙げられます。
これはFirefox Oggビデオバックエンドの最初のバージョンでの不具合によって立証されています。
私達が使っている第三者のライブラリでも返値はチェックされていないことがありました。
ATSでは、
もしエラー時の返値がチェックされない場合コンパイル時エラーになるような型を定義できます。

"curl_easy_setopt"と"curl_easy_perform"の返値は成功もしくは失敗を表わしています。
非ゼロの値はエラーです。

そのような値のチェックを強制する変更を加えたバージョンが
[simple3.dats](http://bluishcoder.co.nz/ats/curl/simple3.dats)
([pretty-printed version](http://bluishcoder.co.nz/ats/curl/simple3.html))
です。
"curl_easy_setopt"の定義が以下のように変更されています。

```ocaml
extern fun curl_easy_setopt {p:type}
  (handle: !CURLptr1 >> opt(CURLptr1, err == 0),
   option: CURLoption, parameter: p)
  : #[err:int] int err = "#curl_easy_setopt"
```

これと同じ変更が"curl_easy_perform"にもほどこされています。
まず"handle"パラメータが
"!CURLptr1 » opt(CURLptr1, err == 0)"
型に変更されています。
"»"の前で定義している型は関数が受け付ける入力です。
"»"の後ろは、関数が返った後の、このパラメータの型です。

つまり関数が返った後では、
"err"がゼロであるなら"handle"の型は"CURLptr1"であるということを表わしています。
"err"は後のコードで整数として定義されています。
これは返値がゼロであることをチェックするコードを呼び出すことを強制します。
このコードは"opt"から展開したCURLptr1型を使って実行を継続します。
修正された"main"関数がこの動作を表わしています。

```ocaml
implement main() = let
  val curl = curl_easy_init();
  val () = assert_errmsg(CURLptr_isnot_null curl, "curl_easy_init failed");
  val res = curl_easy_setopt(curl, CURLOPT_URL, "www.bluishcoder.co.nz");
  val () = assert_errmsg(res = 0, "curl_easy_setopt failed");
  prval () = opt_unsome(curl);
  val res = curl_easy_perform(curl);
  val () = assert_errmsg(res = 0, "curl_easy_perform failed");
  prval () = opt_unsome(curl);
  val ()  = curl_easy_cleanup(curl);
in
 ()
end;
```

関数の返値をassertでチェックしていることに注意してください。
その後"opt_unsome"という呼び出しで、型から"optを外し"て"CURLptr1"として使用できるようにします。
assertチェックもしくは"opt_unsome"をコメントにするとコンパイルは通りません。
0以外の値についてassertチェックが行なわれた場合にもコンパイルは通りません。
簡潔さのためにここではassertを使いましたが、
"if"文を使って結果をチェックすることもできます。
