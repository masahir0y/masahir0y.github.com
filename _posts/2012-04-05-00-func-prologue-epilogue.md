---
layout: post
title: 関数のプロローグとエピローグ
description: ""
tags: [ARM, C]
---
{% include JB/setup %}

関数のプロローグとエピローグにどういう意味があるのか調べてみました。

とりあえず、ARM の gcc コンパイラで実験してみた。

最適化オプション (`-O`) はつけてない。

    void hoge()
    {
    }

をコンパイルすると

    <hoge>:
        push {fp}  ; (str fp, [sp, #-4]!)
        add fp, sp, #0
        add sp, fp, #0
        pop {fp}  ; (ldr fp, [sp], #4)
        bx lr

となった。

`fp` は `r11` のこと。フレームポインタの意味らしい。

- 関数に入るとき（プロローグ）に `fp` をスタックに退避し、 `sp` の値を `fp` にコピー。
- 関数から出るとき（エピローグ）に `fp` を `sp` にコピーしたあと、 `fp` を復元。

となっているが、これだけ見ると、なんでこんな周りくどいことするのかよく理解できない。

そこで

    int hoge()
    {
            int a = 10;
            return a;
    }

をコンパイルしてみた。

    <hoge>:
        push {fp}  ; (str fp, [sp, #-4]!)
        add fp, sp, #0
        sub sp, sp, #12
        mov r3, #10
        str r3, [fp, #-8]
        ldr r3, [fp, #-8]
        mov r0, r3
        add sp, fp, #0
        pop {fp}  ; (ldr fp, [sp], #4)
        bx lr

となった。

これを見ると、 `sp` を `fp` へコピーした後、スタックを `12` 伸ばしている。
この時点で、 `sp = fp - 12` という関係がある。

例えば、関数に入った時点の `sp` の値が `0x80000000` だったとして、
スタックの使い方は

    0x7ffffff0: 未使用   <-- sp の指す位置
    0x7ffffff4: 変数 a
    0x7ffffff8: 未使用
    0x7ffffffc: 復帰用fp <-- fp の指す位置

となっているはず。

さらに

    int hoge()
    {
            int a = 10, b = 20;
            return a + b;
    }

をコンパイルしてみると

    <hoge>:
        push {fp}  ; (str fp, [sp, #-4]!)
        add fp, sp, #0
        sub sp, sp, #12
        mov r3, #10
        str r3, [fp, #-8]
        mov r3, #20
        str r3, [fp, #-12]
        ldr r2, [fp, #-8]
        ldr r3, [fp, #-12]
        add r3, r2, r3
        mov r0, r3
        add sp, fp, #0
        pop {fp}  ; (ldr fp, [sp], #4)
        bx lr

となった。

スタックの使い方は

    0x7ffffff0: 変数 b   <-- sp の指す位置
    0x7ffffff4: 変数 a
    0x7ffffff8: 未使用
    0x7ffffffc: 復帰用fp <-- fp の指す位置

さらにさらに

    int hoge()
    {
            int a = 10, b = 20, c = 30;
            return a + b + c;
    }

は

    <hoge>:
        push {fp}  ; (str fp, [sp, #-4]!)
        add fp, sp, #0
        sub sp, sp, #20
        mov r3, #10
        str r3, [fp, #-8]
        mov r3, #20
        str r3, [fp, #-12]
        mov r3, #30
        str r3, [fp, #-16]
        ldr r2, [fp, #-8]
        ldr r3, [fp, #-12]
        add r2, r2, r3
        ldr r3, [fp, #-16]
        add r3, r2, r3
        mov r0, r3
        add sp, fp, #0
        pop {fp}  ; (ldr fp, [sp], #4)
        bx lr

となった。

スタックは 

    0x7fffffe8: 未使用   <-- sp の指す位置
    0x7fffffec: 変数 c
    0x7ffffff0: 変数 b
    0x7ffffff4: 変数 a
    0x7ffffff8: 未使用
    0x7ffffffc: 復帰用fp <-- fp の指す位置

となっているはず。

以上を見ると、

- `fp` と `sp` の間に自動変数を確保しているらしい。
- `sp` の伸ばす量は 8 ずつ増えている。

という規則があることがわかった。

ただし、上記の例で、 `0x7ffffff8` の位置が常に未使用になっているのがちょっと不思議です。

そこで、以下の関数をコンパイルしてみて納得。

    int hoge()
    {
            int a = 10, b = 20;
            return piyo(a, b);
    }

    <hoge>:
        push {fp, lr}
        add fp, sp, #4
        sub sp, sp, #8
        mov r3, #10
        str r3, [fp, #-8]
        mov r3, #20
        str r3, [fp, #-12]
        ldr r0, [fp, #-8]
        ldr r1, [fp, #-12]
        bl <piyo>
        mov r3, r0
        mov r0, r3
        sub sp, fp, #4
        pop {fp, pc}


となった。

なるほど、この場合は `fp` と `lr` の両方を退避しておく必要があるのね。

    0x7ffffff0: 変数 b   <-- sp の指す位置
    0x7ffffff4: 変数 a
    0x7ffffff8: 復帰用fp
    0x7ffffffc: lr       <-- fp の指す位置

ちなみに、 ABI によると、 `r0-r3`, `r12` は復元しなくてよいレジスタ、 `r4-r11` は復元しなくてはならないレジスタです。
