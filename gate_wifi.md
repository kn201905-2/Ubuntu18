# パーティション
```
・/boot/efi　vfat　256M（RHEL の推奨値が 200M）
・/　ext4　残り全部
・swap　4G
```

---
# 起動時にネットワークの確認で、起動が遅くなることへの対処
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
