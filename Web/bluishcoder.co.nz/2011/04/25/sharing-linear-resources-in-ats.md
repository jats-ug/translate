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

コールバックに渡したリソースを消費したければ、渡したリソースほ保持するために データ観型 (dataviewtype) を使えます。
これを表わすコードは [callback3.dats](http://bluishcoder.co.nz/ats/callback3.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback3.html)) です。
このコードの主な変更点は次のようになります:

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

このコードでは `Container` データ観型が2つのリソースを保持しています。
`r` と `r2` の所有権は `Container` に渡されます。
コールバックへの引数は `!Container` ではなく `Container` で、これはそのコンテナが破棄されることを示しています。
次の行は、コンテナ中にある `r` と `r2` へのオブジェクトの束縛をデコンストラクトして、そのコンテナを破棄します (`~` はこれを行なう表記です):

```ats
val ~Container (r,r2) = c
```

以前の線形クロージャの例よりもこのコードの方が理解しやすいでしょう。
次のアプローチは、渡されたオブジェクトのいくつかを消費し、いくつかを消費しないための方法です。

## 参照渡しを使う

ここでのアプローチは、共有したいリソースを含むレコードを生成することです。
参照渡しを使ってこのレコードを渡します。
参照渡しを使うという点を除いて上記のコンテナの例と同じコードが [callback4.dats](http://bluishcoder.co.nz/ats/callback4.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback4.html)) です。
変更したコードは次です:

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

関数への引数の `&env` 構文は参照で引数を渡すことを意味しています。
この例では、線形リソースは `env` インスタンスと渡される参照に所有されています。
`env` はフラットレコードです。
(`{` レコードを、`@` はそれがフラットであることを意味します。対して `'` はそれがボックス化されていることを意味します。)
フラットレコードはC言語の構造体のようなものです。
実際 ATS から生成されたC言語コードは、スタックに確保するのにC言語の構造体を使います。

## いくつかのリソースのみ消費する

前の「参照渡し」の例を拡張して、リソースのいくつかを消費して、それ意外を消費しない方法を示してみようと思います。
下記に少しの変更を加えた完全なコードは [callback5.dats](http://bluishcoder.co.nz/ats/callback5.dats) ([pretty-printed html](http://bluishcoder.co.nz/ats/callback5.html)) を見てください:

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

消費されるかもしれないリソースのために、ATS prelude で定義された観型 (viewtype) `Option_vt` を使います。
これはコンストラクタ `Some_vt` と `None_vt` を持っています。
前者は値を保持し、後者は値を持ちません。
`_vt` 接尾辞は prelude における命名規約で、それが観型であることを示しています。
また (ガベージコレクションされるオブジェクトである) データ型の `Option` に相当します。
観型の値を保持するために、ここでは観型を使います。

コールバックは値がレコードによって保持されている確認するのに `case` を使います。
もしそれが `Some_vt` ならば、そのリソースを使い、解放し、保持されたリソースがないことを主張するために `None_vt` を割り当てます。
`main` 本体でも、リソースが保持されていたら、同様にそのリソースを解放します。
パターンで `~` を使っているのは `Option_vt` リソースを消費して解放するためであることに注意してください。

## 概要

この最終的な例は、準グローバルなオブジェクトを保持するためにコンテキストオブジェクトを確保するようなC言語プログラムに近いものです。
また必要に応じてそれを渡しまわります。
ATS はそれらのオブジェクトが正しく破棄され、再び使われないことをコンパイル時に検査してくれます。

この形のC言語プログラムの例は、 [libevent](http://monkey.org/~provos/libevent/) を使って HTTP 経由で URL の内容を読む [download.c](http://bluishcoder.co.nz/ats/download.c) (元は [このコードに基づいています](http://archives.seul.org/libevent/users/Sep-2010/msg00050.html)) です。
それは他のオブジェクトへのポインタを含む `download_context` オブジェクトを確保します。
そして、それらのいくつかは破棄され、いくつかは保持し続けることを想定します。
私の線形リソースの共有の調査はこの例を変換することが動機でした。
