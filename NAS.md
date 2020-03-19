# HDD のフォーマット  
lsblk にてデバイスを確認。その後、「# fdisk /dev/sd*」でパーティションの生成を行う。  
* パーティションの削除は「d」  
* パーティションの作成は「n」  
* パーティション操作を指定し終えたら「w」。自動的に fdisk を抜けてしまう。

fdisk を抜けたら、「# mkfs -t ext4 /dev/sd*」でフォーマットを実行する。１分程度で終わるはず。

* ラベルを付けるときは　# e2label /dev/sdb1 APPZ_01  
* ラベルを確認するときは　# e2label /dev/sdb1

---
# samba の設定  
* インストールの確認　# samba -V  
* インストールする場合　# apt install samba  
* samba の設定ファイルは「/etc/samba/smb.conf」

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

* 設定ファイルを設定し終えたら、ファイルの構文チェック　\# testparm  

* 共有ディレクトリの作成  
  \# cd /home  
  \# mkdir shared

* Samba のユーザー登録。Linux に存在するユーザーで登録する必要がある。  
  Linux と Samba では暗号化方式が異なるため、同じパスワードでも両方に登録が必要になる。  
  \# pdbedit -a user-k

* ディレクトリにアクセスできるように権限を変更しておく  
  \# chown user-k:user-k /home/shared/  
  \# chmod 774 /home/shared/
 
* 一応サービスの設定  
  \# systemctl enable smbd  
  \# systemctl disable nmbd（netbios を使用しないため、nmb は必要ない）
  
* TCP 445 を利用するため、FW の設定には注意すること
---

# USB 外付けHDD の自動マウント  
ラベルを用いて管理する方法を用いる。  

* まずは現状の確認　# lsblk  
* ネットの情報にあるように、udev を用いて自動マウントはうまくいかなかった。udisksctl の monitor 機能を用いて、外付けHDDの自動マウントを実現することにした。

* まずは、シェルスクリプトファイルを作成
> \# touch /home/shared/HDD_mount.sh  
> \# vi /home/shared HDD_mount.sh

#!/bin/bash  
udisksctl monitor | while read ev; do  
　　if [[ $ev =~ /dev/disk/by-label/APPZ_01 ]]; then  
　　　　mount -L APPZ_01 /home/shared/APPZ_01  
　　fi  
done  

* HDD_mount.sh を起動時に自動実行するように設定する。Systemd の機能を用いる。  
> /etc/systemd/system/ の下にUnit定義ファイルを作る  
> \# touch /etc/systemd/system/HDD_mount.service  
> \# vi /etc/systemd/system/HDD_mount.service

[Unit]  
Description = HDD-auto-mount daemon  

[Service]  
ExecStart = /home/shared/HDD_mount.sh  
Restart = always  
Type = simple  

[Install]  
WantedBy = multi-user.target  

* 自動起動するように設定しておく  
> \# systemctl start HDD_mount.service  
> \# systemctl enable HDD_mount.service
---

# udev を用いた 外付けHDD の自動マウント  
* 外付け対象となる HDD の情報が知りたい場合　# lsblk -f  
* ネットに掲載されている情報のように、udev から直接 mount コマンドを起動させても、どうもデバイスの準備が間に合っていないらしくてマウントできたなかった。そのため、systemd のサービスを用いて mount コマンドを実行させることにする。

* smartclt をインストールしておく
```
# smartctl --version
# apt install smartmontools
```

* 外付けHDD がマウントされるマウントポイントを作成する　# mkdir /home/shared/APPZ_01

* udev に systemd のサービスを start させる rule を設定する　# vi /etc/udev/rules.d/99-local.rules
```
ACTION=="add", ENV{DEVTYPE}=="partition", ENV{ID_FS_LABEL}=="APPZ_01", RUN+="/bin/systemctl start hdd-automount@%k.service"
```

