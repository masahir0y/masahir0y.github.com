---
layout: post
title: U-Boot と Linux Kernel のメインラインで Zynq を動かす 2014年4月版
description: ""
tags: [Zynq, Linux Kernel, U-Boot]
---
{% include JB/setup %}

[以前]({{ BASE_PATH }}/2014/01/21/00-uboot-linux-on-zynq/)、2014年1月時点での
U-Boot と Linux のメインラインで Zynq を動かす方法を紹介しました。

その後、コード(特に U-Boot)が大きく変わっているので、
2014年4月時点での動かし方を紹介することにします。

使用するのは、

- U-Boot 2014.04
- Linux Kernel 3.14
- Zynq ZC706 ボード

です。

その前に、 Zynq のブートシーケンスを復習しておきましょう。
Active Boot (= JTAG ブート以外)は

1. Boot ROM

2. FSBL

3. U-Boot

4. Linux Kernel

というのが、Xilinx が公式にサポートしているブートシーケンスで、
[Xilinx Wiki ページ](http://www.wiki.xilinx.com/)もこのやり方を紹介しています。

最近の U-Boot では

1. Boot ROM

2. U-Boot SPL

3. U-Boot

4. Linux Kernel

というブートシーケンスも可能になっています。

SPL というのは Secondary Progmam Loader の略です。
DRAM 等のメモリを初期化し、U-Boot 本体を NAND, MMC 等のデバイスからロードするための、
より小さなブートローダーといったものです。
U-Boot の標準のインフラとして用意されています。
詳しく知りたい人は U-Boot の `doc/README.SPL` を読んでください。

これを選択するメリットは、 FSBL (First Stage Boot Loader) を生成するために、
XSDK (Xilinx SDK) を起動しなくても済む、ということです。

さらに簡略化した

1. Boot ROM

2. U-Boot SPL

3. Linux Kernel

というブートシーケンスもあります。 (Falcon ブートといいます)

これのメリットは U-Boot 本体をスキップすることで、より高速に Linux を起動できることです。

ただし、現時点ではサポートが限定的でまともに動かすのは難しいので、今回は割愛します。
おそらく、そう遠くないうちにまともに動かせるようになると思いますが。

以下では、 U-Boot 2014.04 から可能になった SPL を用いたブートのやり方を紹介します。

### STEP1: U-Boot のビルド

##### Input Files Required

- ARM Cross Compiler
- `ps7_init.c`: ISE / Vivado で "Export Hardware for SDK" を実行すると出力される
- `ps7_init.h`: ISE / Vivado で "Export Hardware for SDK" を実行すると出力される

##### Output Files Produced

- `u-boot.bin`: U-Boot 本体の RAWバイナリ
- `u-boot.img`: `u-boot.bin` に uImage ヘッダーをつけたもの
- `spl/u-boot-spl.bin`: U-Boot SPLの RAWバイナリ
- `tools/mkimage`: u-boot で扱うイメージを生成するツール。

##### Task Description

    $ git clone git://git.denx.de/u-boot.git
    $ cd u-boot
    $ git checkout v2014.04

でソース取得して、v2014.04 タグをチェックアウト。
（何かあっても自分で対処できる人は masterブランチでやってもOK）

まず、少々ソースコードをいじらなくてはなりません。

`include/configs/zynq-common.h` を開き、以下のように
`CONFIG_OF_CONTROL` ～ `CONFIG_RSA` までを無効にする。

    --- a/include/configs/zynq-common.h
    +++ b/include/configs/zynq-common.h
    @@ -199,6 +199,7 @@
     #define CONFIG_FIT
     #define CONFIG_FIT_VERBOSE     1 /* enable fit_format_{error,warning}() */
     
    +#if 0
     /* FDT support */
     #define CONFIG_OF_CONTROL
     #define CONFIG_OF_SEPARATE
    @@ -207,6 +208,7 @@
     /* RSA support */
     #define CONFIG_FIT_SIGNATURE
     #define CONFIG_RSA
    +#endif
     
     /* Extend size of kernel image for uncompression */
     #define CONFIG_SYS_BOOTM_LEN   (20 * 1024 * 1024)

なんで、上記を無効にするかというと、前回の U-Boot 2014.01 では U-Boot を
DeviceTree 付きで動かしたのですが、コードのマージが中途半端に行われたために、
2014.04 で再び `CONFIG_OF_CONTROL` が動かなくなってしまったためです。

頑張って動かすことはできるのですが、 Linux Kernel から DeviceTree の記述をいろいろと持ってこなくてはいけなかったり、
SPL から u-boot.img + DeviceTree をロードするのにコード修正したりと、
いろいろと修正箇所が多いので、無効にしてしまった方が楽です。

また、あとで、Kernel を TFTP でダウンロードしたいので、`include/configs/zynq-common.h`
の適当なところに

    #define CONFIG_IPADDR   192.168.11.2
    #define CONFIG_SERVERIP 192.168.11.1
    #define CONFIG_ETHADDR  00:0a:35:00:01:22

の3行を足す。

`CONFIG_IPADDR` は Zynqボードに割り振る IPアドレス、 `CONFIG_SERVERIP` は
TFTP サーバーのアドレスに合わせて下さい。
MACアドレスは、(他のネットワーク機器と被らなければ)適当でいい。

TFTP サーバーがなくても、動かすことはできるので、ない人は上記はスキップして下さい。

さらに、 `ps7_init.c`, `ps7_init.h` を U-Boot の `board/xilinx/zynq` ディレクトリにコピーする。
このファイルが、 FSBL の代わりをするための、肝になるファイルです。

また

    touch board/xilinx/xil_io.h

で、空の `xil_io.h` を作る。
(`ps7_init.c` が `xil_io.h` をインクルードしているので、これがないとエラーになる。)

あとは

    $ make zynq_zc70x_config
    $ make CROSS_COMPILE=arm-linux-gnueabi-

のようにして、コンフィグレーションとビルドをする。

もしくは

    $ make zynq_zc70x CROSS_COMPILE=arm-linux-gnueabi-

のように 1行で、コンフィグレーションとビルドを同時にすることもできる。

`ps7_init.c` と `ps7_init.h` が warning を出しますが、気にしなくてもOK。
気になる人は、関数のプロトタイプの引数部に `void` を足してください。

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
    $ git checkout v3.14

でソース取得して、v3.14 タグをチェックアウト。
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

- `u-boot.bin`: STEP1 で作成したもの
- `fit.itb`: STEP4 で作成したもの
- `xmd`: ISE / Vivado のインストールディレクトリに入っている
- `ps7_init.tcl`: ISE / Vivado から "Export Hardware for SDK" を実行すると出力される
- `stub.tcl`: Xilinx のページからダウンロードできる `ug873-design-files.zip` の中に入っている
- `fpga.bit`: ISE / Vivado で生成した FPGA bit file (Optional)

##### Task Description

`fit.itb` を TFTP の公開ディレクトリに置く。（TFTP 環境のない人はスキップして下さい）

Zynq ボードのブートモードの選択スイッチを JTAG に合わせて電源入れる。JTAG でボードと接続し、XMD を開く。

    $ xmd 

XMD のプロンプトから以下を実行する。(FPGA は必要なければダウンロードしなくても良い)

    XMD% connect arm hw                         ;# Open JTAG connection
    XMD% rst -slcr                              ;# Reset the whole system
    XMD% fpga -f fpga.bit                       ;# Download FPGA bit file (Optional)
    XMD% source ps7_init.tcl
    XMD% ps7_init                               ;# Initialize DDR, IO pins, etc.
    XMD% ps7_post_config                        ;# Enable level shifter
    XMD% source stub.tcl                        ;# start CPU1
    XMD% targets 64                             ;# connect to CPU0
    XMD% dow -data u-boot.bin 0x04000000        ;# Download u-boot to address 0x04000000
    XMD% con 0x04000000                         ;# start CPU0 from address 0x04000000

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

- `u-boot/spl/u-boot-spl.bin`: STEP1 で作成したもの
- `bootgen`: ISE / Vivado のインストールディレクトリに入っている

##### Output Files Produced

- `boot.bin`

##### Task Description

`foo.bif` というファイル(名前適当でよい)に以下のように記述する。

    image:
    {
            [bootloader,load=0x00000000,startup=0x00000000]u-boot/spl/u-boot-spl.bin
    }

そして

    $ bootgen -image foo.bif -w on -o boot.bin

とすると、`boot.bin` ができる。SDカードのブートイメージは必ず `boot.bin`
というファイル名でないといけないので注意する。

### STEP6B: SDカード用のブートイメージを作成する (もうちょっと簡単なやり方)

##### Input Files Required

- `u-boot/spl/u-boot-spl.bin`: STEP1 で作成したもの

##### Output Files Produced

- `boot.bin`

##### Task Description

`bootgen` を使わずに `boot.bin` を作成する方法を紹介します。

u-boot-xlnx のコードを取ってきます。

    git clone git://github.com/Xilinx/u-boot-xlnx.git

`tools` ディレクトリの下に `zynq-boot-bin.py` という Python スクリプトが
入っているので、これを `~/bin` かどこか適当な PATH にコピーする。

あとは

    zynq-boot-bin.py -o boot.bin -u u-boot/spl/u-boot-spl.bin

とすれば、 `boot.bin` ができます。
BIF ファイルを記述しなくてもいい分、こちらの方が簡単だと思います。

なお、 u-boot-xlnx だと、 `zynq-boot-bin.py` が Makefile からフックされていて、
make すると自動で `boot.bin` までできるので、更に楽なのですが、
(しかもローカルでいろいろとソースをいじくらなくても動く)
ここのページではあくまで、メインラインでやることを目指しています。

### Step7: SDカードから U-Boot と Linux Kernel を起動する

##### Input Files Required

- `boot.bin`: STEP6 または STEP6B で作成したもの
- `u-boot.img`: STEP1 で作成したもの
- `fit.itb`: STEP4 で作成したもの

##### Task Description

FAT でフォーマットしたSDカードに `boot.bin`, `u-boot.img`, `fit.itb` をコピー。

SDカードを Zynq ボードに挿し、ブートモードの選択スイッチを SD カードに合わせて電源入れる。

### STEP8: FSBL を使ったブートシーケンス

上記の通り、 SPL を使ったブートシーケンスを紹介しましたが、
従来通りの FSBL を使ったブートシーケンスも可能です。

その場合は U-Boot に `ps7_init.c` と `ps7_init.h` をコピーしたり、
`xil_io.h` を作ったりする必要はないです。

STEP6 の部分を以下のようにアレンジすればよいです。

##### Input Files Required

- `fsbl.elf`: FSBL (First Stage Boot Loader)。XSDK で生成。
- `fpga.bit`: FPGA bit file (Optional)
- `u-boot/u-boot.bin`: STEP1 で作成したもの
- `bootgen`: ISE / Vivado のインストールディレクトリに入っている

##### Output Files Produced

- `boot.bin`

##### Task Description

`foo.bif` というファイル(名前適当でよい)に以下のように記述する。

    image:
    {
            [bootloader]fsbl.elf
            fpga.bit
            [load=0x04000000,startup=0x04000000]u-boot/u-boot.bin
    }

FPGA Bit file のダウンロードが不要なら `fpga.bit` の行は削除してよい。

あとは

    $ bootgen -image foo.bif -w on -o boot.bin

とすると、`boot.bin` ができるので、
FAT でフォーマットしたSDカードに `boot.bin` と `fit.itb` をコピー。

### 今後の開発の行方

Zynq は U-Boot の中でも、特に頻繁に更新されている SoC で、今後もやり方が変わっていく可能性が高いです。
今回、ローカルでソースをいじくっている部分のいくつかは、修正パッチを投稿しておいたので、
次のリリースではもうちょっと楽になるはず。

U-Boot 2014.04 ではまともに動いてませんが、 DeviceTree を使った
U-Boot のコンフィグレーションに移行していくでしょう。

最終的には、 `ps7_init.c` と `ps7_init.h` をコピーすることもなくなり、
すべての情報を DeviceTree から読み込むようになるだろう。
(と、コードを書いている Xilinx のエンジニアは語っておりました。)
