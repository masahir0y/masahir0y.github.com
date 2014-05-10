---
layout: post
title: シェル変数まとめ その1
description: ""
tags: [shell]
---
{% include JB/setup %}

- `${variable:=value}`

    変数 `variable` が空でなければ、 `variable` の値を返す。
    空なら、 `variable` に `value` を代入し、`value` を返す。

- `${variable=value}`

    変数 `variable` が定義済みなら、 `variable` の値を返す。
    未定義なら、 `variable` に `value` を代入し、`value` を返す。

- `${variable:-value}`

    変数 `variable` が空でなければ、 `variable` の値を返す。
    空なら、 `value` を返す。代入処理は行わない。

- `${variable-value}`

    変数 `variable` が定義済みなら、 `variable` の値を返す。
    未定義なら、 `value` を返す。代入処理は行わない。

- `${variable:?message}`

    変数 `variable` が空でなければ、 `variable` の値を返す。
    空なら、 `message` を表示し、シェルスクリプト中ならその場で終了する。

- `${variable?value}`

    変数 `variable` が空でなければ、 `variable` の値を返す。
    空なら、 `message` を表示し、シェルスクリプト中ならその場で終了する。

- `${variable:+value}`

    変数 `variable` が空でなければ、 `value` を返す。
    空なら、空文字列を返す。

- `${variable+value}`

    変数 `variable` が定義済みなら、 `value` を返す。
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

- `${variable#value}`

    `variable` の値の前から `value` を削ったものを返す。最短のマッチ。

- `${variable##value}`

    `variable` の値の前から `value` を削ったものを返す。最長のマッチ。

- `${variable%value}`

    `variable` の値の後ろから `value` を削ったものを返す。最短のマッチ。

- `${variable%%value}`

    `variable` の値の後ろから `value` を削ったものを返す。最長のマッチ。

例)

    $ FOO=foo/bar/baz
    $ echo ${FOO#foo/}     # 前方の foo/ を削ります。
    bar/baz
    $ echo ${FOO#*/}       # ワイルドカードも使えます。前方の */ を削ります。(最短のマッチ)
    bar/baz
    $ echo ${FOO##*/}      # 前方の */ を削ります。(最長のマッチ)
    baz
    $ echo ${FOO%/*}       # 後方の /* を削ります。(最短のマッチ)
    foo/bar
    $ echo ${FOO%%/*}      # 後方の /* を削ります。(最長のマッチ)
    foo
    $ echo ${FOO#foobar}   # マッチしない場合はそのまま
    foo/bar/baz

パスのディレクトリ部分やファイル名部分を取り出すときに、 `dirname` コマンドや `basename` コマンド
を使わなくても、`${FOO%/*}` と `${FOO##*/}` で代用できるということですね。

ここから先は、bash でしか使えない書き方です。
これらをスクリプトで使うときは、shebang を `#!/bin/bash` にすること。

- `${variable^}`

    `variable` の値の 1文字目を大文字に変換する。

- `${variable^^}`

    `variable` の値の 全文字を大文字に変換する。

- `${variable,}`

    `variable` の値の 1文字目を小文字に変換する。

- `${variable,,}`

    `variable` の値の 全文字を小文字に変換する。

- `${variable~}`

    `variable` の値の 1文字目を大文字小文字反転。

- `${variable~~}`

    `variable` の値の 全文字を大文字小文字反転。

- `${variable/before/after/}`

    `variable` の値で、`before` にマッチした文字列を `after` に置換する。(最初にマッチしたもののみ)<br />
    `echo $variable | sed s/before/after/` の動作。

- `${variable//before/after/}`

    `variable` の値で、`before` にマッチした文字列を `after` に置換する。(マッチしたものすべて置換)<br />
    `echo $variable | sed s/before/after/g` の動作。
