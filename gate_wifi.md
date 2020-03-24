# パーティション
```
・/boot/efi　vfat　256M（RHEL の推奨値が 200M）
・/　ext4　残り全部
・swap　4G
```

---
# ~~起動時にネットワークの確認で、起動が遅くなることへの対処~~
* ネットワークが立ち上がってからでないと、起動に失敗するサービス（dhcpなど）があったため、ネットワークの立ち上がりを待つことに変更した。
```
# systemctl unmask systemd-networkd-wait-online.service
# systemctl enable systemd-networkd-wait-online.service
```

* A start job is running for wait for network to be configured. というメッセージが表示されて２分ほど待たされる。  
　systemd-networkd が立ち上があるのを待つために、起動が止まるらしい。  
　以下の設定で、この待ち時間をキャンセルできる。
```
# systemctl disable systemd-networkd-wait-online.service
# systemctl mask systemd-networkd-wait-online.service
```

---
# ip アドレスの設定
* Ubuntu 18 からは、netplan で設定するように変更された。  
* インストール時に、enp1s0 の設定を行っておくと、netplan用の yaml ファイルが生成されている。  
* yaml ファイルがなければ、自分で作成すれば良い。

\# vim /etc/netplan/01-netcfg.yaml
```
network:
　version: 2
　renderer: networkd
　ethernets:
　　enp1s0:
　　　dhcp4: no（false でもよい)
　　　dhcp6: no（false でもよい)
　　　addresses: [192.168.100.2/24]
　　　gateway4: 192.168.100.1
　　　nameservers:
　　　　addresses: [8.8.8.8]
　　enp3s0:
　　　dhcp4: no
　　　dhcp6: no
　　　addresses: [192.168.0.100/24]
```
* yaml ファイルを編集した後は、以下のコマンドで network を更新する。再起動の必要はない。  
\# netplan apply  

* 【注意】  
　enp3s0 に、ネットワーク機器を接続していない場合、ip a としても、enp3s0 には ipアドレスが表示されない。さらに、起動時にネットワークの確認で数分待たされることになるため、上述した「systemd-networkd-wait-online.service」のマスクが必要となる。  
　enp3s0 に、ネットワーク機器を接続すると、自動的にインターフェイスが立ち上がり、inet 192.168.0.100/24 と表示される。  

---
# ipv4 のフォワーディングの設定
* 確認　\# cat /proc/sys/net/ipv4/ip_forward（0 が表示されたらフォワーディングがされていない、ということ）  
* 設定　\# echo 1 > /proc/sys/net/ipv4/ip_forward
* 恒久的な設定　\# vim /etc/sysctl.conf  
コメントアウトされている「net.ipv4.ip_forward=1」という１行を有効化する。

---
# パッケージの更新
```
# apt update（ソフトの更新確認）
# apt upgrade（ソフトの更新）
```

---
# 電源ボタンの設定

\# vim /etc/systemd/logind.conf
```
HandlePowerKey=poweroff or suspend
```

---
# Ramdisk の設定
* gate-wifi は毎日リセットされるため、memory の圧迫をさけるために ramdisk の容量は小さくしている。

\# vim /etc/fstab  
以下を追記
```
# /tmp, /var/tmp -> Ramdisk
tmpfs /tmp tmpfs defaults,size=256m,noatime,mode=1777 0 0
tmpfs /var/tmp tmpfs defaults,size=256m,noatime,mode=1777 0 0

# /var/log -> Ramdisk
tmpfs /var/log tmpfs defaults,size=256m,noatime,mode=0755 0 0
```
noatime： アクセスした際に、アクセス時のタイムスタンプの変更をしない  
mode： アクセス権。先頭の数字はスティッキービット  
５番目の数字： バックアップを作る時を決定するために dump ユーティリティによって使われる。通常は０でよい  
６番目の数字： ファイルシステムをチェックする順番を決めるために fsck によって使われる。RamDiskは０でよい  

* 起動時にRAMディスクにディレクトリを作成するように設定する。一部のサービスは、サービス起動時にワーキングディレクトリの存在が必要となる。  
\# vim /etc/rc.local  
以下を記入する
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
* 上記の rc.local に、実行権限を与える　\# chmod +x /etc/rc.local  
* Ramdisk に移行するフォルダのファイルを空にしておく
```
# find /tmp -type f
# find /tmp -type f -exec cp -f /dev/null {} \;

# find /var/tmp -type f
# find /var/tmp -type f -exec cp -f /dev/null {} \;

# find /var/log -type f
# find /var/log -type f -exec cp -f /dev/null {} \;
```
* 再起動　\# reboot  
* Ramdisk の確認
```
# df -h
# ls -la /var/log
```

---
# code-server のインストール

---
# iptables の設定

* iptables の設定後、情報を永続化させるために以下をインストールする
```
# apt install -y iptables-persistent
```
* iptables を設定し直す毎に、以下のコマンドを実行
```
# netfilter-persistent save
```

---
# ipアドレス、route の設定

\# vim /etc/netplan/01-netcfg.yaml
```
# This file describes the network interfaces available on your system
# For more information, see netplan(5).
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: no
      dhcp6: no
      addresses: [ 192.168.100.3/24 ]
      gateway4: 192.168.100.1
      nameservers:
        addresses: [ 8.8.8.8 ]
      routes:
        - to: 192.168.0.0/24
          via: 192.168.100.2
    enp3s0:
      dhcp4: no
      dhcp6: no
      addresses: [ 192.168.1.100/24 ]
```
