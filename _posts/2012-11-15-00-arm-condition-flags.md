---
layout: post
title: ARM 命令の条件ビット
description: ""
tags: [ARM]
---
{% include JB/setup %}

ARM 命令の条件フィールドについてまとめておきます。

  - N フラグ: 命令の結果を符号付き整数と見て負のときにセット、その他の時にクリアされる
  - Z フラグ: 命令の結果がゼロであるときにセット、その他の時にクリアされる。
  - C フラグ: 比較命令(`CMN`, `CMP`)を含む加減算においてキャリー、
    またはボローが発生したときにセット、その他の時にクリアされる。
    また論理演算時のシフト動作ではシフトによるキャリアウトの結果が C フラグに反映される。
  - `V`フラグ: 加減算命令において符号付きオーバーフローが発生するとセット、
    それ以外の時に零セットされる。

`ADDS` 命令の場合、

    `ADDS <Rd>, <Rn>, <shifter_operand>`

の動作としては以下のようになる。ただし、 `Rd` が `r15(pc)` ではない場合。

    Rd = Rn + shifter_operand
    N Flag = Rd[31]
    Z Flag = if Rd == 0 then 1 else 0
    C Flag = CarryFrom(Rn + shifter_operand)
    V Flag = OverflowFrom(Rn + shifter_operand)

`SUBS` 命令の場合、

    SUBS <Rd>, <Rn>, <shifter_operand>

の動作としては以下のようになる。ただし、 `Rd` が `r15(pc)` ではない場合。

    Rd = Rn - shifter_operand
    N Flag = alu_out[31]
    Z Flag = if alu_out == 0 then 1 else 0
    C Flag = NOT BorrowFrom(Rn - shifter_operand)
    V Flag = OverflowFrom(Rn - shifter_operand)

`CMP` 命令は結果をレジスタにセットしないが、フラグのセットのされ方は `SUBS` 命令と同じ。

C は符号なし整数として、Carry/Borrow が発生したかを示すのに対し、

V は符号あり整数として、Overflow 発生したかを示している。

##### 命令を条件付きで実行する場合の条件フィールド #####

    cond Mnemonic Meaning                       Condition flags
    -----------------------------------------------------------
    0000 EQ       Equal                         Z==1
    0001 NE       Not equal                     Z==0
    0010 CS(HS)   Carry set                     C==1
    0011 CC(LO)   Carry Clear                   C==0
    0100 MI       Minus, negative               N==1
    0101 PL       Plus, positive or zero        N==0
    0110 VS       Overflow                      V==1
    0111 VC       No Overflow                   V==0
    1000 HI       Unsigned higher               C==1 and Z==0
    1001 LS       Unsigned lower or same        C==0 or Z==1
    1010 GE       signed greater than or equal  N==V
    1011 LT       signed less than              N!=V
    1100 GT       signed greater than           Z==0 or N!=V
    1101 LE       Signed less than or equal     Z==1 or N!=V
    1110 None(AL) Always                        Any

  - HS (Unsigned higher or same) は CS
  - LO (Unsigned lower)は CC

と同じ。

やってしまいがちなのは、符号なし整数の比較（例えばアドレスの比較）をしたあと、
条件判定に `lt` や `gt` などを使っているケース。

組み込み系でよくあるブートシーケンスで RAM へコードを展開しているコードですが。。

    copy_loop:
            ldmia  r0!, {r3 - r10}    @ copy from source address [r0]
            stmia  r1!, {r3 - r10}    @ copy to   target address [r1]
            cmp    r0, r2             @ until source end addreee [r2]
            ble    copy_loop

ここで `r1` に `0x70000000`, `r2` に `0x80000000` が入っていたりすると、
意図せずに、ループから抜けてしまいます。

`le` は `signed` での比較になるわけだから、
`0x70000000` と `0x80000000` を比較して「小さい」とはならずに
`0x70000000` と `-0x80000000` を比較して 「大きい」という判定になるはずだから。

というわけで、この場合
`ble` ではなくて `bls` を使うべきですね。
