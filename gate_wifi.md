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

* ~~A start job is running for wait for network to be configured. というメッセージが表示されて２分ほど待たされる。~~  
　~~systemd-networkd が立ち上があるのを待つために、起動が止まるらしい。~~  
　~~以下の設定で、この待ち時間をキャンセルできる。~~
```
# systemctl disable systemd-networkd-wait-online.service
# systemctl mask systemd-networkd-wait-online.service
```

---
# コマンドプロンプトの変更

以下を、~/.bashrc に追記しておけばよい。
```
・緑色に変更
PS1="\[\e[1;32m\][\H: \w]\[\e[0m\]\n\\$ "

緑色： 32、黄色： 33、青色： 34、パープル： 35、シアン： 36、白色: 37

参考 URL：
https://www.atmarkit.co.jp/flinux/rensai/linuxtips/002cngprmpt.html
https://shio-ax.hatenablog.com/entry/2019/05/27/174018
```

---
# タイムゾーンの設定

* タイムゾーンの確認
```
# ls -la /etc/localtime
# date
```

* タイムゾーンの設定
```
# ln -s /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

「/etc/localtime」が存在してエラーになる場合は、/etc/localtime を rm すればよい。
```

* RTC の設定情報の確認
```
# timedatectl status
```

* RTC をローカルタイムにする（BIOS で自動起動設定をするため）
```
# timedatectl set-local-rtc true
```
ただし、上記設定を行うと、timedatectl status で警告が表示されるようになる

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
      dhcp4: no
      dhcp6: no
      addresses:
        - 192.168.100.3/24
      nameservers:
        addresses:
          - 8.8.8.8
      routes:
        - to: 0.0.0.0/0
          via: 192.168.100.1
        - to: 192.168.0.0/24
          via: 192.168.100.2
    enp3s0:
      dhcp4: no
      dhcp6: no
      addresses:
        - 192.168.1.100/24
  bridges:
    br0:
      dhcp4: no
      dhcp6: no
      interfaces: [ enp1s0, enp3s0 ]
      parameters:
        stp: no
```
* yaml ファイルを編集した後は、以下のコマンドで network を更新する。再起動の必要はない。  
\# netplan apply  

* 【注意１】  
　上記の設定でブリッジが張られるため、一度通信が途絶してしまう。  
　ebtables で、v4 パケットをブリッジから drop させることが必要となる。

* 【注意２】  
　enp3s0 に、ネットワーク機器を接続していない場合、ip a としても、enp3s0 には ipアドレスが表示されない。さらに、起動時にネットワークの確認で数分待たされることになるため、上述した「systemd-networkd-wait-online.service」のマスクが必要となる。  
　enp3s0 に、ネットワーク機器を接続すると、自動的にインターフェイスが立ち上がり、inet 192.168.0.100/24 と表示される。  

---
# ebtables の設定
参考：  
https://netplan.io/faq#use-pre-up-post-up-etc-hook-scripts  
https://www7390uo.sakura.ne.jp/wordpress/archives/tag/netplan  

* ネットワークが立ち上がったときに、ebtables のスクリプトを走らせる

\# vim /etc/networkd-dispatcher/routable.d/10-ifup-ebt-broute
```
#!/bin/sh

ebtables -t broute -F
ebtables -t broute -P BROUTING DROP
ebtables -t broute -A BROUTING -p IPv6 -j ACCEPT
```
必要ないと思われるが、一応念のために、chmod +x 10-ifup-ebt-broute としておいた。

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

以下の URL で、ダウンロードするファイルを決める  
https://github.com/cdr/code-server/releases
```
# wget https://github.com/cdr/code-server/releases/download/1.1156-vsc1.33.1/code-server1.1156-vsc1.33.1-linux-x64.tar.gz  
# tar xvzf code-server1.1156-vsc1.33.1-linux-x64.tar.gz（z は gzip の指定）  
```
以上で code-server は利用可能となる。（tar で展開すると、展開したフォルダの中に「code-server」という実行ファイルがある。それを実行すれば、code-server が立ち上がる。）
今回は、以下のような起動用バッチファイルを作成した。
```
#!/bin/bash
code-server1.1156-vsc1.33.1-linux-x64/code-server -p 3010 -N

・-p でリスンするポート番号を指定
・-N でパスワード無しでログイン可能となる
・その他のオプションに関しては、code-server --help で調べると良い
```

* settings.json を、以下の位置にコピー
```
~/.local/share/code-server/User
```

---
# ulogd2 のインストール
```
# apt install ulogd2
```
* ulogd2 の設定  
参考例:  
https://www7390uo.sakura.ne.jp/wordpress/archives/466

\# vim /etc/ulogd.conf
```
[global]
# logfile for status messages
logfile="/var/log/ulog.log"

# loglevel: debug(1), info(3), notice(5), error(7) or fatal(8) (default 5)
loglevel=3

######################################################################
# PLUGIN OPTIONS
######################################################################

plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_inppkt_NFLOG.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_IFINDEX.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_IP2STR.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_PRINTPKT.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_LOGEMU.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_raw2packet_BASE.so"

# IFO
stack=log1:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,print1:PRINTPKT,emu1:LOGEMU

# chk
stack=log2:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,print1:PRINTPKT,emu2:LOGEMU

# Mk
stack=log3:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,print1:PRINTPKT,emu3:LOGEMU

