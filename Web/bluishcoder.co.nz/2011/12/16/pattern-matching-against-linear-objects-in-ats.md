# BLUISH CODER: ATSにおける線形オブジェクトのパターンマッチ

(元記事は http://bluishcoder.co.nz/2011/12/16/pattern-matching-against-linear-objects-in-ats.html です)

この記事では線形リストを扱います。
これは `list_vt` 型で [ATS](http://www.ats-lang.org/) prelude で定義されています。
`list_vt` は Lisp や線形型のない関数型プログラミング言語におけるリスト型に似ています。
そのリストのメモリはガベージコレクタが管理せず、線形オブジェクトへの参照が唯一1つであるという規則を型システムが強制します。
`list_vt` インスタンスにパターンマッチを使うには、少し努力が要求されることがあります。

## パターンマッチ

線形オブジェクトにパターンマッチするとき、デストラクタマッチもしくは非デストラクタマッチを行なうことができます。
前者は自動的にオブジェクトに確保されたメモリを破棄して解放します。
後者はそうではありません。
デストラクタマッチは接頭辞 `~` を付けたパターンマッチ節を使います。
例えば、次のコードは整数リストを印字し、その最中にそのリストを破棄します:

```ats
fun print_list (l: List_vt (int)): void =
  case+ l of
  | ~list_vt_nil () => printf("nil\n", @())
  | ~list_vt_cons (x, xs) => (printf("cons %d\n", @(x)); print_list(xs))

fun test1 (): void = {
  val a = list_vt_cons {int} (1, list_vt_nil)
  val () = print_list (a)
}
```

非デストラクタマッチを使うとうまくいきません。
次のコードは型検査に失敗します:

```ats
fun print_list2 (l: !List_vt (int)): void =
  case+ l of
  | list_vt_nil () => printf("nil\n", @())
  | list_vt_cons (x, xs) => (printf("cons %d\n", @(x)); print_list(xs))

fun test2 (): void = {
  val a = list_vt_cons {int} (1, list_vt_nil)
  val () = print_list2 (a)
  val () = list_vt_free (a)
}
```

この例の問題はそのマッチによって、値 `l` の外にある線形オブジェクトを取っていることです。
`l` は異なった型として放置されていますが、`print_list2` の関数シグニチャでその型は変更も消費もされないと宣言してしまっています。
そのマッチを使ったらすぐに、その線形オブジェクトを `l` に戻す方法が必要なのです。
これを行なうプリミティブが `fold@` で、私のブログエントリ [「ATSにおける線形データ型」](http://bluishcoder.co.nz/2011/02/27/linear-datatypes-in-ats.html) で簡単に説明しました。
`fold@` は `l` の型を元に戻して、パターンマッチ変数へのアクセスを防止します。
その使用例は次のようになります:

```ats
fun print_list2 (l: !List_vt (int)): void =
  case+ l of
  | list_vt_nil () => (fold@ l; printf("nil\n", @()))
  | list_vt_cons (x, !xs) => (printf("cons %d\n", @(x)); print_list2(!xs); fold@ l)

fun test2 (): void = {
  val a = list_vt_cons {int} (1, list_vt_nil)
  val () = print_list2 (a)
  val () = list_vt_free (a)
}
```

このバージョンで注意すべきは、`list_vt_cons` のマッチで `xs` パラメータを `!xs` に変更していることです。
コンスコンストラクタの二番目の引数は線形オブジェクトです。
もしそのオブジェクトを `xs` にマッチさせたら、線形オブジェクトの別名を作るもう1つの例になっていまいます。
`l` を取り出して、元に戻す必要があるのです。
ATS の解決策では、接頭辞 `!` をパターンマッチに付けることです。
これによて `xs` はオブジェクト自身ではなくオブジェクトへのポインタになります。
そのためこの例では `xs` は型 `ptr addr` を持ちます。
このとき `addr` は実際の `List_vt` オブジェクトのアドレスです。
これはまた、`print_list2` の再帰呼び出しにおいて、`xs` に `!` 接尾辞を付ける理由です。
この `!` はポインタのデリファレンスを意味しています。
そのためそれが指している `List_vt` は再帰呼び出しの引数に渡されます。

この方法では線形オブジェクトは取り出されず、ポインタを通じてアクセスするだけです。
その節での `fold@` 呼び出しは `xs` を `List_vt` オブジェクトに戻します。
`fold@` は `!xs` を使った後に呼び出します。
もしそれより前に呼び出すと、デリファレンスできる `xs` を表わす観 (view)にアクセスできなくなってしまいます。
また `fold@` 呼び出しは型検査時のみ使われて、その後削除されるので、`print_list2` は末尾再帰です。

## 線形リストをフィルタする

私のプロジェクトでは線形リストをフィルタする必要がありました。
不幸にも ATS は標準 prelude に線形リストのフィルタ実装を持っていません (永続化リストには有ります)。
私の最初の試みでは `list_vt_filter` を次のように書きました:

```ats
fun list_vt_filter (l: !List_vt (int), f: int -<> bool): List_vt (int) =
  case+ l of
  | list_vt_nil () => (fold@ l; list_vt_nil)
  | list_vt_cons (x, !xs) when f (x) => let
                                          val r = list_vt_cons (x, list_vt_filter (!xs, f))
                                        in
                                          fold@ l; r
                                        end
  | list_vt_cons (x, !xs) => let
                                val r = list_vt_filter (!xs, f)
                              in
                                fold@ l; r
                              end
```

非デストラクタマッチと `fold@` を使う以前のコード `print_list2` と良く似ているため、このコードは馴染みがあるでしょう。
関数 `list_vt_filter` は引数に `list_vt` を取り、リスト中のそれぞれの要素に関数を適用します。
その関数が `true` を返すと、その要素は結果のリストに含まれます。
使い方は次のようになります:

```ats
val a  = list_vt_cons (1, list_vt_cons (2, list_vt_cons (3, list_vt_cons (4, list_vt_nil ()))))
val b  = list_vt_filter (a, lam (x) => x mod 2 = 0)
val () = list_vt_foreach_fun<int> (a, lam(x) =<> $effmask_all (printf("Value: %d\n", @(x))))
val () = list_vt_free (b)
val () = list_vt_free (a)
```

この実装の1つの問題点は、それが末尾再帰ではないことです。
結果のリストサイズに比例してスタックは成長してしまいます。

## 末尾再帰フィルタ

Lisp コードでは私はしばしば、アキュムレータを渡して結果のリストを末尾再帰的に構築します。
そしてそのアキュムレータの先頭に結果の新しい要素を追加するのです。
これは逆順にリストを構築することになるので、返る前にそのリストを逆順にします。
この ATS コードは次のようになります:

```ats
fun list_vt_filter (l: !List_vt (int), f: int -<> bool): List_vt (int) = let
  fun loop (l: !List_vt (int), accum: List_vt (int)):<cloptr1> List_vt (int)  =
    case+ l of
    | list_vt_nil () => (fold@ l; accum)
    | list_vt_cons (x, !xs) when f (x) => let
                                            val r = loop (!xs, list_vt_cons (x, accum))
                                          in
                                            (fold@ l; r)
                                          end
    | list_vt_cons (x, !xs) => let
                                 val r = loop (!xs, accum)
                                in
                                  (fold@ l; r)
                                end
in
  list_vt_reverse (loop (l, list_vt_nil))
end
```

`cloptr1` 関数注釈は、ガベージコレクタの代わりにクロージャ環境のメモリが malloc と free を使ってコンパイラによって管理されるような、クロージャとして内側の関数をマークしています (ガーベッジコレクタを使ったクロージャは `cloref1` で表わします)。
ATS で使われる異なるクロージャと関数については、私のブログポスト [「ATSのクロージャ」](http://bluishcoder.co.nz/2010/06/20/closures-in-ats.html) を参照してください。

Unfortunately the requirement to use `fold@` after we’ve finished with using the pattern matched variables makes the code slightly more verbose as we need to do the tail recursion, obtaining the result, then do the `fold@` and return the result.
Remember that the `fold@` is erased at type checking type which is how this code remains tail recursive even though the code structure makes it look like it isn’t.

One downside to this approach is we iterate over the list twice.
Once to build the result, and once over the result to reverse it.

## シングルパス末尾再帰フィルタ

The creation of the result list can be done in a single pass if we could create a cons with no second argument, and fill in that argument later when we have a result to store there that passes filtering.
ATS allows construction of datatypes with a ‘hole’ that can be filled in later. The ‘hole’ is an unintialized type and we get a pointer to it. An example of doing this is:

```ats
var x = list_vt_cons {int} {0} (1, ?)
```

This creates a `list_vt_cons` with the data set to `1` but no second parameter.
Instead of that parameter being of type `List_vt (int)` it is of type `List_vt (int)?`, the `?` signifying it is uninitialized.
For this example we have to pass the universal type parameters explicitly (the `{int} {0}`) as the ATS type inference algorithm can’t compute them.

To get a pointer to the ‘hole’ we have to pattern match:

```ats
val+ list_vt_cons (_, !xs) = x
val () = !xs := list_vt_nil
val () = fold@ x
```

In this example the `xs` is a pointer, pointing to the `List_vt (int)?`.
It assigns a `list_vt_nil` to this, making the tail of the cons a `list_vt_nil`.
Just like in our previous pattern matching examples using case, the code has to do a `fold@` to change the type of `x` back to that containing a linear object once we’ve finished using xs.

Now that we can get pointers to the tail of the list we can implement a single pass tail recursive filter function:

```ats
fun list_vt_filter (l: !List_vt (int), f: int -<> bool): List_vt (int) = let
  fun loop (l: !List_vt (int), res: &List_vt (int)? >> List_vt (int)):<cloptr1> void =
    case+ l of
    | list_vt_nil () => (fold@ l; (res := list_vt_nil))
    | list_vt_cons (x, !xs) when f (x) => let
                                            val () = res := list_vt_cons {int} {0} (x, ?)
                                            val+ list_vt_cons (_, !p_xs) = res
                                           in
                                             loop (!xs, !p_xs); fold@ l; fold@ res
                                           end
    | list_vt_cons  (x, !xs) => (loop (!xs, res); fold@ l)

  var res: List_vt (int)?
  val () = loop (l, res)
in
  res
end
```

The loop function here no longer turns a result.
Instead the result is passed via a reference (the & signifies ‘by reference’).
When there is something that needs to be stored in the list, a cons is created with a hole in the tail position.
This cons is stored in the result we are passing by reference and we tail recursively call with the hole as the new result.
ATS converts this to nice C code that is a simple loop rather than recursive function calls.

## 雑多なこと

The code examples in this post use `List_vt (a)`.
This is actually a typedef for `list_vt (a,n)` where a is the type and `n` is the length of the list.
The typedef allows shorter examples without needing to specify the sorts for the list length.
Using the full type though has the advantage of being able to specifiy a bit more type safety.
For example, the original filter function would be declared as:

```ats
fun list_vt_filter {n:nat} (l: !list_vt (int,n), f: int -<> bool): [r:nat | r <= n] list_vt (int, r)
```

This defines the type of the result as having a length equal to or less than that of the original list.
This helps prevent errors in the implementatin of the filter - it can’t accidentally leave extra items in the list.
I cover this type of thing in my post on [dependent types](http://bluishcoder.co.nz/2010/09/01/dependent-types-in-ats.html).

Another addition to safety that adding the extra sorts can provide is the ability to check that the function terminates.
This can be done by adding a termination metric to the function definition:

```ats
fun list_vt_filter {n:nat} .<n>. (l: !list_vt (int,n), f: int -<> bool): [r:nat | r <= n] list_vt (int, r)
```

The compiler checks that `n` is decreasing on each recursive call.
If this fails to happen the recursive calls may not terminate and it becomes a compile error.
This is discussed in the Termination-Checking for Recursive Functions section of the [Introduction to Programming in ATS book](http://jats-ug.metasepi.org/doc/ATS2/INT2PROGINATS/index.html).

A description of how `fold@` works is in the [ATS/Anairats User’s Guide PDF](http://ats-lang.sourceforge.net/htdocs-old/DOCUMENT/MISC/manual_main.pdf).
It’s in the ‘Dataviewtypes’ section of the ‘Programming with Linear Types’ chapter and is referred to as folding and unfolding a linear type.

It’s the usage of linear types and dealing with their restrictions that makes my examples a bit more complex.
If you use ATS mainly with non-linear types and link with the garbage collector then it becomes very much like using any other functional programming language, but with additional features in the type system.
My interest has been around avoiding using a garbage collector and having the compiler give errors when memory is not allocated or free’d correctly.
Don’t be put off from using ATS by these complex examples if you’re fine with using garbage collection and non-linear datatypes.
You might never need to deal with the cases that bring in the extra complexity.
