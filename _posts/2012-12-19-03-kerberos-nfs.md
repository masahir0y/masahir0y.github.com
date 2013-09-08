---
layout: post
title: NFSv4 + Kerberos で セキュリティとユーザーマッピングを解決
description: ""
tags: [RHEL/CentOS, Ubuntu]
---
{% include JB/setup %}

NFSv4 と Kerberos を組み合わせることで、NFSのセキュリティとユーザーマッピングの課題を解決してみました。

これまで、自分が NFS を使う際の懸念事項がセキュリティの問題とユーザーマッピングの問題でした。

#### セキュリティについて ####

`/etc/exports` でマウントを許可するクライアントを指定することはできるものの、IPアドレスは基本的に自己申告ですから、LAN内にある全てのマシンが信用できるという場合を除いては、かなり危険だと思います。

ある程度規模の大きい LAN につないでいる場合、中には悪い人がいるかもしれません。

例えば、サーバー側が `/etc/exports` に

    /home    192.168.11.30(rw,sync)

と書いて、公開したとします。

他のマシンから

    $ showmount -e NFS_SERVER_NAME

で問い合わせると

    /home 192.168.11.30

と表示されます。

つまりどのホストに対して、どのディレクトリを公開しているかは一目瞭然です。
その IPアドレスになりすまされると、悪意のある人に簡単にマウントされてしまいます。

#### ユーザーマッピングの問題について ####

NFSv3 までは、パーミッションについては クライアント側の UID と GID がサーバー側でも利用されるので、サーバー/クライアント間で UID と GID  の同期が取れていない場合、別ユーザーになってしまうという問題があります。

例えば、サーバー側では taro が UID 500 だったとします。
もし、クライアント側で taro が UID 600 になっていたとすると、 taro は サーバー上のファイルへアクセスできなくなってしまいます。
しかも、クライアント側で、もし hanako が UID 500 に割り当たっていたとすると、 hanako は サーバー上の taro のファイルにアクセスできてしまうことになる。

ディストリビューションによって、一般ユーザーの UID に使われる範囲が違います。
RedHat系は 500～、Ubuntuなどでは 1000～ が使われるようです。

ですから、 NIS や LDAP などでユーザー情報を一元管理しているか、気をつけてユーザーID を設定している場合を除いては、サーバー/クライアント間で UID や GID は一致しないことが多いです。

そもそも「サーバー/クライアント間でユーザーに同じ UID/GID が使われている」という前提は期待されるべきものでもないと思うのです。

exports の古い man ページを見るとかつて `map_static` というオプションがあって、 サーバー/クライアント間で UID と GID を再マッピングできたみたいなのですが、すでに廃止されていているみたいです。

そこで、NFSv4 + Kerberos version5 をセットアップし、上記の「セキュリティの問題」と「ユーザーマッピングの問題」を解決してみました。

ユーザーマッピングについては NFSv4 で idmapd デーモンがやってくれるので、Kerberos までは必要ないようにも思えます。
ところが、NFSv4 単独だと、ユーザーマッピングがうまく動きませんでした。
検索してみたところ、 「idmapd が期待通り動かない」という書き込みが多かったですし、私も実際試してみたのですが、結局やり方がわかりませんでした。

一方、認証を Kerberos に任せた場合は、ユーザーマッピングもうまく動作しました。

最初、 RHEL/CentOS での設定をまとめます。その後、 Ubuntu での設定も簡単に説明します。

前提として、すでに Kerberos の基本的な設定は済んでいるものとします。
この部分は、前回 [Kerberos を使ってみる(RHEL/CentOS/Ubuntu編)]({{ BASE_PATH }}/2012/12/19/02-kerberos/)の記事で説明しました。

### RHEL/CentOS編 ###

RHEL6.3/CentOS6.3で確認しています。

ここでは説明上、

- NFSサーバーのホスト名: `nfsserver.mydomain.com`
- NFSクライアントのホスト名: `nfsclient.mydomain.com`

とする。

NFSサーバー/NFSクライアント両方とも `/etc/sysconfig/nfs` で

    SECURE_NFS="yes"

の行を有効にしておきます。

さらに、どのホストからでもいいですけど、

    kadmin: addprinc -randkey nfs/nfsserver.mydomain.com
    kadmin: addprinc -randkey nfs/nfsclient.mydomain.com

として、サーバー側、クライアント側の両方のサービスプリンシパルを作成する。

通常のサービスではサーバー側だけサービスプリンシパルを作成するのですが、NFS については サーバー/クライアント両方とも必要です。

