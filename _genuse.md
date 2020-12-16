# パーティションの設定
* 今回はパーティションはインストーラに任せた。基本的に１つのパーティションでの運用とすることにした。  
* ネットワークの設定も、インストール時に行った。  
* OpenSSH のみを追加した。

---
# ~~インストール直後~~
* ssh 接続ができるように、netplan を書き換えた
```
# cd /etc/netplan

network:
    ethernets:
        enp4s0:
            addresses:
            - 192.168.100.5/24
            gateway4: 192.168.100.1
            nameservers:
                addresses:
                - 8.8.8.8
            routes:
            - to: 192.168.0.0/24
              via: 192.168.100.2
    version: 2
```
* 以上で ssh 接続が可能となる。

---
# ssh をルートでログインできるように
```
# vim /etc/ssh/sshd_config

Port ***
PermitRootLogin prohibit-password
```

* 公開鍵の作成
```
# ssh-keygen -t -rsa

# cd .ssh/
# cat id_rsa.pub >> authorized_keys
# systemctl restart sshd
```
* id_rsa を、Win 側に転送し、RLogin に登録する

---
# インストール後に行ったこと
```
# apt update（ソフトの更新確認）
# apt upgrade（ソフトの更新）

# ufw status（inactive の確認）

タイムゾーンの設定（これを設定するだけでタイムゾーンが即時変更される）
# ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
# date（設定変更の確認）
```

* vim の設定  
~/.bashrc に以下を記述
```
PS1="\[\e[1;36m\][SMB \w]\[\e[0m\]\n\\$ "
alias vim='vim -c start'

緑色： 32 / 黄色： 33 / 青色： 34 / パープル： 35 / シアン： 36 / 白色: 37
```

* ディスプレイのスリープの設定  
ログインしなくても、自動的にディスプレイをスリープにさせるために、grub で設定を行う
```
# vim /etc/default/grub

以下を追記
GRUB_CMDLINE_LINUX_DEFAULT="consoleblank=60"

# update-grub
```

* ~~S.M.A.R.T のインストール~~
```
# apt install smartmontools
# smartctl --version（インストールの確認）
```

---
# code-server v3 のインストール
参考URL:  
https://github.com/cdr/code-server  
https://github.com/cdr/code-server/blob/v3.7.4/doc/install.md
```
# curl -fsSL https://code-server.dev/install.sh | sh
...
To have systemd start code-server now and restart on boot:
  sudo systemctl enable --now code-server@$USER
Or, if you don't want/need a background service you can run:
  code-server
```
* 設定ファイルの書き換え
```
# cd ~/.config/code-server
# vim config.yaml

bind-addr: 0.0.0.0:****
（auth: none）
```
* code-server の起動
```
# code-server
```
* 右下の設定アイコンから「Color Theme」を選択 -> Dark テーマに変更  
* 「Ctrl + Shift + p」でコマンドパレットを開き、「setting.json」を選択し、適切に設定する

---
# samba の設定
* 別ページを参照のこと

---
# Ramdisk の設定
* 別ページを参照のこと

---
# ~~外付けHDDとの接続について（UAS無効化）~~
* 別ページを参照のこと

---
# ~~udev を用いた 外付けHDD の自動マウント~~
* 外付け対象となる HDD の情報が知りたい場合　# lsblk -f

* ネットに掲載されている情報のように、udev から直接 mount コマンドを起動させても、デバイスの準備が間に合っていないらしくてマウントできたなかった。そのため、systemd のサービスを用いて mount コマンドを実行させることにする。

* smartclt をインストールしておく
```
# smartctl --version
# apt install smartmontools
```

* 外付けHDD がマウントされるマウントポイントを作成する　# mkdir /home/shared/APPZ_01

* udev に systemd のサービスを start させる rule を設定する　`# vi /etc/udev/rules.d/99-local.rules`  
または、保存されている 99-local.rules を転送すれば良い　`# cp 99-local.rules /etc/udev/rules.d/`
```
ACTION=="add", ENV{DEVTYPE}=="partition", ENV{ID_FS_LABEL}=="APPZ_01", RUN+="/bin/systemctl start hdd-automount@%k.service"
```
上記において、%k は KERNEL を表し、sdb1 などのデバイス名が設定される。  
systemctl start hdd-automount@sdb1.service とすると、hdd-automount@.service が、%i = sdb1 として start される。（%i はインスタンス名という意味らしい）

* systemd に、バッチファイルを起動させるサービスを登録する

