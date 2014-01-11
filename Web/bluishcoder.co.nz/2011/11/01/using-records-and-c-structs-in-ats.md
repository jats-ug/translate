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

この例では typedef を使ってレコードへの型の別名を作っています。
そのため point という名前で型を参照できます。
add1関数は & 接頭辞の付いた参照であるpointを取ります。
これはC++の参照引数のように振舞います。
実際にはこの関数は構造体のインスタンスへのポインタを取り、
関数はそのインスタンスを変更することができます。
ここでは関数に渡されるpointは左辺値(l-value)です。
すなわちミュータブルであるということになります。
これはATSにおいて、varとvalのどちらを使うかによって選択されます。

未初期化のpointオブジェクトを作り、addr1に渡すと型エラーになることに注意してください。
例えば次のコードは型検査エラーになります。

```ocaml
implement main () = {
  var p1: point?
  val () = add1 (p1)
  val () = print_point (p1)
}
```

未初期化である型には ? という接尾辞が付きます。
p1は未初期化であるべきなので、その型は point? です。
add1がpointを取ると型エラーになるわけです。
初期化を行なえば渡すことができるようになります。

```ocaml
implement main () = {
  var p1: point
  val () = p1.x := 5
  val () = p1.y := 10
  val () = add1 (p1)
  val () = print_point (p1)
}
```

関数に参照を渡すのと同様に、ポインタを渡してポインタを直接取り扱うこともできます。
これには証明器を使うことが要求されますが、ここではポインタの取り扱いに進むことにしましょう。
この証明と共に扱う方法は他の記事で解説しようと思います。

```ocaml
fun add1 {l:agz} (pf: !point @ l | p: ptr l): void = {
  val () = p->x := p->x + 1
  val () = p->y := p->y + 1
}

implement main () = {
  var p1 = @{x=10, y=20}
  val () = add1 (view@ p1 | &p1)
  val () = print_point (p1)
}
```

C APIとのインターフェイスにおいては、しばしばC言語の構造体を扱わなければなりません。
ATSでは、ATSのレコードと一致しながらC言語の構造体として振る舞う型を宣言できます。
つまりATSは構造体を読み書きできるのです。
次の例ではC言語で構造体を宣言し、C言語でそれを使う関数を作り、
そしてそれらをどのようにATSでラップするかを示しています。

```ocaml
%{^
typedef struct Point {
  int x;
  int y;
} Point;

void print_point (Point* p) {
  printf("%d@%d\n", p->x, p->y);
}
%}

typedef point = $extype_struct "Point" of {x= int, y= int}
extern fun print_point (p: &point): void = "mac#print_point"

implement main () = {
  var p1: point
  val () = p1.x := 10;
  val () = p1.y := 20;
  val () = print_point (p1)
}
```

$extype_struct キーワードは、与えられた名前のC言語の構造体によって表わされる型を作ります。
of {x= int, y= int} という接尾辞を使うことで、ATSから見たレコードのレイアウトを宣言しています。
これはATSに独自の構造を持つ型を作らせる代わりに、C言語の構造体を使わせます。
main関数からをコンパイルすると次のようなC言語コードが生成されます。

```c
ATSlocal (Point, tmp1) ;

__ats_lab_mainats:
/* Point tmp1 ; */
ats_select_mac(tmp1, x) = 10 ;
ats_select_mac(tmp1, y) = 20 ;
print_point ((&tmp1)) ;
```

これはC言語の構造体の型を直接使っていることに注意してください。

私は zmq_msg_t と zmq_pollitem_t 構造体をラップするラッパーライブラリ
[0MQ](https://github.com/doublec/ats-libzmq)
にこのアプローチを使いました。
