---
layout: post
title: register 変数
description: ""
tags: [ARM, C]
---
{% include JB/setup %}

[前回]({{ BASE_PATH }}/2012/04/05/00-func-prologue-epilogue/)に引き続いて、 レジスタ変数を使った時の逆アセがどうなるかを見てみた。
ARM でやっています。

    int hoge()
    {
            register int a = 10, b = 20;
            return a + b;
    }

をコンパイルしてみると

    <hoge>:
        push {r4, r5, fp}
        add fp, sp, #8
        mov r5, #10
        mov r4, #20
        add r3, r5, r4
        mov r0, r3
        sub sp, fp, #8
        pop {r4, r5, fp}
        bx lr

なるほど、変数 `a`, `b` を `r5`, `r4` に取るのね。
`r4` - `r11` は復元の必要があるので、いったんスタックに退避してます。

では、 register 変数は何個まで取るんだろうという素直に思ったので、やってみた。

    int hoge()
    {
            register int a = 10, b = 20, c = 30, d = 40;
            register int e = 50, f = 60, g = 70, h = 80;
            return a + b + c + d + e + f + g + h;
    }

をコンパイルしてみると

    <hoge>:
        push {r4, r5, r6, r7, r8, r9, sl, fp}
        add fp, sp, #28
        sub sp, sp, #8
        mov r2, #10
        str r2, [fp, #-32] ; 0xffffffe0
        mov r9, #20
        mov sl, #30
        mov r8, #40 ; 0x28
        mov r7, #50 ; 0x32
        mov r6, #60 ; 0x3c
        mov r5, #70 ; 0x46
        mov r4, #80 ; 0x50
        ldr r2, [fp, #-32] ; 0xffffffe0
        add r3, r2, r9
        add r3, r3, sl
        add r3, r3, r8
        add r3, r3, r7
        add r3, r3, r6
        add r3, r3, r5
        add r3, r3, r4
        mov r0, r3
        sub sp, fp, #28
        pop {r4, r5, r6, r7, r8, r9, sl, fp}
        bx lr

変数 `b` ～ `h` は `r4` ～ `r10` に割り当てられているが、変数 `a` はレジスタに取れずに、スタックに確保しているみたい。

ちなみに `sl` は `r10` のこと。

レジスタの別名をまとめてみた。

- `r9` = `sb`: スタティックベース
- `r10` = `sl`: スタックリミット
- `r11` = `fp`: フレームポインタ
- `r12` = `ip`: プロシージャコール内スクラッチ
- `r13` = `sp`: スタックポインタ
- `r14` = `lr`: リンクレジスタ
- `r15` = `pc`: プログラムカウンタ
