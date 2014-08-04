# BLUISH CODER: C言語のプログラムをATSに翻訳する

(元記事は http://bluishcoder.co.nz/2011/04/24/converting-c-programs-to-ats.html です)

私は、C言語よりもより良く安全な低レベルのプログラミング言語として [ATS](http://ats-lang.org/) を使う傾向があります。しばしば私はC言語のプログラムから出発して、少しずつ ATS の機能を付け加えます。このようなことをするために、私は既存のC言語コードを利用します。次に ATS を使って高位の関数を書き、最後にコンパイル時の安全性とリソースの利用を検査を追加するために、C言語の部品を ATS に翻訳します。ATS はプログラム中に直接C言語を埋め込むことを許可しているので、このようなことを容易です。

## http-server.c

最近私は、単純なページを配信し、リクエストをプロキシして修正した後、別のサーバに投げるようなプログラムに組み込むための HTTP サーバが必要になりました。イベントシステムと統合した HTTP 通信をしたかったので、[libevent](http://libevent.org/) を使うことに決めました。libevent 2 配付版の中には http-server サンプルプログラムがありました。そこで [それを少し修正したバージョン](http://bluishcoder.co.nz/ats/http-server.c) から始めることにしました。このC言語サンプルコードは次のようにコンパイルできます:

```
gcc -o http-server http-server.c -levent
```

引数にドキュメントルートのパスを渡してこのプログラムを起動すると 8080 ポートに HTTP サーバが起動します。リクエストを受けるとドキュメントルートからファイルを配信します。特殊な URL '/dump' はリクエストの詳細を標準出力に印字します。

## http-server.dats

ATS プログラムに先のコードを埋め込む最初のステップでは
[http-server.dats](http://bluishcoder.co.nz/ats/http-server.dats)
([pretty-printed html](http://bluishcoder.co.nz/ats/http-server.html))
のように同じC言語コードをそのまま ATS プログラムに埋め込みました。
C言語のインクルードファイルは次のように囲みます:

```ocaml
%{^
#include <stdio.h>
...
#include <event2/event.h>
#}
```

'%{^' という印はその中身がC言語コードで、コンパイル済み ATS コードの先頭に配置すべきであることを ATS に教えます。残りのC言語コードは '%{' を使ってそのまま埋め込みました:

```ocaml
%{
/* Try to guess a good content-type for 'path' */
static const char *
guess_content_type(const char *path)
{
    ...
}
...
%}
```

'%{…%}' で囲まれたコードはコンパイル済み ATS コードに直接挿入されます。C言語の main 関数を削除して、その中身を取り出して http-server にしました。これでそれを呼び出す main を ATS で書けるようになりました。C言語コードを呼び出す ATS コードは次のようになります:

```ocaml
extern fun syntax():void = "mac#syntax"
extern fun http_server(docroot: string):int = "mac#http_server"
exception Error of ()

implement main(argc, argv) =
  if argc < 2 then
    syntax()
  else
    if http_server(argv[1]) = 0 then () else $raise Error ()
```

'extern fun' による定義で、C言語コードを ATS コードから呼び出せるようにしています。それらの関数の本体を書く代わりに "mac#syntax" のような文字列を割り当てています。これはこの関数が で、C言語で実装された関数で、# の後ろに続くC言語関数名であることを ATS に知らせています。# の前にある 'mac' はC言語関数をどのように扱うかを ATS に指示します。[このメーリングリストでの投稿](http://sourceforge.net/p/ats-lang/mailman/message/27338271/) が # 構文についてより詳細に解説しています。

このプログラムは次のコマンドでコンパイルできます:

```
atscc -o http-server http-server.dats -levent
```

C言語プログラムに対して、このプログラムには特に利点はありません。わずかなコンパイル時検査は argv が範囲外参照をしないことですが、ただそれだけです。けれども、これで ATS にコーオを埋め込めたので、コードの翻訳を開始できます。

## http-server2.dats

最初に私が翻訳に取り組んだ関数は http_server でした。結果の ATS コードは
[http-server2.dats](http://bluishcoder.co.nz/ats/http-server.dats)
([pretty-printed html](http://bluishcoder.co.nz/ats/http-server2.html))
です。このコードのためには ATS から libevent の関数を呼び出さなければなりません。libevent は状態を保持するためにC言語の構造体を使います。これらの構造体は抽象的なもので、C言語コードからその中身をのぞけません。それらへのアクセスは libevent の関数を経由させます。http_server で使っている構造体は以下です:

* event_base
* evhttp
* evhttp_request

libevent はこれらの構造体を確保/解放する関数を持っています。もしより安全なC言語の使い方を解説した [私の ATS に関する記事](http://bluishcoder.co.nz/tags/ats/) を読んでいるなら、これらは抽象観型 (abstract viewtypes) で表現できることをご存知でしょう。event_base の型ラッパーは次のようになります:

```ocaml
absviewtype event_base (l:addr)
viewtypedef event_base0 = [l:addr | l >= null ] event_base l
viewtypedef event_base1 = [l:addr | l >  null ] event_base l
```

ここではアドレス l をともなう抽象観型として event_base を定義しています。実際には event_base* 型のC言語ポインタが使われます。viewtypedef キーワードで、NULL になる可能性がある event_base 型 (event_base0) と、NULL にならない event_base 型 (event_base1) が、event_base 型の別名として定義されています。これらの typedef 定義によって、関数が NULL オブジェクトを取るのか取らないのかを表現するのが簡単になります。また、いたるところで全称量化を付けなくても (例えば {l:addr} を関数定義に書かなくても) 関数を定義することができます。evhttp と evhttp_request に対しても同様のラッパーを作ります。

event_base オブジェクトは event_base_new と event_base_free で生成/破棄します。イベントは event_base_dispatch を使ってディスパッチされます。これらの ATS での定義は次のようになるでしょう:

```ocaml
extern fun event_base_new(): event_base0 = "mac#event_base_new"
extern fun event_base_free (p: event_base1):void = "mac#event_base_free"
extern fun event_base_dispatch (base: !event_base1):int = "mac#event_base_dispatch"
```

このコードは、event_base_new が NULL である可能性のある event_base を返すことと、event_base_free が NULL でない event_base を取ることを示しています。抽象観型は線形オブジェクトで、ちゃんと破棄されるか、破棄された後に使われることがないか、コンパイル時型チェックで検査されることを意味しています。event_base_free はその線形型を消費します (型を消費したくない場合には、p: !event_base1 のように ! をともなって引数を定義すべきです)。event_base_dispatch の定義は ! 注釈を引数に持っていて、型を消費しないことを示しています。

event_base を生成/破棄するコードは次のようになるでしょう:

```ocaml
val base = event_base_new()
val () = assert_errmsg(~base, "event_base_new failed")
...
val _  = event_base_dispatch(base)
...
val () = event_base_free(base)
```

assert_errmsg 呼び出しに注意してください。base が NULL でないなら、~ 演算子は真を返します。そのためこの assert は base が NULL でないことをチェックし、型システムはそれを追跡します。この後では base は非 NULL ポインタ (つまり event_base1) であることが分かります。これで base を event_base_free に渡すことができるようになるのです。もしこの assert (もしく別の NULL でないことのチェック) が無ければ、コンパイルエラーになります。

http_server から使うその他の libevent 関数についても、同様のラッパーを作ります。ただし、evhttp_set_cb と evhttp_set_gencb は少し異なります。それらのC言語側定義は次のようになります:

```ocaml
void evhttp_set_gencb(struct evhttp *http,
    void (*cb)(struct evhttp_request *, void *), void *arg);
int evhttp_set_cb(struct evhttp *http, const char *path,
    void (*cb)(struct evhttp_request *, void *), void *cb_arg);
```

これらはコールバックとしてのC言語関数と、そのC言語関数に渡す引数を取ります。特定の URL にアクセスすると、libevent はこのコールバック関数を呼び出します。その引数は void* として型付けされていますが、ATS ではより良い型付けができます。ATS での定義は次のようになります:

```ocaml
typedef evhttp_callback (t1:viewtype) = (!evhttp_request1, !t1) -<fun1> void
extern fun evhttp_set_cb {a:viewtype} (http: !evhttp1,
                                       path: string,
                                       callback: evhttp_callback (a),
                                       arg: !a): int = "mac#evhttp_set_cb"
extern fun evhttp_set_gencb {a:viewtype} (http: !evhttp1,xi
                                          callback: evhttp_callback (a),
                                          arg: !a): void = "mac#evhttp_set_gencb"
```

この typedef はコールバック関数を表わす別名を定義しています。その引数の型は型変数になっています。
上記の定義では evhttp_callback が2つの引数を取るC言語の関数であると宣言しています (fun1 については [Functions in ATS](http://bluishcoder.co.nz/2010/06/13/functions-in-ats.html) を読んでください)。非 NULL の evhttp_request オブジェクトは消費されません。t1 型のオブジェクトはどのような観型も取ることができ、この t1 も消費されません。そしてこの関数は void を返します。従って、evhttp_callback (event_base) は第2引数として event_base を取るC言語の関数になります。

evhttp_set_cb は [多相関数](http://www.ats-lang.org/htdocs-old/DOCUMENT/INTPROGINATS/HTML/x1059.html) です。パラメータ arg はどのような観型をも取ることができます。evhttp_callback のコールバックのパラメータは arg と同じ型でパラメータ化されています。これはこのコールバックが evhttp_set_cb に渡した引数と同じ型の引数のみ受け付けることを意味しています。evhttp_set_cb と evhttp_set_gencb は次のように呼び出されます:

```ocaml
 val _ = evhttp_set_cb {ptr} (http, "/dump", dump_request_cb, null)
 val () = evhttp_set_gencb {string} (http, send_document_cb, docroot)
```

呼び出しにおける静的引数として、引数の型 (この例では ptr と string) を渡していることに注意してください。おかげで ATS は多相関数で使う型を知ることができます。時には ATS がこれを推論してくれるのでこの静的引数が不要にになりますが、この場合において ATS の型推論器は推論できないようです。dump_request_cb と send_document_cb に対する ATS 言語側ラッパーは次のようになります:

```ocaml
extern fun dump_request_cb (request: !evhttp_request1, arg: !ptr): void = "mac#dump_request_cb"
extern fun send_document_cb (request: !evhttp_request1, arg: !string): void = "mac#send_document_cb"
```

http-example2.dats adds an additional callback to cause the server to exit.
The server exit is done by breaking out of the event loop using event_base_loopexit.
That function needs an event_base so I pass this as the callback argument.
In this case there is no C function to wrap so I create the callback function as an ATS anonymous function:

```ocaml
val _ = evhttp_set_cb {event_base1} (http,
                                     "/quit",
                                     lam (req, arg) => ignore(event_base_loopexit(arg, null)),
                                     base)
```

The equivalent callback function in C would need to cast the void* argument to a base_event*. I don’t need to do this in ATS as the callback argument is correctly typed thanks to the definition of evhttp_set_cb and evhttp_callback as described above.

The ignore call is to help make the ATS code a bit more readable. event_base_loopexit returns an integer but the callback returns void. We need to consume the return value of event_base_loopexit. In ATS this needs to be done using the verbose syntax let val _ = ... in () end. ignore is a macro that hides this:

```ocaml
macdef ignore (x) = let val _ = ,(x) in () end
```

This version of http-server2.dats provides a little bit more type safety than the C version. It still utilizes a lot of the C code. There is no overhead from ATS - all the wrappers are defined in terms of the existing C functions. They provide additional checking at compile time, no extra run time code is generated. Although the wrappers look verbose, they’d usually be in a libevent ATS module. The resulting ATS code is a little smaller and clearer if you ignore the wrapper’s the sample code includes.

## Next steps

The next steps would be to start converting the existing functions, or to add extra functionality in ATS (like I did with the quit callback). I’ll do a followup post on anything interesting or new found during the remaining conversion. Otherwise you might like to try it as an exercise in learning ATS.

One interesting thing to look at would be how to handle the very callback oriented libevent code. This can make code difficult to follow. Writing libevent code that does HTTP requests looks something like:

```ocaml
fun handle_result(result: string) = ...
...
var () = http_get("http://www.bluishcoder.co.nz", handle_result)
```

I much prefer something like the following and have the compiler break the code up into the callback style:

```ocaml
var result = http_get("http://www.bluishcoder.co.nz")
```

Languages that have continuations or coroutines make this sort of thing easy. It’d be nice to be able to factor code like this to be easier to read and use in ATS. If anyone has any ideas I’d love to hear them.
