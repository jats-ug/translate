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

tls_process_heartbeat 関数は送信元によって与えられたデータへのポインタと、OpenSSL 内部の構造体からそのデータの総量を得ます。
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

今回のC言語コードが行なうことをモデル化するために
私はメモリ中のデータレイアウトを表現する ATS の view を作りました:

```ocaml
dataview record_data_v (addr, int) =
  | {l:agz} {n:nat | n > 16 + 2 + 1} make_record_data_v (l, n) of (ptr l, size_t n)
  | record_data_v_fail (null, 0) of ()
```

'view' は standard ML の datatype に似ていますが、型検査時にのみ存在します。
これは最終的な実行プログラムでは削除されていて、そのため実行時のオーバーヘッドはありません。
今回の view は2つのコンスラクタを持っています。
1番目のコンストラクタは長さ n のメモリアドレス l に保持されたデータを表わします。
その長さは 16 + 2 + 1 よりも多くなるように強制されています。
これは前に説明した 'byte', 'ushort', 'padding' を足したサイズです。
ここで制約することで、この view を生成するコードに渡された長さを検査することを強制することができます。
2番目のコンストラクタ、record_data_v_fail は無効なレコードバッファの場合を表わします。

この view を生成する関数は次のようになります:

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

ここでは、len と data は SSL 構造体から得ています。
この長さは検査され、そして view が生成されて、データへのポインタと長さをともなって返ります。
もし長さの検査を削除すると、make_record_data_v に設定した強制のためにコンパイルエラーになります。
呼び出し側コードは次のようになります:

```ocaml
val (pf_data | p_data, data_len) = get_record (s)
```

p_data はポインタです。
data_len は符号なしの値、pf_data は view です。
私のコードでは、接尾辞 pf_ はなにかの種 (この例では view) の証明を示し、
p_ はポインタを示します。

## 証明関数

ATS では、ポインタに対してインクリメント、デクリメント、たらい回す、以外にはどのような操作もすることができません。
デクリメントするためには、そのメモリアドレスを指すことを立証する view を獲得しなければなりません。
私達が欲しいデータに特化した特殊な view を得るために、
record_data_v view からメモリアクセスのための view への変換を行なう証明関数をいくつか作りました。
以下にその証明関数を示します:

```ocaml
(* These proof functions extract proofs out of the record_data_v
   to allow access to the data stored in the record. The constants
   for the size of the padding, payload buffer, etc are checked
   within the proofs so that functions that manipulate memory
   are checked that they remain within the correct bounds and
   use the appropriate pointer values
*)
prfun extract_data_proof {l:agz} {n:nat}
               (pf: record_data_v (l, n)):
               (array_v (byte, l, n),
                array_v (byte, l, n) -<lin,prf> record_data_v (l,n))
prfun extract_hbtype_proof {l:agz} {n:nat}
               (pf: record_data_v (l, n)):
               (byte @ l, byte @ l -<lin,prf> record_data_v (l,n))
prfun extract_payload_length_proof {l:agz} {n:nat}
               (pf: record_data_v (l, n)):
               (array_v (byte, l+1, 2),
                array_v (byte, l+1, 2) -<lin,prf> record_data_v (l,n))
prfun extract_payload_data_proof {l:agz} {n:nat}
               (pf: record_data_v (l, n)):
               (array_v (byte, l+1+2, n-16-2-1),
                array_v (byte, l+1+2, n-16-2-1) -<lin,prf> record_data_v (l,n))
prfun extract_padding_proof {l:agz} {n:nat} {n2:nat | n2 <= n - 16 - 2 - 1}
               (pf: record_data_v (l, n), payload_length: size_t n2):
               (array_v (byte, l + n2 + 1 + 2, 16),
                array_v (byte, l + n2 + 1 + 2, 16) -<lin, prf> record_data_v (l, n))
```

証明関数は型検査時のみ実行されます。
それらは証明を扱います。
extract_hbtype_proof 関数が行なっていることを詳しく見てみましょう:

