# パーティションの設定
```
https://shell-mag.com/2nd_linuxoperations/
```
* 今回は、64G-SSD を利用した  
* EFIシステムパーティションを先頭に作る（100M）  
* /boot を作成する（1G）
```
/bootの容量は、カーネルと初期RAMディスクのイメージを合わせて200Mバイト程度です。
ただし、カーネルをアップデートすると、古いイメージが残ったまま新しいイメージが格納されます。
何世代か残しておいても問題ないように、/bootには1G～2Gバイト程度割り当てます
```
* スワップ領域を末尾に作成する（8G）  
* 残りは全て /root に割り当てた（/home, /var を分けても良い）

---
# インストール時の追加パッケージ
* Samba と OpenSSH の２つ

---
# インストール後に行ったこと
```
# sudo apt update（ソフトの更新確認）
# sudo apt upgrade（ソフトの更新）

# ufw status（inactive の確認）

タイムゾーンの設定（これを設定するだけでタイムゾーンが即時変更される）
# ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
# date（設定変更の確認）
```

* vim の設定  
~/.bashrc に以下を記述
```
alias vim='vim -c start'
```

* ディスプレイのスリープの設定  
ログインしなくても、自動的にディスプレイをスリープにさせるために、grub で設定を行う
```
# vim /etc/default/grub

以下を追記
GRUB_CMDLINE_LINUX_DEFAULT="consoleblank=60"

# update-grub
```

* S.M.A.R.T のインストール
```
# apt install smartmontools
# smartctl --version（インストールの確認）
```

---
# samba の設定
* インストールの確認　# samba -V
* インストールする場合　# apt install samba
* samba の設定ファイルを変更　# vim /etc/samba/smb.conf  
以下、変更箇所
```
[global]  
server string = SAMBA SERVER Version %v  
wins support = no  
wins proxy = no  
dns proxy = no（不必要かも？？）  

dos charset = CP932  
load printers = no  
; disable spoolss = yes（不要そうであるため、今は外している）  
; syslog = 0　　syslog は deprecated らしい。log は、全て var/log/samba に書き出すものとする  
disable netbios = yes（netbios は使用しない）  

security = user  
（passdb backend = tdbsam はデフォルトで設定されていた）  
usershare allow guests = no（ユーザー定義共有という新しい機能らしい）  
browseable = yes（no にすると、エクスプローラ上での表示がされなくなった）  

# ネットワークの効率化を狙う、Windows固有の機能。複数のプロセスが同じファイルをロックでき、なおかつクライアントがデータをキャッシュできる。
# この機能は、ファイルの破壊をもたらす可能性があるため、OFF にしておく
oplocks = No
level2 oplocks = No
blocking locks = No


以下のセクションを追加  
[NUC-SMB]（[]の中は好きな文字列でよい。Windowsでアクセスしたときに表示されるフォルダ名となる）  
comment = NUC-SMB  
path = /home/shared  
browseable = yes  
read only = no  
valid users = user-k  
guest ok = no  
writable = yes  
create mode = 0600  
directory mode = 0700  

以下のセクションをすべてコメントアウト（プリンタサーバ機能は利用しない）  
[printers]  
[print$]  
```
* 設定ファイルを設定し終えたら、ファイルの構文チェック　# testparm
* 共有ディレクトリの作成　# mkdir /home/shared
* Samba のユーザー登録。Linux に存在するユーザーで登録する必要がある。  
Linux と Samba では暗号化方式が異なるため、同じパスワードでも両方に登録が必要になる。  
\# pdbedit -a user-k

* ディレクトリにアクセスできるように権限を変更しておく  
\# chown user-k:user-k /home/shared/  
\# chmod 774 /home/shared/  
（後に、/home/shared は、ramdisk に変わるため、この変更は不必要になる。ramdisk の権限で書き換えられてしまう）

* 一応サービスの設定
```
# systemctl enable smbd
# systemctl disable nmbd（netbios を使用しないため、nmb は必要ない）
# systemctl mask nmbd
```
* TCP 445 を利用するため、FW の設定には注意すること

---
# Ramdisk の設定
* 別ページを参照のこと

