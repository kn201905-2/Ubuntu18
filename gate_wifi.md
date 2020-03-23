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

