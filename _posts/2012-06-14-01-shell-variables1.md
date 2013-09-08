---
layout: post
title: シェル変数まとめ その1
description: ""
tags: [shell]
---
{% include JB/setup %}

- `${variable:=value}`

    変数 variable が空でなければ、 variable の値を返す。
    空なら、 variable に value を代入し、value を返す。

- `${variable=value}`

    変数 variable が定義済みなら、 variable の値を返す。
    未定義なら、 variable に value を代入し、value を返す。

- `${variable:-value}`

    変数 variable が空でなければ、 variable の値を返す。
    空なら、 value を返す。代入処理は行わない。

- `${variable-value}`

    変数 variable が定義済みなら、 variable の値を返す。
    未定義なら、 value を返す。代入処理は行わない。

- `${variable:?message}`

    変数 variable が空でなければ、 variable の値を返す。
    空なら、 message を表示し、シェルスクリプト中ならその場で終了する。

- `${variable?value}`

    変数 variable が空でなければ、 variable の値を返す。
    空なら、 message を表示し、シェルスクリプト中ならその場で終了する。

- `${variable:+value}`

    変数 variable が空でなければ、 value を返す。
    空なら、空文字列を返す。

- `${variable+value}`

    変数 variable が定義済みなら、 value を返す。
    未定義なら、空文字列を返す。


`:` (コロン)ありの場合は変数が空でないかどうかで判定するのに対し、`:` (コロン)なしの場合は定義済かどうかで判定します。

例

    $ FOO=                 # 空にする
    $ echo ${FOO:=123}     # 空なので
    123                    # 123 を返す
    $ echo $FOO
    123                    # 代入もされてる
    $ echo ${FOO:=456}     # 今度は空ではないので
    123                    # FOOの値 (123)を返す
    $ FOO=                 # 空にする
    $ echo ${FOO=456}      # 空ですが、定義はされてるので
                           # FOO の値 (NULL) を返す
    $ echo $FOO
                           # 代入もされていない
    $ unset FOO            # 未定義に戻すには unset します
    $ echo ${FOO=456}      # 未定義なので
    456                    # 456 を返す
    $



`=` と `-` の違いは、 `-` は代入まではしないことです。

    $ FOO=                 # 空にする
    $ echo ${FOO:-123}     # 空なので
    123                    # 123 を返す
    $ echo $FOO            # ただし代入はされてない
    
    $

`+` は `-` とは反対のような機能ですね。

    $ FOO=                 # 空にする
    $ echo ${FOO:+123}     # 空なので
                           # 空を返す
    $ FOO=456              # 何か値を入れる
    $ echo ${FOO:+123}     # 空ではないので
    123                    # 123 を返す
    $ echo $FOO
    456                    # 代入はされません

あと、こんなんもあります。
本でみたわけではないけど、人のコードに使われていたのを見て、たぶんこんな感じだと思って使ってます。

- `${variable#value}`

    variable の値の前から value を削ったものを返す

- `${variable%value}`

    variable の値の後ろから value を削ったものを返す

例)

    $ FOO=3.141592
    $ echo ${FOO#3.}       # 前方の3. を削ります
    141592
    $ echo ${FOO#*.}       # ワイルドカードも使えます
    141592
    $ echo ${FOO%92}       # 後ろの 92 を削ります
    3.1415
    $ echo ${FOO%1*}       # 複数のマッチングパターンがあると
    3.14                   # 短い方にマッチするみたい
    $ echo ${FOO%abc}      # マッチしない場合
    3.141592               # そのまま
