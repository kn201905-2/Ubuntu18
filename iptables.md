# iptables に関する syslog を分離する  
* iptables のログ出力は syslog が担っているため、syslog の出力にフィルタを掛ける。  
/etc/rsyslog.d/ にフィルタを記述したファイルを置けば、フィルタが掛かるようになる。

\# vim /etc/rsyslog.d/10-my-iptables.conf
```
:msg,contains,"Drp_" -/var/log/iptables.log
& ~
```
ファイル指定の前の「-」は、ログ出力時のディスクとの同期の抑制を示す。（パフォーマンス向上が狙える）  
「& ~」は、対象としているログを廃棄する、ということを示す。  
```
& は、１つのセレクタに複数のアクションを指定するときに利用する。
FILTER ACTION
& ACTION
& ACTION

~ は、FILTER されたパケットを破棄する。
```

* rsyslog を再起動　# systemctl restart rsyslog  

参考URL：  
https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/7/html/system_administrators_guide/s1-basic_configuration_of_rsyslog  
https://vogel.at.webry.info/201311/article_4.html  

---
# firewall 関連  
* ufw の停止　# ufw disable  
* iptables の設定保存　# apt install iptables-persistent

---
# iptables の設定恒久化
```
# apt install -y iptables-persistent
# netfilter-persistant save
# systemctl status netfilter-persistent
```
