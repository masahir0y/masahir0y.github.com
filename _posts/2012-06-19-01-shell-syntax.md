---
layout: post
title: シェル制御文まとめ
description: ""
tags: [shell]
---
{% include JB/setup %}

### if 文 ###

    if command-list
    then
        command-list
    [elif command-list
    then
        command-list ]
    [else
        command-list ]
    fi

### for 文 ###

    for variable [ in word-list ]
    do
        command-list
    done

### while 文 ###

    while command-list
    do
        command-list
    done

### case 文 ###

    case string in
        [ pattern [ | pattern ] ... ) command-list;; ]
        [ pattern [ | pattern ] ... ) command-list;; ]
        ......
    esac

上記の `command-list` の部分は、改行もしくは `;` で区切れば複数のコマンドを書くことができます。
`[ ... ]` はあってもなくてもいい部分です。

一行で書く時は以下の通り。
ブラウザで見たときに折り返ってるかもしれませんが、一行と思ってください。

### if 文 ###

    if command-list; then command-list; [elif command-list; then command-list;] [else command-list; ] fi

### for 文 ###

    for variable [ in word-list; ] do command-list; done

### while 文 ###

    while command-list; do command-list; done

### case 文 ###

    case string in [ pattern [ | pattern ] ... ) command-list;; ] [ pattern [ | pattern ] ... ) command-list;; ] ...... esac


シェルスクリプトを書くときに一行で書く人はいないでしょうが、makefile など一行で書かないといけないときがあるので、理解しておく必要があります。

どこに `;` を入れるかは、覚えておかなくても理屈で考えればわかります。

    if test -r foo; then

みたいなのは、 `;`がなかったら `then` までが `test` の引数だと解釈されてしまいます。

なので、基本的に `command-list` の後には `;`がいります。

逆に `then` とか `else` とか `do` みたいに、制御用の予約語の後は、 `;` がなくても構文解釈できますので、不要なわけです。
