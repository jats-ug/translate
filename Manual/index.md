# ATS2の公式マニュアルの日本語訳

https://github.com/jats-ug/ATS-Postiats/tree/translate_ja/doc/BOOK

を翻訳する予定。

## 翻訳手順

xxx ...構築中...

http://po4a.alioth.debian.org/

を使ったgettext翻訳をするか考え中。
問題は僕以外の人がgettextを使った翻訳に慣れているか。。。

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
