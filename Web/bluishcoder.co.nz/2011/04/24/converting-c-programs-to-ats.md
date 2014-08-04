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

This defines event_base as an abstract view type with an address, l.
It’s effectively a C pointer of type event_base*.
The two viewtypedef statements define aliases for an event_base type that can be NULL (event_base0) and an event_base type that cannot be NULL (event_base1).
Defining these typedef’s makes it easier to tell which functions accept NULL objects and which don’t.
It also allows defining functions without having to have universal type quantifiers everywhere (eg, the {l:addr} in function definitions).
Similar wrappers are done for evhttp and evhttp_request.

event_base objects are created and destroyed with event_base_new and event_base_free.
Events are dispatched using event_base_dispatch.
The ATS definitions for these look like:

```ocaml
extern fun event_base_new(): event_base0 = "mac#event_base_new"
extern fun event_base_free (p: event_base1):void = "mac#event_base_free"
extern fun event_base_dispatch (base: !event_base1):int = "mac#event_base_dispatch"
```

This says that event_base_new returns a possible NULL event_base and event_base_free takes a non-NULL event_base.
Abstract viewtype’s are linear objects which means the compile time type system checks that they are destroyed and that they aren’t used after destruction.
event_base_free consumes the linear type (For it not to consume the type it would have to define the argument with a ! like p: !event_base1).
You can see this in the definition of event_base_dispatch which has the ! annotation in the argument to say it doesn’t consume the type.

Code to create and destroy an event_base will look like:

```ocaml
val base = event_base_new()
val () = assert_errmsg(~base, "event_base_new failed")
...
val _  = event_base_dispatch(base)
...
val () = event_base_free(base)
```

Note the assert_errmsg call.
The ~ operator returns true if base is not NULL.
So the assert checks that base is non-NULL and the type system tracks this.
It knows that base from then on is a non-NULL pointer (ie.
The event_base1 typedef).
This allows it to be passed to event_base_free.
Without this assert (or other check for non-NULL-ness) there would be a compile error.

Similar wrappers are done for the other libevent functions that http_server uses.
evhttp_set_cb and evhttp_set_gencb are a little different however.
Their C definitions look like:

```ocaml
void evhttp_set_gencb(struct evhttp *http,
    void (*cb)(struct evhttp_request *, void *), void *arg);
int evhttp_set_cb(struct evhttp *http, const char *path,
    void (*cb)(struct evhttp_request *, void *), void *cb_arg);
```

They take a C function as a callback and an argument to pass to that C function.
The callback is called by libevent when a particular URL is accessed.
The argument is typed as a void* but we can do better in ATS.
Here’s the ATS definitions:

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

The typedef defines an alias for referring to the callback function.
It is parameterized over the type of the argument.
The definition states that an evhttp_callback is a C function (the fun1, see
[Functions in ATS](http://bluishcoder.co.nz/2010/06/13/functions-in-ats.html)
) that takes two arguments.
A non-NULL evhttp_request object that is not consumed, and an object of type t1 where t1 is any viewtype.
It is also not consumed.
The function returns void.
An evhttp_callback (event_base) is therefore a C function that takes an event_base as the second argument.

evhttp_set_cb is a
[polymorphic function](http://www.ats-lang.org/htdocs-old/DOCUMENT/INTPROGINATS/HTML/x1059.html).
The arg parameter can be any viewtype.
The callback paramter is a evhttp_callback parameterized over this same type.
This means that the callback must accept as an argument the same type as the argument we pass to evhttp_set_cb.
evhttp_set_cb and evhttp_set_gencb are called like:

```ocaml
 val _ = evhttp_set_cb {ptr} (http, "/dump", dump_request_cb, null)
 val () = evhttp_set_gencb {string} (http, send_document_cb, docroot)
```

Note that we pass the argument type (ptr and string in the example) as a static argument in the call so ATS knows which type to use in the polymorphic function. Sometimes ATS can infer this and the argument is not needed but in this case ATS’ type inference can’t do it. The ATS wrappers for dump_request_cb and send_document_cb are:

```ocaml
extern fun dump_request_cb (request: !evhttp_request1, arg: !ptr): void = "mac#dump_request_cb"
extern fun send_document_cb (request: !evhttp_request1, arg: !string): void = "mac#send_document_cb"
```

http-example2.dats adds an additional callback to cause the server to exit. The server exit is done by breaking out of the event loop using event_base_loopexit. That function needs an event_base so I pass this as the callback argument. In this case there is no C function to wrap so I create the callback function as an ATS anonymous function:

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
