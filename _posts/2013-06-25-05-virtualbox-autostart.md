---
layout: post
title: VirtualBox 自動起動
description: ""
tags: [Ubuntu]
---
{% include JB/setup %}

ホストOSが起動した時に、ゲストOSも自動的に起動して欲しい！そう思ってやってみました。

動作確認は Ubuntu 12.04 LTS および Ubuntu 13.04 で行なっています。

- VirtualBox 本家からダウンロードしてインストールした場合
- Ubuntu 13.04 で apt-get でインストールした場合
- Ubuntu 12.04LTS で apt-get でインストールした場合

の順番に説明します。たぶん、最初のが正式？なやり方。


### 本家からダウンロードする場合 ###

<http://www.virtualbox.org/manual/ch09.html#autostart-linux>
に記載があるのですが、VirtualBox 4.2 から autostart に対応しています。

一応、Ubuntu でやってみましたが、同じやり方でほぼすべてのディストリビューションで動くはず。

#### インストール方法 ####

<https://www.virtualbox.org/wiki/Downloads> のページに行き、
自分のディストリビューション用のパッケージをダウンロードしてきて、インストールする。
(現時点では VirtualBox 4.2.12 for Linux というのがダウンロードできるようだ。)

#### 設定方法 ####

    $ sudo -i
    # mkdir -p /etc/vbox/autostart_db
    # chmod 1777 /etc/vbox/autostart_db
    # echo "default_policy = allow" > /etc/vbox/autostart.cfg

みたいな感じで、データベースディレクトリ `/etc/vbox/autostart_db` およびコンフィグファイル `/etc/vbox/autostart.cfg` を作ります。

名前は必ずしもこの通りでなくてもいいです。
後で `VBOXAUTOSTART_DB` 変数と `VBOXAUTOSTART_CONFIG` 変数にこれらのパスをセットするので。

`/etc/vbox/autostart_db` ディレクトリは一般ユーザーが設定ファイルを作成したり、削除したりするので、全員に書き込み権限を与えます。
ただし、他のユーザーの設定ファイルを勝手に削除できないように、sticky ビットを立てておきます。

`/etc/vbox/autostart.cfg` のコンフィグファイルは

    # Default policy is to deny starting a VM, the other option is "allow".
    default_policy = deny
    
    # Bob is allowed to start virtual machines but starting them
    # will be delayed for 10 seconds
    bob = {
        allow = true
        startup_delay = 10
    }
    
    # Alice is not allowed to start virtual machines, useful to exclude certain users
    # if the default policy is set to allow.
    alice = {
        allow = false
    }

みたいな感じでユーザーごとに設定することもできるが、とりあえず

    default_policy = allow

の1行だけ書いておけば動きます。

次に、 `/etc/vbox/vbox.cfg` (または `/etc/default/virtualbox`) というファイルを作って、データベースディレクトリと設定ファイルへのパスを追加します。

    VBOXAUTOSTART_DB=/etc/vbox/autostart_db
    VBOXAUTOSTART_CONFIG=/etc/vbox/autostart.cfg

みたいに追加します。

続いて、ユーザーごとの設定をします。

    $ VBoxManage setproperty autostartdbpath /etc/vbox/autostart_db
    $ VBoxManage modifyvm VM_NAME --autostart-enabled on

とします。 (`VM_NAME` の部分は動かしたい仮想マシン名に置きかえてください。)

上記をすると、 `~/VirtualBox VMs/VM_NAME/VM_NAME.vbox` に以下のように Autostart enabled の設定が書き込まれます。

             <AttachedDevice passthrough="false" type="DVD" port="0" device="0"/>
           </StorageController>
         </StorageControllers>
    +    <Autostart enabled="true" delay="0" autostop="Disabled"/>
       </Machine>
     </VirtualBox>

さらに `/etc/vbox/autostart_db/` 以下に `USER_NAME.start` というファイルが作られ 中身は `1` が書かれています。
ファイルの中身は、自動起動する仮想マシンの台数を表しています。

もう一台 `--autostart-enabled on` すると `2` になります。
中身が `0` になるとファイル削除されます。

`VM_NAME.vbox` とか `/etc/vbox/autostart_db/` 以下のファイルを直接いじると辻褄が合わなくなるので、理屈を理解しないうちは

    $ VBoxManage modifyvm VM_NAME --autostart-enabled on

とか

    $ VBoxManage modifyvm VM_NAME --autostart-enabled off

