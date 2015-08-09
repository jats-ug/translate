# 型についてさらに

(元記事は http://blog.steinwaywu.com/more-on-types/ です)

(これはボストン大学 CS320 コース用の記事です)

Recall that a type can be seen as a value space.
We also mentioned algebric data type, specifically sum types.

## Abstract Types

In ATS, we encourage programmers to think in an abstract way, to program towards your goal, instead of paying great attention to details at the very beginning.
Therefore ATS provides **abstract types**.

```ats
abstype money
```

The above code introduces a type.
It has a name `mytype`, but it has no concrete value space associated to it.
You can now use the type to write your interface,

```ats
fun get_cent (money): int
fun get_dollar (money): int
fun make_money (dollar: int, cent: int): money

fun money_to_cent (money): int
fun cent_to_money (int): money
```

and use the interface to write the program.

```ats
implement money_to_cent (m) = get_dollar (m) * 100 + get_cent (m)
implement cent_to_money (c) = make_money (c / 100, c mod 100)
```

As you can see, the detail of `money` is **completely unknown**, so are the first three interfaces.
But you can still write programs like the implemtations of the last two interfaces.

However, the implementation of the first three functions, have to know the details of `money`.

```ats
datatype money_type = M of (int, int)
assume money = money_type
```

Here, we define a sum type `money_type` with only one varient `M`.
This is a concrete type.
We then use `assume` to associate the value space of `money_type` to `money` to make it concrete.
From now on, we can implement those functions.

```ats
implement get_cent (m) = c where M(_, c) = m
implement get_dollar (m) = d where M(d, _) = m
implement make_money (d, c) = M(d, c)
```

The benefit here is that you can **change your implementation** as you want.
You just need to `assume` it to another concrete type, and implement these three functions again based on the new associated value space.

## Exercise

Suppose we have an integer arithmatic expression to be evaluated. Please write your implementation based on abstract types and interfaces.

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

And based on the following code

```ats
datatype exp_type = 
    | num of (int)
    | add of (exp_type, exp_type)
    | sub of (exp_type, exp_type)
    | par of (exp_type)
assume exp = exp_type
```

write the implementations for the first five functions.

