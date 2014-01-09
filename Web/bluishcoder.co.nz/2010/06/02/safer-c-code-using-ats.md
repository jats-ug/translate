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

## Ensuring release of resources

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

I only reference one value of these types.
That is CURLOPT_URL and again I reference the C value directly.
Since this is a value rather than a type I use $extval:

```ocaml
macdef CURLOPT_URL = $extval(CURLoption, "CURLOPT_URL")
```

The CURL type is an abstract type in C - we don’t care what it is actually composed of.
A CURL object is always refered to via a pointer.
This is modeled in ATS with:

```ocaml
absviewtype CURLptr (l:addr) // CURL*
```

Think of ‘CURLptr’ as being ‘a reference to a CURL object’.
It’s basically a C ‘CURL*’.
The definition above states that a CURLptr is an object of type ‘CURLptr’ located at a memory address ‘l’.

The C definition of curl_easy_init looks like:

```ocaml
CURL *curl_easy_init(void);
```

The ATS definition is:

xxx
