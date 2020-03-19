# iptables に関する syslog を分離する  
* iptables のログ出力は syslog が担っているため、syslog の出力にフィルタを掛ける。  
/etc/rsyslog.d/ にフィルタを記述したファイルを置けば、フィルタが掛かるようになる。  
\# touch /etc/rsyslog.d/10-my-iptables.conf  
\# vi /etc/rsyslog.d/iptables.conf  
> :msg,contains,"Drp_" -/var/log/iptables.log  
> & ~  

ファイル指定の前の「-」は、ログ出力時のディスクとの同期の抑制を示す。（パフォーマンス向上が狙える）  
「& ~」は、対象としているログを廃棄する、ということを示す。  
・rsyslog の再起動法　# systemctl restart rsyslog  

参考URL：  
http://www.geocities.jp/yasasikukaitou/rsyslog-filter.html  
https://vogel.at.webry.info/201311/article_4.html  

---
# firewall 関連  
* ufw の停止　# ufw disable  
* iptables の設定保存　# apt install iptables-persistent