\# vim /etc/systemd/system/hdd-automount@.service
```
[Unit]
Description = hdd-auto-mount on %i

[Service]
ExecStart = /home/shared/hdd-automount.sh %i
RemainAfterExit = yes
Type = simple

[Install]
WantedBy = multi-user.target
```
* systemd から実行されるバッチファイルを作成する

\# vim /home/shared/hdd-automount.sh
```
#!/bin/bash

DEVBASE=$1
DEVICE="/dev/${DEVBASE}"

# ID_FS_LABEL に値を設定させる
eval $(/sbin/blkid -o udev ${DEVICE})

LABEL=${ID_FS_LABEL}
if [[ -z "${LABEL}" ]]; then
  exit 1
fi

MOUNT_POINT="/home/shared/${LABEL}"
/bin/mount -o async,rw,noatime ${DEVICE} ${MOUNT_POINT}


BASH_ON_START="{ echo; echo '-----------------------------------------------'; date; smartctl -A ${DEVICE/%?}; } >> ${MOUNT_POINT}/smartctl_on_start.log"
/bin/bash -c "${BASH_ON_START}"

BASH_OPR="while true; do sleep 540s; { echo; date; smartctl -A ${DEVICE/%?} | grep -e '1 Raw' -e '5 Real' -e '197 Cur'; } >> ${MOUNT_POINT}/smartctl.log; done"
/bin/bash -c "${BASH_OPR}" &
```

* hdd-automount.sh に実行権限を与える  
\# chmod +x hdd-automount.sh

---
# ~~自動マウントを考える際、有益となるコマンド~~
* udevadm info --query=all -n /dev/sdb1
* udevadm control --reload
* systemctl daemon-reload

---
# ~~(参考) 自動マウントを行うもう１つの方法~~
* udisksctl の monitor 機能を用いて、外付けHDDの自動マウントを実現することも可能であった。  
ただ、ずっと monitor しているのは、あまり好ましいと思わなかったので、この方法は採用していない。

* 現状の確認（ラベルを確認する）　# lsblk

* シェルスクリプトファイルを作成

\# vim /home/shared/HDD_mount.sh
```
#!/bin/bash
udisksctl monitor | while read ev; do
　　if [[ $ev =~ /dev/disk/by-label/APPZ_01 ]]; then
　　　　mount -L APPZ_01 /home/shared/APPZ_01
　　fi
done
```

* HDD_mount.sh を起動時に自動実行するように設定する。Systemd の機能を用いる。

\# vim /etc/systemd/system/HDD_mount.service
```
[Unit]
Description = HDD-auto-mount daemon

[Service]
ExecStart = /home/shared/HDD_mount.sh
Restart = always
Type = simple

[Install]
WantedBy = multi-user.target
```

* 上記で作ったサービスの自動起動の設定
```
# systemctl start HDD_mount.service
# systemctl enable HDD_mount.service
```

---
# ipatables の設定
* メインマシンから、ipt+++.sh を転送して、バッチファイルを実行　# sh ipt+++.sh
* 設定した iptables の恒久化 `# apt install iptables-persistent`（apt を upgrade しておかないと install できないため注意）  
または、`# netfilter-persistent save`。
* 念の為に確認　# ufw status（inactive になっていることを確認。FW の設定は、全て iptables で行う）
* iptables に関する syslog を分離する。（別ページを参照のこと）

---
# ~~IPv6 の無効化~~
* 別ページを参照のこと
* 家庭内LAN においては、NGN を通すわけではないから、v6 と v4 のパケット変換は発生しない。

---
# node.js のインストール
* `# apt install nodejs` とすると、古いバージョンがインストールされる（2020-12 時点では、v8.10）  
* n package を利用して、以下のようにインストールする
```
まず、n package を動かすために、古いものでいいので nodejs をインストールする
# apt install nodejs npm
# npm install n -g

安定バージョンをインストール（新しい nodejs は、/usr/local/bin にインストールされる）
# n stable

新しい nodejs がインストールされたため、古いものは削除
# apt purge -y nodejs npm

環境変数 node を更新するため、シェルを再起動
# exec $SHELL -l

バージョン確認
# node -v

（参考）
以下のようにすると、任意のバージョンに切り替えることができる
# n 9.11.2
```

---
# ~~種々の作業~~
* pv をインストール　`# apt install pv`
* `_ramdisk_copy.sh` をコピー
* `kill_smartctl.js` をコピーして、node で起動するためのバッチファイルを作成

---
# docker のインストール

