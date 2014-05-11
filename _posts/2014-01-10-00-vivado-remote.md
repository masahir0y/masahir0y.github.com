---
layout: post
title: Vivado で FPGA をリモートコンフィグレーション
description: ""
tags: [Zynq]
---
{% include JB/setup %}

### サーバー/クライアントモデルになった FPGA コンフィグレーション

Vivado で便利だと思ったのが、リモートで FPGA のコンフィグレーションが行えることです。

どういうことかというと、Vivado が起動している PC と ボードが繋がっている PC
が別々でもいいのです。


    |-------|          |-------|  JTAG
    |       |   LAN    |       |  cable  |------------|
    |  PC1  |----------|  PC2  |---------| FPGA Board |
    |       |          |       |         |------------|
    |-------|          |-------|

図のように LAN でつながった PC1 と PC2 があったとする。

Vivado は PC1 上で動作している。
Vivado がサクサク動くように PC1 は高速なサーバーマシンだとしよう。

PC2 は作業者が実際に操作するクライアント端末である。非力なマシンでもOK。

PC1 には触らないので、別の部屋にあってもよい。

Zynq ボードは、LED をモニタしたり、ディップスイッチを押したりしたいので、ボードは PC2 に接続する。

この状況で、PC1 上で動作している Vivado から PC2 につながったボード上の
FPGA をコンフィグレーションできます。

#### 手順

作業者は PC2 から PC1 へ ssh などでログインし、Vivado を起動する。

PC2 にボードを接続し、PC2 上で `vcse_server` というのを起動する。

    $ vcse_server 
    Vivado Cse Server Version 2013.2  - Compiled Jun 15 2013 11:11:22
    
    Vivado Cse Server: Opened server at port: 60001
    
      To exit the server, type command 'exit'
    
    VCSE % 

cse_server がポート番号 60001 を開いたというのが表示されました。

PC1 上の Vivado で FPGA の bit stream を作成したら、「Program and Debug」→
「Open Hardware Session」→「Open a new hardware target」と選択すると、
接続する Cse Server を設定するダイアログが開きます。

[![Open Hardware Session](http://4.bp.blogspot.com/-WBzyz9irer8/Us9yhXv-9JI/AAAAAAAAHrk/Lhsz86gp_h0/s320/hardware_session.png)](http://4.bp.blogspot.com/-WBzyz9irer8/Us9yhXv-9JI/AAAAAAAAHrk/Lhsz86gp_h0/s1600/hardware_session.png)

通常は、 Vivado の動作しているPC （この場合はPC1）のIPアドレスがデフォルトで入っていると思いますが、
これをさっき vcse_server を起動したPC (PC2) のIPアドレスに修正します。
ポート番号もさっきの番号 60001 になっているのを確認しておきます。

すると PC2 につながったボードに接続できるので、あとはいつもの感じで
FPGA へ bit stream をダウンロードします。

全く同様に Vivado Logic Analyzer (ISE のときの ChipScope みたいなもの)もリモートで使えます。

### XSDK もリモートでデバッグする

Zynq を動かすには ARM 用のソフトウェアと FPGA のハードウェアの両方を開発する必要があります。

FPGA 開発がリモートで行えるのならば、XSDK (Eclipse を Xilinx 用に拡張したもの)
もリモートでやりたいところです。
2つやり方を見つけたので記録。

#### やり方その1: xmd を使う

ボードを JTAG ケーブルで PC2 に接続し、PC2 上で xmd を起動し、`connect arm hw` を実行。

    $ xmd 
    Xilinx Microprocessor Debugger (XMD) Engine
    Xilinx EDK 14.6 Build EDK_P.68d
    Copyright (c) 1995-2012 Xilinx, Inc.  All rights reserved.
    
    XMD% 
    XMD% connect arm hw
    
    JTAG chain configuration
    --------------------------------------------------
    Device   ID Code        IR Length    Part Name
     1       4ba00477           4        Cortex-A9
     2       23731093           6        XC7Z045
    
    --------------------------------------------------
    Enabling extended memory access checks for Zynq.
    Writes to reserved memory are not permitted and reads return 0.
    To disable this feature, run "debugconfig -memory_access_check disable".
    
    --------------------------------------------------
    
    CortexA9 Processor Configuration
    -------------------------------------
    Version.............................0x00000003
    User ID.............................0x00000000
    No of PC Breakpoints................6
    No of Addr/Data Watchpoints.........4
    
    Connected to "arm" target. id = 64
    Starting GDB server for "arm" target (id = 64) at TCP port no 1234
    XMD% 

TCPポート 1234 番が開いたのがわかる。

PC1 上で動作している XSDK で プロジェクトを右クリックし、
「Debug As」→「Debug Configurations...」と選ぶ。

[![Debug Configuration](http://2.bp.blogspot.com/-4D5fw2VGva8/Us-cQbDXU9I/AAAAAAAAHs8/CxPcp8ZEnUY/s320/debug_config.png)](http://2.bp.blogspot.com/-4D5fw2VGva8/Us-cQbDXU9I/AAAAAAAAHs8/CxPcp8ZEnUY/s1600/debug_config.png)

Xilinx C/C++ application (GDB) を右クリックし、「New」を選択。

「Remote Debug」というタブがあるので、「IP Address」の欄の localhost を消し、
PC2 の IPアドレスを入力する。
「Port」の欄には、 PC2 側が待っている 1234 番が入っているのを確認する。

あとは「Debug」ボタンを押す。ステップ実行などができる。

#### 方法その2: hw_server を使う

基本的に

[http://www.xilinx.com/support/documentation/sw_manuals_j/xilinx14_6/SDK_Doc/tasks/sdk_t_tcf_remote_debug.htm](http://www.xilinx.com/support/documentation/sw_manuals_j/xilinx14_6/SDK_Doc/tasks/sdk_t_tcf_remote_debug.htm)

にしたがってやれば OK。

PC2 上で `hw_server` 起動

    $ hw_server -s tcp::3122
    
    ****** Xilinx hw_server v2013.4
      **** Build date : Dec  9 2013-17:26:10
        ** Copyright 1986-1999, 2001-2013 Xilinx, Inc. All Rights Reserved.
    
    INFO: hw_server application started
    INFO: Use Ctrl-C to exit hw_server application

PC1 上で動いている XSDK 上で

メニュー「Xilinx Tools」→「Configure JTAG Settings」を選び、

  - Type: Xilinx Hardware Server
  - Hostname: hw_server を起動したPC (この場合PC2) のIPアドレス
  - Port: hw_server で開いているポート (この場合 3122)

を設定する。

プロジェクトを右クリックし、「Debug As」→「Launch Hardware (System Debugger)」を選ぶ。
