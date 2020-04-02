# ulogd2

* インストール
```
# apt install ulogd2
```

* /etc/ulogd.conf を設定する

参考例  
https://www7390uo.sakura.ne.jp/wordpress/archives/466
```
[global]
logfile="/var/log/ulogd.log"
loglevel=3

plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_inppkt_NFLOG.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_IFINDEX.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_IP2STR.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_PRINTPKT.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_LOGEMU.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_SYSLOG.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_raw2packet_BASE.so"

stack=log1:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,print1:PRINTPKT,emu1:LOGEMU

[log1]
group=0

[emu1]
file="/var/log/ulog_syslogemu.log"
sync=1
```

* ulogd.conf の内容が問題ないかを確認する
```
# systemctl restart ulogd
# systemctl status ulogd

何か問題があれば、
# journalctl -xe
などで調べる。現時点では、/run/ulog/ulogd.pid が読み取れない、というエラーが表示されてるが、、、
```

* システムを再起動
```
# reboot
```
再起動後に、iptables で、NFLOG チェーンが使えるようになるようである。

---
# iptables への書き方

```
iptables -N U_LOG
iptables -A U_LOG -j NFLOG --nflog-group 0 --nflog-prefix 'iptables:' --nflog-threshold 5

--nflog-group: [log1] で指定したグループの番号。log1:NFLOG となっている。
--nflog-threshold: ulogdが処理を開始するまでにキューへ溜め込むパケット数を2から50の間で指定する。
```
