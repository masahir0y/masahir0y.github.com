---
layout: post
title: ARM Linux で start_kernel関数までを追いかける
description: ""
tags: [Linux Kernel, ARM]
---
{% include JB/setup %}

ARM Linux のカーネルのエントリーから、 `start_kernel` 関数にジャンプするまでのコードを読んでみました。

ARM にそこそこ詳しくないと理解しづらいかもしれませんが、 ARM ARM を見ながら根気強く読んでいくと、Linux Kernel だけでなく ARM の勉強にもなります。

カーネルのエントリーは `arch/arm/kernel/head.S` にあります。
バージョンによってちょっと違うかもしれないけど、 エントリー部分のコードは以下のような感じになっています。 (以下は v2.6.32 のコードです。)

    ENTRY(stext)
            setmode PSR_F_BIT | PSR_I_BIT | SVC_MODE, r9 @ ensure svc mode
                                                    @ and irqs disabled
            mrc     p15, 0, r9, c0, c0              @ get processor id
            bl      __lookup_processor_type         @ r5=procinfo r9=cpuid
            movs    r10, r5                         @ invalid processor (r5=0)?
            beq     __error_p                       @ yes, error 'p'
            bl      __lookup_machine_type           @ r5=machinfo
            movs    r8, r5                          @ invalid machine (r5=0)?
            beq     __error_a                       @ yes, error 'a'
            bl      __vet_atags
            bl      __create_page_tables
    
            /*
             * The following calls CPU specific code in a position independent
             * manner.  See arch/arm/mm/proc-*.S for details.  r10 = base of
             * xxx_proc_info structure selected by __lookup_machine_type
             * above.  On return, the CPU will be ready for the MMU to be
             * turned on, and r0 will hold the CPU control register value.
             */
            ldr    r13, __switch_data               @ address to jump to after
                                                    @ mmu has been enabled
            adr    lr, BSYM(__enable_mmu)           @ return (PIC) address
     ARM(   add    pc, r10, #PROCINFO_INITFUNC      )
     THUMB( add    r12, r10, #PROCINFO_INITFUNC     )
     THUMB( mov    pc, r12                          )
    ENDPROC(stext)

2行目の `setmode` はマクロで、定義は `arch/arm/include/asm/assembler.h` にあります。

    mrc     p15, 0, r9, c0, c0

のコードは processor id を r9 に入れている。

`__lookup_processor_type` は r9 に入っている processor id を元に、該当する `proc_info_list` 構造体へのポインタを得て r5 に入れます。
`struct proc_info_list` の型定義は `arch/arm/include/asm/procinfo.h` にあります。
各 processor ごとの `proc_info_list` 構造体は `arch/arm/mm/` 以下で定義されています。

`__lookup_processor_type` の関数の中身は `arch/arm/kernel/head-common.S` で定義されています。

ここはまだ物理アドレスで動いている部分。つまり、リンク時のシンボル情報と異なるアドレスで動いています。
なので、相対アドレス参照はそのままで動きますが、絶対アドレス参照の部分は、論理アドレスから物理アドレスに変換しながら動かしています。

`__lookup_machine_type` も `__lookup_processor_type` と同様の動きです。
ただし、マシーン番号は自分ではわからないので、ブートローダーから渡された値(r1に入っている)を使って、`machine_desc` 構造体へのポインタを得て r5 に入れています。

`__arch_info_begin` と `__arch_info_end` は `.arch.info.init` セクションの最初と最後を表しています。
`.arch.info.init` セクションには `machine_desc` 構造体がずらずらっと集められているので、r1 の番号に一致するのをサーチしています。

`machine_desc` 構造体のインスタンスを記述するには、 `MACHINE_START` (`arch/arm/include/asm/mach/arch.h` で定義) というマクロで使うらしい。
各ボードごとの `machine_desc` 構造体は `arch/arm/mach-XX` の下で定義されている。

途中に出てくる `#MACHINFO_TYPE` などのマクロは `asm/asm-offsets.h` で定義されているが、これは `arch/arm/kernel/asm-offsets.s` から変換して作られているっぽい。ややこしいな。

続いて `__create_page_tables` ルーチンへジャンプし、MMUテーブルを作成する。
MMUテーブルはカーネルのエントリーアドレスから `0x4000' を引いたアドレスに置かれます。
ここはまだページテーブルを作るだけで、まだMMU ONにはしません。

    ldr    r13, __switch_data

という行は `__switch_data` に書かれているデータを r13(sp) にセットしています。

__switch_data:
        .long   __mmap_switched

となっているので、r13 に `__mmap_switched` の「仮想アドレス」が入ることになります。

    adr    lr, BSYM(__enable_mmu)

の部分は `__enable_mmu` の「物理アドレス」をr14(lr)にセットしています。

     ARM(   add    pc, r10, #PROCINFO_INITFUNC      )
     THUMB( add    r12, r10, #PROCINFO_INITFUNC     )
     THUMB( mov    pc, r12                          )

はthum命令がON/OFFかによって、どっちかが残るようになっています。
`proc_info_list` 構造体の `__cpu_flush` メンバーのアドレスが pc に入ります。

`arch/arm/mm/proc-XXX.S` を見ますと、 `__cpu_flush` メンバーのところには

    b       __v7_setup

みたいなジャンプ命令が書いてあります。

つまり、ここから ARM のアーキテクチャバージョン依存のセットアップ関数にジャンプします。
D Cacheクリーンとか、ARM のアーキテクチャバージョンによってコード実装の異なる処理がここで行われます。

`__v7_setup` を抜けるときに、lr に入っているアドレスへリターンする。
つまり `__enable_mmu` に飛ぶ。

`__enable_mmu` からさらに `__turn_mmu_on` へと飛んで、MMU ON にします。
そして、

    mov     r3, r13
    mov     pc, r3

の部分で `__mmap_switched` にジャンプする。

ここでようやく、仮想アドレスでプログラムが実行される。つまり、リンク時のシンボル情報と実際に動いているアドレスが一致する。

`__mmap_switched` では、 `.data` セクションをロードアドレスから仮想アドレスへコピーするが、普通はロードアドレスと仮想アドレスは同じになっているので、すっ飛ばされる。

続いて、 `.bss` セクションをゼロクリア。

processor ID と machine type と ATAGS pointer を後で参照できるように外部変数に保存する。

`cr_alignment` というのはよくわかりませんでした。

最後に `init_thread_union + THREAD_START_SP` の値を sp レジスタの初期値としてセットしています。

この部分は、 `thread_info` 構造体とスタックの関係を理解してないといけないとピンと来ないと思います。
Linuxでの実装では `thread_info` 構造体とカーネルスタックは常に同じ領域 (サイズは普通 8192バイト)を共有しています。
このようにしておくと、スタックの値の下位13bitを0マスクするだけで、現在動作中の `thread_info` 構造体へのポインタ求められるため、メリットが大きい。

`init_thread_union` (`arch/arm/kernel/init_task.c` で定義)というのは一番最初のthread_infoで、これにカーネルスタックサイズの`THREAD_START_SP` (= 8192 - 8) を足したものが、スタックの初期値になるわけですね。

最後に、`start_kernel` 関数 (`init/main.c` で定義)にジャンプする。
ここからは基本的には arch 非依存な部分で、C言語で記述されています。
