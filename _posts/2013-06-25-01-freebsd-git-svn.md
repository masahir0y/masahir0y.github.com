---
layout: post
title: FreeBSD のソースを git-svn で取ってきた
description: ""
tags: [FreeBSD, git]
---
{% include JB/setup %}

以前 [git-svn]({{ BASE_PATH }}/2012/10/09/02-git-svn/) でも書いたのですが、プロジェクトの管理が cvs とか subversion とか聞いただけでも嫌になってしまう。

現在、FreeBSDは Subversion で管理されている。(以前は cvs だった。)

リポジトリの構造が、Subversion標準の

- trunk
- branches
- tags

になっていれば、git に変換するときにわかりやすいんですが、

    $ svn ls http://svn.freebsd.org/base

で見てみると、そうはなっていないようです。

リポジトリの構造についての説明は

- <http://people.freebsd.org/~peter/svn_notes.txt>
- <http://www.freebsd.org/doc/en/articles/committers-guide/article.html#subversion-primer>
- <http://svn.freebsd.org/base/ROADMAP.txt>

にあたりに説明があるようですが、最後のリンクが一番わかりやすいと思う。

- base/head → trunk
- base/stable → branches
- base/releng → branches
- base/release → tags

という対応のようです。
(tags なのか branches なのかは、実はそれほど気にしなくてもよい。結局 git-svn で変換すると、 branches も tags も git 上ではブランチになってしまうので。)

base/projects の下はトピックブランチのようなのですが、 base/projects/<topic> の下にさらに trunk と branches があったりして、ポリシーが一定していないので、変換はあきらめ。

他にも、 base/vendor, base/vendor-sys とかいくつかのワークエリアがあるようだが、これらも無視した。

というわけで、

    $ git svn clone -T head -t release -b releng -b stable http://svn.freebsd.org/base freebsd_base

という感じで変換して取って来ました。

途中何度か、サーバーのエラーか何かで止まったのですが、その度に `git svn fetch` で再開。

なんだか、むちゃくちゃ時間がかかる。しまった、 trunk だけにしとけばよかったか、と思いつつも、なんとか変換が終わりました。
(time コマンドで測ってみたが、58時間くらいかかった。。)

ちなみに、

    head -> stable/* -> releng/*.* -> release/*.*.*

という風に、なっているので、基本的にブランチ(タグ)名はかぶらないようになっているはずなのだが、最初の頃はポリシーが徹底されていなかったのか、

`base/stable/2.0.5` と `base/releng/2.0.5` の名前がかぶっています。git に変換したときに 2.0.5 のブランチ名がかぶってしまうのですが、この処理がどうなっているのか不明。
