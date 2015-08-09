# 自由変数、関数、そしてクロージャ

(元記事は http://blog.steinwaywu.com/free-variables-functions-and-closures/ です)

(これはボストン大学 CS320 コース用の記事です)

環境、クロージャ、関数について議論するために、はじめに自由変数と束縛変数について議論しようと思います。

## 自由変数と束縛変数

次の例を見てみましょう。

```ats
lam (x) => x + y
```

ここでは上記の `x`, `x`, `y` を (`x` と `y` の) **出現 (occurrence)**と呼びます。
最初の `x` は `x` の**束縛する出現 (binding occurrence)**と呼ばれ、二番目の `x` は**束縛される出現 (bound occurrence)**と呼ばれ、そして `y` は**自由な出現 (free occurrence)**です。

引数リスト中の `x` はプレースホルダーのための名前を宣言していて、本体中の `x` はプレースホルダーになります。
そのような無名関数がパラメータと共に呼び出されると、全てのプレースホルダー `x` は実際のパラメータ値で置換 (もしくは束縛) されます。
けれども、`y` が引数リスト中で定義されていないので、それはプレースホルダーとして扱われません。
この無名関数を呼び出しても、`y` は具体的な値を持っていません。
それは束縛されておらず、自由なのです。

```ats
fun add (x:int, y:int): int = x + y
```

ここでは、本体中の全ての変数の出現は束縛されています。
引数リスト中のそれらの出現は束縛する出現なのです。

```ats
fun addclo (x:int):<cloref1> int = x + y
```

ここでは、本体中の `y` は自由な出現です。

## 環境と評価

評価器がプログラムを評価するとき、変数の値を知る必要があります。
パラメータ `123` を共なって `addclo` を呼び出すことを考えてみましょう。

1. Interpreter sees the binding occurrence of `x`, therefore it binds `123` onto `x`, which is actually putting a key/value pair `(x,123)` into the environment.
2. Interpreter then evaluates the body. When it sees the bound occurence of `x`, it looks up the environment, and replace it with `123`.
3. When it sees the free occurence of `y`, or we say it can't find a binding of `y` in the current environment, it will take further actions.
    * It either reports an error, and stops evaluating.
    * Or continues to look up `y` in a parent scope (if `y` is defined somewhere outside this function).

We can see that a free variable may appear locally, but in a correct program, all variables have to be bound eventually.

## 関数とクロージャ

In a function, every local variable has to be bound.
This means every variable should have at least a binding occurence (either in the argument list, or in the body), and probably several bound occurences.

Closure is actually a function, **plus an environment**.
The current environment is recorded by the interpreter when it first sees the definition of a closure.
Later when it starts to evaluate the application of a closure, it will retrieve the original environment, and evaluate the closure application under it.
We need the environment, and there is a reason.

In a closure definition, variables are not required to be bound (in its defining scope).
But as we mentioned, it has to be bound somewhere else in order to be evaluated. Look at this.

```ats
implement main0 () = let
    val y = 1
    fun addclo (x:int):<cloref1> int = x + y
    val z = addclo(123)
in
    ()
end
```

Here, when the interpreter sees the definition main,

1. Bind `1` onto `y` (put `(y,1)` into the environment).
2. Record the definition of `addclo`. That is to put the function/environemnt pair into a table. The environment contains `(y,1)`.
3. It sees a closure application.
    1. Retrieve `(addclo, env)` from the table
    2. `x` is a bound variable, it is bound to `123` because of the binding occurrence of `x` in the argument list.
    3. `y` **is bound to** `1` **according to the** `env`**, which is the environment when we define** `addclo`**.**
    4. Return `124` (123 + 1), and bind it onto `z`.
4. ...

As you can see, if the closure is not accompanied by its environment, we will never know the value of `y`.
However, since all variables are bound in a function, it does not need to be accompanied by an environment.
