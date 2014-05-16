---
layout: post
title: ISE & Vivado 設定 on Ubuntu
description: ""
tags: [Zynq]
---
{% include JB/setup %}

前回記事 [Zynq 入門中]({{ BASE_PATH }}/2013/12/16/00-zynq-intro/) の続きです。

ISE と Vivado を Ubuntu にインストールします。

前回も述べたように、Ubuntu はサポート外なので、そのままでは動かない部分があります。

バージョンによって若干の差があるかもしれないので、ここでは以下のバージョンとします。

- Vivado 2013.2
- ISE 14.6
- Ubuntu 2013.04 64bit

### Vivado のセットアップ

#### インストール

Xilinx のページから tar をダウンロードして、以下を実行。

    $ Xilinx_Vivado_SDK_2013.2_0616_1.tar
    $ cd Xilinx_Vivado_SDK_2013.2_0616_1
    $ sudo ./xsetup

途中のダイアログで Cable Driver をインストールする選択肢があるが、選んでも多分うまくいかないので、

    $ cd /opt/Xilinx/Vivado/2013.2/data/xicom/cable_drivers/lin64/digilent$
    $ sudo ./install_digilent.sh

で直接インストール。

#### 文字化け(豆腐)の修正

planAHEAD は文字化けしないのだが、 Vivado はステータスバーの表示が豆腐になってしまう。

修正の仕方がわからないが、とりあえず日本語である必要はないので、メニューから

Tools -> Options... -> General -> Language and Tooltips とたどり

Japanese から English へ変更。

#### Documentation のLink が開かない場合の対処方法

IP などの Documentation を開こうとすると、

    /opt/Xilinx/Vivado/2013.2/ids_lite/ISE/lib/lin64/libstdc++.so.6:
     version `GLIBCXX_3.4.15' not found (required by /usr/lib/firefox/libxul.so)

のようなエラーが端末に出て開かない。

Vivado から起動された firefox がライブラリ検索で
`/opt/Xilinx/Vivado/2013.2/ids_lite/ISE/lib/lib64/` 以下を見に行ってしまって、
しかも Ubuntu に元々インストールされているものより古いのが原因のようだ。

以下のように Vivado 添付のライブラリを退避し、`/usr/lib` 以下のものへシンボリックリンクを貼ることで動くようになった。

    $ cd /opt/Xilinx/Vivado/2013.2/ids_lite/ISE/lib/lin64
    # mv libstdc++.so libstdc++.so.bak
    # mv libstdc++.so.6 libstdc++.so.6.bak
    # ln -s libstdc++.so.6 libstdc++.so
    # ln -s /usr/lib/x86_64-linux-gnu/libstdc++.so.6

### ISE のセットアップ

#### インストール

Xilinx のページから tar をダウンロードして、以下を実行。


    $ tar xf Xilinx_ISE_DS_14.6_P.68d_3.tar
    $ cd Xilinx_ISE_DS_14.6_P.68d_3
    $ sudo ./xsetup


#### PlanAhead が起動しない場合の対処方法

PlanAhead が異常終了し、コンソールに以下のようなメッセージが出る。

    /opt/Xilinx/14.6/ISE_DS/PlanAhead/bin/rdiArgs.sh: 95 行: 13848 Segmentation fault      (コアダンプ) "$RDI_PROG" "$@"

以下のようにして、ISE に付属の libjvm.so を置き換える。

    $ sudo apt-get install openjdk-7-jre
    $ cd /opt/Xilinx/14.6/ISE_DS/PlanAhead/tps/lnx64/jre/lib/amd64/server/
    $ sudo mv libjvm.so libjvm.so.bak
    $ sudo ln -s /usr/lib/jvm/java-7-openjdk-amd64/jre/lib/amd64/server/libjvm.so

[ここのページ](http://forums.xilinx.com/t5/Installation-and-Licensing/RHEL5-64-bit-ISE-13-1-PlanAhead-launch-from-w-in-ISE-fails/td-p/148624/page/2)
を参考にしました。

#### XilinxNotify のエラーダイアログの対処方法

root 権限でインストールした場合、PlanAhead 起動時に以下のようなエラーダイアログボックスが出る。

    A disk write failure occurred. There may be insufficient disk space or you may
    not have write permisson at the following directory.
    
    /opt/Xilinx/14.6/ISE_DS/.xinstall
    Press Retry to try again, press Cancel to exit XilinxNotfy

エラーの原因は、 root にしか書き込み権限のない `/opt/Xilinx/14.6/ISE_DS/.xinstall/`
の下に `xilinxnotify.log` というファイルを作ろうとしているため。

インストール時は、root 権限、アプリ実行は 一般ユーザーで行うのが普通なのに、なんでこんな場所にログファイル作っちゃうんだろうね？
クオリティ低いですな。

とりあえず以下のようにして回避。

    $ sudo chmod a+w /opt/Xilinx/14.6/ISE_DS/.xinstall

#### Xilinx License Configuration Manager からライセンス取得できない場合の対処方法

Connect Now ボタンを押しても、ブラウザが開かずに端末に以下のメッセージが出ている。

    /usr/lib/chromium-browser/chromium-browser: /opt/Xilinx/14.6/ISE_DS/common//lib/lin64/libstdc++.so.6: version `GLIBCXX_3.4.9' not found (required by /usr/lib/chromium-browser/chromium-browser)

