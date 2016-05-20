# Specific Endpoints

(元記事は http://www.stackbuilders.com/news/specific-endpoints です)

> 「日々増加しているのではなく日々減少しているのだ。余計なものを削ぎ落せ。」
>
> -- ブルース・リー

Web アプリケーション開発者として、私が書いた xUnit スタイルテストの多くはデータ間の単純な関係を表明していました。
When applications begin to grow, the constraints imposed by these relationships are sometimes strengthened through addition or relaxed for convenience;
they sometimes converge and become redundant requiring removal or come into conflict and require reconciliation.
Furthermore, as application data becomes more complicated tests begin to implicitly depend on several sets of conditions aside from the one that is explicitly stated in a given test.
This leads to situations where harmless code modifications begin to cause test failures leaving only lengthy back traces to hint at the violated assumption which caused the failure.
In these situations it is no seldom occurrence that a test fails for not the betrayal of an explicit assertion in a test case but because of the invalidity of an implicit assumptions made in the test setup.
In larger projects the maintenance costs of x-unit testing efforts can be such that they require a developer to spend much more time in the update and correction of tests and test data than in the development of production code.
The question is then "how can we greatly minimize maintenance costs while losing little of the confidence gained by the construction of heavy x-unit test suites?"

Haskell web developers claim that encoding constraints as types (as opposed to tests) reduces the growth rate of test suites which encourages sustained developer productivity.
As a Haskeller, I am inclined to agree with that notion.
I firmly believe that constraints encoded at the level of static verification offer faster and more relevant failure information often with pinpoint precision with respect to the location in the code that caused the failure.
With a degree of the low-level testing burden mitigated by types, Haskell programmers can spend more time focusing on application code and write fewer tests and at a higher level.
But why stop there?
Are there emerging techniques that can further improve savings in the test maintenance department by delegating more of our verification effort to static analysis?
Are there existing systems that can, today, offer some glimpse of how we might perform our redundant checks in a manner less costly?
In this post we briefly explore one promising solution through an example combining the use of both the languages **ATS** and **Haskell**.

> 「ねぇ教えて、どっちへ行けばいいの？」
> 「そりゃー、アンタがどこへ行きたいかによるにゃー」と猫は言った。
>
> (不思議の国のアリス)

Consider a hypothetical business specification which resembles much of what I, as an everyday web developer, encounter on a regular basis.
We are going to encode much of this specification at the type level in effort to eliminate the maintenance costs of encoding our specs as x-unit tests cases.

Let's begin with our specification:

A book dealer wants to offer shipping discounts to her customers but only those customers classified as "VIP"; regular customers are not eligible for a shipping discount.
The discount percentage is 2% off the shipping amount for every book beyond the 2nd in a given transaction.
If a VIP does not have more than 2 books the customer does not receive a discount on shipping.

Although quite realistic, the specification is intentionally kept simple.
Please try the mock VIP bookstore view below to get a concrete feel for the described behavior.

![](img/bookstore.png)

The book dealer's technical lead, after analyzing the above specification, asks for an endpoint to be added to the API she has developed in Haskell. She would like the endpoint to collect the customer's identification as well as the current number of books the customer intends to buy. The endpoint is to return, in JSON format, the estimated shipping discount.

Our first task is to formalize the technical specification of the endpoint which was defined by the technical lead. The technical lead has been developing her API using Servant which we are very happy about given that the package gives us a good deal of flexibility in encoding the specification at the type level which is our goal.

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

The type `BookStoreAPI` strongly (if not exactly) expresses the endpoint specification. The type unambiguously represents the URL `/user/:userid/discount/:book-count` which collects the user's identification and a tentative count of books for the estimated discount calculation. The API type dictates the type of the web handler used to serve the content. With Servant we can rest assured that the entry and exit to/from our handler is in harmonious accordance with the URL specification.

If you haven't already, please read the [Servant tutorial](https://haskell-servant.github.io/tutorial/) to get an understanding of how the BookStoreAPI type captures the requirements and provides us with a good degree of type safety by facilitating the "correct by construction" philosophy that many Haskell developers favor.

Now that entry point to our functionality is nicely constrained to specification, how then do we implement the handler `calculateDiscount` (as seen in the above code) and how can we ever hope to encode the constraints on the discount logic in a way as precise and easy as Servant has allowed for the API's endpoint specification?

```haskell
foreign import ccall unsafe calculate_discount :: CInt -> CInt -> CInt

calculateDiscount :: Integer -> Integer -> Integer
calculateDiscount userId bookCount =
  fromIntegral $ calculate_discount (fromIntegral userId)
                                    (fromIntegral bookCount)
```

The foreign call hints that we intention to leave the elegant world of Haskell in order to claim the prize of implementing our constraints statically. We will be using a language called ATS which has been described as an advanced type system layered on top of the C programming language. ATS' primitive types mirror that of C's and therefore interfacing using Haskell's excellent FFI is as easy as delicious cake.

Update: Note that it is not a necessity to leave Haskell to achieve what follows. [LiquidHaskell](http://goto.ucsd.edu/~rjhala/liquid/haskell/blog/about/) offers a way to remain in Haskell and yet achieve the same guarantees found in the ATS code we develop here. Many thanks to Professor Ranjit Jhala for providing [an equivalent LiquidHaskell example](http://goto.ucsd.edu:8090/index.html#?demo=permalink%2F1449031696_5061.hs)!!!

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

ATS is designed to make the expression of propositions "programmer centric," and as a programmer I find the clean separation between its concepts of "statics" and "dynamics" refreshingly simple. The `datasort` declaration is a simple type declaration only it is in the static world. The `datatype` declaration (considered a dynamic construct in ATS) should look very familiar to any Haskell programmer with the only strangeness being that a datatype declaration may be parameterized by a sort thereby constructing a conduit between the two dimensions of statics and dynamics.

In the static world we construct properties. In the dynamic world we construct programs and manipulate properties to constrain our programs. The datatype `customer` in the example above has two data constructors both of which are singleton, that is, they are constrained to represent only one value, that value represented in the static world as `vip` and `regular` respectively. This means that when we write functions using these data constructions in the dynamic world their use can be constrained to the properties we have defined on their counterparts in the static world. Which, of course, means that any x-unit testing for behavior precluded by static constraints would be redundant and serve no purpose.

> 「それが黒いなら、どんな色も手にできる。」
>
> -- ヘンリー・フォード

```ats
dataprop CUSTOMER_GETS_DISCOUNT (customer, int, bool) =
  | {i:int | GT_BOOK_THRESHOLD i}  VIP_ENOUGH_BOOKS     (vip, i, true)
  | {i:int | LTE_BOOK_THRESHOLD i} VIP_NOT_ENOUGH_BOOKS (vip, i, false)
  | {i:int}                        REGULAR              (regular, i, false)
```

The dataprop declaration above describes a relationship between the sorts customer, int, and bool. The first case of our dataprop states that for all ints, as long as they are greater than a defined number of books, the relationship between the customer, those particular ints, and the bool should be that the customer is a vip, the int is greater than the threshold, and the bool equals true. This case is named `VIP_ENOUGH_BOOKS` to denote that it is the case where a vip customer is purchasing enough books to get a discount on his shipping cost.

The second constructor of the dataprop declares another valid relationship between the sorts customer, int, and bool. `CUSTOMER_GETS_DISCOUNT` allows the customer to be VIP, the int to be less than the threshold, and the bool (which represents whether or not the customer is entitled to a discount) to be false. This case is named `VIP_NOT_ENOUGH_BOOKS`.

Finally, the constructor `REGULAR` encodes the relationship "regular customers are not eligible for a shipping discount"; it doesn't matter how many books a regular customer intends to buy, they receive no discount regardless. We now have our constraints encoded at the type level; we only need to put them to use.

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

We use the dataprop `CUSTOMER_GETS_DISCOUNT` in the description of the predicate function `customer_gets_discount`. The type expresses that this function, for all customers and all ints, takes as arguments a value of type customer and a value of type int and returns a value of type bool. The type signature is simple enough yet it says something more and outside of what most strongly typed programmers would normally expect; it says that the post-condition relationship between the customer, int, and bool are confined to only those relationships we defined as part of the algebraic property `CUSTOMER_GETS_DISCOUNT`.

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

In the tradition of `type-driven development` we start with a type definition, here an alias, detailing what we want to have returned from our `calculate_discount'` function. The typedef discount says "for all ints 'book_count' we will return an int 'result' such that the result will be book_count minus our threshold thus multiplied by the discount percentage". That wasn't hard, and if you ask me it is pretty specific -- which is exactly what we want.

The type signature of `calculate_discount'` takes things a step further by facilitating the tracking of the static information asserting that its argument int is indeed a part of a relationship that entitles a discount in the first place. `calculate_discount'` will not accept an int that cannot be shown to be of the `CUSTOMER_GETS_DISCOUNT` relationship.

The implementation of `calculate_discount'` is of course a simple arithmetic routine that uses `g1int_mul2` which is a function that generates that static information needed to appease the `discount` constraint attached to our result type -- in other words, `g1int_mul2` not only multiplies its operands but returns evidence that the operations actually took place. We will need this evidence later.

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

`calculate_discount` brings it all together and is the function that is being called from Haskell (and by the bookstore demo above). We call `customer_type` to check the customer's type and then call `customer_gets_discount'` with the customer's type and his count of books. As stated above, the call to `customer_gets_discount'` returns both a bool denoting whether the customer gets a discount as well as the proof of the relationship backing the determination of its decision. If the customer is entitled, we will calculate the discount by supplying the proof given to us by `customer_gets_discount` and the count of books to `calculate_discount'`. This all works out because, like a guardian angel, there is an omnipresence checking that the relationship between our three pieces of important information; the customer type, his current book count, and his discount eligibility; remain valid throughout the code. That presence is an applied type system.

The [source](https://github.com/stackbuilders/high-spec) is available for anyone who wants to check how the project is built (by dynamic linking because it's easier than static when it comes to GHC and Cabal).

In conclusion, we have provided test coverage of the specifications we had been handed by both bookseller and the tech-lead. What is more is that we did so using only static analysis and without the use of a single run-time unit test. Thus, we are free to focus our run-time testing on system boundaries and higher-level user-facing behavior keeping our maintenance costs low. I hope for more in the direction of simplifying the use of static verification so that it becomes accessible to a wider audience and a larger number of development efforts can realize the tremendous savings made possible by such techniques. While it is likely that some form of run-time testing will always be of necessity, maybe in the near future minimizing run-time testing in favor of static analysis will be in fact... the way to go.
