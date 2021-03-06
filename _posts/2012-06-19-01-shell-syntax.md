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


シェルスクリプトを書くときに一行で書く人はいないでしょうが、
makefile など一行で書かないといけないときがあるので、理解しておく必要があります。

どこに `;` を入れるかは、覚えておかなくても理屈で考えればわかります。

    if test -r foo; then

みたいなのは、 `;`がなかったら `then` までが `test` の引数だと解釈されてしまいます。

なので、基本的に `command-list` の後には `;`がいります。

逆に `then` とか `else` とか `do` みたいに、制御用の予約語の後は、 `;` がなくても構文解釈できますので、不要なわけです。

### Here Documents ###

    command <<EOF
    ......
    ......
    EOF

`command` としてよく使われるのは `cat` です。
`EOF` の代わりに `END` とかもよく使われます。

`command <<\EOF` または `command <<'EOF'` のようにすると `......` の部分全体がクォートされることになり、変数は展開されなくなります。

また `command <<-EOF` のようにすると `......` 中の行頭のタブは無視されます。

### 入力リダイレクト拡張 ###

bash でのみ使える機能ですが、

    command <<< string

`command` の標準入力にリダイレクトで文字列 `string` を流す。

    echo foo | sed 's/foo/bar'

を

    sed 's/foo/bar/' <<< foo

のように書くことができる。

### 標準エラー出力のパイプ ###

これも bash でのみ使える機能ですが、

    command1 |& command2

は `command1` の標準エラー出力をパイプに流す。

例えば、

    command1 |& tee log

は `command1` の標準エラー出力を `log` ファイルに記録することができる。
