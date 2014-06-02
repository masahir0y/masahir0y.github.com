---
layout: post
title: キーボードの LED を光らせるドライバ
description: ""
tags: [Linux Kernel]
---
{% include JB/setup %}

「Linuxデバイスドライバプログラミング」の 7-2 章にキーボードの LED
を光らせるやり方が書いてあったので、その通りやってみたけど、あれれ？光らないよ。。

そこで、ちょっと調べてみました。

PS/2規格は以下のページを参考にさせてもらいました。

[PS/2 インターフェイスの研究](http://hp.vector.co.jp/authors/VA037406/html/ps2interface.htm)

0x64 のステータスバイトを polling するのがミソらしい。

以下のコードでできました。

    #include <linux/init.h>
    #include <linux/module.h>
    #include <asm/io.h>

    MODULE_LICENSE("GPL");

    static int sample_init(void)
    {
            /* wait while buffer if full*/
            while (inb(0x64) & 2);
            /* set mode indicator */
            outb(0xED, 0x60);
            /* wait while buffer if full*/
            while (inb(0x64) & 2);
            /* bit1: Num Lock */
            outb(0x02, 0x60);

            return 0;
    }

    static void sample_exit(void)
    {
            while (inb(0x64) & 2);
            outb_p(0xED, 0x60);
            while (inb(0x64) & 2);
            outb_p(0x00, 0x60);
    }

    module_init(sample_init);
    module_exit(sample_exit);


`insmod` で Num Lock が光ります。
`rmmod` で消えます。

`outb(0x02, 0x60);` の行を `outb(0x04, 0x60);` や `outb(0x01, 0x60);` にすると
Caps Lock や Scroll Lock が光ります。
