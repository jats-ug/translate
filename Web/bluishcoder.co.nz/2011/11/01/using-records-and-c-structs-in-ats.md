# BLUISH CODER: ATSでレコードとC言語の構造体を使う

(元記事は http://bluishcoder.co.nz/2011/11/01/using-records-and-c-structs-in-ats.html です)

[ATS](http://www.ats-lang.org/) はレコード型を持っています。
これは名前によってそれぞれの要素を参照する点を除いてタプルに似ています。
これらはC言語の構造体とても近く、生成されるC言語コードはこの方法で表現されます。
次のコードは二次元平面上の点を表現するxとyの値を保持するレコードを使った例です。

```ocaml
fun print_point (p: @{x= int, y= int}): void =
  printf("%d@%d\n", @(p.x, p.y))

implement main () = {
  val p1 = @{x=10, y=20}
  val () = print_point (p1)
}
```

この例でのレコードオブジェクトのリテラルは @{ ... } という構文を使って生成されています。
レコードのフィールドをデリファレンスするには"."を使います。
レコードの表現がC言語では構造体であることを生成されたC言語コードは示しています。

```c
typedef struct {
  ats_int_type atslab_x ;
  ats_int_type atslab_y ;
} anairiats_rec_0 ;
```

@ 構文はレコードのリテラルとして使われて、
型はレコードが"フラット"レコードであることを示します。
フレットレコードは値の意味論をもっており、
この型の値は根底となるC言語の構造体のサイズに等しいバイトサイズを持っています。
上記の上のmain関数から生成されたC言語コードを次に示します。

```c
ATSlocal (anairiats_rec_0, tmp4) ;

__ats_lab_mainats:
tmp4.atslab_x = 10 ;
tmp4.atslab_y = 20 ;

/* tmp3 = */ print_point_0 (tmp4) ;
```

ここでのtmp4変数は先のATSコードにおけるp1に相当します。
これはレコードを表現するC言語の構造体のインスタンスで、スタックに確保されます。
初期化された後、print_point関数に値で渡されます。

```c
ats_void_type
print_point_0 (anairiats_rec_0 arg0) {
  ATSlocal (ats_int_type, tmp1) ;
  ATSlocal (ats_int_type, tmp2) ;

  ...
}
```

レコードは '{...} という構文を使っても定義できます。
@ の代わりに ' を使っていることに注意してください。
このリテラルはヒープに確保されGCによってメモリ管理されるボックス化レコードです。
ボックス化レコードはポインタの意味論を持っていて、
ポインタのサイズと同じバイトサイズを持っています。

```ocaml
implement main () = {
  val x = sizeof<@{x=int}>
  val y = sizeof<'{x=int}>
  val a = int_of_size x
  val b = int_of_size y
  val () = printf ("%d %d\n", @(a, b))
}
```

このコードは"4 8"を印字します
これはフラット型はintのサイズを持っており、
ボックス化型はポインタのサイズを持っていることを示しています。
私がこれまでのATSにおけるレコード経験では、
GCの使用を避けるためにフラットレコードを多用します。

フラットレコードへの参照を渡すために、
当該の関数の引数に & を使って"参照渡し"のマークをつけます。

```ocaml
typedef point = @{x= int, y= int}

fun print_point (p: point): void =
  printf("%d@%d\n", @(p.x, p.y))

fun add1 (p: &point): void = {
  val () = p.x := p.x + 1
  val () = p.y := p.y + 1
}

implement main () = {
  var p1 = @{x=10, y=20}
  val () = add1 (p1)
  val () = print_point (p1)
}
```

xxx
