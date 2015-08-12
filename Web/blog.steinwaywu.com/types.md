# 型について

(元記事は http://blog.steinwaywu.com/types/ です)

(これはボストン大学 CS320 コース用の記事です)

A type is a category of things having common characteristics.
Putting it into our context, it is a **classification** identifying one of various types of data that usually determines
([参考](https://ja.wikipedia.org/wiki/%E3%83%87%E3%83%BC%E3%82%BF%E5%9E%8B))

* the **possible values** for that type;
* the operations that can be done on values of that type;
* the meaning of the data;
* and the way values of that type can be stored.

Or, you can think of a type as a value space, a **set** of all possible values.

* 整数型: `{... -2, -1, 0, 1, 2, ...}`
* 真理値型: `{true, false}`
* 文字型: `{a, b, c, ...}`

## プリミティブ型/複合型

Usually, we call them primitive type: `int`, `bool`, `string`, `float`, etc. They are building blocks for composite types.
Composite types are types formed by combining other types.

## 代数的データ型

ADT is a kind of composite type.
Depending on how the value space is determined by its composing types, we have **product types** and **sum types**.
([参考](https://ja.wikipedia.org/wiki/%E4%BB%A3%E6%95%B0%E7%9A%84%E3%83%87%E3%83%BC%E3%82%BF%E5%9E%8B))

### 直積型

If P is a product type formed by A and B, then the value space of P is the **cartisian product** of the value space of A and B.

Suppose we have a pair of type `(bool, int)`, then it's value space should be `{..., (T -1), (F -1), (T 0), (F 0), (T 1), (F 1), ...}`.
It is easily seen that this is simply a cartisian product of `{T, F}` and `{..., -1, 0, 1, ...}`.

These composing types are usually called **fields**.
The above type contains two fields: the first is a bool field, the second is an int field.
Fields may or may not have names/labels.

Examples of product types are: pairs, tuples, records, etc.

### 直和型

If S is a sum type formed by A and B, then the value space of S is the **union** of the value space of A and B.

For a sum type, each of its composing types is called a **variant**, and each variant is identified by a **constructor**.
Constructors may or may not **carry data** of any type.

Suppose we have a `intlist` type:

```ats
datatype intlist = 
    | list_cons of (int, intlist)
    | list_nil of ()
```

The integer list type has two variants, identified by its two constructors: `list_cons` and `list_nil`.

* `list_cons` variant correspond to the set of all non-empty integer lists.
* `list_nil` variant correspond to the set of empty integer list.

And we also see, `list_cons` is carrying two pieces of data: an integer (representing the current/head list node), and an integer list (representing the rest of the list).
But `list_nil` is carrying nothing.

## 型の値を定義する

For primitive types and product types, just write down the literal value of that type (which should be a member of the value space set).

```ats
val x = 3
val y = true
val z1 = (x, y)  // an unboxed/flat tuple
val z2 = '(x, y) // a boxed tuple

typedef point = @{x = double, y = double} // an unboxed record
val p = @{x = 0.0, y = 0.0 } : point 

typedef student = `{name = string, id = int} // a boxed record
```

For sum types, we should instead **use the constructor**.

```ats
val l = list_nil ()
val ls = list_cons (1, list_nil ())

// 1 -> (2 -> (3 -> nil))
val lss = list_cons (1, list_cons (2, list_cons (3, list_nil ()))
```

After constructing sum type values, we can use **pattern matching** to **retrieve** the data we previously put into the constructor.

```ats
case+ lss of
    | list_nil => println! ("Empty list!")
    | list_cons (x, y) => println! (x)
```

Because constructors can identify different variants of a sum type, it is perfect for pattern matching, which will help us interpret the ADT value, and reverse the process of constructing to get the data inside it.
Here, `x` is matched with `1`, the head of the list.
`y` is matched with `list_cons (2, list_cons (3, list_nil ()))`, the rest of the list.

However, this is only a list of integers.
It can't hold other types of data.
We will solve this by polymorphic types later.
