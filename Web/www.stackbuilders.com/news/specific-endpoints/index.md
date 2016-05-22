# Specific Endpoints

(元記事は http://www.stackbuilders.com/news/specific-endpoints です)

> 「日々増加しているのではなく日々減少しているのだ。余計なものを削ぎ落せ。」
>
> -- ブルース・リー

Web アプリケーション開発者として、私が書いた xUnit スタイルテストの多くはデータ間の単純な関係を表明していました。
アプリケーションが正常しはじめると、追加や利便性のために、時々それらの関係によって強制された制約は増加します;
時々それらは集中し、冗長で除去が必要になり、競合すると調停を要求したりするのです。
その上、アプリケーションのデータがより複雑になると、テストは与えられたテストで明示的に宣言した状態とは別の状態の集合に暗黙的に依存しはじめます。
これによってささいなコード変更がテストの失敗をを引き起こすようになり、その失敗の原因である犯された仮定のヒントになる長く退屈なバックトレースが残されます。
このような状況において、テストケースにおける明示的な表明に対する違反はめったに起きず、テストセッアップにおける暗黙的な仮定の無効によるテスト失敗が起きます。
より大きなプロジェクトでは、xUnit テストの管理コストによって、製品コードの開発よりもテストとテストデータの更新や修正に開発者がより多くの時間を費やすようになります。
ここでの問は「どうすれば xUnit テストスイートの構築で得られる信頼を損なうことなく、管理コストを最小化できるのか？」ということです。

Haskell Web 開発者は、(テストと対照的に) 型によって制約をエンコードすることで開発者の生産性を維持するテストスイートの成長速度を減少させることができる、と主張します。
一人の Haskeller として、この意見に同意したいと思います。
静的な検証のレベルでエンコードされた制約は、より速く、より失敗を引き起こしたコードの位置に精度高く関連した失敗の情報を提供します。
型によって低レベルのテストの負担が軽減するので、
Haskell プログラマはアプリケーションコードにより多くの時間を使うことができ、少ないテストを書くだけで済み、高レベルなプログラミングができます。
しかしなぜここで満足するのでしょう？
私達の検証努力の多くを静的解析に移譲することで、テスト管理の範囲をより抑える手法はないのでしょうか？
私達の冗長なチェックをコストの小さくする方法を今日垣間見るような、既存のシステムはないのでしょうか？
この記事では、[ATS](http://www.ats-lang.org/) と [Haskell](https://www.haskell.org/) 言語の両方を混合して使った例を通して、期待できる解決策を簡潔に探索します。

> 「ねぇ教えて、どっちへ行けばいいの？」
> 「そりゃー、アンタがどこへ行きたいかによるにゃー」と猫は言った。
>
> (不思議の国のアリス)

日常的な Web 開発者として私が習慣的に出会う仕様と共通点がある、仮想上のビジネス仕様を考えましょう。
その仕様を xUnit テストケースとしてエンコードする管理コストを取り除くために、型レベルでこの仕様の多くをエンコードします。

その仕様は次のようなものです:

ある書籍販売業者は "VIP" として分類された顧客に対してのみ購入割引を提供したいとします;
通常の顧客は購入割引の資格はありません。
1取引において2冊を超えた本毎に購入合計に対して、割引割合は 2% OFF になります。
もし VIP が2冊以上購入しなければ、その顧客は購入割引を受けられません。

現実的ですが、その仕様は意図的に単純です。
表現された挙動を具体的に体感するために、次の **VIP** ブックストアビューのモックを試しましょう。
(訳注: 動作させたい方は [原文](http://www.stackbuilders.com/news/specific-endpoints) のモックを試してください。)

![](img/bookstore.png)

上記の仕様を解析した後、書籍販売業者の技術主任は、彼女が Haskell で開発した API へのエンドポイント追加を要求します。
彼女はそのエンドポイントで、顧客の識別子とその顧客が購入するつもりの本の冊数を受け取りたいのです。
そのエンドポイントは JSON フォーマットで見積った購入割引を返します。

私達の最初の仕事は、技術主任によって定義されたエンドポイントの技術的な仕様を形式化することです。
技術主任はその API を Servant を使って開発しています。
このパッケージは私達のゴールである型レベルの仕様のエンコードに大幅な柔軟性を実現してくれます。

```haskell
type BookStoreAPI = "user" :> Capture "userid" Integer
                           :> "discount"
                           :> Capture "book-count" Integer
                           :> Get '[JSON] Integer

handleCalculateDiscount :: Monad m => Integer -> Integer -> m Integer
handleCalculateDiscount id bookCount =
  return $ calculateDiscount id bookCount

bookStoreAPI :: Proxy BookStoreAPI
bookStoreAPI = Proxy

server :: Server BookStoreAPI
server = handleCalculateDiscount
```

(正確とは言わないまでも) 型 `BookStoreAPI` はエンドポイントの仕様を表現しています。
この型は明確に URL `/user/:userid/discount/:book-count` が user の識別子と割引計算を見積るための仮の冊数を受け取ることを表わしています。
この API 型はその中身を供給するのに使われる Web ハンドラの型を示しています。
Servant を用いると、このハンドラに対する出入りがその URL の仕様に従っていると安心できます。

もしまだ読んでいなければ、どのように BookStoreAPI 型が要求を捕捉し、多くの Haskell 開発者が好む「構文による正しさ (correct by construction)」哲学を促進することによって型安全を提供するか理解するために、[Servant チュートリアル](https://haskell-servant.github.io/tutorial/) を読んでください。

これでこの機能へのエントリイポイントには仕様が強制されました。
すると、どうやって (上記のコードで見た) ハンドラ `calculateDiscount` 実装し、
どうやって API のエンドポイント仕様を許すような Servant と同様に正確/簡単な方法で割引ロジックに制約をエンコードできるのでしょうか？

```haskell
foreign import ccall unsafe calculate_discount :: CInt -> CInt -> CInt

calculateDiscount :: Integer -> Integer -> Integer
calculateDiscount userId bookCount =
  fromIntegral $ calculate_discount (fromIntegral userId)
                                    (fromIntegral bookCount)
```

この FFI 呼び出しは、Haskell のエレガントな世界をはなれて、静的に制約を実装を主張することを意図しています。
C 言語の上に階層化された高度な型システムとして表現された ATS と呼ばれる言語を使います。
ATS のプリミティブ型はC言語の型をミラーしているので、Haskell の FFI を使ったインターフェイスを取るのは簡単です。

更新: 次のように、Haskell を使わない必要はありません。
[LiquidHaskell](http://goto.ucsd.edu/~rjhala/liquid/haskell/blog/about/) は Haskell に留まり、私達が開発した ATS コードで発見した保証を同じく実現する方法を提供します。
[等価な LiquidHaskell の例](http://goto.ucsd.edu:8090/index.html#?demo=permalink%2F1449031696_5061.hs) を提供してくれた Ranjit Jhala 教授ありがとうございます!!!

```ats
#define DISCOUNT_PERCENTAGE   2 (* percent *)
#define BOOK_THRESHOLD        2 (* books *)
#define LTE_BOOK_THRESHOLD(i) (i <= BOOK_THRESHOLD)
#define GT_BOOK_THRESHOLD(i)  (i > BOOK_THRESHOLD)

datasort customer =
  | vip
  | regular

datatype customer(customer) =
  | vip (vip)
  | regular (regular)
```

ATS は「プログラマ中心」に命題の式を作るようにデザインされていて、
プログラマとして私は「静的」と「動的」のコセンプトの間に明確な分離を見つけました。
`datasort` 宣言は単純な型宣言で、それは静的な世界に属します。
(ATS で動的にコンストラクトすると考えられる) `datatype` 宣言は Haskell プログラマに馴染み深いはずですが、
唯一不思議な点は静的と動的の2つの次元の間の導線をコンストラクトすることで datatype 宣言は種 (sort) によってパラメータ化できることです。

静的な世界では、性質をコンストラクトします。
動的な世界では、プログラムをコンストラクトし、そのプログラムを制約するために性質を操ります。
上記の例の datatype `customer` はシングルトンである2つのデータコンストラクタを持ちます。
すなわち、それらは唯一1つの値を表わすことを強制されており、その値はそれぞれ `vip` と `regular` として静的な世界で表現されます。
これは、動的な世界でこれらのデータコンストラクタを使って関数を書く際、それらの使用は静的な世界でのそれらの対応で定義した性質を強制できることを意味しています。
もちろんこれは、静的な制約で除外された挙動を表わす xUnit テストは余分で、無意味であることを意味しています。

> 「それが黒いなら、どんな色も手にできる。」
>
> -- ヘンリー・フォード

```ats
dataprop CUSTOMER_GETS_DISCOUNT (customer, int, bool) =
  | {i:int | GT_BOOK_THRESHOLD i}  VIP_ENOUGH_BOOKS     (vip, i, true)
  | {i:int | LTE_BOOK_THRESHOLD i} VIP_NOT_ENOUGH_BOOKS (vip, i, false)
  | {i:int}                        REGULAR              (regular, i, false)
```

The dataprop declaration above describes a relationship between the sorts customer, int, and bool.
The first case of our dataprop states that for all ints, as long as they are greater than a defined number of books, the relationship between the customer, those particular ints, and the bool should be that the customer is a vip, the int is greater than the threshold, and the bool equals true.
This case is named `VIP_ENOUGH_BOOKS` to denote that it is the case where a vip customer is purchasing enough books to get a discount on his shipping cost.

The second constructor of the dataprop declares another valid relationship between the sorts customer, int, and bool.
`CUSTOMER_GETS_DISCOUNT` allows the customer to be VIP, the int to be less than the threshold, and the bool (which represents whether or not the customer is entitled to a discount) to be false.
This case is named `VIP_NOT_ENOUGH_BOOKS`.

Finally, the constructor `REGULAR` encodes the relationship "regular customers are not eligible for a shipping discount";
it doesn't matter how many books a regular customer intends to buy, they receive no discount regardless.
We now have our constraints encoded at the type level;
we only need to put them to use.

```ats
extern fn customer_gets_discount:
  {c: customer}{i: int} (customer c, int i) ->
  [b:bool] (CUSTOMER_GETS_DISCOUNT(c,i,b) | bool b)

implement customer_gets_discount(c,i) =
   case (c, i) of
   | (regular (), i) => (REGULAR | false)
   | (vip(), i)      => if GT_BOOK_THRESHOLD i
                          then (VIP_ENOUGH_BOOKS | true)
                          else (VIP_NOT_ENOUGH_BOOKS | false)
```

We use the dataprop `CUSTOMER_GETS_DISCOUNT` in the description of the predicate function `customer_gets_discount`.
The type expresses that this function, for all customers and all ints, takes as arguments a value of type customer and a value of type int and returns a value of type bool.
The type signature is simple enough yet it says something more and outside of what most strongly typed programmers would normally expect;
it says that the post-condition relationship between the customer, int, and bool are confined to only those relationships we defined as part of the algebraic property `CUSTOMER_GETS_DISCOUNT`.

```ats
typedef discount (book_count:int) =
  [res:int] (MUL( book_count - BOOK_THRESHOLD
                , DISCOUNT_PERCENTAGE
                , res
                ) | int(res))

extern fn calculate_discount':
  {c: customer}{i:int}
  (CUSTOMER_GETS_DISCOUNT(c,i,true) | int(i)) ->
  discount(i)

implement calculate_discount' (_ |bookCount) =
  g1int_mul2 (bookCount - BOOK_THRESHOLD, DISCOUNT_PERCENTAGE)
```

In the tradition of `type-driven development` we start with a type definition, here an alias, detailing what we want to have returned from our `calculate_discount'` function.
The typedef discount says "for all ints 'book_count' we will return an int 'result' such that the result will be book_count minus our threshold thus multiplied by the discount percentage".
That wasn't hard, and if you ask me it is pretty specific -- which is exactly what we want.

The type signature of `calculate_discount'` takes things a step further by facilitating the tracking of the static information asserting that its argument int is indeed a part of a relationship that entitles a discount in the first place.
`calculate_discount'` will not accept an int that cannot be shown to be of the `CUSTOMER_GETS_DISCOUNT` relationship.

The implementation of `calculate_discount'` is of course a simple arithmetic routine that uses `g1int_mul2` which is a function that generates that static information needed to appease the `discount` constraint attached to our result type -- in other words, `g1int_mul2` not only multiplies its operands but returns evidence that the operations actually took place.
We will need this evidence later.

```ats
extern fn calculate_discount: (int, int) -> int = "mac#"
implement calculate_discount (userid, bookCount) =
  let
    val c = customer_type userid
    val count = g1ofg0 (bookCount)
    val (pf | gets_discount) = customer_gets_discount (c, count)
  in
    if gets_discount
      then let
            val (_ | discount) = calculate_discount' (pf | count)
           in
            discount
           end
      else 0
  end
```

`calculate_discount` brings it all together and is the function that is being called from Haskell (and by the bookstore demo above).
We call `customer_type` to check the customer's type and then call `customer_gets_discount'` with the customer's type and his count of books.
As stated above, the call to `customer_gets_discount'` returns both a bool denoting whether the customer gets a discount as well as the proof of the relationship backing the determination of its decision.
If the customer is entitled, we will calculate the discount by supplying the proof given to us by `customer_gets_discount` and the count of books to `calculate_discount'`.
This all works out because, like a guardian angel, there is an omnipresence checking that the relationship between our three pieces of important information;
the customer type, his current book count, and his discount eligibility;
remain valid throughout the code.
That presence is an applied type system.

The [source](https://github.com/stackbuilders/high-spec) is available for anyone who wants to check how the project is built (by dynamic linking because it's easier than static when it comes to GHC and Cabal).

In conclusion, we have provided test coverage of the specifications we had been handed by both bookseller and the tech-lead.
What is more is that we did so using only static analysis and without the use of a single run-time unit test.
Thus, we are free to focus our run-time testing on system boundaries and higher-level user-facing behavior keeping our maintenance costs low.
I hope for more in the direction of simplifying the use of static verification so that it becomes accessible to a wider audience and a larger number of development efforts can realize the tremendous savings made possible by such techniques.
While it is likely that some form of run-time testing will always be of necessity, maybe in the near future minimizing run-time testing in favor of static analysis will be in fact... the way to go.
