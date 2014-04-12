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

I’ve written about this approach before.
ATS allows embedding C directly so the first start was to embed the dtls1_process_heartbeat C code in an ATS file,
call that from an ATS function and expose that ATS function as the real dtls1_process_heartbeat.
The code for this is in my first attempt of d1_both.dats.

## 安全でない ATS

The second stage was to write the functions using ATS but unsafely.
This code is a direct translation of the C code with no additional typechecking via ATS features.
It uses usafe ATS code.
The rewritten d1_both.dats contains this version.

The code is quite ugly but compiles and matches the C version.
When installed on a test system it shows the heartbleed bug still.
It uses all the pointer arithmetic and hard coded offsets as the C code.
Here’s a snippet of one branch of the function:

```ats
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

It should be pretty easy to follow this comparing the code to the C version.
```

xxx
