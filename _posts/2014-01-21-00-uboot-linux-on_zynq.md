---
layout: post
title: U-Boot と Linux Kernel のメインラインで Zynq を動かす
description: ""
tags: [Zynq,Linux Kernel,U-Boot]
---
{% include JB/setup %}

[以前]({{ BASE_PATH }}/2014/01/08/00-zynq-get-started2/)、U-Boot と Linux をソースからビルドして
Zynq 上で動かすには、[Wiki Page](http://www.wiki.xilinx.com/)
を読むのがわかりやすいと書きました。
このページを参照しながら、やっている人も多いのではと思います。

Xilinx は GitHub に[ローカルな開発リポジトリ](https://github.com/Xilinx)
を持っていて、 Wiki Page もこのリポジトリを使っています。

ここでは、Xilinx のリポジトリではなく、U-Boot と Linux Kernel のメインラインからコードを持ってきて、
Linux をブートさせるやり方を紹介します。

ちょうど、本日(=2014.1.21) U-Boot 2014.01 がリリースされました。
このリリースから Zynq の各種ボードサポートが U-Boot のメインラインにマージされたので、
Xilinx ローカルリポジトリを追跡しなくても動かせるようになりました。

また、前日 (=2014.1.20) には Linux 3.13 がリリースされましたので、これを使ってみます。

今回使うのは

  - U-Boot 2014.01
  - Linux Kernel 3.13
  - Zynq ZC706 ボード

です。

ただし、Wiki Page で紹介されているやり方と結構違います。主な違いは

  - U-Boot 自身のコンフィグレーションにも Device Tree が必要
  - Kernel のイメージは、従来のレガシー uImage ではなく、 FIT (Flattened uImage Tree) を使う
  - Kernel の defconfig は `multi_v7_defconfig` を使う

入門者には何言っているかわからないかもしれないが、以下で丁寧に手順を説明します。

### STEP0: DTC (Device Tree Compiler) の準備

##### Input Files Required

  - None

##### Output Files Produced

  -  `dtc`: Device Tree Compiler

##### Task Description

U-Boot を Device Tree 付きでビルドするには version 1.4 以降の DTC が必要になる。
ディストリビューションに標準で用意されている DTC では不足するかもしれないのでバージョンを確認。

    $ dtc -v
    Version: DTC 1.3.0

バージョンが 1.4.0 よりも古い場合は、自分でビルドします。

    $ git clone git://git.jdl.com/software/dtc.git

でソースを取ってきて

    $ make
    $ make install

でビルドとインストールができる。

`$(HOME)/bin` の下に `dtc` が入るので PATH を通しておく。

### STEP1: U-Boot のビルド

##### Input Files Required

  - `dtc`
  - ARM Cross Compiler

##### Output Files Produced

  - `u-boot-dtb.bin`: u-boot の RAWバイナリと u-boot をコンフィグレーションする DTB を連結したもの
  - `tools/mkimage`: u-boot で扱うイメージを生成するツール。

(Wiki Page では `u-boot` を使っているが、これは使わない。)

##### Task Description

    $ git clone git://git.denx.de/u-boot.git
    $ cd u-boot
    $ git checkout v2014.01

でソース取得して、v2014.01 タグをチェックアウト。
（何かあっても自分で対処できる人は masterブランチでやってもOK）

あとで、Kernel を TFTP でダウンロードしたいので、`include/configs/zynq-common.h`
を開いて、適当なところに

    #define CONFIG_IPADDR   192.168.11.2
    #define CONFIG_SERVERIP 192.168.11.1
    #define CONFIG_ETHADDR  00:0a:35:00:01:22

の3行を足す。

`CONFIG_IPADDR` は Zynqボードに割り振る IPアドレス、 `CONFIG_SERVERIP` は
TFTP サーバーのアドレスに合わせて下さい。
MACアドレスは、(他のネットワーク機器と被らなければ)適当でいい。

TFTP サーバーがなくても、動かすことはできるので、ない人は上記はスキップして下さい。

あとは

    $ make zynq_zc70x_config
    $ make CROSS_COMPILE=arm-linux-gnueabi- DEVICE_TREE=zynq-zc706

のようにして、コンフィグレーションとビルドをする。

もしくは

    $ make zynq_zc70x CROSS_COMPILE=arm-linux-gnueabi- DEVICE_TREE=zynq-zc706

のように 1行で、コンフィグレーションとビルドを同時にすることもできる。

### STEP2: Linux Kernel のビルド

##### Input Files Required

  - ARM Cross Compiler

##### Output Files Produced

  - `arch/arm/boot/zImage`: Kernel Image
  - `arch/arm/boot/dts/zynq-zc706.dtb`: Kernel をコンフィグレーションする DTB (Device Tree Blob)

(従来、 U-Boot から Kernel を起動するときは `arch/arm/boot/uImage` を使っていたが、これは使わない。)

##### Task Description

    $ git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
    $ cd linux
    $ git checkout v3.13

でソース取得して、v3.13 タグをチェックアウト。
（何かあっても自分で対処できる人は masterブランチでやってもOK）

以下のようにして ARMv7 Multi な設定にする。

    $ make ARCH=arm multi_v7_defconfig

ここで

    $ make ARCH=arm menuconfig

をして、少々設定をいじる。

    Device Drivers  --->
      Block devices  --->
        [*] RAM block device support
         (16384)  Default RAM disk size (kbytes)

のようにたどり、 `RAM block device support` にチェックを入れ、
`Default RAM disk size` を `16384` に設定する。

もう一つ

    Device Drivers  --->
      Character devices --->
        [ ] Legacy (BSD) PTY support

のようにたどり、`Legacy (BSD) PTY support` のチェックを外す。

あとは

    $ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-

でビルド。

### STEP3: Ramdisk のダウンロード

##### Input Files Required

  - None

##### Output Files Produced

  - `arm_ramdisk.image.gz`: Kernel がマウントする init ramdisk (を gzip 圧縮したもの)

##### Task Description

[こちら](http://www.wiki.xilinx.com/Build+and+Modify+a+Rootfs) から
arm_ramdisk.image.gz をダウンロードする。

### STEP4: ITB (Image Tree Blob) の生成

##### Input Files Required

  - `linux/arch/arm/boot/zImage`
  - `arm_ramdisk.image.gz`
  - `linux/arch/arm/boot/dts/zynq-zc706.dtb`
  - `u-boot/tools/mkimage`

カレントディレクトリから見て、上記の配置になっているとする。

##### Output Files Produced

  - `fit.itb`: U-Boot から Kernel を起動するためのイメージ

##### Task Description

Kernel Image, Ramdisk, DTB (Device Tree Blob) を一つにまとめた ITB というのを作ります。
ITB を作るには、 ITS(Image Tree Source) を記述して、 `mkimage` に食わせます。

以下の内容を `fit.its` というファイルに記述する。

    /dts-v1/;
    
    / {
            description = "Kernel, ramdisk and FDT blob";
            #address-cells = <1>;
    
            images {
                    kernel@1 {
                            description = "Linux Kernel 3.13 configured with multi_v7_defconfig";
                            data = /incbin/("linux/arch/arm/boot/zImage");
                            type = "kernel";
                            arch = "arm";
                            os = "linux";
                            compression = "none";
                            load = <0x00008000>;
                            entry = <0x00008000>;
                            hash@1 {
                                    algo = "md5";
                            };
                    };
    
                    ramdisk@1 {
                            description = "Ramdisk for Zynq";
                            data = /incbin/("arm_ramdisk.image.gz");
                            type = "ramdisk";
                            arch = "arm";
                            os = "linux";
                            compression = "gzip";
                            load = <0x00000000>;
                            entry = <0x00000000>;
                            hash@1 {
                                    algo = "sha1";
                            };
                    };
    
                    fdt@1 {
                            description = "FDT for ZC706";
                            data = /incbin/("linux/arch/arm/boot/dts/zynq-zc706.dtb");
                            type = "flat_dt";
                            arch = "arm";
                            compression = "none";
                            hash@1 {
                                    algo = "crc32";
                            };
                    };
    
            };
    
            configurations {
                    default = "config@1";
    
                    config@1 {
                            description = "Zynq ZC706 Configuration";
                            kernel = "kernel@1";
                            ramdisk = "ramdisk@1";
                            fdt = "fdt@1";
                    };
            };
    };

あとは以下のようにすれば、`fit.itb` ができる。

    $ u-boot/tools/mkimage -f fit.its fit.itb

### STEP5: JTAG (Slave Boot) から U-Boot と Linux を起動する

##### Input Files Required

  - `u-boot-dtb.bin`: STEP1 で作成したもの
  - `fit.itb`: STEP4 で作成したもの
  - `xmd`: Vivado (または ISE)のインストールディレクトリに入っている
  - `ps7_init.tcl`: Vivado (または ISE)から "Export Hardware for SDK" を実行すると出力される
  - `stub.tcl`: Xilinx のページからダウンロードできる `ug873-design-files.zip` の中に入っている
  - `fpga.bit`: Vivado (または ISE)で生成した FPGA bit file (Optional)

##### Task Description

`fit.itb` を TFTP の公開ディレクトリに置く。（TFTP 環境のない人はスキップして下さい）

Zynq ボードのブートモードの選択スイッチを JTAG に合わせて電源入れる。JTAG でボードと接続し、XMD を開く。

    $ xmd 

XMD のプロンプトから以下を実行する。(FPGA は必要なければダウンロードしなくても良い)

    XMD% connect arm hw                         # Open JTAG connection
    XMD% rst -slcr                              # Reset the whole system
    XMD% fpga -f fpga.bit                       # Download FPGA bit file (Optional)
    XMD% source ps7_init.tcl
    XMD% ps7_init                               # Initialize DDR, IO pins, etc.
    XMD% ps7_post_config                        # Enable level shifter
    XMD% source stub.tcl                        # start CPU1
    XMD% targets 64                             # connect to CPU0
    XMD% dow -data u-boot-dtb.bin 0x04000000    # Download u-boot to address 0x04000000 
    XMD% con 0x04000000                         # start CPU0 from address 0x04000000

なお、毎回これを打ち込むのも面倒ですので、 `foo.tcl` に書いておきましょう。

    XMD% source foo.tcl

で XMD から読み込むか、シェルから

    $ xmd -tcl foo.tcl

とすればよいです。

U-Boot のプロンプトが出た後、放っておくと、自動的に TFTPサーバーから `fit.itb`
をダウンロードして、 Linux が起動する。

TFTP サーバーがない場合は、`con 0x04000000` の前に

    XMD% dow -data fit.itb 0x02000000

とすれば、 JTAG 経由で `fit.itb` をダウンロードできるので(時間かかりますが、、)
あとは U-Boot のプロンプトから

    > bootm 2000000

と入力して Linux を起動させる。

### STEP6: SDカード用のブートイメージを作成する

##### Input Files Required

  - `fsbl.elf`: FSBL (First Stage Boot Loader)。XSDK で生成。
  - `fpga.bit`: FPGA bit file (Optional)
  - `u-boot/u-boot-dtb.bin`: STEP1 で作成したもの
  - `bootgen`: Vivado (または ISE)のインストールディレクトリに入っている

##### Output Files Produced

  - `boot.bin`

##### Task Description

`foo.bif` というファイル(名前適当でよい)に以下のように記述する。

    image:
    {
            [bootloader]fsbl.elf
            fpga.bit
            [load=0x04000000]u-boot/u-boot-dtb.bin
    }

FPGA Bit file のダウンロードが不要なら `fpga.bit` の行は削除してよい。

    $ bootgen -image foo.bif -w on -o boot.bin

とすると、`boot.bin` ができる。SDカードのブートイメージは必ず `boot.bin`
というファイル名でないといけないので注意する。

### Step7: SDカードから U-Boot と Linux Kernel を起動する

##### Input Files Required

  - `boot.bin`: STEP6 で作成したもの
  - `fit.itb`: STEP4 で作成したもの

##### Task Description

FAT でフォーマットしたSDカードに `boot.bin` と `fit.itb` をコピー。

SDカードを Zynq ボードに挿し、ブートモードの選択スイッチを SD カードに合わせて電源入れる。

### ZC706 以外のボードの場合

動作確認はしていないが、上記とほぼ同じやり方でできずはず。

U-Boot のビルドの部分が、ZC702 ボードなら

    $ make zynq_zc70x CROSS_COMPILE=arm-linux-gnueabi-

ZED ボードならば

    $ make zynq_zed CROSS_COMPILE=arm-linux-gnueabi-

となる。

U-Boot の `boards.cfg` というファイルを見れば、自分のボードに対応するコマンドがわかる。

Linux の Device Tree も `arch/arm/boot/dts` ディレクトリに各ボードごとの DTB
ができているので、それを使う。

### その他参照資料

  - Device Tree を用いた U-Boot のコンフィグレーションについて知りたい場合は
    U-Boot ソースツリーの `doc/README.fdt-control` を参照

  - ITS (Image Tree Source) の書き方を知りたい場合は U-Boot ソースツリーの
    `doc/uImage.FIT/` 以下のドキュメント参照

  - XMD の使い方は Xilinx の資料 "Embedded System Tools Reference Manual" 参照

  - FSBL の作り方は Xilinx の資料
    "Zynq-7000 All Programmable SoC: Concepts, Tools, and Techniques" 5章を参照

  - Bootgen をもっと詳しく知りたい場合は Xilinx の資料
    "Zynq-7000 All Programmable SoC Software Developers Guide" 参照

### ご注意

ここに記載の内容は 2014年1月時点のソースでのやり方ですので、今後もこのやり方が通用するかはわかりません。
コードはめまぐるしく変わっていきますので。
