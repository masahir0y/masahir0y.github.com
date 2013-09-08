---
layout: post
title: バイナリファイルを逆アセンブル
description: ""
tags: []
---
{% include JB/setup %}

elf とかではなく、バイナリーイメージになっている場合。

i386の場合

    $ objdump -b binary -m i386 -D foo.bin

ARMの場合

    $ arm-none-eabi-objdump -b binary -m arm -D foo.bin

でできた。
