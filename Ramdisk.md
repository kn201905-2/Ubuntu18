# Ramdisk

* マウント先の作成　# mkdir /home/shared/ramdisk
* fstab の書き換え

\# vim /etc/fstab
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
（参考）現在のところ、１回の使用時間が短い場合は size=256m、使用時間が長い場合は size=512m としている。

> noatime： アクセスした際に、アクセス時のタイムスタンプの変更をしない  
> mode： アクセス権。先頭の数字はスティッキービット  
> ５番目の数字： バックアップを作る時を決定するために dump ユーティリティによって使われる。通常は０でよい  
> ６番目の数字： ファイルシステムをチェックする順番を決めるために fsck によって使われる。RamDiskは０でよい  


* 起動時にRAMディスクにディレクトリを作成するように設定する。一部のサービスは、サービス起動時にワーキングディレクトリの存在が必要となる。

\# vim /etc/rc.local  
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
* smbd より先に rc.local が実行されるように設定する

\# vim /lib/systemd/system/smbd.service
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
