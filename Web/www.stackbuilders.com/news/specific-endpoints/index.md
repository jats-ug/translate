# 仕様付きエンドポイント

(元記事は http://www.stackbuilders.com/news/specific-endpoints です)

> 「日々増加しているのではなく日々減少しているのだ。余計なものを削ぎ落せ。」
>
> -- ブルース・リー

Web アプリケーション開発者として、私が書いた xUnit スタイルテストの多くはデータ間の単純な関係を表明していました。
アプリケーションが成長しはじめると、追加や利便性のために、時々それらの関係によって強制された制約は増加します;
時々それらは集中し、冗長で除去が必要になり、競合すると調停を要求したりするのです。
その上、アプリケーションのデータがより複雑になると、テストは与えられたテストで明示的に宣言した状態とは別の状態の集合に暗黙的に依存しはじめます。
これによってささいなコード変更がテストの失敗をを引き起こすようになり、その失敗の原因である犯された仮定のヒントになる長く退屈なバックトレースが残されます。
このような状況においては、テストケースにおける明示的な表明に対する違反はめったに起きず、テストセッアップにおける暗黙的な仮定の無効によってテスト失敗します。
より大きなプロジェクトでは、xUnit テストの管理コストによって、製品コードの開発よりもテストとテストデータの更新や修正に開発者がより多くの時間を費やすようになります。
ここでの問は「xUnit テストスイートの構築で得られる信頼を損なうことなく、どうすれば管理コストを最小化できるのか？」ということです。

Haskell Web 開発者は、(テストと対照的に) 型によって制約をエンコードすることで開発者の生産性を維持するテストスイートの成長速度を減少させることができる、と主張します。
一人の Haskeller として、この意見に同意したいと思います。
静的な検証のレベルでエンコードされた制約は、より速く、より失敗を引き起こしたコードの位置に精度高く関連した失敗の情報を提供します。
型によって低レベルのテストの負担が軽減するので、
Haskell プログラマはアプリケーションコードにより多くの時間を使うことができ、少ないテストを書くだけで済み、高レベルなプログラミングができます。
しかしなぜここで満足するのでしょう？
私達の検証努力の多くを静的解析に移譲することで、テスト管理の範囲をより抑える手法はないのでしょうか？
私達の冗長なチェックのコストを小さくする方法を今日垣間見れるような、既存のシステムはないのでしょうか？
この記事では、[ATS](http://www.ats-lang.org/) と [Haskell](https://www.haskell.org/) 言語の両方を混合して使った例を通して、期待できる解決策を簡潔に探索します。

> 「ねぇ教えて、どっちへ行けばいいの？」
> 「そりゃー、アンタがどこへ行きたいかによるにゃー」と猫は言った。
>
> (不思議の国のアリス)

日常的な Web 開発者として私が習慣的に出会う仕様と共通点がある、仮想上のビジネス仕様を考えましょう。
その仕様を xUnit テストケースとしてエンコードする管理コストを取り除くために、型レベルでこの仕様の多くをエンコードします。

その仕様とは次のようなものです:

ある書籍販売業者は "VIP" として分類された顧客に対してのみ購入割引を提供したいとします;
通常の顧客は購入割引の資格はありません。
1つの取引において2冊を超えた本毎に購入合計に対する割引割合は 2% OFF になります。
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
[等価な LiquidHaskell の例](http://goto.ucsd.edu:8090/index.html#?demo=permalink%2F1449031696_5061.hs) を提供してくれた Ranjit Jhala 教授に感謝します!!!

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

上記の dataprop 宣言は種 customer, int, bool の間の関係を表現しています。
この dataprop の最初のケースは、全ての int について定義された冊数以上なら、customer, int, bool の関係は、customer が vip で、int は閾値より大きく、bool は true であることを表明しています。
このケースは `VIP_ENOUGH_BOOKS` と名付けられ、vip customer が出荷費用の割引を受けるに十分な本を購入していることを意味しています。

dataprop の二番目のコンストラクタは種 customer, int, bool 間の別の有効な関係を宣言しています。
`CUSTOMER_GETS_DISCOUNT` は customer が VIP で、int が閾値以下で、(customer が割引を受ける権利があるのかどうかを表わす) bool が false であることを許可しています。
このケースの名前は `VIP_NOT_ENOUGH_BOOKS` です。

最後に、コンストラクタ `REGULAR` は「regular customer には出荷割引の資格がない」関係をエンコードしています;
regular customer が購入しようとしている冊数に関係なく、regular customer は割引を受けられません。
これで型レベルにエンコードされた制約ができました;
それらを利用してみましょう。

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

述語関数 `customer_gets_discount` の記述において dataprop `CUSTOMER_GETS_DISCOUNT` を使います。
この型は、全ての customer と全ての int について、この関数が型 customer の値と型 int の値を引数に取り、型 bool の値を返すことを表明しています。
この型シグニチャは単純ですが、多くの強い型付けプログラマの予想以上のことを表現しています;
このシグニチャは customer, int, bool 間の事後条件の関係が代数的性質 `CUSTOMER_GETS_DISCOUNT` として定義された関係のみに制限されていることを主張しています。

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

「型駆動開発」の習慣では、ここでは別名ですが、型定義からはじめます。
`calculate_discount'` 関数が何を返したいのか詳細化するのです。
この typedef discount は「全ての int `book_count` について、book_count から閾値を引いてから割引率を掛けた返り値 int を返す」と主張しています。
これは難しくありませんでした。
私に言わせれば、それは小さな仕様 -- 正確に私達が欲しいものです。

`calculate_discount'` の型シグニチャは、そもそも引数 int が割引の権利を持つ関係にあることを表明する静的な情報の追跡で促進されています。
`calculate_discount'` は `CUSTOMER_GETS_DISCOUNT` の関係で示されない int をを受け付けません。

もちろん `calculate_discount'` の実装は `g1int_mul2` を使った単純な算術ルーチンです。
このとき関数は返り値の型に付随した `discount` 制約を満たす静的な情報を生成します
-- 別の言い方をすると、`g1int_mul2` はオペランドを乗算をするだけでなく、その演算が実際に行なわれた証拠も返します。
後でこの証拠が必要になります。

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

           end
      else 0
  end
```

`calculate_discount` はそれらを統合し、(上記のブックストアデモから呼び出される) Haskell コードから呼び出される関数です。
`customer_type` を呼び出して customer の型をチェックし、それから customer 型と彼の冊数と共に `customer_gets_discount` を呼び出します。
上記で示したように、`customer_gets_discount` 呼び出しは customer が割引を受けられるかどうかを示す bool と共に、その判定を指示する関係の証明を返します。
もし customer に権利があれば、`customer_gets_discount` で得られた証明と冊数を `calculate_discount'` に渡して、割引を計算します。
守護天使のように重要な情報の3つの部品、customer 型, 冊数, 割引権利、の間の関係が偏在しているために、これは解決します;
コードの至るところでそれは有効です;
これは applied type system と呼ばれます。

(GHC と Cabal で静的リンクより簡単な動的リンクによって) このプロジェクトがどのようにビルドされるかチェックしたい方は [ソースコード](https://github.com/stackbuilders/high-spec) を取得できます。

結論として、書籍販売業者と技術主任の双方によって扱われた仕様のテストカバレッジを提供しました。
さらに静的解析のみを使い、単純な実行時のユニットテストは使いませんでした。
そのため、システム境界と高レベルのユーザに面した挙動に対する実行時テストから解放されて、管理コストを低く抑えることができました。
プログラマが広く使用できるように静的検証を使用がより簡単になり、そのような技術によって多大な開発努力が大幅に省力できればと願っています。
なんらかの実行時テストは常に必要になりそうですが、近い将来、静的解析で支持された最小の実行時テストが実現するでしょう。
