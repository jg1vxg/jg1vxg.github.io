+++
author = "akirakko"
title = "eBayで安く100GbEの夢を見れるか"
date = "2025-03-19"
description = "Chelsio T6 + FINISAR 100G-CWDM4を使ってみた話"
tags = [
    "100Gbps",
    "Chelsio T6",
    "FINISAR",
    "eBay",
    "NIC",
]
noindex = false
+++

100GbE の NIC を購入したので、動作確認をした話

<!--more-->

## はじめに

常日頃、NW をもう少し高速化したいけど少しお値段がな〜と思っていた。
そんな中、

- 100Gbps の NIC は安い（スイッチはともかくとして）
- 光トランシーバーも eBay にてどうやら投げ売りしている

というの書き込みを見かけたので、PCtoPC の直接接続を目的に買ってみることに。

## 構成

### NIC 関係

- NIC: Chelsio Communications 110-1220-60
  - 100GbE 対応の Chelsio T6 チップが乗っている 2 ポート NIC
  - どうやら T62100-LP-CR らしい
  - PCI ブラケットも別で売ってる
- 光トランシーバー:Finisar FTLC1157RGPL6FB1 100G-CWDM4 Lite
  - 話題のやつ
  - もう少し安いのもあったが、シングルモードファイバを使いたかった
- 光ファイバー: 10Gtek OS2-LC-LC-D10M
  - Amazon でまともそうで安かったのでコレ

### PC 関係

今回は baltazar と melchior と名付けた（仮初） 2 台を直結する。

- Ubuntu 24.04 LTS
  - `Linux ohm 6.11.0-19-generic #19~24.04.1-Ubuntu SMP PREEMPT_DYNAMIC Mon Feb 17 11:51:52 UTC 2 x86_64 x86_64 x86_64 GNU/Linux`
  - カーネルチューニングなどは特段無し
- Intel Xeon Gold 6530(HT 無効) 4.0GHz x 2way
- DDR5-4800 256GB

## 導入・インストール

