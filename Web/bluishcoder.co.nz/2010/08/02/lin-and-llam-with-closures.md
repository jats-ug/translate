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

A (somewhat contrived) example of usage below defines a function ‘test’ that takes as an argument a closure tagged as lin,cloptr1.
It calls the closure and then frees it.
In the ‘main’ function I use the cURL library to call ‘curl_global_init’.
This returns a ‘pf_gerr’ resource that is used to track if the correct cURL cleanup routine is called.
The following example compiles and runs fine:

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

Compile with something like:

```
atscc -o example example.dats -lcurl
```

Notice that in the closure passed to ‘test’ the ‘pf_gerr’ resource is used. The means that resource is captured by the closure. A version without using ‘lin’ and ‘llam’ is:

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

This produces the compile error showing that without the ‘lin’ tag a closure cannot capture resources:

```
example.dats: ...: the linear dynamic variable [pf_gerr] is expected to be local but it is not.
```

A version that works without using ‘lin’:

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

Here we have to explicitly pass the resource around in a number of places. The use of ‘lin’ and ‘llam’ makes the code shorter and easier to understand. In a later reply Hongwei goes on to say:

> While linear closures containing resources may not be common, linear proof functions are common and very useful for manipulating and tracking resources.
