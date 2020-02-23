---
title: cksum(checksum)incorrect
comments: false
toc: false
date: 2020-02-23 16:27:30
categories: network
tags:
---

測試環境為 CentOS x86_64 (虛擬機)

``` sh
04:12:03.916512 IP (tos 0x12,ECT(0), ttl 64, id 55822, offset 0, flags [DF], proto TCP (6), length 664) localhost.localdomain.ssh > 192.168.95.1.55154: Flags [P.], cksum 0x4277 (incorrect -> 0x5557), seq 7810800:7811412, ack 5365, win 292, options [nop,nop,TS val 2588788 ecr 595141790], length 612
```

> Flags – DF (Don’t fragment).

上網查了一下,跟 網卡的 TOE(TCP/IP offload engine)或是 TCO(TCP/IP offload) 有關,有這一類功能的網卡其 checksum 是透過網卡本身來運作(降低使用系統 CPU 資源),透過 tcpdump 抓取的封包還沒到 NIC 做實際的 checksum ,所以可以看到 cksum 都是臨時的.
>什麼是 TOE,要先說到 SAN.
>
>SAN(Storage area Network),他是一種資料儲存裝置(Raw Device),最早的 SAN 傳輸介面為光纖,所以每單位成本很高,還要加上光纖 switch 的成本.但是隨著 1G/10/40/100 Gb 網卡的上市,網卡的傳輸頻寬增加,網卡也可以擔任光纖提供的頻寬(光纖的頻寬為 2/4/8/16 Gb,光纖短波-850 mm,長波-1310 mm),這個技術稱為 iSCSI(Internet SCSI),也因此以網卡為傳輸為介質的 SAN,被稱為 IP-SAN.但是傳統的網卡資料是透過 CPU 去解碼運算,所以當做資料儲存裝置(Raw Device)時會消耗太多系統 CPU 資源.
>
>TOE(Broadcom TCP/IP offload engine)或是 Intel® I/O 加速技術 (I/OAT),在網卡上多加入一個專門在運算 TCP/IP 封包的處理器.它可以針對 TCP/IP 的封包直接在網路卡上運算所以不會因此佔用系統上的 CPU 的使用率.
>
> NIC:网络接口控制器(network interface controller)，又称网络适配器（network adapter），网卡（network interface card）或局域网接收器（LAN adapter），是一块被设计用来允许计算机在计算机网络上进行通讯的计算机硬件

那要怎麼確認是被 NIC 的 TOE / TCO 所影響的,可以先檢查 NIC 的 rx-checksumming 以及 tx-checksumming 是否開啟.
透過 ethtool 參數 -k | –show-features | –show-offload devname 來查詢的

``` sh
[root@localhost ~]# ethtool -k ens33 | grep checksum
rx-checksumming: on
tx-checksumming: on
                    tx-checksum-ipv4: on
                    tx-checksum-ip-generic: off [fixed]
                    tx-checksum-ipv6: on
                    tx-checksum-fcoe-crc: off [fixed]
                    tx-checksum-sctp: off [fixed]
```
> 小写k

on 表示為開啟,先關閉來進行測試,透過 ethtool 參數 -K|–features|–offload 來設定.

``` sh
[root@localhost ~]# ethtool -K ens33 tx off rx off
Actual changes:
tx-checksumming: off
                    tx-checksum-ip-generic: off
tcp-segmentation-offload: off
                    tx-tcp-segmentation: off [requested on]
```

> 大写K

透過 tcpdump 再抓一次封包,確認一下 checksum incorrect 是否解決了,如果確認是這個問題,要記得把設定值改回來.

``` sh
[root@localhost ~]# ethtool -K ens33 tx on rx on
Actual changes:
rx-checksumming: on
tx-checksumming: on
                    tx-checksum-ip-generic: on
tcp-segmentation-offload: on
                    tx-tcp-segmentation: on
```
