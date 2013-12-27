# ATS2の公式マニュアルの日本語訳

https://github.com/jats-ug/ATS-Postiats/tree/translate_ja/doc/BOOK

を翻訳する予定。

## 翻訳手順

xxx ...構築中...

http://po4a.alioth.debian.org/

を使ったgettext翻訳をするか考え中。
問題は僕以外の人がgettextを使った翻訳に慣れているか。。。

docbookであるmain.dbをpo4aにかけると煩雑なpoファイルが出てしまう。
大本のmain.atxtファイルをpo4aにかけてしまうの無難か。
ところが、jwコマンドでエラーになる。xmlのスタイルシートをなんか指定しないとダメ？

```
$ pwd
/home/kiwamu/src/ATS-Postiats.jats-ug/doc/BOOK/INT2PROGINATS
$ git diff preface.atxt
diff --git a/doc/BOOK/INT2PROGINATS/preface.atxt b/doc/BOOK/INT2PROGINATS/preface.atxt
index 4ff8dae..821c2b5 100644
--- a/doc/BOOK/INT2PROGINATS/preface.atxt
+++ b/doc/BOOK/INT2PROGINATS/preface.atxt
@@ -30,7 +30,7 @@ implements according to its specification.  If I could associate only one
 single word with ATS, I would choose the word
 <emphasis>precision</emphasis>.  Programming in ATS is about being precise
 and being able to effectively enforce precision. This point will be
-demonstrated concretely and repeatedly in this book.\
+demonstrated concretely and repeatedly in this book. 日本語\
 
 ')
$ make html
--snip--
jw -b html --output HTML/ main.db
Using catalogs: /etc/sgml/catalog
Using stylesheet: /usr/share/docbook-utils/docbook-utils.dsl#html
Working on: /home/kiwamu/src/ATS-Postiats.jats-ug/doc/BOOK/INT2PROGINATS/main.db
jade:/usr/share/sgml/declaration/xml.dcl:31:27:W: characters in the document character set with numbers exceeding 65535 not supported
jade:/home/kiwamu/src/ATS-Postiats.jats-ug/doc/BOOK/INT2PROGINATS/preface.db:23:54:E: non SGML character number 151
jade:/home/kiwamu/src/ATS-Postiats.jats-ug/doc/BOOK/INT2PROGINATS/preface.db:23:57:E: non SGML character number 156
jade:/home/kiwamu/src/ATS-Postiats.jats-ug/doc/BOOK/INT2PROGINATS/preface.db:23:61:E: non SGML character number 158
make: *** [html] エラー 8
```

## マニュアルのビルド手順

まずATS-Anairiatsのアーカイブを以下のURLから取ってきます。

http://sourceforge.net/projects/ats-lang/

取得したATS-Anairiatsをビルドしましょう。
インストールはしなくても大丈夫です。

```
$ tar xfz ats-lang-anairiats-0.2.11.tgz
$ cd ats-lang-anairiats-0.2.11
$ ./configure
$ make
--snip--
ATS/Anairiats has been built up successfully!
The value of ATSHOME for this build is "/home/kiwamu/src/ats-lang-anairiats-0.2.11".
The value of ATSHOMERELOC for this build is "ATS-0.2.11".
```

ATSHOMEとATSHOMERELOCを環境変数に設定しましょう。

```
$ export ATSHOME=/home/kiwamu/src/ats-lang-anairiats-0.2.11
$ export ATSHOMERELOC=ATS-0.2.11
```

準備がととのったので、ATS-Postiatsのマニュアルをビルドしましょう。

```
$ sudo apt-get install docbook-utils
$ git clone https://github.com/jats-ug/ATS-Postiats.git
$ cd ATS-Postiats
$ git checkout translate_ja
$ cd doc/BOOK/INT2PROGINATS
$ make html
$ ls HTML/index.html
HTML/index.html
$ make pdf
$ ls PDF/main.pdf
PDF/main.pdf
```
