* 公式オプション  
https://www.wireshark.org/docs/man-pages/tshark.html

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
```

* パケットの詳細を表示 -V
```
# tshark -i 1 -V -f 'port 53'
```

* 簡易な統計 -z
```
# tshark -r packet -z conv,ip
# tshark -r packet -z http_req,tree
# tshark -r packet -z hosts

# tshark -z http,stat
# tshark -z http,tree
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

ファイルサイズが10キロバイトに達したら別のファイルに切り替え。ファイル数が5つになったら1番古いファイルを削除して新しいファイルで上書きする
# tshark -w packet -b filesize:10 -b files:5
```


