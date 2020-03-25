
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

