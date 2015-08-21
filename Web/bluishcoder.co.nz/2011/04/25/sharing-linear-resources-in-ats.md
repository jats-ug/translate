# BLUISH CODER: ATSにおける共有した線形リソース

(元記事は http://bluishcoder.co.nz/2011/04/25/sharing-linear-resources-in-ats.html です)

私の以前の記事[「C言語のプログラムをATSに翻訳する」](http://bluishcoder.co.nz/2011/04/24/converting-c-programs-to-ats.html)では、線形リソースをコールバック関数に渡す例を紹介しました。
そのコードは次のようなものでした:

```ats
typedef evhttp_callback (t1:viewtype) = (!evhttp_request1, !t1) -<fun1> void
extern fun evhttp_set_cb {a:viewtype} (http: !evhttp1,
                                       path: string,
                                       callback: evhttp_callback (a),
                                       arg: !a): int = "mac#evhttp_set_cb"
...
val _ = evhttp_set_cb {event_base1} (http,
                                     "/quit",
                                     lam (req, arg) => ignore(event_base_loopexit(arg, null)),
                                     base)
```

コールバック関数に渡される `base` 引数は線形型のインスタンスです:

```ats
absviewtype event_base (l:addr)
viewtypedef event_base0 = [l:addr | l >= null ] event_base l
viewtypedef event_base1 = [l:addr | l >  null ] event_base l
```

コールバックプログラミングにおいては、1つより多くの引数をコールバックに渡したくなるものです。
多くのリソースをまとめて、それらをコールバックに渡し、それらのいくつかを消費するかもしれないコールバックを作る方法が必要でした。

[ats-lang-users メーリングリスト](https://lists.sourceforge.net/lists/listinfo/ats-lang-users) で
[助言を求める](http://sourceforge.net/p/ats-lang/mailman/message/27404524/) までは、この問題を解決するためにスタブを使っていました。
[Hongwei Xi の返信](http://sourceforge.net/p/ats-lang/mailman/message/27405027/) はこの問題に対する良い助言です。

コールバック問題の例を切り崩し、解決の最初の試みを説明して、Hongweiの返答に基づく解決策に繋げようと思います。

## コールバックの例

次のプログラム [callback1.dats](http://bluishcoder.co.nz/ats/callback1.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback1.html))
は、上記の libevent の例のように、1つの線形リソースをコールバックに渡すコールバックプログラムの小さなテストケースです。
メインコードは次のようなものです:

```ats
absviewtype resource (l:addr)
viewtypedef resource1 = [l:addr | l > null] resource l
extern fun resource_new():resource1 = "mac#resource_new"
extern fun resource_use(r: !resource1):void = "mac#resource_use"
extern fun resource_free(r: resource1):void = "mac#resource_free"

typedef callback = (!resource1) -<fun1> void
extern fun do_something(f: callback, arg: !resource1):void = "mac#do_something"

implement main() = let
  val r = resource_new()
  val () = do_something(lam (r) => resource_use(r), r)
  val () = resource_free(r)
in
  ()
end
```

この例では、線形オブジェクト `r` をコールバックと一緒に `do_something` に渡しています。
それから `do_something` はそのコールバックに引数である線形オブジェクトを渡します。
このコールバックはそのオブジェクトを使用し、そして消費しません。
次の作業は複合的な線形オブジェクトを渡す問題に対処することです。
およそ次のようになるでしょう:

```ats
implement main() = let
  var r = resource_new()
  var r2 = resource_new()
  val () = do_something(lam (r_and_r2) => (resource_use(r);resource_use(r2)),
                        ...want_to_pass_r_and_r2_here...)
  val () = resource_free(r)
  val () = resource_free(r2)
in
  ()
end
```

## 線形型を含む線形クロージャ

コールバックへ複合的な値を渡す最初の試みは、コールバック関数への引数としてクロージャを使うことでした。
そうすれば複合オブジェクトに対処できます。
このコードは [callback2.dats](http://bluishcoder.co.nz/ats/callback2.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback2.html)) です。
主な変更点は次のようなものです:

```ats
viewtypedef func = () -<lincloptr1> void
extern fun do_something(f: (func) -<fun1> void, arg: func):void = "mac#do_something"

fun callback(f: func):void = let val () = f() in cloptr_free(f) end

implement main() = let
  extern prfun __borrow {l:addr}
                        (pf: ! resource1 @ l): (resource1 @ l, resource1 @ l -<lin,prf> void)
  var r = resource_new()
  var r2 = resource_new()
  prval (pf, pff) = __borrow(view@ r)
  prval (pf2, pff2) = __borrow(view@ r2)

  val () = do_something(callback,
                        llam () => let
                          val () = resource_use(r)
                          val () = resource_use(r2)
                          prval () = pff(pf)
                          prval () = pff2(pf2)
                        in () end)
  val () = resource_free(r2)
  val () = resource_free(r)
in
  ()
end
```

このコードでは、実際のコールバックは引数として、引数のない別のコールバックを取って呼び出すような関数です。
この新しいコールバックはクロージャです。
そのコードは `r` と `r2` の値とそれらへの `resource_use` 呼び出しを含んでいます。
`r` と `r2` は線形リソースなので、それらに対して通常のクロージャを使うことができません。
「線形型を含む線形クロージャ」を使い、そのクロージャの型は `<lincloptr1>` タグで識別されます:

```ats
viewtypedef func = () -<lincloptr1> void
```

これはこのクロージャがそのスコープにおいて、線形リソースへの参照を含むことができるように型システムにしらせます。
他に、そのスコープから線形オブジェクトを明示的に借りる (borrow) ための証明システムを使うことが必要になります。
これは次のコードを使って行なわれます:

```ats
extern prfun __borrow {l:addr} (pf: ! resource1 @ l): (resource1 @ l, resource1 @ l -<lin,prf> void)
...
prval (pf, pff) = __borrow(view@ r)
```

証明関数 `__borrow` は観 (view) を取り、その観と借りたリソースを返却するために呼び出す証明関数を表わす、新しい証明の値を返します。
このクロージャの中ではこの証明関数に受け取った新しい証明の値を渡して、そのリソースの借用が完了したことを申告します。
これは次のコードのようになります:

```ats
prval () = pff(pf)
```

もしこのコードは `__borrow` でリソースを借りていなければ、型システムはクロージャ中の線形リソースは消費されていると期待します。
この例では、クロージャの外で `resource_free` を使ってそれらを消費したくはありません。
`__borrow` を「型システムさん、他の関数で一時的にこのリソースを借りますよ、使い終わったらしらせますから」と主張していると考えてみてください。

クロージャを使うとき、そのメモリは閉じた環境を保管するために確保されます。
これについて [ATSのクロージャ](http://bluishcoder.co.nz/2010/06/20/closures-in-ats.html) の記事で議論しました。
線形クロージャのメモリは呼び出し元によって自動的に解放されます。
けれどもこの場合、呼び出し元はC言語で実装された関数です。
そのため ATS はそのメモリの解放呼び出しを挿入できません。
このコードの `cloptr_free` 呼び出しによるメモリの手動解放を行なわないと、型検査エラーを引き起こします。
もし ATS コードからそのコールバックを呼び出していたら、それがなくても型検査エラーを起こさないで欲しいのです。
線形リソースと線形クロージャについて、より多くのことを [lin と llam を伴うクロージャ](http://bluishcoder.co.nz/2010/08/02/lin-and-llam-with-closures.html) に書きました。

この例は動作し、どのようなオブジェクトもクロージャで捕捉して渡すことができるという点で柔軟です。
オブジェクトは消費されなければならないという点で複雑です。

## データ観型 (dataviewtype) コンテナを使う

If I want to consume the resources I pass to the callback I can use a dataviewtype to hold the resources I pass.
The code for this is in [callback3.dats](http://bluishcoder.co.nz/ats/callback3.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback3.html)).
The changed parts of this code are:

```ats
dataviewtype container = Container of (resource1, resource1)
typedef callback = (container) -<fun1> void
extern fun do_something(f: callback, arg: container):void = "mac#do_something"

fun mycallback(c: container): void = let
  val ~Container (r,r2) = c
  val () = resource_use(r)
  val () = resource_use(r2)
  val () = resource_free(r)
  val () = resource_free(r2)
in
  ()
end

implement main() = let
  val r = resource_new()
  val r2 = resource_new()

  val c = Container(r, r2)
  val () = do_something(mycallback, c)
in
  ()
end
```

In this code the Container dataviewtype holds the two resources.
Ownership of r and r2 are passed to the Container.
The argument for the callback is Container rather than !Container to indicate that the container itself gets destroyed.
The following line does a deconstructing bind of the objects in the container to r and r2 and destroys the container (The ~ is the notation that does this):

```ats
val ~Container (r,r2) = c
```

Hopefully that code is easier to understand than the linear closure example previously.
The next approach is how to deal with passing objects where some are to be consumed and some aren’t.

## 参照渡しを使う

The approach here is to create a record containing the resources I want to share.
I pass this record around using call by reference.
The equivalent of the container example above, but using call-by-reference, is in [callback4.dats](http://bluishcoder.co.nz/ats/callback4.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback4.html)).
The changed code is:

```ats
viewtypedef env (l1:addr, l2:addr) = @{r1= resource l1, r2= resource l2}
viewtypedef env = [l1,l2:agz] env(l1, l2)
typedef callback = (&env) -<fun1> void
extern fun do_something(f: callback, arg: &env):void = "mac#do_something"

fun mycallback(e: &env): void = let
  val () = resource_use(e.r1)
  val () = resource_use(e.r2)
in
  ()
end

implement main() = let
  var env: env(null, null)
  val () = env.r1 := resource_new()
  val () = env.r2 := resource_new()

  val () = do_something(mycallback, env)

  val () = resource_free(env.r1)
  val () = resource_free(env.r2)
in
  ()
end
```

The &env syntax in the argument to a function means pass that argument by reference.
In this example the linear resources are owned by the env instance and a reference to that is passed around.
`env` is a flat record (the { says it’s a record, the @ says it’s flat vs ' for boxed).
A flat record is like a struct in C.
In fact the generated C code from ATS uses a C struct allocated on the stack.

## いくつかのリソースのみ消費する

Extending the previous ‘call-by-reference’ example I show how to consume some of the resources but not others.
See [callback5.dats](http://bluishcoder.co.nz/ats/callback5.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback5.html)) for the full code with the changed bits below:

```ats
viewtypedef env (l1:addr) = @{r1= resource l1, r2= Option_vt (resource1)}
viewtypedef env = [l1:agz] env(l1)
typedef callback = (&env) -<fun1> void
extern fun do_something(f: callback, arg: &env):void = "mac#do_something"

fun mycallback (e: &env): void = let
  val () = resource_use(e.r1)
  val () = case+ e.r2 of
           | ~Some_vt(x) => let
                              val () = resource_use(x)
                              val () = resource_free(x)
                              val () = e.r2 := None_vt
                            in () end
           | ~None_vt () => let
                              val () = e.r2 := None_vt
                            in () end
in
 ()
end

implement main() = let
  var env: env(null)
  val () = env.r1 := resource_new()
  val () = env.r2 := Some_vt (resource_new())

  val () = do_something(mycallback, env)

  val () = resource_free(env.r1)
  val () = case+ env.r2 of 
           | ~Some_vt (x) => resource_free(x)
           | ~None_vt () => ()
in
  ()
end
```

For resources that can be consumed I use the `Option_vt` viewtype defined in the ATS prelude.
This has constructors `Some_vt` and `None_vt`.
The former holds a value and the latter means no value is contained.
The `_vt` suffix is the prelude’s naming standard to say that this is a viewtype.
There also exists an equivalent to Option for datatypes (garbage collectable objects).
We use a viewtype here since we a holding on to values that are viewtypes.

The callback uses case to check if a value is held by the record.
if it is `Some_vt` I use the the resource, free it, and assign `None_vt` back to say there is no longer a resource being held.
In the main body I do the same to free the resource if a resource is being held.
Note the use of the `~` in the patterns to consume and free the `Option_vt` resources.

## 概要

This final example most closely follows the way some C programs work, allocating a context object to hold semi-global objects.
And passing that around as needed.
What ATS buys us is the checking at compile time that these objects are correctly destroyed and not used again.

An example of this type of C program is [download.c](http://bluishcoder.co.nz/ats/download.c) (originally [based on this code](http://archives.seul.org/libevent/users/Sep-2010/msg00050.html)) which uses [libevent](http://monkey.org/~provos/libevent/) to read the contents of a URL via HTTP.
It allocates a download_context object which contains pointers to other objects, some of which are destroyed and some are supposed to be retained.
My explorations into linear resource sharing was motivated by converting this example.
