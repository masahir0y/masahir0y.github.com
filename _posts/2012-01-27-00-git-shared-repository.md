---
layout: post
title: git 共用リポジトリ作成
description: ""
tags: [git]
---
{% include JB/setup %}

普通、プライマリーグループはユーザー名と同じ名前になっている。 
なので、 gitusers というグループを作って、gitを使うユーザーをそのグループに入れる。 

    # groupadd gitusers
    # gpasswd -a user1 gitusers
    # gpasswd -a user2 gitusers

みたいな感じ。 
 
`/var/git/hoge.git` というリポジトリ作る場合、 以下のようにします。

    # mkdir -p /var/git/hoge.git
    # cd /var/git/hoge.git
    # chmod 2775 .
    # chgrp gitusers .
    # git --bare init --shared

`git init` の前に sgid ビットを立てておかないと root 以外のユーザーが push できなくなる。 
