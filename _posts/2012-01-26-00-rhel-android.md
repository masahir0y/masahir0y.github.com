---
layout: post
title: RHELでAndroidビルド
description: ""
tags: [Android, RHEL/CentOS]
---
{% include JB/setup %}

Android のビルドには Ubuntuを使うのが簡単ですが、会社では一応 RHEL を使えとなっている。

そこで、 RHEL6 64bit 上で Androidをビルドできるように環境を作ってみたので、その時のメモです。

Ubuntuでのビルドの仕方は [Android Open Source Project](http://source.android.com/) に書いてあるので参考に。

まずは repo で Android のソースコードを取ってきます。
 
「Initializing the Build Enrironment」のページを見ながら、ビルドに必要なパッケージを入れていきます。

git, gnupg, flex, bison, gperf, zip, curl あたりは `yum install` ですんなり入った。
あとは Ubuntu と同じ名前のパッケージが rpm にないっぽい。

とりあえず、無視して先に進むことにして、エラーが出るたびに、メッセージを見ながら必要なものを入れていくという作戦にした。

`lunch` すると

    /lib/ld-linux.so.2: bad ELF interpreter: そのようなファイルやディレクトリはありません

とエラー。

    # yum install ld-linux.so.2

ですんなり入った。

また、javac が入ってないので、ORACLEのダウンロードページから 「Java SE6 Update 30」 の rpm を取ってきてインストールした。
RHEL には OpenJDKというのが最初から入っているけど、これはダメみたいです。

`build/core/main.mk` のコード見てみると 

    # Check for the correct version of java
    java_version := $(shell java -version 2>&1 | head -n 1 | grep '^java .*[ "]1\.6[\. "$$]')
    ifneq ($(shell java -version 2>&1 | grep -i openjdk),)
    java_version :=
    endif
    ifeq ($(strip $(java_version)),)

という感じで java コマンドのバージョンをチェックする行があって、OpenJDKははじかれるようになっています。

続いて、make すると

    /bin/bash: g++: コマンドが見つかりません

とエラー。

g++ を提供するパッケージ名を知りたいのだが、こういう時は `yum whatprovides` が便利。

    yum whatprovides */g++

と打つと、 gcc-c++ を入れればよいことがわかる。

さらに進むと、今度は

    /usr/include/gnu/stubs.h:7:27: error: gnu/stubs-32.h: そのようなファイルやディレクトリはありません。

`yum whatprovides` で調べると、対応するパッケージは glibc-devel.i686 らしい。

glibc-devel.x86_64 はすでにインストールされていたが、 glib-devel.i686 の方もインストールしたところ stubs-32.h が入った。

さらに進んで、

    /usr/bin/ld: skipping incompatible /usr/lib/gcc/x86_64-redhat-linux/4.4.6/libstdc++.a when searching for -lstdc++
    /usr/bin/ld: cannot find -lstdc++

というエラー。 

libstdc++ は既に入っていたが libstdc++.x86_64 ではダメで、libstdc++.i686 がいるようだ。

その後も途中で何度か止まったが、エラーメッセージを見ながら `yum whatprovides` でパッケージ名を調べて、
zlib-devel.i686, ncurses-devel.i686, libX11-devel.i686, mesa-libGL-devel.i686
あたりを `yum install`。

これを繰り返しているうちにとりあえず、ビルドが完了したようだ。

ちょっと不安だけど、一応OKということにしとこう。

今回勉強になったのは2点。

どのパッケージを入れるかを覚えても応用が効かないので、やり方を覚えておくこと。 

1. 必要なパッケージ名がわからんときは `yum whatprovides` で調べる。
2. (64ビットマシンを使っていて).x86_64 を入れててうまくいかないときは .i686 を入れる。
