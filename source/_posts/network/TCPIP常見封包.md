---
title: TCPIP常見封包
comments: false
toc: false
date: 2020-02-23 14:58:15
categories: network
tags:
---

測試環境 CentOS7 x86_64

## TCP 3 way handshack
所謂的 TCP 三方交握(TCP 3-way handshake) 就是 Server 與 Client 間透過 3 次的溝通,才會進行資料的傳輸。這步驟如下:
1. Client 用戶端向 Server 伺服器發送一個 "SYN" 訊息,跟 Server 伺服器請求連線.
2. 如果 Server 伺服器準備好與 Client 用戶端連線,就會回傳一個 "SYN-ACK" 的訊息.
3. 如果 Client 用戶端接受到剛剛的 "SYN-ACL" 而且也準備好,就會向 Server 伺服器發送一個 "ACK"訊號,讓 Server 伺服器知道現在要開始傳送資料了.

``` sh
15:44:32.253563 IP 172.16.0.14.39137 > 172.16.0.2.80: Flags [S], seq 3326256915, win 14600, options [mss 1460,sackOK,TS val 541699350 ecr 0,nop,wscale 7], length 0
15:44:32.253768 IP 172.16.0.2.80 > 172.16.0.14.39137: Flags [S.], seq 1389324781, ack 3326256916, win 28960, options [mss 1460,sackOK,TS val 98054855 ecr 541699350,nop,wscale 7], length 0
15:44:32.253794 IP 172.16.0.14.39137 > 172.16.0.2.80: Flags [.], ack 1, win 115, options [nop,nop,TS val 541699351 ecr 98054855], length 0
```

* S(SYN) : Synchronize sequence numbers.
* S. : SYN-ACK .
* . : no flags.

## 資料的傳送
Client 用戶端每送一筆資料 Server 伺服器收到資料都會回傳給 Client 用戶端一個 ACK 訊號.但這樣未免太沒效率,所以通常會一次傳送多筆資料過去,這就是 TCP Sliding Window – http://benjr.tw/3020

TCP Sliding Window 可以區分為 Send / Receive Window 兩種.

1. TCP 傳送端(Client 用戶端)的 Sliding Window 被稱為 Send Window,有收到接收端的 ACK 訊號才會傳送下一筆資料.並移動 Send Window.
2. TCP 接收端(Server 伺服器)的 Sliding Window 被稱為 Receive Window,只有最前端的資料收到才會送出 ACK 訊號,並移動 Receive Window.

Send / Receive window size 的值通常非固定但雙方會協議出一個數值.

``` sh
15:44:32.276142 IP 172.16.0.2.80 > 172.16.0.14.39137: Flags [P.], seq 2897:5206, ack 281, win 235, options [nop,nop,TS val 98054868 ecr 541699351], length 2309
15:44:32.276170 IP 172.16.0.14.39137 > 172.16.0.2.80: Flags [.], ack 5206, win 160, options [nop,nop,TS val 541699373 ecr 98054868], length 0
```

> P(PSH) : Asks to push the buffered data to the receiving application.

## 結束資料傳送
雷同 三方交握(TCP 3-way handshake) .

``` sh
15:44:37.416686 IP 172.16.0.2.80 > 172.16.0.14.39138: Flags [F.], seq 10060, ack 966, win 252, options [nop,nop,TS val 98060018 ecr 541699549], length 0
15:44:37.418618 IP 172.16.0.14.39138 > 172.16.0.2.80: Flags [F.], seq 966, ack 10061, win 205, options [nop,nop,TS val 541704516 ecr 98060018], length 0
15:44:37.418912 IP 172.16.0.2.80 > 172.16.0.14.39138: Flags [.], ack 967, win 252, options [nop,nop,TS val 98060020 ecr 541704516], length 0
```

> F(FIN) : Last packet from sender.

## ARP
ARP (Address Resolution Protocol) 地址解析協議,也就是你主機 NIC 使用的 MAC (固定不變) 和實際使用的 IP (會依據環境做改變) 相對應的列表.

``` sh
15:44:32.253383 ARP, Request who-has 172.16.0.2 tell 172.16.0.14, length 28
15:44:32.253553 ARP, Reply 172.16.0.2 is-at 00:21:5e:6d:77:ee, length 46
```

## DHCP
DHCP Client 向 DHCP Server (使用 UDP) 要求 IP 時主要的四個動作( DHCPDISCOVER , DHCPOFFER , DHCPREQUEST , DHCPACK ),如果能看到這四個動作,這就代表 Client 已經成功獲得 IP ,下面 Client 是正確透過 DHCP 獲取 IP 的過程.

``` sh
15:42:31.013004 IP 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from 00:02:b3:af:b2:82, length 314
15:42:32.013978 IP 192.10.0.2.67 > 192.10.0.20.68: BOOTP/DHCP, Reply, length 300
15:42:32.014176 IP 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from 00:02:b3:af:b2:82, length 320
15:42:32.039599 IP 192.10.0.2.67 > 192.10.0.20.68: BOOTP/DHCP, Reply, length 300
```

雖然看到的封包都是 Request & Reply 而已.這時候可以透過 tcpdump -vv 來看詳細內容的確是 DISCOVER , OFFER , REQUEST , ACK .

``` sh
DHCP-Message Option 53, length 1: Discover
DHCP-Message Option 53, length 1: Offer
DHCP-Message Option 53, length 1: Request
DHCP-Message Option 53, length 1: ACK
```