---
# 外付けHDDとの接続について（UAS無効化）
* JMS567 との相性が悪いとき（転送速度が 100MB/s を下回るとき）は、UASドライバーを無効化すると良い  
（通常の usb-storage ドライバに変更する）
```
まず、JMS567 のベンダIDとデバイスIDを調べる
# lsusb
# lsusb -t
ベンダIDとデバイスIDが、152d:0567 と分かると思う。

/etc/default/grub に以下を追記
GRUB_CMDLINE_LINUX="usb_storage.quirks=152d:0567:u"

# update-grub
# reboot

「Driver=usb-storage」となっているかを確認
# lsusb -t

参考：
https://www.smartmontools.org/wiki/SAT-with-UAS-Linux
https://www.atmarkit.co.jp/ait/articles/1902/01/news045.html
```

---
# udev を用いた 外付けHDD の自動マウント
* 外付け対象となる HDD の情報が知りたい場合　# lsblk -f
* ネットに掲載されている情報のように、udev から直接 mount コマンドを起動させても、どうもデバイスの準備が間に合っていないらしくてマウントできたなかった。そのため、systemd のサービスを用いて mount コマンドを実行させることにする。
* 外付けHDD がマウントされるマウントポイントを作成する　# mkdir /home/shared/APPZ_01
* udev に systemd のサービスを start させる rule を設定する　# vi /etc/udev/rules.d/99-local.rules  
または、保存されている 99-local.rules を転送すれば良い　# cp 99-local.rules /etc/udev/rules.d/
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
# 自動マウントを考える際、有益となるコマンド
* udevadm info --query=all -n /dev/sdb1
* udevadm control --reload
* systemctl daemon-reload

---
# ipatables の設定
* メインマシンから、ipt+++.sh を転送して、バッチファイルを実行　# sh ipt+++.sh
* 設定した iptables の恒久化　# apt install iptables-persistent（apt を upgrade しておかないと install できないため注意）
* 念の為に確認　# ufw status（inactive になっていることを確認。FW の設定は、全て iptables で行う）
* ptables に関する syslog を分離する。  
iptables のログ出力は syslog が担っているため、syslog の出力にフィルタを掛ける。  
/etc/rsyslog.d/ にフィルタを記述したファイルを置けば、フィルタが掛かるようになる。  

\# touch /etc/rsyslog.d/10-my-iptables.conf  
\# vim /etc/rsyslog.d/10-my-iptables.conf
```
:msg,contains,"Drp_" -/var/log/iptables.log
& ~
```
ファイル指定の前の「-」は、ログ出力時のディスクとの同期の抑制を示す。（パフォーマンス向上が狙える）  
「& ~」は、対象としているログを廃棄する、ということを示す。

* rsyslog を再起動　# systemctl restart rsyslog  
  
参考URL：  
http://www.geocities.jp/yasasikukaitou/rsyslog-filter.html  
https://vogel.at.webry.info/201311/article_4.html

---
# IPv6 の無効化（不要な inbound 通信を防ぐため）

* まず、IPv6 のアドレスを持っていることを確認　# ip addr 
```
ens2:  mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:08:d2:ef brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.40/24 brd 192.168.10.255 scope global ens2
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe08:d2ef/64 scope link
       valid_lft forever preferred_lft forever
```
* /etc/default/grub を編集する。以下の文言を追加
```
GRUB_CMDLINE_LINUX_DEFAULT="ipv6.disable=1"
GRUB_CMDLINE_LINUX="ipv6.disable=1"
```
* \# update-grub 
* \# reboot
* inet6 の行が消えていることを確認する　# ip addr

---
# IPv6 の無効化２（種々の不具合を避けるために、リンクリーカルのみを残す方法）

* IPv6 の RA を受け取らないようにする（accept-ra: no）
```
vim /etc/netplan/01-netcfg.yaml

network:
  version: 2
  renderer: networkd
  ethernets:
    enp4s0:
      dhcp4: no
      dhcp6: no
      accept-ra: no
      addresses: [ 192.168.100.5/24 ]
      gateway4: 192.168.100.1
      nameservers:
          addresses: ["8.8.8.8"]
```
* \# reboot
* \# ip a
* inet6 に、リンクローカルアドレスのみがあることを確認する

---
# ssh ポート番号の変更等
* \# vim /etc/ssh/sshd_config
```
Port 3002
（？？これはエラーとなった）AddressFamily inet
（？？これはエラーとなった）ListenAddress 192.168.100.2   # GATE_study
上記の代替策として、iptables で受け入れ先を絞ることにした
```

---
# iptables の変更
```
sh ipt-...
/etc/init.d/netfilter-persistent save
/etc/init.d/netfilter-persistent reload
```

