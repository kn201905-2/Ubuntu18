
# samba の設定  
* インストールの確認　# samba -V  
* インストールする場合　# apt install -y samba  
* samba の設定ファイルを設定

\# vim /etc/samba/smb.conf  
以下、変更箇所  
```
[global]  
server string = SAMBA SERVER Version %v  
wins support = no
wins proxy = no
dns proxy = no  # 不必要かも？？

dos charset = CP932
load printers = no
; disable spoolss = yes  # 不要そうであるため、今は外している
; syslog = 0  # syslog は deprecated らしい。log は、全て var/log/samba に書き出すものとする  
disable netbios = yes  # netbios は使用しない

security = user
passdb backend = tdbsam  # これは、デフォルトで設定されていた
usershare allow guests = no  # ユーザー定義共有という新しい機能らしい
browseable = yes  # no にすると、エクスプローラ上での表示がされなくなった

# ネットワークの効率化を狙う、Windows固有の機能
# 複数のプロセスが同じファイルをロックでき、なおかつクライアントがデータをキャッシュできる機能
# しかし、この機能は、ファイルの破壊をもたらす可能性があるため、OFF にしておく
oplocks = No
level2 oplocks = No
blocking locks = No


# 以下のセクションを追加  
[NUC-SMB]  # []の中は好きな文字列でよい。Windowsでアクセスしたときに表示されるフォルダ名となる
comment = NUC-SMB
path = /home/shared  
browseable = yes  
read only = no  
valid users = user-k  
guest ok = no  
writable = yes  
create mode = 0600  
directory mode = 0700  

# 以下のセクションをすべてコメントアウト（プリンタサーバ機能は利用しない）  
[printers]  
[print$]  
```

* 設定ファイルを設定し終えたら、ファイルの構文チェック  
\# testparm  

* 共有ディレクトリの作成  
 \# mkdir /home/shared

* Samba のユーザー登録。Linux に存在するユーザーで登録する必要がある。  
Linux と Samba では暗号化方式が異なるため、同じパスワードでも両方に登録が必要になる。  
\# pdbedit -a user-k

* ディレクトリにアクセスできるように権限を変更しておく  
\# chown user-k:user-k /home/shared/  
\# chmod 774 /home/shared/
 
* サービスの自動起動の設定
```
# systemctl enable smbd
# systemctl disable nmbd  # netbios を使用しないため、nmb は必要ない
# systemctl mask nmbd
```
  
* TCP 445 を利用するため、FW の設定には注意すること


