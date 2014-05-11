---
layout: post
title: Zynq 事始め2
description: ""
tags: [Zynq]
---
{% include JB/setup %}

[前回]({{ BASE_PATH }}/2013/12/27/00-zynq-get-started1/)は、ツールに慣れる意味でも、
Zynq の FPGA 部だけを動かしてみました。

しかし、ARM を起動しないと Zynq を使う意味がありませんので、次は ARM + FPGA
で動かしましょう。

ここから当面は Xilinx のドキュメントに沿って進めるのがよいと思います。

### Zynq-7000 All Programmable SoC: Concepts, Tools, and Techniques (CTT)

入門者にとっておすすめのドキュメントは
"Zynq-7000 All Programmable SoC: Concepts, Tools, and Techniques (CTT)" というチュートリアルです。
表紙のリリース日を見て、新しいものを選ぶようにしてください。
日本語版も出ているがバージョンが古いのでおすすめできません。

非常にわかりやすく、これに沿って5章ぐらいまでやっていけば、基本的な使い方は覚えられるでしょう。
唯一の欠点は、現時点では ISE を対象に書かれていて、Vivado に対応していないことです。

そこで、いったんドキュメント通りに ISE でやってみて、
同じことを Vivado でもやってみるのがよいと思います。
Vivado を選択するほうが、最終的に使い勝手がよいと思いますが、
ISE も一応理解はしておいた方がよいと思います。

以下、各章でやることをさらっとまとめました。

#### Chapter 1: Introduction

ツールや用語の説明。 PS と PL という用語は何度も出てくるので抑えておきましょう。

  - PS (Processing System): CPU 部のこと。ARM と周辺のハード。
  - PL (Programmable Logic): FPGA 部のこと

#### Chapter 2: Embedded System Design Using the Zynq Processing System

ARM を動かし、UART から Hello World を出す。FPGA は動かさない。

XPS で PS の設定をし、ハードウェア情報をエクスポート。
それを XSDK にインポートし、Hello World のスタンドアローンアプリを作成します。

#### Chapter 3: Embedded System Design Using the Zynq Processing System and Programmable Logic

ARM の周りにいくつか IP をくっつけて、ARM + FPGA で動かす。
JTAG 経由で FPGA のビットストリームと ARM 用のプログラムをダウンロードします。

#### Chapter4: Debugging with SDK and ChipScope

XSDK を用いたソフトウェアのデバッグ、ChipScope を用いたハードウェアのデバッグ

#### Chapter5: Linux Booting and Application Debugging Using SDK

U-Boot と Linux を起動する。
U-Boot も Linux Kernel もコンパイル済みのイメージをダウンロードしてきて使用します。

XMD を開いて、JTAG 経由で U-Boot と Linux をダウンロードし、起動する。(Slave Boot)

Master Boot に使うブートイメージの作り方を学ぶ。
そのブートイメージを U-Boot を使って、 QSPI に書き込み、 QSPI から起動させる。
また SDカードからも起動させる。(Master Boot)

### チュートリアルの内容を Vivado でやるときの差分

基本的に、チュートリアルの内容と同等のことが Vivado できるはずですが、差分をいくつかピックアップ。

  - ISE で XPS でやっている部分は、Vivado では 「IP Integrator」 → 「Create Block Design」で行う。

  - Chapter3 で「Chipscope AXI Monitor」という IP を接続していますが、
    これは Vivado にはないので、飛ばします。
    Vivado には Chipscope 自体がありませんので。
    普通に Vivado Logic Analyzer で AXI バスをモニタすればいいので、なくても困りません。

  - Chapter6 で 「AXI Central DMA」 というIP を接続していますが、
    Vivado では 「AXI Central Direct Memory Access」という名前になっています。
    検索ボックスに 「DMA」といれてもヒットしなくて、しばらく考え込みました。。

### 起動シーケンスまとめ

どういう仕組みで動いているのかがわかった方がいいので、簡単にまとめておきます。

Zynq は 5つの起動モードがあります。

電源投入時に、特定の端子が Pull Up されているか Pull Down されているかで起動モードを選択できます。

  - QSPI
  - NAND
  - NOR
  - SD card
  - JTAG

QSPI, NAND, NOR, SD card から起動するのを Master Boot といいます。
JTAG ブートは Slave Boot といいます。

Master Boot は ARM が自発的に起動しますが、 Slave Boot は JTAG コマンド待ちのループに入るので、
JTAG からプログラムをダウンロードし、適切なアドレスから実行させる必要があります。

当然ながら最終製品では Master Boot を使いますが、開発中に便利なのは Slave Boot です。

#### Master Boot 用のブートイメージ

    |------------------------|
    |        Header          |
    |------------------------|
    | (IMG1) FSBL            |
    |------------------------|
    | (IMG2) FPGA bit stream |
    |------------------------|
    | (IMG3) User Program    |
    |------------------------|
    |          ...           |


