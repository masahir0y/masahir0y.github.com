---
layout: post
title: RAID 上の Ubuntu をアップグレードする
description: ""
tags: [Ubuntu]
---
{% include JB/setup %}

Ubuntu 12.04 LTS から Ubuntu 13.04 にアップグレードしてみました。

マシンは、以前 [Linuxインストール on RAID (Intel RST)]({{ BASE_PATH }}/2012/09/27/00-linux-raid/) で書いた SDD 4 台で RAID5 構成です。

いきなり 13.04 にアップグレードできないので、 12.04 ⇒ 12.10 ⇒ 13.04 へとアップグレードすることにしますう。

アップデートマネージャーで 「設定...」を選び、

Ubuntu の新バージョンの通知:

を「長期サポート(LTS)版」から「すべての新バージョン」へ変更する。

すると、「Ubuntu の新しいリリース '12.10'が利用可能です」というのが出てくる。

普通ならこれで簡単にアップグレードできるはずなのですが、困ったことに RAID を組んでいると、GUI でうまくアップグレードしてくれないみたいです。

Googleで調べつつやって、うまくいったので、やり方をまとめておきます。

とりあえず、アップデートマネージャーでアップグレードしていきました。

途中で、「起動デバイスへの GRUB の書き込みが失敗しました。続行しますか？」と出てくる。

[![grub-install failed](http://2.bp.blogspot.com/-RIBjtrk9jNg/Ua7h7tH-oAI/AAAAAAAAAGQ/iiU_cTiYHGU/s320/grub-fail.png)](http://2.bp.blogspot.com/-RIBjtrk9jNg/Ua7h7tH-oAI/AAAAAAAAAGQ/iiU_cTiYHGU/s1600/grub-fail.png)

とりあえず、続行ボタンを押して、後で手動で GRUB をインストールすることにする。

で、最後までいって、再起動する直前に端末から

    $ sudo grub-install /dev/mapper/isw_bdafdjfgcf_Volume0
    Path `/boot/grub' is not readable by GRUB on boot. Installation is impossible. Aborting.

とやったのですが、失敗。ガビーン。
(`isw_xxxxxxxxxx_Volume0` の部分は人によって違うでしょう。)


このまま再起動するも、やっぱり起動せず。

こういう困った時のために [Boot Repair](https://help.ubuntu.com/community/Boot-Repair) という素晴らしいツールがあります。
メディアに書き込んで、Boot Repair からブート。

起動したら「おすすめの修復」というのを選択。

途中で、「ターミナルを開いて、次のコマンドを入力（またはコピー&amp;ペースト）してください。」という指示が出るので、言われるままに実行していきます。

最後にGRUBをインストールする画面が出てくるので
「/dev/mappder/isw_xxxxxxxxxx_Volume0」を選択します。

これで再起動すると、 Ubuntu 12.10 が起動するようになった。
困った時の Boot Repair 、覚えておこう。