```ocaml
prfun extract_hbtype_proof {l:agz} {n:nat}
               (pf: record_data_v (l, n)):
               (byte @ l, byte @ l -<lin,prf> record_data_v (l,n))
```

この関数は1つの引数 pf をを取ります。
この引数はアドレス l で長さ n の record_data_v のインスタンスです。
この証明引数は消費されます。
一度呼び出されると、再度アクセスすることはできません (つまり線形の証明です)。
この関数は2つのことを返します。
1つ目は証明 byte @ l で、"アドレス l には byte が保管されている" と主張しています。
2つ目は線形証明関数で、はじめに私達が返した証明を取り、それを消費して再利用できなくした後、引数として渡した元の証明を返します。

これは、ATS では非常に一般的なイディオムです。
証明を取り、それを消費し、新しい証明を返します。
そして新しい証明を消費する方法と古い証明に戻す方法を提供します。
次にこの関数の使い方を示します:

```ocaml
prval (pf, pff) = extract_hbtype_proof (pf_data)
val hbtype = $UN.cast2int (!p_data)
prval pf_data = pff (pf)
```

prval は証明変数の宣言です。
pf は私の慣用的な名前で証明を表わします。
pff を私が使った場合、証明を消費してオリジナルの証明を返す証明関数を表わします。

!p_data はC言語の *p_data に似ています。
それはポインタに保持されている領域をデリファレンスします。
これが起きると、ATS は p_data
に位置するメモリにアクセスすることができることを示す証明を探索します。
この得られた証明 pf は、私達は byte @ p_data
を持っていて、そこから byte を取り出せるということを主張しています。

より複雑な証明関数は次のようなものです:

```ocaml
prfun extract_payload_length_proof {l:agz} {n:nat}
               (pf: record_data_v (l, n)):
               (array_v (byte, l+1, 2),
                array_v (byte, l+1, 2) -<lin,prf> record_data_v (l,n))
```

array_v view はメモリの連続した配列を表わします。
それが取る3つの引数はそれぞれ、配列に保存されたデータの型と、開始アドレス、要素の数です。
この関数は record_data_v を消費して、証明を産み出します。
その証明は、record view に保持されたオリジナルのメモリアドレスから1バイトオフセットした場所から2要素が存在することを主張しています。
この証明にアクセスできたとしても、SSL
レコードによって保持されたメモリバッファ全体にアクセスすることはできません。
1番目の要素のオフセットから2バイトのみしかアクセスできないのです。

## 安全な memcpy

より複雑な view を見てみましょう:

```ocaml
prfun extract_payload_data_proof {l:agz} {n:nat}
               (pf: record_data_v (l, n)):
               (array_v (byte, l+1+2, n-16-2-1),
                array_v (byte, l+1+2, n-16-2-1) -<lin,prf> record_data_v (l,n))
```

This returns a proof for an array of bytes starting at the 3rd byte of the record buffer.
Its length is equal to the length of the record buffer less the size of the payload, and first two data items.
It’s used during the memcpy call like so:

```ocaml
prval (pf_dst, pff_dst) = extract_payload_data_proof (pf_response)
prval (pf_src, pff_src) = extract_payload_data_proof (pf_data)
val () = safe_memcpy (pf_dst, pf_src 
           add_ptr1_bsz (p_buffer, i2sz 3),
           add_ptr1_bsz (p_data, i2sz 3),
           payload_length)
prval pf_response = pff_dst(pf_dst)
prval pf_data = pff_src(pf_src)
```

By having a proof that provides access to only the payload data area we can be sure that the memcpy can not copy memory outside those bounds.
Even though the code does manual pointer arithmetic (the add_ptr1_bszfunction) this is safe.
An attempt to use a pointer outside the range of the proof results in a compile error.

The same concept is used when setting the padding to random bytes:

```ocaml
prval (pf, pff) = extract_padding_proof (pf_response, payload_length)
val () = RAND_pseudo_bytes (pf |
                            add_ptr_bsz (p_buffer, payload_length + 1 + 2),
                            padding)
prval pf_response = pff(pf)a
```

## 実行時の検査

xxx
