# BLUISH CODER: 安全なプログラミング言語を使って heartbleed を防ぐには

(元記事は http://bluishcoder.co.nz/2014/04/11/preventing-heartbleed-bugs-with-safe-languages.html です)

OpenSSL の [heartbleed バグ](http://heartbleed.com/) はインターネット全体に多大なダメージを与えました。
このバグはとても単純で、C言語のような安全でない言語を使ったプログラミングがなぜ問題になるのかを示す教科書的な例です。

より安全なシステムプログラミング言語がこのバグを防止できるかどうかの実験として、
今回の問題の関数を [ATS プログラミング言語](http://www.ats-lang.org/) を用いて書き直してみます。
私は以前、[より安全なC言語として ATS について紹介しました](http://bluishcoder.co.nz/tags/ats)。
この記事では現実のテストケースを取り扱います。
ここでは ATS2 と呼ばれる ATS の最新版を使っています。

ATS はC言語コードにコンパイルします。
生成された関数インターフェイスは既存のC言語関数に厳密にマッチし、さらにC言語から呼び出すこともできます。
OpenSSL の dtls1_process_heartbeat と tls1_process_heartbeat 関数を ATS 言語を使って置換するために、私はこの機能を使いました。
これら2つの関数は heartbleed バグを修正するためにパッチがあてれた箇所です。

私が取ったアプローチは
[ATS メーリングリストでの John Skaller による次の発言](http://sourceforge.net/p/ats-lang/mailman/message/32204291/)
とよく似たものです:

> ATS は基本的により良い型システムを持つC言語だ。恐しい依存型なしに、かなり低レベルなC言語コードを書くとしよう。そしてC言語のコードが手に入る。もし君がミスをしていたらクラッシュするだろう。
>
> もし高レベルな型システムを使えば、コーディングにはより多くの作業とより多くの考慮が要求されることになる。でも君はプログラムの正確さについてより強い保証が得られるんだ。それも OCaml や Haskell を使うよりもさらに強い保証だ。さらに実行時の検査を省略することで、君はC言語より *より良い* 効率を望むことだってできる。これらは型システムによる正確さの証明によっている。クリティカルなソフトウェアにおいて、君のコードの 50% 以上はそのような証明だと思う。ひょっとすると君の能力の 90% はアルゴリズムを実装するのではなく、そのような証明できることを構築することに費やされているかもしれない。これはパラダイムシフトだよ。

私はまずC言語コードを直接ラップして、ATS から呼び出すことからはじめました。
それからC言語コードを安全でない ATS コードに置換しました。
それが終わったら、エラーを見つけるために型を付けました。

修正した OpenSSL コードは [github](https://github.com/doublec/openssl/branches)
に置いてあります。
そこには2つのブランチ、ats と ats_safe があり、これから示す ATS における関数実装の2つの段階を表わします。

私が取った異なる道筋について概要を示した後
今回のエラーを発見するためにどのように ATS を使ったのか詳細を説明します。

## C言語コードをラップする

[以前このアプローチについて紹介](http://bluishcoder.co.nz/2011/04/24/converting-c-programs-to-ats.html)しました。
ATS には直接C言語を埋め込めます。
そこで、手始めに dtls1_process_heartbeat のC言語コードを ATS のファイルに組み込みました。
そして ATS 関数からそれを呼び出すようにして、その ATS 関数を真の dtls1_process_heartbeat として公開します。
このコードは私の最初の試みで、[d1_both.dats](https://github.com/doublec/openssl/blob/f40838fd8b2c4ba907865eef54e5cca96dc0c62f/ssl/d1_both.dats) にあります。

## 安全でない ATS

2番目の段階は ATS を使って当該の関数を書き直すことでした。
しかしこの書き直した関数は安全ではありません。
このコードは ATS の機能である追加の型検査なしに、直接C言語コードに翻訳されます。
これは ATS コードを使っています。
[書き換えられた d1_both.dats](https://github.com/doublec/openssl/blob/ats/ssl/d1_both.dats)
はこのバージョンのコードを含んでいます。

このコードはまったく醜いものですが、コンパイルでき、C言語の実装に一致しています。
テスト対象にインストールすると、やはり heartbleed バグは再現します。
このコードはポインタ演算を使っており、またC言語コードのオフセットがハードコードされています。
以下にこの関数の一部を抜き出してみます:

```ocaml
val buffer = OPENSSL_malloc(1 + 2 + $UN.cast2int(payload) + padding)
val bp = buffer

val () = $UN.ptr0_set<uchar> (bp, TLS1_HB_RESPONSE)
val bp = ptr0_succ<uchar> (bp)
val bp = s2n (payload, bp)
val () = unsafe_memcpy (bp, pl, payload)
val bp = ptr_add (bp, payload)
val () = RAND_pseudo_bytes (bp, padding)
val r = dtls1_write_bytes (s, TLS1_RT_HEARTBEAT, buffer, 3 + $UN.cast2int(payload) + padding)
val () = if r >=0 && ptr_isnot_null (get_msg_callback (s)) then
           call_msg_callback (get_msg_callback (s),
                              1, get_version (s), TLS1_RT_HEARTBEAT,
                              buffer, $UN.cast2uint (3 + $UN.cast2int(payload) + padding), s,
                              get_msg_callback_arg (s))
val () = OPENSSL_free (buffer)
```

このコードとC言語コードを比較することは容易でしょう。

## より安全な ATS

3番目の段階では、ポインタ演算が妥当であることと境界エラーがないことを検査するために、安全でない ATS コードに型を付けました。
もし境界チェックがアサートされた場合、
[このバージョンの d1_both.dats](https://github.com/doublec/openssl/blob/12b89f1b2d714835b9257c10bcc5fd210714d07d/ssl/d1_both.dats)
はコンパイルに失敗します。
もし行番号 [123](https://github.com/doublec/openssl/blob/12b89f1b2d714835b9257c10bcc5fd210714d07d/ssl/d1_both.dats#l123),
[178](https://github.com/doublec/openssl/blob/12b89f1b2d714835b9257c10bcc5fd210714d07d/ssl/d1_both.dats#l178),
[193](https://github.com/doublec/openssl/blob/12b89f1b2d714835b9257c10bcc5fd210714d07d/ssl/d1_both.dats#l193)
の assertloc を削除すると、制約エラーが発生します。
このエラーは heartbleed バグを効果的に防止しています。

## テスト可能なバージョン

最後の段階で私が行なったことは、tls1_process_heartbeat
に対する修正を実装したことと、共用できるように補助ルーチンのいくつかを取り除いたことです。
このバージョンは [ats_safe ブランチ](https://github.com/doublec/openssl/tree/ats_safe)
にあり、新しい修正がほどこしてあります。
このバージョンでは assertloc は使わず、実地でテストできるように不具合を修正しています。

私はこのバージョンの OpenSSL をテストして、heartbleed
テストプログラムがメモリダンプに失敗することを確かめました。

## 安全へのアプローチ

tls_process_heartbeat 関数は送信元によって与えられたデターへのポインタと、OpenSSL 内部の構造体からそのデータの総量を得ます。
そのデータは次のようなフォーマットであると期待しています:

```
 byte = hbtype
 ushort = payload length
 byte[n] = bytes of length 'payload length'
 byte[16]= padding
```

既存のC言語コードは、送信者から供給された 'payload length' を信頼して、それを memcpy に渡たすというミスを犯しています。
もしデータの実際の長さが 'payload length' よりも小さければ、
その応答にメモリ中のランダムなデータを送ってしまうことになります。

ATS では、ポインタを自由に操作することができます。
しかしスコープにそのメモリアドレスに配置されたことを立証する view がない場合には、
それらはデリファレンスされたり使われたりすることはありません。
view と view のサブセットをたらい回しすることによって、ATS
コードがアクセスすべきでないメモリにアクセスしていないことを検査することが可能になるのです。
view はケーパビリティのようなものです。
そのメモリに安全にアクセスするケーパビリティを持つコードが欲しい時にそれらを渡し、
アクセスが完了した時にそれを取り戻します。

## View

To model what the C code does I created an ATS view that represents the layout of the data in memory:

```ocaml
dataview record_data_v (addr, int) =
  | {l:agz} {n:nat | n > 16 + 2 + 1} make_record_data_v (l, n) of (ptr l, size_t n)
  | record_data_v_fail (null, 0) of ()
```

A ‘view’ is like a standard ML datatype but exists at type checking time only.
It is erased in the final version of the program so has no runtime overhead.
This view has two constructors.
The first is for data held at a memory address l of length n.
The length is constrained to be greater than 16 + 2 + 1 which is the size of the ‘byte’, ‘ushort’ and ‘padding’ mentioned previously.
By putting the constraint here we immediately force anyone creating this view to check the length they pass in.
The second constructor, record_data_v_fail, is for the case of an invalid record buffer.

The function that creates this view looks like:

```ocaml
implement get_record (s) =
  val len = get_record_length (s)
  val data = get_record_data (s)
in
  if len > 16 + 2 + 1 then
    (make_record_data_v (data, len) | data, len)
  else
    (record_data_v_fail () | null_ptr1 (), i2sz 0)
end
```

Here the len and data are obtained from the SSL structure.
The length is checked and the view is created and returned along with the pointer to the data and the length.
If the length check is removed there is a compile error due to the constraint we placed earlier on make_record_data_v.
Calling code looks like:

```ocaml
val (pf_data | p_data, data_len) = get_record (s)
```

p_data is a pointer.
data_len is an unsigned value and pf_data is our view.
In my code the pf_ suffix denotes a proof of some sort (in this case the view) and p_ denotes a pointer.

xxx