`ktadd` はおのおののホスト上で行わないといけません。

NFSサーバー (`nfsserver.mydomain.com`)上で

    kadmin: ktadd nfs/nfsserver.mydomain.com

NFSクライアント(`nfsclient.mydomain.com`)上で

    kadmin: ktadd nfs/nfsclient.mydomain.com

とする。

それぞれのホストの `/etc/krb5.keytab` にキーが保存されるはず。

ちゃんと保存されているか念のために確認してみます。

NFSサーバー側の結果：

    [root@nfsserver] # klist -e -k /etc/krb5.keytab
    Keytab name: WRFILE:/etc/krb5.keytab
    KVNO Principal
    ---- --------------------------------------------------------------------------
       2 nfs/nfsserver.mydomain.com@MYDOMAIN.COM (aes256-cts-hmac-sha1-96)
       2 nfs/nfsserver.mydomain.com@MYDOMAIN.COM (aes128-cts-hmac-sha1-96)
       2 nfs/nfsserver.mydomain.com@MYDOMAIN.COM (des3-cbc-sha1)
       2 nfs/nfsserver.mydomain.com@MYDOMAIN.COM (arcfour-hmac)
       2 nfs/nfsserver.mydomain.com@MYDOMAIN.COM (des-hmac-sha1)
       2 nfs/nfsserver.mydomain.com@MYDOMAIN.COM (des-cbc-md5)

NFSクライアント側の結果：

    [root@nfsclient]# klist -e -k /etc/krb5.keytab
    Keytab name: WRFILE:/etc/krb5.keytab
    KVNO Principal
    ---- --------------------------------------------------------------------------
       3 nfs/nfsclient.mydomain.com@MYDOMAIN.COM (aes256-cts-hmac-sha1-96)
       3 nfs/nfsclient.mydomain.com@MYDOMAIN.COM (aes128-cts-hmac-sha1-96)
       3 nfs/nfsclient.mydomain.com@MYDOMAIN.COM (des3-cbc-sha1)
       3 nfs/nfsclient.mydomain.com@MYDOMAIN.COM (arcfour-hmac)
       3 nfs/nfsclient.mydomain.com@MYDOMAIN.COM (des-hmac-sha1)
       3 nfs/nfsclient.mydomain.com@MYDOMAIN.COM (des-cbc-md5)

また、NFSサーバー側の `/etc/exports` の書き方は

    /home    gss/krb5(fsid=0,crossmnt,rw,sync)

などとなります。公開先のホスト名の部分を `gss/krb5` とする部分がポイントです。

あとはクライアント側から

    # mount -t nfs4 -o sec=krb5 nfsserver:/  hogehoge

といった感じでマウントができます。 (実は `-t nfs4` の部分はなくてもよい。)

マウントはこれでできますが、実際にアクセスするには、ユーザーは Kerberos 認証を済ませないといけません。

    $ kinit

でTGT を入手しておけば、アクセス可能になります。

なお、NFSv4 + Kerberos を解説したページで `ktadd` するときに `-e des-cbc-crc:normal` オプションを付けると説明しているページもありますが、現在はこのオプションは不要ですので、このオプションは付けないでください。
`dec-cbc-crc` 自体が廃止で、エラーになります。

### Ubuntu編 ###

Ubuntu 12.04 LTS で確認しました。

やり方は、RHEL/CentOS とほとんど同じなので、差分だけ説明します。

NFSサーバーは `nfs-kernel-server` パッケージ、NFSクライアントは `nfs-common` パッケージのインストールが必要です。

RHEL/CentOS で `/etc/sysconfig/nfs` で

    SECURE_NFS="yes"

を設定するとなっている部分は Ubuntu では不要です。

その代わり、Ubuntu では、NFSサーバーの `/etc/default/nfs-kernel-server` に

    NEED_SVCGSSD=yes

NFSクライアントの `/etc/default/nfs-common` に

    NEED_GSSD=yes

を追加します。

あとは、同じやり方でできます。

### RHEL/CentOS と Ubuntu の組み合わせ ###

RHEL/CentOS 同士、 Ubuntu 同士の組み合わせで説明しましたが、もちろん、 CentOSのNFSサーバー と Ubuntu クライアントといった組み合わせでも動きました。
CentOS では UID は 500～ から使われていて、Ubuntuでは UID は 1000～ なので、当然ながら UID/GID はまったくバラバラなのですが、うまくユーザーマッピングして動いてくれています。
