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

xxx
