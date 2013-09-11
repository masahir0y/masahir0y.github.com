---
layout: post
title: git 共用リポジトリ作成
description: ""
tags: [git]
---
{% include JB/setup %}

例えば、 gitusers というグループに所属しているユーザーであれば自由に push できる共用リポジトリを作る場合。

まずは、 gitusers というグループを作って、開発に参加するユーザーをそのグループに入れる。

    # groupadd gitusers
    # gpasswd -a user1 gitusers
    # gpasswd -a user2 gitusers
 
続いて `/var/git/hoge.git` というリポジトリ作る場合、 以下のようにします。

    # mkdir -p /var/git/hoge.git
    # cd /var/git/hoge.git
    # chmod 2775 .
    # chgrp gitusers .
    # git --bare init --shared
    # touch gitweb-export-ok               # if you want to publish this repository with gitweb
    # touch git-daemon-export-ok           # if you want to publish this repository with git-daemon
    # echo "Description of this repo" > description

`git init` の前に sgid ビットを立てておかないと root 以外のユーザーが push できなくなる。
