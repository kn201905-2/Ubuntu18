# samba に接続したい場合
* \# apt install -y cifs-utils  
* \# mount -t cifs -o username=testuser,password=testpass,vers=1.0 //192.168.0.80/Public /home/shared/cifs

* 115 エラーが出る場合は、firewall の設定を確認すること
