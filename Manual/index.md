# ATS2の公式マニュアルの日本語訳

## 文書

### ATSプログラミング入門

* 日本語: http://jats-ug.metasepi.org/doc/INT2PROGINATS/
* 元データ: https://github.com/jats-ug/ATS-Postiats/tree/translate_ja/doc/BOOK

## 翻訳環境

### 概要

xxx 図

翻訳環境は大きく翻訳作業と発行作業の2つに分れています。
発行作業は
[JATS-UGのメンバー](https://github.com/jats-ug?tab=members)
しかできませんが、翻訳作業はどなたでも可能です。
誤字や未訳を見つけたらpull requestをいただけたら助かります。

githubリポジトリにたまった翻訳は適切なタイミングで
[JATS-UGのメンバー](https://github.com/jats-ug?tab=members)
が発行します。

### 翻訳作業

xxx

### 発行作業

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
$ make html_ja -j4
$ ls HTML/index.html
HTML/index.html
```

あとは
[jats-ug.github.io](https://github.com/jats-ug/jats-ug.github.io)
リポジトリにコピーして、git pushすれば発行作業は完了です。

```
$ make publish_jats_ug
  cp -rf HTML/* /home/kiwamu/doc/jats-ug.github.io/doc/INT2PROGINATS/
$ cd ~/doc/jats-ug.github.io
$ git add .
$ git commit -m "Update doc"
$ git push
```

## 古いメモ

[古いメモ](memo.md)
を試行錯誤の履歴として保管していあります。
