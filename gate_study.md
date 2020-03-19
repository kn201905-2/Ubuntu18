# ipv4 のフォワーディングの設定  

* 確認　# cat /proc/sys/net/ipv4/ip_forward（0 が表示されたらフォワーディングがされていない、ということ）  
* 設定　# echo 1 > /proc/sys/net/ipv4/ip_forward  
* 恒久的な設定　# vim /etc/sysctl.conf  
「net.ipv4.ip_forward=1」という１行を有効化する。

---
# ip アドレスの設定  
* Ubuntu 18 からは、netplan で設定するように変更された　# vim /etc/netplan/+++++.yaml  
以下は YAML ファイルとなるため、インデントは半角スペースでそろえること。  

> network:  
>　version: 2  
>　renderer: networkd  
>　ethernets:  
>　　enp1s0:  
>　　　dhcp4: no（false でもよい)  
>　　　dhcp6: no（false でもよい)  
>　　　addresses: [192.168.100.2/24]  
>　　　gateway4: 192.168.100.1  
>　　　nameservers:  
>　　　　addresses: [8.8.8.8]  
>　　enp3s0:  
>　　　dhcp4: no  
>　　　dhcp6: no  
>　　　addresses: [192.168.0.100/24]

* 編集した YAML ファイルの読み込み　# netplan apply  

---
# パッケージの更新

\# apt update（ソフトの更新確認）  
\# apt upgrade（ソフトの更新）  

---
# iptables の設定  
* メインマシンから、ipt+++.sh を転送　# sh ipt+++.sh  
* 設定した iptables の恒久化　# apt install iptables-persistent（apt を upgrade しておかないと install できないため注意）  

* この時点で、他の 192.168.0.0/24 のマシンからもインターネットへの通信が可能となる  
* 念の為に確認　# ufw status（inactive になっていることを確認。FW の設定は、全て iptables で行う）  

* iptables に関する syslog を分離する。  
iptables のログ出力は syslog が担っているため、syslog の出力にフィルタを掛ける。  
/etc/rsyslog.d/ にフィルタを記述したファイルを置けば、フィルタが掛かるようになる。 

\# touch /etc/rsyslog.d/10-my-iptables.conf  
\# vi /etc/rsyslog.d/10-my-iptables.conf  

> :msg,contains,"Drp_" -/var/log/iptables.log  
> & ~  

ファイル指定の前の「-」は、ログ出力時のディスクとの同期の抑制を示す。（パフォーマンス向上が狙える）  
「& ~」は、対象としているログを廃棄する、ということを示す。  

* rsyslog を再起動　# systemctl restart rsyslog  

参考URL：  
http://www.geocities.jp/yasasikukaitou/rsyslog-filter.html  
https://vogel.at.webry.info/201311/article_4.html  

---
# タイムゾーンの設定  
* 次のコマンドで、即時に恒久的に変更される　# ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime  
* JST になっていることを確認　# date  

---
# vim の設定  
* ~/.bashrc に以下を記述  
> alias vim='vim -c start'

---
# 電源ボタンの設定  
* パワーオフ or サスペンド　# vim /etc/systemd/logind.conf  
> HandlePowerKey=poweroff or suspend  

---
# Ramdisk の設定  
* fstab に、以下の行を追加　# vim /etc/fstab  

> tmpfs /tmp tmpfs defaults,size=512m,noatime,mode=1777 0 0  
> tmpfs /var/tmp tmpfs defaults,size=512m,noatime,mode=1777 0 0  
> tmpfs /var/log tmpfs defaults,size=512m,noatime,mode=0755 0 0  

noatime： アクセスした際に、アクセス時のタイムスタンプの変更をしない  
mode： アクセス権。先頭の数字はスティッキービット  
５番目の数字： バックアップを作る時を決定するために dump ユーティリティによって使われる。通常は０でよい  
６番目の数字： ファイルシステムをチェックする順番を決めるために fsck によって使われる。RamDiskは０でよい  

* 起動時にRAMディスクにディレクトリを作成するように設定する。一部のサービスは、サービス起動時にワーキングディレクトリの存在が必要となるため　
\# touch /etc/rc.local  
\# vim /etc/rc.local  

> #!/bin/bash  
>  
> mkdir -p /var/log/apt  
> mkdir -p /var/log/fsck  
>  
> mkdir -p /var/log/samba  
> chown root.adm /var/log/samba  
>  
> touch /var/log/lastlog  
> touch /var/log/wtmp  
> touch /var/log/btmp  
> chown root.utmp /var/log/lastlog  
> chown root.utmp /var/log/wtmp  
> chown root.utmp /var/log/btmp  

* \# chmod +x /etc/rc.local  
補足： rc.local は、「/lib/systemd/system/rc-local.service」により、systemd が起動を実行する。  

* reboot して、「df -h」、「ls /var/log」によって、ramdisk が機能していることを確認する  

---
# ログローテーション  
* /etc/logrotate.d に設定ファイルを作成することにより、ローテートを行う  
2019-10-20　ローテーションの設定ファイルは、「/etc/logrotate.d/rsyslog」を参考にして作成した。設定ファイルの書き方に関するネットでの情報は、Linux のディストリビューションや、バージョンごとの差異があり不明な点が多かったため、現時点では、現OSが走っている設定ファイルを真似することにした。  

\# touch /etc/logrotate.d/iptables  
\# vim /etc/logrotate.d/iptables  

> /var/log/iptables.log  
> {  
> 　missingok  
> 　notifempty  
> 　daily  
> 　rotate 15  
> 　create 766  
> 　dateext  
> 　dateformat _%Y-%m-%d  
> 　postrotate  
>　　systemctl kill -s HUP rsyslog.service  
> 　endscript  
> 　su syslog adm  
> }  

* 設定ファイルの動作確認　# logrotate -d /etc/logrotate.d/iptables  
「-d」は dry run を表す。実際には実行せずに、動作確認だけを行う。  

---
# USBメモリの設定  
* ラベルの確認　# lsblk -f  
* 起動時にマウントできるようにする　# vim /etc/fstab  

> LABEL="gate-study-USB" /home/shared-USB ext4 defaults,noatime 0 0

---
# Samba の設定  
* 別ファイルを参考にして設定する  
* gate-study は、Ramdisk を利用しているため、/var/log/samba 等が生成された後に smbd が起動するように設定する。  
\# vim /lib/systemd/system/smbd.service  
[Unit] の After を以下のように変更する。  

After=network.target rc-local.service  
（nmbd と winbind は利用しないため、削除してよい。winbind は security=domain などの場合に利用されるデーモン）  