Bootgen というプログラムを使って、いくつかのバイナリーを結合し、Master Boot 用の起動イメージを作ります。
先頭にヘッダー情報が付加されます。各イメージの先頭にも小さなヘッダーが付きます。

1番目のイメージは FSBL (First Stage Boot Loader)です。
FSBL を自前で記述することも可能ですが、普通はツール (XSDK) が作ってくれるものをそのまま使います。
ISE/Vivado から Export した ハードウェア情報を XSDK にインポートし、FSBL を生成します。

通常、2番目のイメージは FPGA の bit stream ですが、FPGA を動かさないなら、なくても構いません。

3番目以降がユーザープログラムです。
これは、XSDK で作成したスタンドアロンプログラムだったり、U-Boot などの別のブートローダーだったりします。

上記の起動イメージを QSPI, NAND, NOR, SD Card のいずれかに書き込みます。

SD Card の場合、FAT でフォーマットされたカードに `boot.bin` というファイル名でコピーする必要があります。

なお、ピン数の関係で 5つの起動モードをすべて同時に使えるわけではありません。
私の使っている ZC706 というボードでは QSPI, SD Card, JTAG が使えます。

余談ですが、ZC706 にのっている QSPI の容量が 16MB しかありません。
ZC706 にのっている Zynq (xc7z045ffg900) の FPGA の bit stream は 13MB 近くあります。
残り 3MB で Linux Kernel やら Ramdisk やらを格納するのはほぼ無理です。
つまり、FPGA 動かすなら、QSPI は使えないですね。
なんで、もうちょっと容量の大きいのをのせてくれなかったんだろう？

#### Master Boot のブートシーケンス

Zynq はリセット解除後、(Master Boot も Slave Bootも)必ず Boot ROM から実行を開始します。

  1. Boot ROM

  Mask Rom なので変更不可。
  端子情報を元に、 QSPI/NAND/Nor/SD card のいずれかから起動イメージを読み出す。
  FSBL を SRAM へロードし、FSBL に制御を移す。

  2. FSBL (First Stage Boot Loader)

  (XSDK で生成した FSBL の場合)
  起動イメージに FPGA の bit stream が含まれていれば、FPGA へダウンロードする。
  必要なピン設定や DDR の初期化を行い、 User Program を DDR へロードし、 User Program に制御を移す。

  3. User Program

  何をするかは User Program の作り方次第ですが、Linux を動かすのなら、
  この User Program の部分は U-Boot でしょう。
  U-Boot がなんらかの不揮発デバイスから（またはネットワーク経由で）Kernel をロードします。

#### Slave Boot のブートシーケンス

JTAG ブートの場合、起動後、簡単な初期化をした後、すぐに JTAG コマンド待ちのループに入ります。
ARM が自発的に何かをすることがないので、Slave Boot と言います。

SoC の初期化や、プログラムコードをメモリ上に置き、ARM をキックするところまで
JTAG 経由で行う必要があります。

以下が、そのシーケンスです。

  1. Boot ROM

  ARM は簡単な初期化をした後、JTAG コマンド待ちのループに入る。

  2. FPGA コンフィグレーション

  (もし必要なら）JTAG経由で FPGA の bit stream をダウンロードする。

  3. JTAG から ピン設定や DDR の初期化を行う。

  4. JTAG から User Program を DDR (または SRAM) へダウンロードし、CPU を走らせる。

チュートリアルの Chapter 2 ～ Chapter 4 でやっているのはこの Slave Boot です。

Master Boot と違って、FSBL なしで、User Program を実行することができます。
その代わり、 FSBL が行うのと同等の設定を JTAG 経由で行うための Tcl スクリプトが用意されています。

ISE/Vivado でハードウェア情報をエクスポートしたときに吐き出されるファイルの中に
`ps7_init.tcl` みたいなものあると思いますが、これです。

「3. JTAG から ピン設定や DDR 初期化」 にはこの Tcl スクリプトを用います。

`ps7_init.c` (FSBL を作るために必要な Cコード) と、内容は同等のはずです。

XMD のプロンプトから

    % source ps7_init.tcl
    % ps7_init
    % ps7_post_config

とやれば OK。

あれ？ チュートリアルの Chapter 2 ～ Chapter 4 をやったときには、いちいち
`ps7_init.tcl` を実行しなかったよ？と思うかもしれません。

はい。 XSDK は、User Program ダウンロード時に、`ps7_init.tcl` も自動で実行してくれています。
XMD の場合は、自分で実行する必要があります。

### 次は何をやるか

CTT の Chapter 5 くらいまでやれば、ツールの基本的な使い方は身につくと思います。

Chapter 6 以降は必要に応じてやっていけばいいと思います。
Zynq の仕様についてもっと知るために
"Zynq-7000 All Programmable SoC Technical Reference Manual" も合わせて読んでいくのもよいでしょう。

私は基本的にソフトの人なので、引き続き [Wiki Page](http://www.wiki.xilinx.com/)
に沿って動かしてみました。
U-Boot や Linux Kernel をソースからコンパイルして動かす方法が非常に丁寧に説明してあります。
