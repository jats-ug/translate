# BLUISH CODER: より安全なATSによるC言語の使い方

(元記事は http://bluishcoder.co.nz/2010/11/23/more-on-type-safety-using-c-and-ats.html です)

以前 [ATSプログラミング言語](http://www.ats-lang.org/) を使った
[安全なC言語の使い方](http://bluishcoder.co.nz/2010/06/02/safer-c-code-using-ats.html) を紹介しました。
私は最近いくつかのプロジェクトでATSを使っていて、C言語ライブラリをATSから扱わねばなりませんでした。
この記事ではある種類のエラーを型検査のエラーにすることで、
C言語ライブラリの不正な使用を防止するより多くの方法を紹介します。

私のプロジェクトでは [JSON](http://json.org/) リクエストを構文解析する必要がありました。
この用途に適したC言語ライブラリは [jansson](http://digip.org/jansson) です。
このライブラリは良い実装と
[良いドキュメント](http://www.digip.org/jansson/doc/1.3) を合わせ持っています。

janssonのAPIで使われるJSONオブジェクトはリファレンスカウントによって管理されていて、
ドキュメントにはどのAPIが参照の所有権を持ち、どのAPIが持たないかが明記されています。
それでも一見正しいコードが問題を引き起こすことはあります。
そのようなコーナーケースは、ATSのラッパーがコンパイル時に検出することが望まれます。

## リファレンスカウント

リファレンスカウントの操作ミスは明らかにエラーの発生源です。

次のC言語プログラムはJSONオブジェクトのデリファレンスに失敗しています。
結果としてメモリリークを引き起こしてしまいます。

```c
#include <stdio.h>
#include <jansson.h>
    
int main() {
  json_t* a = json_string("this is a string");
  const char* s = json_string_value(a);
  printf("String value: %s\n", s);
  return 0;
}
```
