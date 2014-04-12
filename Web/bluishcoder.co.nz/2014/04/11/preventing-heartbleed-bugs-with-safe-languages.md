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

## Safer ATS

The third stage was adding types to the unsafe ATS version to check that the pointer arithmetic is correct and no bounds errors occur.
This version of d1_both.dats fails to compile if certain bounds checks aren’t asserted.
If the assertloc at line 123, line 178 or line 193 is removed then a constraint error is produced.
This error is effectively preventing the heartbleed bug.

## Testable Vesion

The last stage I did was to implement the fix to the tls1_process_heartbeat function and factor out some of the helper routines so it could be shared.
This is in the ats_safe branch which is where the newer changes are happening.
This version removes the assertloc usage and changes to graceful failure so it could be tested on a live site.

I tested this version of OpenSSL and heartbleed test programs fail to dump memory.

## The approach to safety

The tls_process_heartbeat function obtains a pointer to data provided by the sender and the amount of data sent from one of the OpenSSL internal structures.
It expects the data to be in the following format:

```
 byte = hbtype
 ushort = payload length
 byte[n] = bytes of length 'payload length'
 byte[16]= padding
```

The existing C code makes the mistake of trusting the ‘payload length’ the sender supplies and passes that to a memcpy.
If the actual length of the data is less than the ‘payload length’ then random data from memory gets sent in the response.

In ATS pointers can be manipulated at will but they can’t be dereferenced or used unless there is a view in scope that proves what is located at that memory address.
By passing around views, and subsets of views,
it becomes possible to check that ATS code doesn’t access memory it shouldn’t.
Views become like capabilities.
You hand them out when you want code to have the capability to do things with the memory safely and take it back when it’s done.

xxx