以下のようにして shared object 置き換え。

    $ cd /opt/Xilinx/14.6/ISE_DS/common/lib/lin64
    # mv libstdc++.so libstdc++.so.bak
    # mv libstdc++.so.6 libstdc++.so.6.bak
    # ln -s libstdc++.so.6 libstdc++.so
    # ln -s /usr/lib/x86_64-linux-gnu/libstdc++.so.6

#### Document が開かない場合の対処方法

ドキュメントをブラウザで開こうとするが、GLIBC の互換性の問題で開かない。

上記と同じなので、端末のエラーメッセージを見ながら `libstdc++.so` を置き換える。


### fpga editor が起動しない場合の対処方法 ###

コンソールに以下のようなエラーメッセージが出る。

    /opt/Xilinx/14.6/ISE_DS/ISE/bin/lin64/_fpga_editor: error while loading shared libraries: libXm.so.3: cannot open shared object file: No such file or directory

ここでも shared object の問題のようだ。

パッケージ名がわからない場合は `apt-file` で調べられる。

    $ apt-file search libXm.so.3
    libmotif4: /usr/lib/x86_64-linux-gnu/libXm.so.3
    $ sudo apt-get install libmotif4

一歩前進したが、まだ以下のようなエラーメッセージが出る。

    /opt/Xilinx/14.6/ISE_DS/ISE/bin/lin64/_fpga_editor: error while loading shared libraries: libstdc++.so.5: cannot open shared object file: No such file or directory

以下をインストール。

    $ sudo apt-get install libstdc++5

まだ以下のようなエラーメッセージが出る。

    Wind/U X-toolkit Error: wuDisplay: Can't open display

環境変数が `DISPLAY=:0.0` になっているのを修正

    $ export DISPLAY=:0

まだ以下のエラーメッセージが出る。

    Wind/U Error (193): X-Resource: DefaultGUIFontSpec (-*-helvetica-medium-r-normal-*-14-*) does not fully specify a font set for this locale

フォントをインストール。

    $ sudo apt-get install xfonts-75dip xfonts-100dpi

ようやく起動した。

#### ChipScope が起動しないときの対処方法

ChipScope Analyzer が起動せずに

    /opt/Xilinx/14.6/ISE_DS/ISE/bin/lin64/unwrapped/analyzer: 72: /opt/Xilinx/14.6/ISE_DS/ISE/bin/lin64/unwrapped/cs_common.sh: XIL_DIRS[0]=/opt/Xilinx/14.6/ISE_DS/ISE/: not found
    /opt/Xilinx/14.6/ISE_DS/ISE/bin/lin64/unwrapped/analyzer: 73: /opt/Xilinx/14.6/ISE_DS/ISE/bin/lin64/unwrapped/cs_common.sh: count++: not found
    /opt/Xilinx/14.6/ISE_DS/ISE/bin/lin64/unwrapped/analyzer: 152: /opt/Xilinx/14.6/ISE_DS/ISE/bin/lin64/unwrapped/cs_common.sh: Syntax error: Bad for loop variable

というエラーメッセージが端末に出る。

これは `cs_common.sh` 中で bash の拡張機能を使っているにも関わらず、shebang が `#/bin/sh` になっているから。

Xilinx がサポートしている RedHat系ディストリビューションでは `/bin/sh` が
`/bin/bash` へのシンボリックリンクになっているために、__たまたま__ 動いてしまう。
Ubuntu では動かない。(Ubuntu では `/bin/sh` は `/bin/dash` へのシンボリックリンクになっている。)

そこで、 `/opt/Xilinx/14.6/ISE_DS/ISE/bin/lin64/unwrapped/cs_common.sh`
を開き、一行目を `#!/bin/bash` に書き換えるが、やはり動かない。。