電源落として NIC を PCIe へ差し込み、起動するだけで NIC が認識される。
[AMD の資料](https://www.amd.com/content/dam/amd/en/documents/epyc-business-docs/performance-briefs/100g-network-performance-for-amd-epyc.pdf)では Chelsio Unified Wire を導入しているが、今回は出来合いのものを利用する。

光トランシーバーは不調のものをいくつか引いた。今回は 10 個購入したうち 2 個はリンクアップせず、また ethtool でも読めなかった。

![shindeta-transi-ba-.png](https://blog.akirakko.com/post/ebay-gekiyasu-100gbe-1/shindeta_toransi-ba-.png)

生きてそうな光トランシーバーでは次のように読めた。

```bash
user@baltazar:~$ ethtool ens5f4 -m
Settings for ens5f4:
	Supported ports: [ FIBRE ]
	Supported link modes:   1000baseT/Full
	                        10000baseKR/Full
	                        40000baseSR4/Full
	                        25000baseCR/Full
	                        50000baseCR2/Full
	                        100000baseCR4/Full
	Supported pause frame use: Symmetric Receive-only
	Supports auto-negotiation: No
	Supported FEC modes: RS	 BASER
	Advertised link modes:  100000baseCR4/Full
	Advertised pause frame use: Symmetric
	Advertised auto-negotiation: No
	Advertised FEC modes: RS
	Link partner advertised link modes:  Not reported
	Link partner advertised pause frame use: Symmetric
	Link partner advertised auto-negotiation: No
	Link partner advertised FEC modes: None
	Speed: 100000Mb/s
	Duplex: Full
	Auto-negotiation: off
	Port: FIBRE
	PHYAD: 255
	Transceiver: internal
netlink error: Operation not permitted
        Current message level: 0x000000ff (255)
                               drv probe link timer ifdown ifup rx_err tx_err
	Link detected: yes

```

## iperf

対向 1 ポートを直結し、適当に ip アドレスを振り、MTU9000 に設定した。
その後 iperf2 でピーク性能を見る。

### サーバー側コマンド

```bash
$ sudo apt update
$ sudo apt install iperf

$ iperf -s -B 192.168.101.111 # 諸事情で100GのみBindする
```

### クライアント側コマンド

```bash
$ sudo apt update
$ sudo apt install iperf

$ iperf -c 192.168.101.111 -P 2
```

### 結果

並列数を 2,4,8,16 で試したところ、8 並列で 98.4Gbps とほぼ限界値が出た。

（（（ログ中の IP アドレスなどはホスト名に雑置換している）））

```bash
user@melchior:~$ iperf -c baltazar -P 2
------------------------------------------------------------
Client connecting to baltazar, TCP port 5001
TCP window size: 16.0 KByte (default)
------------------------------------------------------------
[  2] local melchior port 53994 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/333)
[  1] local melchior port 54006 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/330)
[ ID] Interval       Transfer     Bandwidth
[  1] 0.0000-10.0096 sec  40.5 GBytes  34.7 Gbits/sec
[  2] 0.0000-10.0097 sec  42.3 GBytes  36.3 Gbits/sec
[SUM] 0.0000-10.0002 sec  82.7 GBytes  71.1 Gbits/sec


user@melchior:~$ iperf -c baltazar -P 4
------------------------------------------------------------
Client connecting to baltazar, TCP port 5001
TCP window size: 16.0 KByte (default)
------------------------------------------------------------
[  1] local melchior port 39792 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/317)
[  2] local melchior port 39800 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/263)
[  3] local melchior port 39806 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/206)
[  4] local melchior port 39804 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/299)
[ ID] Interval       Transfer     Bandwidth
[  1] 0.0000-10.0051 sec  34.0 GBytes  29.2 Gbits/sec
[  2] 0.0000-10.0048 sec  34.8 GBytes  29.9 Gbits/sec
[  3] 0.0000-10.0050 sec  17.9 GBytes  15.4 Gbits/sec
[  4] 0.0000-10.0050 sec  18.0 GBytes  15.5 Gbits/sec
[SUM] 0.0000-10.0001 sec   105 GBytes  89.9 Gbits/sec


user@melchior:~$ iperf -c baltazar -P 8
------------------------------------------------------------
Client connecting to baltazar, TCP port 5001
TCP window size:  325 KByte (default)
------------------------------------------------------------
[  3] local melchior port 43964 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/282)
[  1] local melchior port 43958 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/295)
[  2] local melchior port 43980 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/305)
[  5] local melchior port 43998 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/186)
[  6] local melchior port 44004 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/283)
[  7] local melchior port 43982 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/302)
[  4] local melchior port 43994 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/286)
[  8] local melchior port 43966 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/334)
[ ID] Interval       Transfer     Bandwidth
[  2] 0.0000-10.0092 sec  16.3 GBytes  14.0 Gbits/sec
[  1] 0.0000-10.0092 sec  16.3 GBytes  14.0 Gbits/sec
[  7] 0.0000-10.0091 sec  8.26 GBytes  7.09 Gbits/sec
[  3] 0.0000-10.0092 sec  16.4 GBytes  14.1 Gbits/sec
[  5] 0.0000-10.0093 sec  16.4 GBytes  14.1 Gbits/sec
[  4] 0.0000-10.0093 sec  16.4 GBytes  14.1 Gbits/sec
[  6] 0.0000-10.0093 sec  8.15 GBytes  7.00 Gbits/sec
[  8] 0.0000-10.0092 sec  16.3 GBytes  14.0 Gbits/sec
[SUM] 0.0000-10.0013 sec   115 GBytes  98.4 Gbits/sec


user@melchior:~$ iperf -c baltazar -P 16
------------------------------------------------------------
Client connecting to baltazar, TCP port 5001
TCP window size: 16.0 KByte (default)
------------------------------------------------------------
[  4] local melchior port 41592 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/283)
[  5] local melchior port 41572 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/334)
[  1] local melchior port 41570 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/351)
[  3] local melchior port 41618 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/337)
[ 11] local melchior port 41662 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/207)
[  9] local melchior port 41632 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/314)
[  6] local melchior port 41642 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/247)
[ 12] local melchior port 41648 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/267)
[ 10] local melchior port 41706 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/47)
[  2] local melchior port 41568 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/351)
[ 14] local melchior port 41720 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/72)
[  8] local melchior port 41676 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/243)
[ 16] local melchior port 41566 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/362)
[ 15] local melchior port 41574 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/339)
[  7] local melchior port 41590 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/343)
[ 13] local melchior port 41674 connected with baltazar port 5001 (icwnd/mss/irtt=87/8948/159)
[ ID] Interval       Transfer     Bandwidth
[ 14] 0.0000-10.0062 sec  3.26 GBytes  2.80 Gbits/sec
[  5] 0.0000-10.0061 sec  6.45 GBytes  5.53 Gbits/sec
[  6] 0.0000-10.0063 sec  12.5 GBytes  10.7 Gbits/sec
[  2] 0.0000-10.0061 sec  6.61 GBytes  5.67 Gbits/sec
[  8] 0.0000-10.0057 sec  6.56 GBytes  5.63 Gbits/sec
[ 10] 0.0000-10.0058 sec  3.30 GBytes  2.84 Gbits/sec
[ 12] 0.0000-10.0057 sec  3.32 GBytes  2.85 Gbits/sec
[  1] 0.0000-10.0056 sec  6.52 GBytes  5.59 Gbits/sec
[  7] 0.0000-10.0063 sec  6.49 GBytes  5.57 Gbits/sec
[ 11] 0.0000-10.0063 sec  6.45 GBytes  5.54 Gbits/sec
[  3] 0.0000-10.0059 sec  12.5 GBytes  10.7 Gbits/sec
[  9] 0.0000-10.0058 sec  6.45 GBytes  5.53 Gbits/sec
[ 15] 0.0000-10.0053 sec  6.58 GBytes  5.65 Gbits/sec
[  4] 0.0000-10.0061 sec  6.50 GBytes  5.58 Gbits/sec
[ 16] 0.0000-10.0064 sec  3.30 GBytes  2.83 Gbits/sec
[ 13] 0.0000-10.0055 sec  6.37 GBytes  5.47 Gbits/sec
[SUM] 0.0000-10.0048 sec   103 GBytes  88.6 Gbits/sec
```

ちゃんとするなら PCIe スロットと CPU の接続、iperf が実行されたコア、UPI リンクなどを吟味する必要がある。
が、今回はこれで十分なのでヨシとする。
結構アチアチになるので、ちゃんと風を当てたほうがいいかも。

次はこの NIC を使って 3 ノード MPI・OpenMP 並列を試してみる。

## 参考文献

- <https://blog.nishi.network/2020/08/26/Mellanox-100GbE-1/>
- <https://qiita.com/shirok/items/531a86d635e524a9c6ac>
