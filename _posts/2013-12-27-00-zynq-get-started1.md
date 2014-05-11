---
layout: post
title: Zynq 事始め1
description: ""
tags: [Zynq]
---
{% include JB/setup %}

[前回]({{ BASE_PATH }}/2013/12/17/00-vivado-ise-on-ubuntu/)、ISE/Vivado
等のツール類のセットアップは済ませました。

では何から始めましょうか。

例えば、Zynq の評価ボードを渡され、「じゃ、立ち上げよろしく！」と言われたとしましょう。
（私が言われたみたいに。。）

膨大な資料を前に途方にくれてしまうかもしれません。
似たような状況にある人の手助けになるかどうかわかりませんが、私のたどったステップを記録しておこうと思う。

進めるには、何らかの Zynq ボードが必要となります。
私は ZC706 というボードを使っていますが、他のボードでも似たような感じだと思います。

### まずはツールに慣れる

Zynq を手にとっているということは FPGA ぐらいは経験があるという人が多いと思います。
ISE は使ったことはあるが、Vivado は初めてという人もいるかもしれませんが、
今後のことを考えれば Vivado を使いこなせた方がよいと思います。

とりあえず、ISE および Vivado で簡単な RTL のプロジェクトを作成し、FPGA を
Configuration するところまでやってみます。
すでに知っている人は飛ばしてください。

まずは、 ARM は動かさずに、Zynq を純粋な FPGA として使います。
(Zynq の通常の使い方としては、ARM が最初に起動し、ARM から FPGA を
Configuration するのですが、それはもうちょっと先に回します。)

#### まずは planAhead でやってみる

ISE のインストールディレクトリに `settings64.sh` (または `settings64.csh`)
というのがあるので、それを `source` したあと、

    $ planAhead

で起動。

「Create New Project」→「RTL Project」を選択。

「Add File...」 か 「Create File...」 で RTL (Verilog or VHDL) を追加。
とりあえず練習ですので、プッシュボタンを押すと LED が点灯するぐらいの RTL でよいです。

ウィザードの最後で Parts で Zynqチップ名 (xc7z0xx) か評価ボード名を選ぶ。

制約ファイルは自分でいちいち書くのは面倒ですが、大抵はすでに用意してくれている。
自分の場合は、 ZC706 ボードなので、Xilinx のサイトから
`zc706-ucf-xdc-rdf0207-rev1-0.zip` というのをダウンロード。

RTL と制約ファイル(UCF)を追加できたら、Synthesis, Implement, Generate Bitstream と進む。

PC とボードの JTAG ポートをつなぐ。
ZC706 の場合は、ボード上に JTAG <-> USB 変換チップが載っているので、
USBケーブルでつなぐだけです。

iMPACT を起動すると、「zynq7000_arm_xxx」(=ARM) というのと「xc7z0xx」(=FPGA) というが見えるはず。
2つ見えるのは、Zynq のパッケージ内で ARM コアと FPGA が JTAGデイジーチェーンで繋がっているためです。

FPGA の方を選んで、bit ファイルをダウンロードすると、うまくいくはず。
LED が光ることを確認して下さい。

#### Vivado でもやってみる

Vivado のインストールディレクトリに `settings64.sh` (または `settings64.csh`)
というのがあるので、それを `source` したあと、

    $ vivado

で起動。

planAhead と操作手順はほぼ同じですが、制約ファイルが従来のUCF ではなくて、
XDC というファイルです。
Tcl の文法で書かれている。

Synthesis, Implement, Generate Bitstream と進み、最後は、 iMPACT ではなくて、
「Open Hardware Session」というのを選ぶ。

「Open Hardware Session」→「Open a new hardware target」を選び、
`Server name <host:port>` で PC の IPアドレスを入力する(普通はデフォルトでそうなっているはず)。
ポート番号はそのままでよい。
あとは、「Program device」を選べば、 FPGA を Configuration できる。

ちなみに、「Launch iMPACT」というのも選べますが、Vivado 自体は iMPACT を含んでいませんので、
iMPACT を使いたいならば、ISE もインストールし、 `impact` が PATH に含まれていないといけません。

「Open Hardware Session」 というのは、ISE の時の iMPACT と ChipScope の置き換えで、たぶんこっちの方が便利だと思います。
サーバー＆クライアントで使用できます。
このやり方は[後日]({{ BASE_PATH }}/2014/01/10/00-vivado-remote/)紹介します。

### もうちょっと RTL を足してみる

上記ができたら、クロックを入れて、LED 点滅とかもやってみます。

クロックラインを BUFG に通す、タイミング制約の追加とかが必要です。
特に Vivado が初めての場合は、 XDC でのタイミング制約の書き方を確認しておきます。

「Project Manager」 → 「Templates」とたどれば、書き方がわかるので、コピペでいけますが。
