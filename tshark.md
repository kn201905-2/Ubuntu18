* 公式オプション  
https://www.wireshark.org/docs/man-pages/tshark.html

その他の参考  
https://qiita.com/hana_shin/items/0d997d9d9dd435727edf

* インストール
```
# apt install tshark
```

* パケットをキャプチャするインタフェイスを調べる -D
```
# tshark -D

調べたいインタフェイスの番号を確認する
```

* パケットのファイルへの書き出し -w、読み込み -r 
```
# tshark -i 1 -w test.pcap

# tshark -r test.pcap
# tshark -r test.pcap -x -Y 'frame.number==45'

-x : パケットの中身を 16進ダンプをする
```

* フィルタ -f  
https://wiki.wireshark.org/CaptureFilters#CaptureFilters-1  
https://www.wireshark.org/docs/man-pages/pcap-filter.html
```
# tshark -i 1 -f 'host 101.111.73.127'
# tshark -i 1 -f 'port 53'
# tshark -i 1 -f 'port 80 and tcp'
# tshark -i 1 -f 'host www.example.com and not port 80'
# tshark -i 1 -f 'net 192.168.1.0/24'
# tshark -i 1 -f 'src net 192.168.1.0/24'
# tshark -i 1 -f 'dst net 192.168.1.0/24'
# tshark -i 1 -f 'host www.example.com and not (port 80 or port 25)'
# tshark -i 1 -f 'host www.example.com and not port 80 and not port 25'（上と同じ意味となる）

他のフィルタ例
(tcp[0:2] > 1500 and tcp[0:2] < 1550) or (tcp[2:2] > 1500 and tcp[2:2] < 1550)
tcp[0:2] = tcpヘッダの 0バイト目から２バイト取り出す

tcp portrange 1501-1549
ip（arp や stp を取り除きたい場合など）
not broadcast and not multicast

Capture IPv6 "all nodes" (router and neighbor advertisement) traffic. Can be used to find rogue RAs: 
dst host ff02::1

dst port 135 and tcp port 135 and ip[2:2]==48
icmp[icmptype]==icmp-echo and ip[2:2]==92 and icmp[8:4]==0xAAAAAAAA

その他の書き方
# tshark -i eth0 -Y 'tcp.dstport==80' -n
# tshark -i eth0 -n -Y 'tcp.flags.syn==1 and tcp.flags.ack==0'
# tshark -i eth0 -Y 'icmp.type==8' -n
# tshark -i eth0 -Y  'icmp.type==0 or icmp.type==8'
```

* パケットの詳細を表示 -V
```
# tshark -i 1 -V -f 'port 53'
```

* 簡易な統計 -z
```
# tshark -r packet -z conv,ip （conv = conversation）
# tshark -r packet -z conv,tcp
# tshark -r packet -z http_req,tree
# tshark -r packet -z hosts

# tshark -z http,stat
# tshark -z http,tree

全プロトコルの統計情報を表示する方法（-q は出力表示の簡略化）
# tshark -qr test.cap -z io,phs
```

* キャプチャの自動停止
```
10秒間で停止
# tshark -a duration:10

ファイルサイズが10キロバイトに達すると停止
# tshark -w packet -a filesize:10

ファイルサイズが10キロバイトに達すると次のファイルに記録。ファイル数が5つになったら停止
# tshark -w packet -a filesize:10 -a files:5

パケット数で指定
# tshark -c 10

「-b」オプションを指定するとファイルのバッファサイズを指定できる
基本は「-a」と同じですが、ログローテーションのようにファイルを切り替えるタイミングを指定するだけで、停止はしないので注意。

ファイルサイズが10キロバイトに達したら別のファイルに切り替え。
ファイル数が5つになったら1番古いファイルを削除して新しいファイルで上書きする
# tshark -w packet -b filesize:10 -b files:5
```

* 基本的には「-f」で取得したいデータをフィルタして、「-a」でファイルに取得する時間を指定。「-z」で統計を確認。
問題がありそうなパケットに関しては「-x」や「-V」で詳細を確認。といった流れで大抵の需要は満たせると思います。

* pcap ファイルから、条件を指定して表示させる方法
```
# tshark -r test.cap 'tcp.port==80' -n
-n : すべての名前解決を無効にする。(デフォルト:有効）

# tshark -r test.cap 'tcp.dstport==80' -n
# tshark -r test.cap 'tcp.srcport==80' -n

# tshark -r test.cap 'tcp.flags.syn==1' and 'tcp.flags.ack==0' -n
# tshark -r test.cap 'tcp.flags.fin==1' -n

# tshark -r test.cap 'ip.src==192.168.0.100' -n
# tshark -r test.cap 'ip.dst==192.168.0.100' -n

# tshark -r ping.cap 'icmp.type==8'  # echo request
# tshark -r ping.cap 'icmp.type==0'  # echo reply

# tshark -r test.cap -Y 'frame.number>=10 and frame.number<=20'
```

* パケットの取得時刻
```
パケットの取得時刻を表示する場合（-ta）
# tshark -r dns.cap -ta

パケットの取得時刻の差分（１つ１つが「何秒」ずれて取得されたかが表示される）
# tshark -r dns.cap -td

```
