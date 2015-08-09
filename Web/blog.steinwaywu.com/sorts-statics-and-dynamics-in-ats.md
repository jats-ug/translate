# ATSでの種、静的な世界、そして動的な世界

(元記事は http://blog.steinwaywu.com/sorts-statics-and-dynamics-in-ats/ です)

これは私の指導教官である Xi による興味深い話です。
そして、そのアイデアをここに適切に記そうと努力しようと思います。

## 型システムの要約

型システムとは型に関する体系です。
それらは**基本型**、**複合型**、そして**基本型を使って複合型を構築するための規則**を持っています。
また型システムには他に、いくつかの規則を使って項や式の型をどのように決定するかのような、本質的な理論があります。
Applied Type System を表わす ATS は型システムです。
それは独自の基本型、複合型を表現する規則、そして多くの関連する理論を持ちます。

型システムには異なる種類があります。
複合型を表現するために基本型を使い、さらなる複合型を表現するために複合型を使うかもしれません。
これは階層的な構造を持っています。
それらは**階層的な型システム**であると言えるのです。
その他の型システムは階層を持ちません。
これは Martin-Lof の型理論のような**循環した定義**を含んでいます。
ATS はこの階層的な型システムを選択しています。
(しかし、私の指導教官は彼のアイデアを、循環した定義の体系がいくつかの点でより先進的であると表現しました。)

ATS には型の基礎的なレベルと、その基礎の上に2つの型のレベルがあります。
それらはそれぞれ、**種 (sorts)**、**静的な世界 (statics)**、と**動的な世界 (dynamics)** です。
種は基礎的な型で、静的な世界を表現したりコンストラクトしたりします。
そして静的な世界は、動的な世界で使われる複合型をコンストラクトするために使われます。

## 静的な世界 (Statics)

静的な世界の前に、型の基礎的なレベルである種について説明しましょう。
ATS の論文では、種は次のように定義されています:

![](img/sorts-statics-and-dynamics-in-ats/1.png)

![](img/sorts-statics-and-dynamics-in-ats/3.png) は基礎種を表わし、それらは `bool`, `addr`, `type`, `t@ype` のようなそれ以上分解できない種です。
また ![](img/sorts-statics-and-dynamics-in-ats/2.png) も種として定義されています。
それらは関数を表わす種として考えることができます。
しかし現実の ATS の静的な世界では、それまだサポートされていません。
なぜなら実際にはラムダ抽象と関数適用は存在しないからです。
**これらの種は、静的な項を表現したりコンストラクトしたりするのに使われます。**
項はラムダ項を参照していて、これは後のブログエントリで説明しようと思います。
私の理解では、**項は静的な世界と動的な世界の中心部分です。***
それは他のプログラミング言語でも同様でしょう。
それはプログラミング言語において、変数、命令文、関数、関数呼び出しなどのような構造的な部品を表わします。
型付き言語 (少なくとも ATS) では、**それぞれの項には型が割り当てられます。**
特にそれぞれの静的な項には、いくつかの規則を使って、種が割り当てられます。
これが「静的な項を表現したりコンストラクトするために種を使う」という意味です。

**静的な世界は、複合の静的な項を表現するための言語として考えることができます。**
その構文は論文で良く説明されていて、最初の構文は上記に示した種の構文です。
残りは以下のようなものです:

* 静的な項: ![](img/sorts-statics-and-dynamics-in-ats/4.png)
* 静的な変数コンテキスト: ![](img/sorts-statics-and-dynamics-in-ats/5.png)
* シグニチャ: ![](img/sorts-statics-and-dynamics-in-ats/6.png)
* 静的な置換: ![](img/sorts-statics-and-dynamics-in-ats/7.png)

静的な項は、型付きラムダ計算におけるラムダ項によく似ています。
![](img/sorts-statics-and-dynamics-in-ats/8.png) は変数です。
![](img/sorts-statics-and-dynamics-in-ats/9.png) はラムダ抽象で、関数を表わします。
![](img/sorts-statics-and-dynamics-in-ats/10.png) は関数適用もしくは関数呼び出しです。
唯一の違いは ![](img/sorts-statics-and-dynamics-in-ats/11.png) です。
それは関数適用の特別な種類で、静的な定数 ![](img/sorts-statics-and-dynamics-in-ats/12.png) を静的な項 ![](img/sorts-statics-and-dynamics-in-ats/13.png) に適用します。

Static constants include static constant constructors like int(n), and static constant functions like + operator.
Or you can just call them static constant functions, because they are essentially the same.
Although they could both be classified as lambda abstractions or functions (or more precisely, predefined functions in ATS), ATS does so because of some practical implementation considerations.
If you try to encode the plus operator using pure typed lambda calculus, that could be a lot of work.
So ATS just extract them into a separate definition.
I don't know too much about it currently, but I may explain it later when I learn more.
Come back to static constants. Their definitions are described in ![](img/sorts-statics-and-dynamics-in-ats/14.png).
The first part ![](img/sorts-statics-and-dynamics-in-ats/15.png) contains  some predefined basic definitions.
And the second part is an expansion rule which expands ![](img/sorts-statics-and-dynamics-in-ats/16.png) to include a new signature ![](img/sorts-statics-and-dynamics-in-ats/17.png) of the "sort" ![](img/sorts-statics-and-dynamics-in-ats/18.png).
In order to avoid the conflict between the sort for static constants, and the sort defined previously, ATS calls "sc-sort" for the sort of static constants.
(There ARE lambda abstractions and function applications in ATS statics.
 You can define a sort in the form of ![](img/sorts-statics-and-dynamics-in-ats/19.png).
 And you can apply it, too.)
If you would like to gain some real feel, skip to the end for examples.

## 動的な世界 (Dynamics)

Dynamics are all about proofs and programs.
In a word, you use dynamics to construct a real program, while you use statics to describe dynamics, or to write specifications of dynamics.
Proofs are also programs, but they produce propositions to prove something.
Dynamics are very similar to statics.
Dynamic terms will be assigned types.
Those types are static terms of sort type(or t@ype).
There is also a formal definition for dynamic terms, dynamic values, variable context, and substitution.
But it is too complex for me now.
I will cover that later on.
Not that there is a "dynamic values".
As far as I know, they are dynamic terms that couldn't be further simplified, like a termination token in a programming language grammar.

## 例

```ats
datasort mylist =  
 | nil of ()
 | cons of (int, mylist)
stadef list = cons (1, nil())
```

In this statics example, we defined a sort called mylist, which is part of the base sort ![](img/sorts-statics-and-dynamics-in-ats/20.png) in the paper.
The consis a static constant of sc-sort ![](img/sorts-statics-and-dynamics-in-ats/21.png).
The nil() and cons (1, nil()) are static constant applications, which are defined to be static terms, of sort mylist.
The listis a static term, of sort mylist.
Note that, the static constant should be of form ![](img/sorts-statics-and-dynamics-in-ats/22.png) instead of ![](img/sorts-statics-and-dynamics-in-ats/23.png).

```ats
fun myplus (a:int, b:int):int
```

In this dynamics example, we defined a function myplus, which is a dynamic term, of type ![](img/sorts-statics-and-dynamics-in-ats/24.png).
The type ![](img/sorts-statics-and-dynamics-in-ats/24.png) is a static term of sort type.
This static term is of the form of static constant application with the static constant being the ![](img/sorts-statics-and-dynamics-in-ats/25.png) symbol.
It is a static constant who takes static terms (of sort type) and produces static terms (of sort type).
The ![](img/sorts-statics-and-dynamics-in-ats/25.png) is defined as a member of ![](img/sorts-statics-and-dynamics-in-ats/26.png) in the statics syntax.
論文を参照してください。

## 参考文献

ATS の論文は [ここ](http://www.ats-lang.org/MYDATA/ATS-types03.pdf) から入手できます。