# Kc
stack=log4:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,print1:PRINTPKT,emu4:LOGEMU

# mac
stack=log5:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,print1:PRINTPKT,emu5:LOGEMU


# ------------------------

[log1]
# Group O is used by the kernel to log connection tracking invalid message
group=0  # 0 は特別な意味があるらしい

[log2]
group=2

[log3]
group=3

[log4]
group=4

[log5]
group=5

# ------------------------

[emu1]
file="/var/log/Ip4_IFO.log"
sync=1

[emu2]
file="/var/log/Ip4_chk.log"
sync=1

[emu3]
file="/var/log/Ip4_Mk.log"
sync=1

[emu4]
file="/var/log/Ip4_Kc.log"
sync=1

[emu5]
file="/var/log/Ip4_mac.log"
sync=1
```

* ulogd.conf の内容が問題ないかを確認する
```
# systemctl restart ulogd
# systemctl status ulogd

何か問題があれば、
# journalctl -xe
などで調べる。現時点では、/run/ulog/ulogd.pid が読み取れない、というエラーが表示されてるが、、、

https://kb.gtkc.net/iptables-with-ulogd-quick-howto/
```

* システム再起動後に、iptables で NFLOG チェーンが使えるようになるようである。

* （参考）iptables への書き方
```
iptables -N U_LOG
iptables -A U_LOG -j NFLOG --nflog-group 0 --nflog-prefix 'iptables:' --nflog-threshold 5

--nflog-group: [log1] で指定したグループの番号。log1:NFLOG となっている。
--nflog-threshold: ulogdが処理を開始するまでにキューへ溜め込むパケット数を2から50の間で指定する。
```

---
# iptables の設定

* iptables の設定を永続化させるために以下をインストールする
```
# apt install -y iptables-persistent
```
* iptables を設定し直す毎に、以下のコマンドを実行
```
# netfilter-persistent save
```

* iptables の設定

---
# DHCP サーバー
* 今回は、isc-dhcp-server を利用した。（Internet Systems Consortium）  
\# apt install -y isc-dhcp-server  

* 設定ファイル isc-dhcp-server の設定  
\# vim /etc/default/isc-dhcp-server
```
・実際に必要になるのは、以下の３行のみ（他は全部コメントアウト）

DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
DHCPDv4_PID=/var/run/dhcpd.pid
INTERFACESv4="enp3s0"

（注）２行目のファイルの指定位置と、Systemd の起動設定ファイルに齟齬があるようなので、
　　　/lib/systemd/system/isc-dhcp-server.service の修正が必要となるようである。（バグ？？）
```

* 設定ファイル /etc/dhcp/dhcpd.conf の設定  
（参考URL）http://park12.wakwak.com/~eslab/pcmemo/linux/dhcpd/dhcpd3.html  
\# vim /etc/dhcp/dhcpd.conf
```
・実際に必要になるのは、以下の９行のみ（実際には８行かも？）

# option domain-name はコメントアウトしておけばよい。DNSサフィックスが空欄になるだけ。
option domain-name-servers 8.8.8.8;

# 以下の数字は「秒」
default-lease-time 86400;  # １日
max-lease-time 259200;  # ３日

# DHCP 処理を行ったクライアント情報（IP アドレス・ホスト名等）の、
# （ DNS サーバーが管理する）ゾーンデータ更新スタイルを指定する。
# http://park12.wakwak.com/~eslab/pcmemo/linux/dhcpd/dhcpd3.html
ddns-update-style none;

# この行は、特に必須ではないと思う、、、
authoritative;

# この行で、アドレスの範囲と、next hop を指定する。
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.200 192.168.1.254;
  option routers 192.168.1.100;
}

上記のサブネットが、「INTERFACESv4」で指定したインターフェイスのサブネットと合致しない場合、
dhcpサーバの起動に失敗するので注意。
何が原因で起動に失敗したのかなどのメッセージがないため、起動に失敗した場合は、
subnet の項目を点検すること。
```
* Systemd の起動設定ファイルの調整  
systemctl start isc-dhcp-server のときは問題ないが、電源投入後の自動起動時に問題が発生する。以下のようなエラーが出力される。
```
Can't create PID file /run/dhcp-server/dhcpd.pid
```
おそらくバグであろうと思われる。とりあえず以下のように修正した。  
\# vim /lib/systemd/system/isc-dhcp-server.service
```
下から４行目を以下のように変更した。
-pf /run/dhcp-server/dhcpd.pid
　↓
-pf /run/dhcpd.pid
```
* dhcpサーバの自動起動の設定  
\# systemctl enable isc-dhcp-server  
* dhcpサーバの起動  
\# systemctl start isc-dhcp-server

---
# AM 1:00 に電源オフ
\# crontab -e  
初めて crontab -e するときは、使用するエディタを尋ねてくる。vim.basic を選択。
```
以下の１行を追記。不要なコメントはすべて消しておいてよい。
00 01 * * * /home/shutdown.sh
```
* 確認  
\# crontab -l  
* shutdown.sh の作成  
\# vim /home/shutdown.sh
```
#!/bin/sh
/sbin/shutdown -h now  # おそらく、-h は必要ない
```
* shutdown.sh が実行可能なようにする  
\# chmod +x shutdown.sh

---
# AM 6:30 に電源オン
* BIOS で設定を行う
