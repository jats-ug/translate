# 自由変数、関数、そしてクロージャ

(元記事は http://blog.steinwaywu.com/free-variables-functions-and-closures/ です)

(これはボストン大学 CS320 コース用の記事です)

In order to discuss environments, closures and functions, I would like to discuss free and bound variables first.

## Free/Bound Variables

Take this as an example

```ats
lam (x) => x + y
```

Here we call `x`, `x`, and `y` **occurrences** (of `x` and `y`).
The first `x` is called a **binding occurence** of `x`, the second `x` is called a **bound occurence** of `x`, and the `y` is a **free occurence**.

The `x` in the argument list is like to decalre a name for the placeholder, and then the `x` in the body will become a placeholder.
Whenever such anonymous function get called with a parameter, all the placeholder `x` will be replaced (or actually, bound) by the actual paremeter value.
However, the `y` is not defined in the argument list, thus it is not treated as a placeholder.
However I call this anonymous function, `y` won't have a concrete value. It is not bound, it's free.

```ats
fun add (x:int, y:int): int = x + y
```

Here, all the occurrences of variables in the body are bound, while those occurrences in the argument list are binding occurrences.

```ats
fun addclo (x:int):<cloref1> int = x + y
```

While here the `y` in the body is a free occurrence.

## Environments and Evaluation

When the interpreter evaluates a program, it needs to know the values of variables.
Suppose I call addclo with parameter 123, then

1. Interpreter sees the binding occurrence of `x`, therefore it binds `123` onto `x`, which is actually putting a key/value pair `(x,123)` into the environment.
2. Interpreter then evaluates the body. When it sees the bound occurence of `x`, it looks up the environment, and replace it with `123`.
3. When it sees the free occurence of `y`, or we say it can't find a binding of `y` in the current environment, it will take further actions.
    * It either reports an error, and stops evaluating.
    * Or continues to look up `y` in a parent scope (if `y` is defined somewhere outside this function).

We can see that a free variable may appear locally, but in a correct program, all variables have to be bound eventually.

## Functions and Closures

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
