---
layout: post
title: Shared Object 入門
description: ""
tags: [C]
---
{% include JB/setup %}

最近の gcc だとシンボル解決がうまくいかないといかないことがあったので、ちょっと調べました。

次のような 3つのソースファイルがあったとして、ダイナミックリンクで作ってみましょう。

`main.c`

    int foo(void);
    int main(void)
    {
            return foo() + 1;
    }

`foo.c`

    int bar(void);
    int foo(void)
    {
            return bar() + 1;
    }

`bar.c`

    int bar(void)
    {
            return 0;
    }

shared object を作るのに必要なのは、コンパイル時に `-fPIC` オプションをつけることと、リンク時に `-shared` オプションをつけることの 2点です。

    $ gcc -fPIC -c -o bar.o bar.c
    $ gcc -shared -o libbar.so bar.o

のように中間オブジェクト `bar.o` を作ってもいいですし

    $ gcc -shared -fPIC -o libbar.so bar.c

のように一度にやってもよいです。いずれにせよ `libbar.so` が作られます。

では、最後までやってみます。

    $ gcc -shared -fPIC -o libbar.so bar.c
    $ gcc -shared -fPIC -o libfoo.so foo.c
    $ gcc -L . -lfoo -lbar main.c
    /tmp/ccqQv8uv.o: In function `main':
    main.c:(.text+0x5): undefined reference to `foo'
    collect2: ld returned 1 exit status

あれれ、 `foo` が見つからないと言われました。。。
gcc のバージョンによっては、上記のやり方でもエラーなしにリンクできる場合もあるかもしれません。
gcc のバージョンが新しいとうまくいかないようです。
私の手元の gcc のバージョンは 4.6.3 です。

ちなみに、

    $ gcc -L . main.c -lfoo -lbar

のようにすればうまくいきました。

関数が `main()` ->　`foo()` -> `bar()` のようにコールしているならば、
リンク時にも `main.c -lfoo -lbar` の順に並べないと、リンク解決できないようです。

とは言うものの、リンク時に順番をいちいち考えるのも面倒です。

順番を気にせずうまくやってくれるためには、リンカーに、 `-no-as-needed` を渡せばよいようです。

    $ gcc -Wl,-no-as-needed -L. -lfoo -lbar main.c

でも

    $ gcc -Wl,-no-as-needed -L. -lfoo main.c -lbar

でもうまくいきます。

実行してみます。
`libfoo.so` と `libbar.so` は検索パスに置いてないので、 `LD_LIBRARY_PATH` を指定します。

    $ LD_LIBRARY_PATH=$PWD ./a.out
    $ echo $?
    2

うまくいきました。

ちなみに、リンク時に `-R`  または `-rpath` でライブラリ検索パスを指定しておけば、実行時に `LD_LIBRARY_PATH` は指定しなくてもいいです。

    $ gcc -Wl,-no-as-needed -Wl,-R$PWD -L . -lfoo -lbar main.c

または

    $ gcc -Wl,-no-as-needed -Wl,-rpath $PWD -L . -lfoo -lbar main.c

まだよくない点があります。

関数 `main()` から直接的にコールしているのは `foo()` だけであって、 `bar()` を直接的にコールしているわけではないです。

なので、関数 `foo()` をコールするために

    $ gcc -Wl,-no-as-needed -L. -lfoo -lbar main.c

のように `-lfoo` も `-lbar` もリンクしないといけないのは変な話です。

関数 `foo()` は関数 `bar()` をコールしているので、 `libfoo.so` は `libbar.so` に依存していることになります。

そこで、shared object 間の依存関係を考慮することにします。

`libfoo.so` を作るときにも `-Wl,-no-as-needed` を指定すればうまくいきます。

    $ gcc -shared -fPIC -o libbar.so bar.c
    $ gcc -shared -fPIC -Wl,-no-as-needed -L . -lbar -o libfoo.so foo.c
    $ gcc -Wl,-no-as-needed -Wl,-rpath $PWD -Wl,-rpath-link $PWD -L . -lfoo main.c

最終行で、 `-lbar` は不要になっています。 ただし、 `-rpath-link` オプションがいります。

`ldd` コマンドで依存関係を見ると、 libfoo.so は libbar.so を必要としています。

    $ ldd libfoo.so
     linux-vdso.so.1 =>  (0x00007fff1fbff000)
     libbar.so => not found
     libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f5141d0d000)
     /lib64/ld-linux-x86-64.so.2 (0x00007f51422ea000)

ちなみに、shared object の勉強は
[共有ライブラリーを解剖する](http://www.ibm.com/developerworks/jp/linux/library/l-shlibs/)
というページがわかりやすかったです。
