---
layout: post
title: Ubuntu 13.04 アップグレード後始末
description: ""
tags: [Ubuntu]
---
{% include JB/setup %}

[前回]({{ BASE_PATH }}/2013/06/25/03-ubuntu-raid-upgrade/)の記事で、Ubuntu 13.04 にアップグレードしました。
アップグレード後、ログイン時に以下のようなメッセージが出るようになりました。

    Welcome to Ubuntu 13.04 (GNU/Linux 3.8.0-23-generic x86_64)
    
     * Documentation:  https://help.ubuntu.com/
    
    0 packages can be updated.
    0 updates are security updates.
    
    New release '13.04' available.
    Run 'do-release-upgrade' to upgrade to it.

`Welcome to Ubuntu 13.04` と `New release '13.04' available` 矛盾していますね。

検索してみると、バグとして報告されているようです。

とりあえず、直し方は単純に以下のようにファイル1個消すだけでいいようです。

    $ suro rm /var/lib/ubuntu-release-upgrader/release-upgrade-available

`/etc/update-motd.d/91-release-upgrade` から呼ばれた
`/usr/lib/ubuntu-release-upgrader/release-upgrade-motd`
が必要に応じてまた作ってくれるみたいです。