とかコマンドで操作するのがよいです。面倒ですけどね。。

ちなみに

    $ VBoxManage modifyvm VM_NAME --autostop-type acpishutdown

として autostop の設定もすることはできますが、現時点では autostop は動かないらしい。。

以上で、設定は完了です。

再起動すると、VirtualBox も自動起動するはず、、、と思ったら起動しない。

ちょっと落とし穴がありました。

`/etc/init.d/vboxautostart-service` の中身を見てみると

    vboxdrvrunning() {
        lsmod | grep -q "vboxdrv[^_-]"
    }

のように vboxdrv モジュールがロードされているかどうかを確認するコードがあります。
vboxdrv をロードしているのは `/etc/init.d/vboxdrv` です。

つまり、 `/etc/init.d/vboxdrv` より後に `/etc/init.d/vboxautostart-service` が実行されないといけません。

`/etc/rc2.d/` の下を `ls -l` で見てみると、

    S20vboxautostart-service -> ../init.d/vboxautostart-service
    S20vboxdrv -> ../init.d/vboxdrv

となっています。怪しいな。。

そこで `S20vboxdrv` を `S19vboxdrv` にリネームしてみました。

    S19vboxdrv -> ../init.d/vboxdrv
    S20vboxautostart-service -> ../init.d/vboxautostart-service

これで再起動すると、、バンザーイ。動きました。

`/etc/init.d/virtualbox` は簡単なシェルスクリプトなので読んでみると、起動時のシーケンスは

1. `/etc/init.d/vboxautostart-service` を実行
2. `/etc/vbox/vbox.cfg` と `/etc/default/virtualbox` 読み込む
3. `/etc/vbox/autostart_db/*.start` を見て、ユーザーを確認
4. ユーザーごとに `/usr/lib/virtualbox/VBoxAutostart` を実行。引数に `/etc/vbox/autostart.cfg` を渡す

という感じになっているようです。

### Ubuntu 13.04 で apt-get でインストールした場合 ###

Ubuntu 13.04 で

    $ sudo apt-get install virtualbox

でインストールした場合です。

この場合、VirtualBox4.2.10 がインストールされますが、本家サイトからパッケージをダウンロードしてきた場合と、 etc ファイルの構成がかなり異なります。

本家パッケージをインストールしたときにあった

- `/etc/init.d/vboxautostart-service`
- `/etc/init.d/vboxballoonctrl-service`
- `/etc/init.d/vboxdrv`
- `/etc/init.d/vboxweb-service`
- `/etc/vbox` ディレクトリ

が、Ubuntu 13.04 で apt でインストールした場合にはありません。

その代わり、

- `/etc/init.d/virtualbox` : `/etc/init.d/vboxdrv` の代わり
- `/etc/defauult/virtualbox`

があります。

`/etc/init.d/vboxautostart-service` が存在しませんので、代わりとして `/etc/rc.local` に `VBoxAutostart` を叩くコードを追加してみました。以下のような感じです。

    binary=/usr/lib/virtualbox/VBoxAutostart
    /sbin/start-stop-daemon --background --chuid USER_NAME --start --exec $binary -- --background --start --config /etc/vbox/autostart.cfg

`USER_NAME` の部分は、動かしたい仮想マシンの所有者に置き換えてください。

次に `/etc/vbox/autostart.cfg` を作り、以下の内容を記載。

    default_policy = allow

あとは自動起動したい仮想マシンの `VirtualBox VMs/VM_NAME/VM_NAME.vbox` を開き、以下のように1行追加。

             <AttachedDevice passthrough="false" type="DVD" port="0" device="0"/>
           </StorageController>
         </StorageControllers>
    +    <Autostart enabled="true" delay="0" autostop="Disabled"/>
       </Machine>
     </VirtualBox>

### Ubuntu 12.04LTS で apt-get でインストールした場合 ###

Ubuntu 12.04 LTS で

    $ sudo apt-get install virtualbox

でインストールした場合です。

VirtualBox のバージョンは 4.1.12 で `/usr/lib/virtualbox/VBoxAutostart` も存在しないので、上記のやり方は使えません。

`/etc/rc.local` に以下の行を追加しておきます。

    su - USER_NAME -c "VBoxHeadless -startvm VM_NAME &"

`USER_NAME` の部分はユーザー名に、`VM_NAME` の部分は動かしたいバーチャルマシンに置き換えてください。

というか、これが一番簡単かも。VirtualBox 4.2 以前でも使えるし。
