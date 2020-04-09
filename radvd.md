# RA の設定
* IPv6 のパケットフォワーディングの設定を行う。  
/etc/sysctl.conf の「net.ipv6.conf.all.forwarding=1」を有効にする

* radvd（Router Advertisement daemon）をインストールする。
```
# apt install radvd
インストール後に、自動起動をするが、radvd.conf がないため、起動に失敗する、、、
```

* radvd.conf を準備する

参考：  
https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/7/html/networking_guide/sec-configuring_the_radvd_daemon_for_ipv6_routers  
https://linux.die.net/man/5/radvd.conf

\# vim /etc/radvd.conf
```
# 以下の設定は、RedHat が公開している例に沿っている

interface enp3s0
{
        AdvSendAdvert on;
        MinRtrAdvInterval 200;
        MaxRtrAdvInterval 600;
        prefix 240b:x:x:x::/64
        {
                AdvOnLink on;
                AdvAutonomous on;
                AdvRouterAddr off;
        };
};
```

* radvd の起動
```
# systemctl enable radvd
# systemctl start radvd
# systemctl status radvd
```
