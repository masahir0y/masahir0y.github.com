---
layout: post
title: RAM を一切使わずにプリントデバッグ
description: ""
tags: []
---
{% include JB/setup %}

Linux Kernel のコードを見ていると、 `arch/arm/kernel/debug.S` にかなりローレベルなプリントデバッグ用のサブルーチンがありますね。

これをちょっと真似して、より少ない資源で、ARM の起動初期のプリントデバッグができるように改良したものを作ってみました。

- `printhex2`: 2桁の16進数を表示する
- `printhex4`: 4桁の16進数を表示する
- `printhex8`: 8桁の16進数を表示する
- `printunsigned`: unsigned な10進数を表示する
- `printsigned`: signed な10進数を表示する
- `printascii`: 文字列表示
- `printch`: 一文字表示

表示したい数値もしくは文字列を `r0` にセットして、上記のマクロを call すれば使えます。

`printhex2`, `printhex4`, `printhex8`, `printascii`, `printch` は Linux Kernel にもあるのですが、Linux Kernel に含まれているバージョンとの違いは RAM を一切使用せず、破壊するレジスタは `r0`, `r1`, `r2`, `r3` のみです。  
`printunsigned` と `printsigned` はオリジナルで作りました。

これら用途としては、ブートのかなり初期で、RAM もまだ使えない状態で、簡単にプリントデバッグしたいときのために作ったものなのです。  
(LEDぐらいしか点灯しないとなるとかなりデバッグが苦しいですので。)  
私は U-Boot で `lowlevel_init` 関数とかのデバッグ用途に使っていました。

以下、コードです。

レジスタビューは 16550 を想定してます。
最初に `early_serial_init`を call する必要があります。
`DEBUG_UART_BASE`, `UART_SHIFT`, `DIVISOR` あたりは自分の環境に合わせて変更して下さい。

{% highlight gas linenos %}
#define DEBUG_UART_BASE    0xXXXXXXXXX
#define UART_SHIFT 1

/*
 * DIVISER = ((BAUDCLK / 8 / BAUDRATE + 1) / 2)
 */
//#define DIVISOR  40 /* 19200 bps */
//#define DIVISOR  20 /* 38400 bps */
#define DIVISOR  7 /* 115200 bps */

#define UART_TX  0 /* Out: Transmit buffer */

#define UART_LCR 3 /* Out: Line Control Register */
#define UART_LCR_DLAB  0x80 /* Divisor latch access bit */
#define UART_LCR_WLEN8  0x03 /* Wordlength: 8 bits */

#define UART_LSR 5 /* In:  Line Status Register */
#define UART_LSR_TEMT  0x40 /* Transmitter empty */
#define UART_LSR_THRE  0x20 /* Transmit-hold-register empty */

/*
 * DLAB=1
 */
#define UART_DLL 0 /* Out: Divisor Latch Low */
#define UART_DLM 1 /* Out: Divisor Latch High */


/*
 * Call the routine once
 * before calling printhex8/4/2, printascii, printch.
 */
ENTRY(early_serial_init)
  addruart r3
  mov r0, #UART_LCR_DLAB
  strb r0, [r3, #UART_LCR &lt;&lt; UART_SHIFT]
  mov r0, #(DIVISOR &amp; 0xff)   @ LSB of divisor
  strb r0, [r3, #UART_DLL &lt;&lt; UART_SHIFT]
  mov r0, #((DIVISOR &gt;&gt; 8) &amp; 0xff) @ MSB of divisor
  strb r0, [r3, #UART_DLM &lt;&lt; UART_SHIFT]
  mov r0, #UART_LCR_WLEN8
  strb r0, [r3, #UART_LCR &lt;&lt; UART_SHIFT]
  mov pc, lr
ENDPROC(early_serial_init)


  .macro addruart, rx
  ldr \rx, =DEBUG_UART_BASE
  .endm

  .macro senduart,rd,rx
  strb \rd, [\rx, #UART_TX &lt;&lt; UART_SHIFT]
  .endm

  .macro busyuart,rd,rx
1002:  ldrb \rd, [\rx, #UART_LSR &lt;&lt; UART_SHIFT]
  and \rd, \rd, #UART_LSR_TEMT | UART_LSR_THRE
  teq \rd, #UART_LSR_TEMT | UART_LSR_THRE
  bne 1002b
  .endm

/*
 * Useful debugging routines
 *
 * printhex8, printhex4, printhex2:
 *  print a number in 8, 4, 2 digits hexadecimal, respectively.
 *  r0 = number
 *  r1-r3 corrupted
 *
 * printunsigned, printsigned
 *  print a number in decimal.
 *  r0 = number
 *  r1-r3 corrupted
 *
 * printascii
 *  print a string
 *  r0 = pointer to string
 *  r0-r3 corrupted
 *
 * printch
 *  print a character
 *  r0 = character code
 *  r0-r3 corrupted
 */
ENTRY(printhex8)
  mov r2, #28
  b printhex
ENDPROC(printhex8)

ENTRY(printhex4)
  mov r2, #12
  b printhex
ENDPROC(printhex4)

ENTRY(printhex2)
  mov r2, #4
printhex:
  addruart r3
1:  mov r1, r0, lsr r2
  and r1, r1, #15
  cmp r1, #10
  addlt r1, r1, #'0'
  addge r1, r1, #'a' - 10
  senduart r1, r3
  busyuart r1, r3
  subs r2, r2, #4
  bge 1b
  mov pc, lr
ENDPROC(printhex2)

ENTRY(printunsigned)
  mov r1, #1
  mov r2, #0
1:  mov r1, r1, lsl #1  @ r1 = 2 * r1
  add r1, r1, r1, lsl #2 @ r1 = 5 * r1
  add r2, r2, #1
  cmp r2, #10   @ prevent overflow of r1
  bge 2f
  cmp r1, r0
  bls 1b
2:  mov r3, r2   @ number of digits
3:  mov r1, #1
  mov r2, #0
4:  add r2, r2, #1
  cmp r2, r3
  movlt r1, r1, lsl #1  @ r1 = 2 * r1
  addlt r1, r1, r1, lsl #2 @ r1 = 5 * r1
  blt 4b
  mov r2, #0
5:  cmp r1, r0
  subls r0, r0, r1
  addls r2, r2, #1
  blo 5b
  add r2, r2, #'0'
  addruart r1
  senduart r2, r1
  busyuart r2, r1
  subs r3, r3, #1
  bgt 3b
  mov pc, lr
ENDPROC(printunsigned)

ENTRY(printsigned)
  cmp r0, #0
  bge printunsigned  @ signed greater or equal
  addruart r3
  mov r1, #'-'
  senduart r1, r3
  busyuart r1, r3
  rsb r0, r0, #0  @ r0 = 0 - r0
  b printunsigned
ENDPROC(printsigned)

ENTRY(printascii)
  addruart r3
  b 2f
1:  senduart r1, r3
  busyuart r2, r3
  teq r1, #'\n'
  moveq r1, #'\r'
  beq 1b
2:  teq r0, #0
  ldrneb r1, [r0], #1
  teqne r1, #0
  bne 1b
  mov pc, lr
ENDPROC(printascii)

ENTRY(printch)
  addruart r3
  mov r1, r0
  mov r0, #0
  b 1b
ENDPROC(printch)
{% endhighlight %}
