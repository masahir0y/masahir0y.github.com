---
layout: post
title: scull ドライバ
description: ""
tags: [RHEL/CentOS]
---
{% include JB/setup %}

O'REILLY の LINUXデバイスドライバ 第3版を勉強中。

サンプルコードは以下からダウンロードできました。  
<ftp://ar.linux.it/pub/ldd3/>

scull を make してみるものの、Kernelのversionによってちょこちょこ直さないといけない。

修正箇所を以下にメモ。(手元の Kernel のバージョンは 2.6.32-40)

Makefile の

    CFLAGS += $(DEBFLAGS)
    CFLAGS += -I$(LDDINC)

を

    EXTRA_CFLAGS += $(DEBFLAGS)
    EXTRA_CFLAGS += -I$(LDDINC)

に修正。

`main.c` の

    #include <linux/config.h>

を

    #include <linux/autoconf.h>

に変更。

`pipe.c` と `access.c` に

    #include <linux/sched.h>

を追加。

`access.c` 中の `current->uid` と `current->euid` をそれぞれ `current->cred->uid` と `current->cred->euid` に変更。

これで makeが通った。

    $ sudo ./scull_load

とするとエラーが出るので、

    major=$(awk "\\$2==\"$module\" {print \\$1}" /proc/devices)

を

    major=$(awk "\$2==\"$module\" {print \$1}" /proc/devices)

に修正するとうまくいきました。
