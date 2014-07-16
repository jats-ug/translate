# BLUISH CODER: lin と llam を伴うクロージャ

(元記事は http://bluishcoder.co.nz/2010/08/02/lin-and-llam-with-closures.html です)

[Pthreads](http://en.wikipedia.org/wiki/POSIX_Threads) の [ATS](http://www.ats-lang.org/)
ラッパーを使っていると、これまで見たことがないクロージャの使い方に私は気が付きました。クロージャをスレッドとして作る関数のシグニチャは次のようなものです:

```ocaml
fun pthread_create_detached_cloptr
  (f: () -<lin,cloptr1> void): void
// end of [pthread_create_detached_cloptr]
```

'cloptr1' と同時に 'lin' を使っていることに注意してください。次のコンパイルエラーのために、この関数では 'lam' を使ってクロージャを作ることができません:

```ocaml
(* Using *)
val () = pthread_create_detached_cloptr(lam () => print_string("hello thread 1\n"));

(* Gives *)
test.dats: ...: error(3) nonlinear function is given a linear type .
```

正しいやり方は 'llam' を使うことです:

```ocaml
val () = pthread_create_detached_cloptr(llam () => print_string("hello thread 1\n"));
```

[ATSfloat](http://scg.ece.ucsb.edu/ats.html) のサイトから入手できる "Tips for using ATS"
という論文には 'llam' が次の 'lam' の使い方に相当すると言及されています:

```ocaml
val () = pthread_create_detached_cloptr(lam () =<lin,cloptr1> print_string("hello thread 1\n"));
```

探してもそれ以上 lin と cloptr の使い方は見つかりませんでした。そこで私がこの件を
[メーリングリストで質問](http://sourceforge.net/p/ats-lang/mailman/ats-lang-users/thread/Pine.LNX.4.64.1008011044170.11828@csa2.bu.edu/)
したところ、Hongwei は 'lin' を使用する場合と使用しない場合の差異を以下のように説明しました:

> ATS では、その環境にリソースを含むクロージャを作ることができる。
> 上記の例では、[f] がその環境にリソースを含む可能性のあるクロージャであることを示すために、'lin' を使っている。

Hongwei はこの返信で、'lin' を使わないことによってクロージャの環境にリソースを含むことができないために、より制限された pthread_create_detached_cloptr
のバージョンを示しています。

次の(いくらか不自然な)使用例では、lin,cloptr1 とタグのついたクロージャを引数に取る関数 'test'
を定義しています。この例ではクロージャを呼び出した後、解放しています。'main'
関数は 'curl_global_init' 関数を呼び出して cURL ライブラリを使っています。この関数は、cURL
のクリーンアップルーチンが正しく呼び出されるかどうか追跡するために使う 'pf_gerr'
リソースを返します。次の例はコンパイルして実行できます:

```ocaml
staload "contrib/cURL/SATS/curl.sats"

fun test(f: () -<lin,cloptr1> void) = (f(); cloptr_free(f))

implement main() = let
  val (pf_gerr | gerr) = curl_global_init (CURL_GLOBAL_ALL)
  val () = assert_errmsg (gerr = CURLE_OK, #LOCATION)

  val () = test(llam () => curl_global_cleanup (pf_gerr | (*none*)))
in
  ()
end;
```

次のようにしてコンパイルできます:

```
atscc -o example example.dats -lcurl
```

クロージャは 'test' 関数から渡された 'pf_gerr'
リソースを使っていることに注意してください。これは、リソースがクロージャによって捕捉されることを意味しています。'lin' と 'llam'
を使わないバージョンは次のようになります:

```ocaml
staload "contrib/cURL/SATS/curl.sats"

fun test(f: () -<cloptr1> void) = (f(); cloptr_free(f))

implement main() = let
  val (pf_gerr | gerr) = curl_global_init (CURL_GLOBAL_ALL)
  val () = assert_errmsg (gerr = CURLE_OK, #LOCATION)

  val () = test(lam () => curl_global_cleanup (pf_gerr | (*none*)))
in
  ()
end;
```

このコードは、'lin'
タグがないのでクロージャがリソースを捕捉できないために、コンパイルエラーを引き起こします:

```
example.dats: ...: the linear dynamic variable [pf_gerr] is expected to be local but it is not.
```

次のように修正すれば 'lin' を使わずに動作します:

```ocaml
staload "contrib/cURL/SATS/curl.sats"

fun test {v:view}
  (pf: v | f: (v | (*none*)) -<cloptr1> void) = (f(pf | (*none*)); cloptr_free(f))

implement main() = let
  val (pf_gerr | gerr) = curl_global_init (CURL_GLOBAL_ALL)
  val () = assert_errmsg (gerr = CURLE_OK, #LOCATION)

  val () = test(pf_gerr | lam (pf | (*none*)) => curl_global_cleanup (pf | (*none*)))
in
  ()
end; 
```

このコードでは、複数の箇所でリソースを明示的に渡しています。'lin' と 'llam'
を使うことで、コードを短かくそして理解しやすくできます。後の返信で、Hongwei は続けて次のようにコメントしています:

> リソースを含む線形クロージャは一般的ではないかもしれないが、線形証明関数は一般的でリソースを操作し追跡するためにとても有用だ。
