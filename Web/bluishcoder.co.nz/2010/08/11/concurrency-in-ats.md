# BLUISH CODER: ATSにおける並行プログラミング

(元記事は http://bluishcoder.co.nz/2010/08/11/concurrency-in-ats.html です)

[ATS](http://www.ats-lang.org/)
ではオペレーティングシステムのネイティブスレッドを使った並行プログラミングが可能です。
[Pthreads](http://en.wikipedia.org/wiki/POSIX_Threads)
のラッパーと、ワーカースレッドの生成する高レベル API、そしてそれらを使って
'parworkshop' と呼ばれる並列計算を提供しています。

メーリングリストのアナウンスでは次のように 'parworkshop' を解説しています:

> …'parworkshop' と名付けられたパッケージは、プログラマが 'workshop' を作り、'worker' を借りて、'work' を投稿することなどを可能にします。
> おそらく、もうお分かりでしょう :) 現時点では、それぞれの 'worker' は Pthreads 上に構築されています。
> 長期的には、ハードウェア/アーキティクチャを下層として利用する、より洗練された workshop を構築できるでしょう。

この記事では、'parworkshop' の使い方を学ぶために2つの例を紹介します。1つ目は実際には何も計算しないシンプルなものです。2つ目は、最近
reddit に投稿したプログラム例に基づいています。

## 単純な例

'parworkshop' を使うために、2つのファイルをインクルードします:

```ocaml
staload "libats/SATS/parworkshop.sats"
staload "libats/DATS/parworkshop.dats"
```

この SATS ファイルは関数とデータ型の宣言を含んでいます。この DATS ファイルはテンプレート関数のいくつかを実装するために必要です。

'parworkshop' を使うステップは次のようなものです:

1. 'workshop_make' を用いて workshop を生成します。
2. 'workshop_add_worker' を用いて workshop に worker を追加します。それぞれの worker はネイティブスレッドです。
3. 'workshop_insert_work' を用いて worker が処理する複数の work を挿入します。
4. 'workshop_wait_blocked_all' を用いて全ての work が完了するのを待ちます。
5. worker を終了させます。'workshop_insert_work' を用いて番兵に worker を終了させることができます。
6. 'workshop_wait_quit_all' を用いて全ての worker が終了するのを待ちます。
7. 'workshop_free' もしくは 'workshop_free_vt_exn' を用いて worker を解放します。

上記ステップ2で生成される worker はネイティブスレッドです。生成された後、このスレッドはキューでブロックします。(上記ステップ3で)
'workshop_insert_work' はキューに要素を追加します。worker スレッドの1つが起きて、引数として渡された work
を受け取って関数を呼び出します。そしてキューのブロックを解除します。worker スレッドによって呼び出された関数は、ステップ1で
'workshop_make' に渡した引数の1つです。

'workshop_make' の定義は次のようになります:

```ocaml
fun{a:viewt@ype} workshop_make {n:pos} (
  qsz: size_t n
, fwork: {l:agz} (!WORKSHOPptr (a, l), &a >> a?) -<fun1> int
) : WORKSHOPptr a
```

上記のコードから、'workshop_make' は2つの引数を取る関数テンプレートであることが分かります。これは
(この定義では 'a' である) work 要素の型がパラメータ化された関数テンプレートです。2つの引数の内、1つ目は
work 要素を追加するのに使われるキューの初期サイズを表わす数です。2つ目は work
要素がキューに追加された時に worker スレッドで呼び出される関数へのポインタです。
その関数は次の型を持つ必要があります:

```ocaml
{l:agz} (!WORKSHOPptr (a, l), &a >> a?) -<fun1> int
```

この型は、worker スレッド関数が生成された
(すなわち、'workshop_make' の返値である) workshop
と work 要素へのリファレンスを引数として取ることを意味しています。その返値の型は
'int' です。この worker 関数の返値は、worker
スレッドが終了すべきか継続してキューの要素を処理すべきか、当該スレッドに通知します。その返値が:

* > 0 なら worker は実行継続します
* = 0 なら worker は終了します
* = ~1 なら worker は休止します

例えば、'int' 型のwork の要素を取りその値を印字する worker 関数は、次のようになります:

```ocaml
fun fwork {l:addr} (
  ws: !WORKSHOPptr(work,l),
  x: &int >> int?
  ): int = let
  val () = printf("x = %d\n", @(x))
in 
 if x < 0 then 0 else 1
end
```

この例では、もし 'int' の値がゼロより小さければ worker は終了します。この関数を用いて
workshop といくつかの worker を生成します:

```ocaml
val ws = workshop_make<int>(1, fwork)
val _ = workshop_add_worker(ws)
val _ = workshop_add_worker(ws)
```

work 要素の型 (つまり 'int') がテンプレートパラメータとして 'workshop_make'
に渡されていることに注意してください。これはテンプレート関数の明示的な使用という点で
C++ の構文と良く似ています。次に work 要素を追加します:

```ocaml
val () = workshop_insert_work(ws, 1)
val () = workshop_insert_work(ws, 2)
val () = workshop_insert_work(ws, 3)
```

このコードは3つの要素をキューに挿入し、スレッドを起こしてそれらを処理させます。それから 'workshop_wait_blocked_all'
を用いてそれらが完了するのを待ち合わせて、worker スレッドを終了させる work 要素を挿入します:

```ocaml
val () = workshop_wait_blocked_all(ws)
val nworker = workshop_get_nworker(ws)
var i: Nat = 0
val () = while(i < nworker) let
           val () = workshop_insert_work(ws, ~1)
         in i := i + 1 end  
```

このコードは 'workshop_get_nworker' を用いて、現存する worker スレッドの数を得ます。それからスレッドの数だけループして、それらの
worker スレッドを終了させるようにゼロより小さい work 要素を挿入します。最後のステップではそれらが終了するまで待ち合わせた後、workshop
を解放します:

```ocaml
val () = workshop_wait_quit_all(ws)
val () = workshop_free(ws)
```

完全なプログラムは次のようになります:

```ocaml
staload "libats/SATS/parworkshop.sats"
staload "libats/DATS/parworkshop.dats"

typedef work = int

fun fwork {l:addr} (
  ws: !WORKSHOPptr(work,l),
  x: &work >> work?
  ): int = let
  val () = printf("x = %d\n", @(x))
in 
  if x < 0 then 0 else 1
end

implement main() = let
  val ws = workshop_make<work>(1, fwork)
  val status = workshop_add_worker(ws)
  val status = workshop_add_worker(ws)
 
  val () = workshop_insert_work(ws, 1)
  val () = workshop_insert_work(ws, 2)
  val () = workshop_insert_work(ws, 3)

  val () = workshop_wait_blocked_all(ws)
  val nworker = workshop_get_nworker(ws)
  var i: Nat = 0
  val () = while(i < nworker) let
             val () = workshop_insert_work(ws, ~1)
           in i := i + 1 end  

  val () = workshop_wait_quit_all(ws)
  val () = workshop_free(ws)
in
  ()
end
```

このコード (そのファイル名が 'eg1.dats' だとすると)
は次のようにコンパイルできます:

```
atscc -o eg1 eg1.dats -D_ATS_MULTITHREAD -lpthread
```

実行すると次の出力が得られます:

```
$ ./eg1
x = 1
x = 2
x = 3
x = -1
x = -1
```

'parworkshop' は型システムを利用して、リソースが正しく使用されることを保証していることに注意してください。上記の例で
'workshop_free' を呼び出さない場合、コンパイル時エラーになります。

## より複雑な例

前の例は単純なものでしたが、ほとんど 'parworkshop' の使い方に必要な基本的なステップを示しています。しかしほとんどの並列プログラミングでは、worker
スレッドになんらかのデータを処理させて、どうにかして結果を返したくなるでしょう。それからこの結果は後の処理に使われれるでしょう。ATS
の配付版にはこのようなやり方を示す
[いくつかの例](https://svn.code.sf.net/p/ats-lang/code/trunk/doc/EXAMPLE/MULTICORE/)
があります。以下ではこのアプローチを解説しようと思います。

[programming.reddit.com のあるスレッド](http://www.reddit.com/r/programming/comments/cy4m6/and_e_appears_from_nowhere_quick_numeric/)
ではある単純なプログラミングタスクを様々なプログラミング言語で実装していました。それらのいくつかは並行な言語で実装されていました。そこで私は
'parworkshop' を用いて ATS のバージョンを作ってみることにしました。

そのタスクは次のアプローチを使って数学定数 'e' の近似値を求めるものです (詳細は [こちら](http://www.mostlymaths.net/2010/08/and-e-appears-from-nowhere.html) を見てください):

> 0 と 1 の間のランダムな数を選びます。
> さらに別のランダムな数を選び、それを1つ目に加えます。
> これをランダムな数に対して繰り返しします。
> 合計が 1 より大きくなるのに必要だったランダムな数の個数の平均が e です。

このタスクに対応する ATS コードはまったく単純で、この記事の最後に完全なコードを入手できます。反復回数を取って試行回数の合計値を返す
'n_attempts' と呼ばれる関数を作りました。そしてこれを反復回数で割ると平均が得られるので、'e' を見積ることができます:

```ocaml
fun n_attempts (n:int): int
...
var e = n_attempts(10000000) / double_of_int(10000000)
```

マルチコアでを活用するために、それぞれのコアに worker スレッドを生成することにしました。これらそれぞれの
worker スレッドは要求された反復回数全体の一部を実行しようとします。work
要素は次の2つの事柄を表わしうるデータ型 'command' です:

1. 'Compute' は worker スレッドに反復を実行させます。'Compute' は worker スレッドが生成した結果を保持する整数を指すポインタを含んでいます。
2. 'Quit' は worker スレッドを終了させます。

このデータ型は次のように定義されます:

```ocaml
dataviewtype command = 
  | {l:agz} Compute of (int @ l | ptr l, int)
  | Quit 

viewtypedef work = command
```

'datatype' の代わりに 'dataviewtype'
を使っていることに注意してください。前者を使うには確保したデータ型を回収するガベージコレクタをリンクしなければなりません。後者は明示的に解放される必要がある
'線形データ型' をもたらし、そのためガベージコレクタが不要なのです。私のゴールはガベージコレクタが不要な実装を作ることでした。

worker スレッド関数は次のようになります:

```ocaml
fun fwork {l:addr}
  (ws: !WORKSHOPptr(work,l), x: &work >> work?): int = 
  case+ x of
  | ~Compute (pf_p | p, iterations)  => let 
       val () = !p := n_attempts(iterations)
       extern prfun __unref {l:addr} (pf: int @ l):void
       prval () = __unref(pf_p)
     in 1 end
  | ~Quit () => 0
```

このコードは基本的に 'case' 式を用いて work 要素の値で分岐しています。'Quit'
の場合、worker スレッドを終了させるために '0' を返します。'Compute'
の場合、反復回数ともなって n_attempts を呼び出して、結果を整数へのポインタに格納します。そして
worker スレッドに '1' を返して、次の work 要素を待ち合わせさせます。'case' 式での型の直前に '~' を使うことで ATS
に、線形型 ('Compute' と 'Quit') のメモリを解放させています。そのため手動で解放する必要がありません。

work 要素を workshop に追加するとき、結果を保持する 'int' の配列生成します。ここでは線形配列を使って、手動でメモリ管理をしています:

```ocaml
#define NCPU 2
...
val (pf_gc_arr, pf_arr | arr) = array_ptr_alloc<int>(NCPU)
val () = array_ptr_initialize_elt<int>(!arr, NCPU, 0)
...
val () = array_ptr_free {int} (pf_gc_arr, pf_arr | arr)
```

work 要素を挿入するとき、結果を格納する 'int' へのポインタを渡して 'Compute'
を生成しなければなりません。このポインタは先の配列の単一の要素を指します。そのコードは次のようになります:

```ocaml
fun loop {l,l2:agz} {n:nat} .< n >. (
      pf: !array_v(int, n, l2)
    | ws: !WORKSHOPptr(work, l), 
      p: ptr l2, 
      n: int n, 
      iterations: int)
    : void =
  ...
  prval (pf1, pf2) = array_v_uncons{int}(pf)
  extern prfun __ref {l:addr} (pf: ! int @ l): int @ l
  prval xf = __ref(pf1)
  val () = workshop_insert_work(ws, Compute (xf | p, iterations))
  val () = loop(pf2 | ws, p + sizeof<int>, n - 1, iterations)
  prval () = pf := array_v_cons{int}(pf1, pf2)
  ...
```

このコードは、配列全体へのアクセスを提供する証明を取り、一部の要素へアクセスする証明を返すような証明操作をしています。これは
worker スレッド関数が単一の要素のメモリにのみアクセスできることをコンパイル時に保証します。その魔法は 'array_v_uncons' にあります:

```ocaml
prval (pf1, pf2) = array_v_uncons{int}(pf)
```

'pf' は配列全体にアクセスする証明 ('array_ptr_alloc' から返される 'pf_arr') です。'array_v_uncons'
はこの証明を消費して2つの証明を返します。1つ目 'pf1' は配列の最初の要素にアクセスするための
'int @ l' です。2つ目 'pf2' は配列の残りにアクセスするための証明です。

これで整数へのポインタとともに 'pf1' を 'Compute'
に渡すことができます。これを処理する配列の要素がなくなるまで繰り返します。'loop'
への末尾呼出が 'int' サイズだけポインタを増加させていることに注意してください。ATS
はC言語のメモリを効率良く扱えますが、安全のためにメモリアクセスの証明と有効なデータを含むことを要求します。

'loop' 再帰呼出の後に次の行があります:

```ocaml
prval () = pf := array_v_cons{int}(pf1, pf2)
```

この行は、'array_v_uncons' によって 'pf' に保持された証明が消費されてしまっているために必要です。けれども
("pf: !array_v(int, n, l2)" に '!' が付いているため) 'loop' の定義では 'pf' は消費されてはいけないのです。この最後の行は
'pf' を消費したときに生成した 'pf1' と 'pf2' を一緒に繋げることで元の 'pf' の証明を復元しています。

他にも少しトリッキーな箇所があります:

```ocaml
extern prfun __ref {l:addr} (pf: ! int @ l): int @ l
prval xf = __ref(pf1)
```

'Compute' コンストラクタ渡された証明を消費します。もし 'pf1'
を渡してしまうと、それは消費されてしまい、後の 'array_v_cons'
呼び出しで使うことができません。このコードは既にある証明 'pf1'
と同じ新しい証明 'xf' を生成して、それを 'Compute' コンストラクタに渡します。後で
'fwork' 関数の中でこの証明を手動で消費します。以前の例の
'fwork' 中で、以下のコードに気がついていたかもしれません:

```ocaml
extern prfun __unref {l:addr} (pf: int @ l):void
prval () = __unref(pf_p)
```

このコードについて ATS メーリングリストにて、まだ私が使ったことがなかった '__borrow' 証明関数を用いたより良い方法について議論がありました。

プログラム全体は以下のようになります。前の単純な例と上記の少しこみいった説明で理解しやすいと思います:

```ocaml
staload "libc/SATS/random.sats"
staload "libc/SATS/unistd.sats"
staload "libats/SATS/parworkshop.sats"
staload "libats/DATS/parworkshop.dats"
staload "prelude/DATS/array.dats"

#define ITER 100000000
#define NCPU 2

fn random_double (buf: &drand48_data): double = let
  var r: double
  val _ = drand48_r(buf, r)
in
  r
end

fn attempts (buf: &drand48_data): int = let 
  fun loop (buf: &drand48_data, sum: double, count: int): int = 
    if sum <= 1.0 then loop(buf, sum + random_double(buf), count + 1) else count
in
  loop(buf, 0.0, 0)
end

fun n_attempts (n:int): int = let
  var buf: drand48_data
  val _ = srand48_r(0L, buf)
  fun loop (n:int, count: int, buf: &drand48_data):int =
    if n = 0 then count else loop(n-1, count + attempts(buf), buf)
in
  loop(n, 0, buf)
end

dataviewtype command = 
  | {l:agz} Compute of (int @ l | ptr l, int)
  | Quit 

viewtypedef work = command

fun fwork {l:addr}
  (ws: !WORKSHOPptr(work,l), x: &work >> work?): int = 
  case+ x of
  | ~Compute (pf_p | p, iterations)  => let 
       val () = !p := n_attempts(iterations)
       extern prfun __unref {l:addr} (pf: int @ l):void
       prval () = __unref(pf_p)
     in 1 end
  | ~Quit () => 0

fun insert_all {l,l2:agz}
               {n:nat | n > 0}
    (pf_arr: !array_v(int, n, l2) | ws: !WORKSHOPptr(work, l),
     arr: ptr l2, n: int n, iterations: int):void = let
  fun loop {l,l2:agz} {n:nat} .< n >. (
      pf: !array_v(int, n, l2)
    | ws: !WORKSHOPptr(work, l), 
      p: ptr l2, 
      n: int n, 
      iterations: int)
    : void =
    if n = 0 then () else let
      prval (pf1, pf2) = array_v_uncons{int}(pf)
      extern prfun __ref {l:addr} (pf: ! int @ l): int @ l
      prval xf = __ref(pf1)
      val () = workshop_insert_work(ws, Compute (xf | p, iterations))
      val () = loop(pf2 | ws, p + sizeof<int>, n - 1, iterations)
      prval () = pf := array_v_cons{int}(pf1, pf2)
    in
      // nothing
    end
in
  loop(pf_arr | ws, arr, n, iterations / NCPU)
end

implement main() = let 
 val ws = workshop_make<work>(NCPU, fwork)

  var ncpu: int
  val () = for(ncpu := 0; ncpu < NCPU; ncpu := ncpu + 1) let 
            val _ = workshop_add_worker(ws) in () end
 
  val nworker = workshop_get_nworker(ws)

  val (pf_gc_arr, pf_arr | arr) = array_ptr_alloc<int>(NCPU)
  val () = array_ptr_initialize_elt<int>(!arr, NCPU, 0)
  prval pf_arr  = pf_arr

  val () = insert_all(pf_arr | ws, arr, NCPU, ITER)

  val () = workshop_wait_blocked_all(ws)

  var j: Nat = 0;
  val () = while(j < nworker) let
             val () = workshop_insert_work(ws, Quit ())
           in 
             j := j + 1
           end  

  val () = workshop_wait_quit_all(ws)
  val () = workshop_free_vt_exn(ws)

  var k: Nat = 0;
  var total: int = 0;
  val () = for(k := 0; k < NCPU; k := k + 1) total := total + arr[k]
  val avg = total / double_of_int(ITER)
  val () = printf("total: %d\n", @(total))
  val () = print(avg)
in
  array_ptr_free {int} (pf_gc_arr, pf_arr | arr)
end
```

Hongwei Xi が '__ref' と '__unref' を '__borrow' で置き換え、スタックに確保された線形配列に結果を格納することで、このコードを少し綺麗にしてくれました。そのコードは
[randcomp_mt.dats](https://svn.code.sf.net/p/ats-lang/code/trunk/doc/EXAMPLE/MULTICORE/randcompec_mt.dats)
として ATS の配付版に含まれています。私が書いたコードと比較することで、その仕組みを理解する助けになればと思います。

[redditに投稿](http://www.reddit.com/r/programming/comments/cy4m6/and_e_appears_from_nowhere_quick_numeric/c0w6upk)
したように、単一スレッドのバージョンを実行すると、私のデュアルコアのマシンで 30 秒かかりました。'parworkshop'
を使ったバージョンは 16 秒でした。単一スレッドのバージョンでは 'drand48' を、'parworkshop' バージョンでは 'dthread48_r'
を使いました。後者を使わないと、乱数の状態のために 'drand48'
でのロックが発生して、スレッドがシリアライズされて、結果として単一スレッドのバージョンよりも実行時間が遅くなってしまいます。また速度をかせぐために
'atscc' に '-O3' オプションを渡してコンパイルしました。

'parworkshop' を使ったこのタスクに対する最初の試みは [この gist](https://gist.github.com/doublec/514898)
です。work 要素を保持するための線形データ型の代わりに、線形クロージャを作り、結果を取得するために 'fwork'
関数中で呼び出しました。ATS に既にある例 MULTICORE
に基づいてこのアプローチを取りました。結局、線形データ型のアプローチを選択したことで、より明確になったと感じました。
