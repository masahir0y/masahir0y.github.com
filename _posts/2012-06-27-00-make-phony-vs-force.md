---
layout: post
title: Makefile での .PHONY と FORCE の違い
description: ""
tags: [Makefile]
---
{% include JB/setup %}

[以前]({{ BASE_PATH }}/2012/02/12/03-kbuild4/)、 `.PHONY:` 指定と `: FORCE` 指定について触れたことがあるのですが、
明らかに両者を使い分けるべきケースがあることに最近気がついた。

まずは、 make の初歩ですが、 PHONY ターゲットからおさらいします。

次のような `clean` ターゲットがあったとする。

    clean:
            $(RM) $(programs) $(objects)

もし、カレントディレクトリにたまたま `clean` という名前のファイルが存在すると、
この `clean` は実行してくれません。

    $ touch clean
    $ make clean
    make: `clean' is up to date.

そこで、

    .PHONY: clean
    clean:
            $(RM) $(programs) $(objects)

としておくと、 `clean` という名前のファイルがあったとしても、必ず `clean` の生成ルールを実行できます。

一方、次のような書き方をしても、必ず `clean` を実行させることはできます。

    clean: FORCE
            $(RM) $(programs) $(objects)
    FORCE:
    .PHONY: FORCE

`FORCE` の部分は別の名前でもいいですが、とにかくダミーの `PHONY` ターゲットを作ります。

このダミーターゲット `FORCE` を依存関係に引き連れていると、日付とかに関係なく問答無用で、
生成ルールを実行させることができるので、 `FORCE` という名前が分かりやすいと思います。

一般的に、 `all` や `clean` などファイル実体をもたないものは `.PHONY: `を、
ファイルだけども、毎回必ず作り直したい場合は `: FORCE` を指定すればよいです。

以下では、 `.PHONY: ` と `: FORCE` で挙動に差が出る場合を見てみます。

以下のような内容の `Makefile`, `foo.c`, `bar.c` という 3 つのファイルを用意します。

Makefile

    all: foo bar
    
    %: %.c config.h
            gcc -include config.h -o $@ $<

    config.h: FORCE
            @ echo -n > $@.tmp
            @ $(if $(FOO), echo "#define CONFIG_FOO \"$(FOO)\"" >> $@.tmp)
            @ $(if $(BAR), echo "#define CONFIG_BAR \"$(BAR)\"" >> $@.tmp)
            @ if [ -r $@ ] && cmp -s $@ $@.tmp; then \
                    rm $@.tmp; \
                    echo $@ was NOT updated.; \
            else \
                    mv $@.tmp $@; \
                    echo $@ was updated.; \
            fi
    
    clean:
            rm -f foo bar config.h

    FORCE:
    .PHONY: all clean FORCE


foo.c

    #include <stdio.h>
    
    int main()
    {
    #ifdef CONFIG_FOO
            puts(CONFIG_FOO);
    #else
            puts("CONFIG_FOO is not defined.");
    #endif
            return 0;
    }

bar.c

    #include <stdio.h>
    
    int main()
    {
    #ifdef CONFIG_BAR
            puts(CONFIG_BAR);
    #else
            puts("CONFIG_BAR is not defined.");
    #endif
            return 0;
    }

ちょっと取ってつけたような例ですが、 make の最初に必ず、 `config.h` という設定ファイルを生成し、
全ソースファイルから `config.h` を include するようにしています。

    %: %.c config.h

の依存関係がありますので、 `foo` と `bar` より前に `config.h` が作られます。

また、

    config.h: FORCE

となっているので、 `config.h` の生成ルールは make のたびに必ず走ります。
ただし、いったん、 `config.h.tmp` を作り、もし内容が更新されているときのみ、

    mv config.h.tmp config.h

とします。

さて、実行してみましょう。

    $ make
    config.h was updated.
    gcc -include config.h -o foo foo.c
    gcc -include config.h -o bar bar.c
    $ make
    config.h was NOT updated.
    $ cat config.h
    $ ./foo ; ./bar
    CONFIG_FOO is not defined.
    CONFIG_BAR is not defined.
    $ make FOO="hello, world" BAR="good morinig"
    config.h was updated.
    gcc -include config.h -o foo foo.c
    gcc -include config.h -o bar bar.c
    $ make FOO="hello, world" BAR="good morinig"
    config.h was NOT updated.
    $ cat config.h
    #define CONFIG_FOO "hello, world"
    #define CONFIG_BAR "good morinig"
    $ ./foo ; ./bar
    hello, world
    good morinig

期待通りに、

- make のたびに必ず `config.h` の更新チェックが入っています。
- `config.h` の内容が前回から書き変わったときにだけ、 `foo` と `bar` は再コンパイルされている。

さてさて、Makefile をちょっと書き変えて

    config.h: FORCE

の部分を

    .PHONY: config.h
    config.h:

としてみましょう。

    $ make
    config.h was updated.
    gcc -include config.h -o foo foo.c
    gcc -include config.h -o bar bar.c
    $ make
    config.h was NOT updated.
    gcc -include config.h -o foo foo.c
    gcc -include config.h -o bar bar.c

あれあれ、今度は `config.h was NOT updated.` の時でも、毎回 `foo` と `bar` が再コンパイルされています。

これが `.PHONY:` 指定と `: FORCE` 指定の決定的な違いだと思います。

PHONY ターゲットを依存関係につれていると、「必ず」生成ルールが実行されてしまいます。
そうではなくて、 `config.h` と `foo` および `bar` の間の日付関係は考慮して欲しいわけですから
`.PHONY: config.h` はまずいということになります。

同様の例で、 `prepare` ターゲットみたいなのを考えてみます。

例えば、ビルドを開始する前に必ず `prepare` に指定されたルールを走らせたい場合、
PHONYターゲットを使って以下のように書くことができます。

    all: foo bar
    if_old = if [ ! -f $@ ] || [ "$$(find $(filter-out $(PHONY), $^) -newer $@)" ]; then $1; fi

    compile = gcc -o $@ $(filter %.c, $^)

    foo: foo1.c foo2.c foo.h prepare
            $(call if_old, $(compile))

    bar: bar.c bar1.h bar2.h prepare
            $(call if_old, $(compile))

    prepare:
            @echo "This is always processed first."

    clean:
            rm -f foo bar

    PHONY := all prepare clean
    .PHONY: $(PHONY)

`foo` と `bar` の生成ルール自体は毎回実行されてしまうので、
`if_old` 関数を定義して必要な時しか、 `$(comiple)` が実行されないようにしています。
（Linuxカーネルの makefile の `if_changed` 関数とか `if_changed_dep` 関数とかを参考に書いてみた）

しかし、以下のように `.prepare` 隠しファイルを使った方が、 `if_old` 関数も不要なので、簡単のような気がします。

    all: foo bar
    
    compile = gcc -o $@ $(filter %.c, $^)
    
    foo: foo1.c foo2.c foo.h .prepare
            $(compile)
    
    bar: bar.c bar1.h bar2.h .prepare
            $(compile)
    
    .prepare: FORCE
            @echo "This is always processed first."
            @if [ ! -f $@ ]; then touch $@; fi
    
    clean:
            rm -f foo bar .prepare
    
    FORCE:
        .PHONY: all clean FORCE