上記において、%k は KERNEL を表し、sdb1 などのデバイス名が設定される。  
systemctl start hdd-automount@sdb1.service とすると、hdd-automount\@.service が、%i = sdb1 として start される。（%i はインスタンス名という意味らしい）  

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
\#vi /home/shared/hdd-automount.sh  
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
# Ramdisk の設定
* マウント先の作成　# mkdir /home/shared/ramdisk
* fstab の書き換え　# vim /etc/fstab  
以下を追記
```
# /tmp, /var/tmp -> Ramdisk
tmpfs /tmp tmpfs defaults,size=256m,noatime,mode=1777 0 0
tmpfs /var/tmp tmpfs defaults,size=256m,noatime,mode=1777 0 0

# /var/log -> Ramdisk
tmpfs /var/log tmpfs defaults,size=256m,noatime,mode=0755 0 0

# 追加の Ramdisk
tmpfs /home/shared/ramdisk tmpfs defaults,size=12G,noatime,mode=0700,uid=user-k,gid=user-k 0 0
（rsync -a を用いると、ramdisk の属性が、転送先フォルダの属性にコピーされるため注意すること）
```
（参考）NUC-SMB は、１回の使用時間が短いため、Ramdisk のサイズを size=256m とした

noatime： アクセスした際に、アクセス時のタイムスタンプの変更をしない  
mode： アクセス権。先頭の数字はスティッキービット  
５番目の数字： バックアップを作る時を決定するために dump ユーティリティによって使われる。通常は０でよい  
６番目の数字： ファイルシステムをチェックする順番を決めるために fsck によって使われる。RamDiskは０でよい  

* 起動時にRAMディスクにディレクトリを作成するように設定する。一部のサービスは、サービス起動時にワーキングディレクトリの存在が必要となる。  
\# touch /etc/rc.local（CentOS とはファイル位置が異なるため注意）  
\# vim /etc/rc.local  
以下を追記する
```
#!/bin/bash

mkdir -p /var/log/apt
mkdir -p /var/log/dist-upgrade
mkdir -p /var/log/fsck
mkdir -p /var/log/installer
mkdir -p /var/log/journal
mkdir -p /var/log/landscape
mkdir -p /var/log/lxd
mkdir -p /var/log/samba
mkdir -p /var/log/unattended-upgrades

chown root.systemd-journal /var/log/journal
chown landscape.landscape /var/log/landscape
chown root.adm /var/log/samba
chown root.adm /var/log/unattended-upgrades

# Create Lastlog, wtmp, btmp
touch /var/log/lastlog
touch /var/log/wtmp
touch /var/log/btmp

chown root.utmp /var/log/lastlog
chown root.utmp /var/log/wtmp
chown root.utmp /var/log/btmp
```

* 上記の rc.local に、実行権限を与える　# chmod +x /etc/rc.local
* smbd より先に rc.local が実行されるように設定する　# vim /lib/systemd/system/smbd.service
```
[Unit] の After を以下のように変更する。

After=network.target rc-local.service
（nmbd と winbind は利用しないため、削除してよい。winbind は security=domain などの場合に利用されるデーモン）
```


* Ramdisk に移行するフォルダのファイルを空にしておく
```
# find /tmp -type f
# find /tmp -type f -exec cp -f /dev/null {} \;

# find /var/tmp -type f
# find /var/tmp -type f -exec cp -f /dev/null {} \;

# find /var/log -type f
# find /var/log -type f -exec cp -f /dev/null {} \;
```
（参考）ネット上の情報では「# find /var/log/ -type f -name * -exec cp -f /dev/null {} ;」となってたが、-name オプションは不要と判断した

* 再起動　# reboot
* Ramdisk の確認
```
# df -h
# ls -la /var/log
# systemctl status smbd
```

---
# ipatables の設定
* メインマシンから、ipt+++.sh を転送して、バッチファイルを実行　# sh ipt+++.sh
* 設定した iptables の恒久化　# apt install iptables-persistent（apt を upgrade しておかないと install できないため注意）
* 念の為に確認　# ufw status（inactive になっていることを確認。FW の設定は、全て iptables で行う）
* iptables に関する syslog を分離する。  
iptables のログ出力は syslog が担っているため、syslog の出力にフィルタを掛ける。  
/etc/rsyslog.d/ にフィルタを記述したファイルを置けば、フィルタが掛かるようになる。  

\# touch /etc/rsyslog.d/10-my-iptables.conf  
\# vi /etc/rsyslog.d/10-my-iptables.conf  
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
