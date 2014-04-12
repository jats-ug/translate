# BLUISH CODER: 安全なプログラミング言語を使って heartbleed を防ぐには

(元記事は http://bluishcoder.co.nz/2014/04/11/preventing-heartbleed-bugs-with-safe-languages.html です)

OpenSSL の [Heartbleed バグ](http://heartbleed.com/) はインターネット全体に多大なダメージを与えました。
このバグはとても単純で、C言語のような安全でない言語を使ったプログラミングがなぜ問題になるのかを示す教科書的な例です。

より安全なシステムプログラミング言語がこのバグを防止できるかどうかの実験として、
今回の問題の関数を [ATS プログラミング言語](http://www.ats-lang.org/) を用いて書き直してみます。
私は以前、[より安全なC言語として ATS について紹介しました](http://bluishcoder.co.nz/tags/ats)。
この記事では現実のテストケースを取り扱います。
ここでは ATS2 と呼ばれる ATS の最新版を使っています。

ATS はC言語コードにコンパイルします。
The function interfaces it generates can exactly match existing C functions and be callable from C.
I used this feature to replace the dtls1_process_heartbeat and tls1_process_heartbeat functions in OpnSSL with ATS versions.
These two functions are the ones that were patched to correct the heartbleed bug.

The approach I took was to follow something similar to that outlined by John Skaller on the ATS mailing list:

```
ATS on the other hand is basically C with a better type system.
You can write very low level C like code without a lot of the scary
dependent typing stuff and then you will have code like C, that
will crash if you make mistakes.

If you use the high level typing stuff coding is a lot more work
and requires more thinking, but you get much stronger assurances
of program correctness, stronger than you can get in Ocaml
or even Haskell, and you can even hope for *better* performance
than C by elision of run time checks otherwise considered mandatory,
due to proof of correctness from the type system. Expect over
50% of your code to be such proofs in critical software and probably
90% of your brain power to go into constructing them rather than
just implementing the algorithm. It's a paradigm shift.
```

xxx
