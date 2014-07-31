# BLUISH CODER: ATSにおける線形データ型

(元記事は http://bluishcoder.co.nz/2011/02/27/linear-datatypes-in-ats.html です)

ATS におけるデータ型 (datatype) は ML や Haskell のような言語において定義される型と似ています。それらはガベージコレクタによって管理されたヒープに確保されるオブジェクトです。プログラマが明示的にメモリを解放したり、オブジェクトの生存期間を管理する必要はありません。

データ観型 (dataviewtype) もヒープに確保されるオブジェクトであるという点においてはデータ型と似ています。データ観型は線形型です。プログラマは確保されたメモリを明示的に解放しなければなりませんし、オブジェクトの生存期間を制御しなければなりません。型検査器は線形型の使用を追跡して、それが消費されてその宙ぶらりんな参照が保持されないことを型レベルで保証します。

データ型とデータ観型を比較するために、データ型を用いたリスト型の単純な定義から始めましょう:

```ocaml
datatype list (a:t@ype) =
  | nil (a)
  | cons (a) of (a, list a)

fun {seed:t@ype}{a:t@ype} fold(f: (seed,a) -<> seed, s: seed, xs: list a): seed =
  case+ xs of
  | nil () => s
  | cons (x, xs) => fold(f, f(s, x), xs)

implement main() = let
  val a = cons(1, cons(2, cons(3, cons(4, nil))))
  val b = fold(add_int_int, 0, a)
  val c = fold(mul_int_int, 1, a)
  val () = printf("b: %d c: %d\n", @(b, c))
in
  ()
end
```

このプログラムは、特定の型のインスタンスを保持するリスト型を定義しています。関数 fold がリストの fold を行なうように定義されています。main ルーチンはリスト a を作り、2回 fold を行ないます。その1つはリストの和を求め、もう1つは積を求めます。最後にそれらの結果が印字されます。

fold 関数は C++ のテンプレート関数に似ています。構文 fun {seed:t@type}{a:t@ype} は2つの引数を取る関数テンプレートを定義しています。それらは fold 関数のための seed の型と、リスト要素の型です。fold が呼び出されると、この関数は実際に使われる型でインスタンス化されます。

add_int_int と mul_int_int は2つの整数についてそれぞれ加算と乗算を行なうように定義された ATS prelude の関数です。

手動のメモリ管理が不要であることに注意してください。a に保持されているリストはガベージコレクタによって管理されたヒープの中にあります。この ATS プログラムは、ガベージコレクタとリンクして list 型が使うメモリを回収するために -D_ATS_GCATS 付きでビルドする必要があります:

```
atscc -D_ATS_GCATS test.dats
```

上記の例は、データ型を使ったプログラミングが他のガベージコレクタを持つ関数型プログラミング言語ととても良く似ていることを示しています。

次のコードは同じプログラムをデータ観型を使うように書き直したものです:

```ocaml
dataviewtype list (a:t@ype) =
  | nil (a)
  | cons (a) of (a, list a)

fun {a:t@ype} free(xs: list a): void =
  case+ xs of
  | ~nil () => ()
  | ~cons (x, xs) => free(xs)

fun {seed:t@ype}{a:t@ype} fold(f: (seed,a) -<> seed, s: seed, xs: !list a): seed =
  case+ xs of
  | nil () => (fold@ xs; s)
  | cons (x, !xs1) => let val s = fold(f, f(s, x), !xs1) in fold@ xs; s end

implement main() = let
  val a = cons(1, cons(2, cons(3, cons(4, nil))))
  val b = fold(add_int_int, 0, a)
  val c = fold(mul_int_int, 1, a)
  val () = printf("b: %d c: %d\n", @(b, c))
in
  free(a)
end
```

このリストの定義はキーワード dataviewtype を使っていることを除いて同じに見えます。このコードはリストを線形型で作っています。線形型を受け取る関数は、線形型を消費するか無変更のままにするかを引数リストに明示します。これは型の直前に付ける感嘆符で表わします。例えば:

```ocaml
// Consumes (free's) the linear type
fun consume_me(xs: list a) = ...

// Does not consume the linear type
fun use_me(xs: !list a) = ...

implement main() = let
  val a = nil
  val () = use_me(a)
  // Can use 'a' here because 'use_me' did not consume the linear type
  val () = consume_me(a)
  // The following line is a compile error because
  // 'consume_me' consumed the type
  val () = use_me(a)
in () end
```

fold の新しい定義では、線形型を消費しないことを意味する感嘆符が xs 引数の型の直前に付いていることに注意してください。このおかげで、同じリストに fold を何度でも呼び出すことができます。

fold の実装は線形型を消費せずに使うために、少し変更する必要がありました。関数本体は次のようになります:

```ocaml
case+ xs of
| nil () => (fold@ xs; s)
| cons (x, !xs1) => let val s = fold(f, f(s, x), !xs1) in fold@ xs; s end
```

fold@ 呼び出しと ! が使われていることに注意してください。xs は線形値 (つまり線形リスト) です。線形値は case 文で ! をともなってパターンマッチされるので、an_address をメモリアドレスとすると線形値 xs の型は cons(int, an_address) に変化します。xs1 には、xs の値が an_address アドレスに格納された型 ptr(an_address) が割り当てられます。これがその後で !xs1 が必要になる理由です。ここで xs1 はポインタなので、fold に実際のリストを再度渡す前にデリファレンスされなければなりません。このパターンマッチの処理はドキュメントでは '線形値の unfold (unfolding a linear value)' と表現されています。

xs の型はパターンマッチ中では変化してしまっているので、パターン本体の中で元に戻す必要があります。fold@ 呼び出しは xs の型を cons(int,an_address) から cons(int,list int) に戻します。これはドキュメントでは '線形値の fold' と呼ばれています。

main の中でリスト a を使った後はメモリを解放してやる必要があります。free 関数を使えば、リストのそれぞれの要素について繰り返し解放してくれます。free 関数は引数の型の前に感嘆符を持たないことに注意してください。

free 関数の case 文のそれぞれの行では、パターンマッチするコンストラクタの前に ~ が付いています。これはデストラクトパターン (destruction pattern) と呼ばれます。もし線形値がそのパターンにマッチしたら、パターン変数が束縛されて線形値は解放されます。その結果 free はリストを辿ってそれぞれの要素を解放するのです。

```ocaml
case+ xs of
  | ~nil () => ()
  | ~cons (x, xs) => free(xs)
```

内部について詳細な違いはありますが、2つのバージョンのコードはとても良く似ています。手動によるメモリ管理の複雑さの扱いを、ATS は比較的容易にしてくれます。さらにもしその管理が誤っていたらコンパイル時エラーにしてくれます。ATS に対する私の興味の大部分は、どうやって安全にメモリ管理をする低レベルのコードを書くか、ということにあります。