よく見ると、 analyzer から `cs_common.sh` を source している。
なので `/opt/Xilinx/14.6/ISE_DS/ISE/bin/lin64/unwrapped/analizer` を開き、一行目を
`#!/bin/bash` に書き換えると、、ChipScope が起動するようになりました。

#### Export Hardware for SDK が失敗する場合の対処方法

ダイアログボックスが開き、

    ERROR: [Common 17-49] Internal Data Exception: xps application failed!

というメッセージが出る。`gmake` がいるようだ。

    ln -s /usr/bin/make /usr/local/bin/gmake

### ISE と Vivado 共通のセッティング

#### Xilinx SDK で arm-xilinx-eabi-gcc が not found の場合の対処方法

xsdk を起動して、プロジェクトをビルドするときに

    /bin/sh: 1: arm-xilinx-eabi-gcc: not found

のエラーがでる場合がある。

`settings64.sh` を source すれば、 `arm-xilinx-eabi-gcc` への PATH も通るはず。

端末上から `arm-xilinx-eabi-gcc` が見えているのに相変わらず動かない場合。

    $ which arm-xilinx-eabi-gcc
    /opt/Xilinx/14.6/ISE_DS/EDK/gnu/arm/lin/bin/arm-xilinx-eabi-gcc
    $ arm-xilinx-eabi-gcc --version
    arm-xilinx-eabi-gcc: No such file or directory

原因は、`arm-xilinx-eabi-gcc` が 32bit ホスト用であるため。以下を実行してみればわかる。

    $ file `which arm-xilinx-eabi-gcc`
    /opt/Xilinx/14.6/ISE_DS/EDK/gnu/arm/lin/bin/arm-xilinx-eabi-gcc: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.2.5, stripped

これを 64bit ホストで動かすには 32bit 用のライブラリが必要なので、以下のようにインストールする。

    $ sudo apt-get install ia32-libs
    
もはや 32bit ホストで FPGAやるのは難があるのだから、
64bit ネイティブでツールをそろえて欲しいところですが。。

#### ノードロックライセンスを全ユーザーから使えるようにする

通常はライセンスファイルは ~/.Xilinx に置かれるが、環境変数で変更できるので、

/opt/Xilinx 以下に置き、各ユーザーが

    export XILINXD_LICENSE_FILE=/opt/Xilinx/Xilinx.lic

の環境変数設定。

### 端末表示関連

Zynq 上の ARM と UART で通信しなくてはならないので、そのセットアップ。

最近のデスクトップPC にはシリアルポートがついてなかったりするのですが、
評価ボード上には USB-UART 変換(CP210X)が載っているので問題なし。

#### USB-UART ドライバ

Windows ならば、SiliconLab のページから CP210X のドライバをダウンロードしてくるところですが、
Linux では、Kernel のソースツリーに CP210X のドライバが含まれているので、何もしなくてよい。楽だな。

Ubuntu ならば `CONFIG_USB_SERIAL_CP210X=m` でビルドされているので、ボードと
USBケーブルでつなぐだけで勝手にモジュールがロードされるはず。
試しに、ボードとUSBケーブルでつないでみて、

    $ lsmod | grep ^cp210x
    cp210x                 17382  0 

みたいに表示されれば、ちゃんとロードされている。

ちなみに、ドライバは `/lib/modules/3.*.*-*-generic/kernel/drivers/usb/serial/cp210x.ko`
にインストールされている。

モジュールのロードと同時に `/dev/ttyUSB0` が作られているはず。

    $ ls -l /dev/ttyUSB0 
    crw-rw---- 1 root dialout 188, 0  8月 22 11:08 /dev/ttyUSB0

でわかるように、dialout に所属していないとアクセスできないので、以下のようにする。

    $ sudo gpasswd -a <user_account> dialout

#### ckermit メモ

ターミナルソフトは、 Windows なら TeraTerm が有名ですが、Linux なら ckermit がおすすめです。

    $ sudo apt-get install ckermit

でインストール。

    $ kermit

で起動。

    (/home/masahiro/) C-Kermit>set line /dev/ttyUSB0
    (/home/masahiro/) C-Kermit>set carrier-watch off
    (/home/masahiro/) C-Kermit>set speed 115200
    /dev/ttyUSB0, 115200 bps
    (/home/masahiro/) C-Kermit>connect

でつながるが、毎回入力するのが面倒なので `~/.kermrc` に

    set line /dev/ttyUSB0
    set carrier-watch off
    set speed 115200
    connect

を書いておきましょう。
閉じるときは `Ctrl-\` のあとに `c`。
