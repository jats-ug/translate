# 型についてさらに

(元記事は http://blog.steinwaywu.com/more-on-types/ です)

(これはボストン大学 CS320 コース用の記事です)

型を値の空間として考えることができたことを思い出しましょう。
また代数的データ型、特に直和型についても学びました。

## 抽象型

ATS では、設計の最初期に詳細について注意を払う代わりに、ゴールに向かってプログラムを進める際に抽象的な方法で考えることをプログラマに促しています。
そのために ATS は**抽象型**を提供しています。

```ats
abstype money
```

上記のコードは型を導入しています。
それは名前 `mytype` を持っていますが、それに関連した具体的な値の空間を持っていません。
この型を使って次のようにインターフェイスを書くことができます:

```ats
fun get_cent (money): int
fun get_dollar (money): int
fun make_money (dollar: int, cent: int): money

fun money_to_cent (money): int
fun cent_to_money (int): money
```

そしてこのインターフェイスを使ってプログラムを書くことができます。

```ats
implement money_to_cent (m) = get_dollar (m) * 100 + get_cent (m)
implement cent_to_money (c) = make_money (c / 100, c mod 100)
```

想像通り、`money` の詳細は**完全に未知**で、最初の3つのインターフェイスについても同様です。
しかし最後の2つのインターフェイスの実装のようなプログラムを書き下すこともできます。

けれども、最初の3つの関数の実装は `money` の詳細について知らなければなりません。

```ats
datatype money_type = M of (int, int)
assume money = money_type
```

ここで、唯一のバリアント `M` を持つ直和型 `money_type` 定義しました。
これは具体的な型です。
それから具体化するために、`assume` を使って `money_type` 値の空間を `money` に関連付けます。
これで、それらの関数を実装することができます。

```ats
implement get_cent (m) = c where M(_, c) = m
implement get_dollar (m) = d where M(d, _) = m
implement make_money (d, c) = M(d, c)
```

この恩恵は、望むときに**実装を変更**できることです。
他の具体的な型に変更したい時は `assume` を使い、新しい関連した値の空間に基づいてこれら3つの関数を再度実装するだけです。

## 練習問題

評価したい整数の代数式があると仮定します。
次の抽象型とインターフェイスに基づいてあなたの実装を書いてください。

```ats
abstype exp

fun get_num (exp): int
fun is_num (exp): bool
fun is_add (exp): bool
fun is_sub (exp): bool
fun fst (exp): exp      // e.g. (pseudo) fst ("1+2") = "1"
fun snd (exp): exp      // e.g. (pseudo) snd ("1+2") = "2"

fun eval (exp): int     // please implement this
```

また次のコードに基づき...

```ats
datatype exp_type =
    | num of (int)
    | add of (exp_type, exp_type)
    | sub of (exp_type, exp_type)
    | par of (exp_type)
assume exp = exp_type
```

最初の5つの関数の実装を書いてください。

