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

The definition of the list looks the same but uses the keyword dataviewtype. This makes list a linear type. Functions that accept linear types will show in the argument list whether they consume or leave unchanged the linear type. This is denoted by an exclamation mark prefixed to the type. For example:

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

Notice in the new definition of fold that the xs argument type is prefixed with an exclamation mark meaning it does not consume the linear type. This allows calling fold multiple times on the same list.

The implementation of fold had to be changed slightly to use the linear type without consuming it. The body of the function is:

```ocaml
case+ xs of
| nil () => (fold@ xs; s)
| cons (x, !xs1) => let val s = fold(f, f(s, x), !xs1) in fold@ xs; s end
```

Notice the fold@ call and the use of !. xs is a linear value (ie. our linear list). Because the linear value matches a pattern in a case statement that has a ! then the type of the linear value xs changes to be cons(int, an_address) where an_address is a memory address. xs1 is assigned the type ptr(an_address) where the value of xs is stored at address an_address. This is why the !xs1 is needed later. Since xs1 is now a pointer it must be dereferenced before calling fold again to pass the actual list. This pattern matching process is known as ‘unfolding a linear value’ in the documentation.

As the type of xs has changed during the pattern matching we need to change it back in the body of the pattern. The fold@ call returns the type of xs from cons(int,an_address) back to cons(int,list int). This is referred to as ‘folding a linear value’ in the documentation.

After using the list a in main we need to free the memory. This is done with the free function which iterates over the list consuming each element. Notice that free does not have an exclamation mark prefixed to the argument type.

Each line in the case statement in free has a ~ prefixed to the constructor to be pattern matched against. This is known as a destruction pattern. If a linear value matches the pattern then the pattern variables are bound and the linear value is free’d. The result of this is free walks the list freeing each element.

```ocaml
case+ xs of
  | ~nil () => ()
  | ~cons (x, xs) => free(xs)
```

There’s a lot of detail going on under the hood but the code between the two versions is very similar. ATS makes it relatively easy to deal with the complexities of manual memory management and fails at compile time if it’s not managed correctly. Much of my interest in ATS is how to write low level code that manages memory safely.
