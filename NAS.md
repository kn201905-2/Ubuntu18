
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
（別ページを参照）

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
