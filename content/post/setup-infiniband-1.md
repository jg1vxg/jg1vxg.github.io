+++
author = "akirakko"
title = "Infinibandを使ったときのメモ"
date = "2024-04-16"
description = "Infinibandを使ったときにどんなコマンドを実行したかのメモ"
tags = [
    "Infiniband",
    "Mellanox",
]
noindex = false
+++

Infinibandを使ったときにどんなコマンドを実行したかのメモ
<!--more-->


# 環境
1:1の対向
## master (subnetmanager)
lumen
```sh
(base) testuser@lumen:~$ uname -a
Linux lumen 5.15.0-102-generic #112~20.04.1-Ubuntu SMP Thu Mar 14 14:28:24 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
(base) testuser@lumen:~$ cat /etc/os-release 
NAME="Ubuntu"
VERSION="20.04.6 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.6 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal
```

## client
```sh
root@ampere:~# uname -a
Linux ampere 6.5.0-27-generic #28~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Fri Mar 15 10:51:06 UTC 2 x86_64 x86_64 x86_64 GNU/Linux
root@ampere:~# cat /etc/os-release 
PRETTY_NAME="Ubuntu 22.04.2 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.2 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```

## 初期設定
### master
```sh
apt install ibverbs-utils infiniband-diags perftest opensm
```

モジュール追記 + 読み込み
```
root@ampere:~# nano /etc/modules
```
```
# For Infiniband
mlx4_ib
rdma_ucm
ib_umad
ib_uverbs
ib_ipoib
```

```
root@ampere:~# modprobe mlx4_ib
root@ampere:~# modprobe rdma_ucm
root@ampere:~# modprobe ib_umad
root@ampere:~# modprobe ib_uverbs
root@ampere:~# modprobe ib_ipoib

opensm start 
```

### client
```sh
apt install ibverbs-utils infiniband-diags perftest 
```

モジュール追記 + 読み込み
```
root@ampere:~# nano /etc/modules
```
```
# For Infiniband
mlx4_ib
rdma_ucm
ib_umad
ib_uverbs
ib_ipoib
```

```
root@ampere:~# modprobe mlx4_ib
root@ampere:~# modprobe rdma_ucm
root@ampere:~# modprobe ib_umad
root@ampere:~# modprobe ib_uverbs
root@ampere:~# modprobe ib_ipoib
```


## 動作確認
###  ibping
#### node1（Pingサーバー側）
``` sh
root@ampere:~# ibstat
CA 'mlx4_0'
	CA type: MT26428
	Number of ports: 1
	Firmware version: 2.9.1000
	Hardware version: b0
	Node GUID: 0x0002c903002b9476
	System image GUID: 0x0002c903002b9479
	Port 1:
		State: Active
		Physical state: LinkUp
		Rate: 40
		Base lid: 2
		LMC: 0
		SM lid: 1
		Capability mask: 0x02500868
		Port GUID: 0x0002c903002b9477
		Link layer: InfiniBand

root@ampere:~# ibping -S
```

#### node2（Pingクライアント側）
```sh
root@lumen:~# ibnodes 
Ca	: 0x0002c903002b9476 ports 1 "MT25408 ConnectX Mellanox Technologies"
Ca	: 0x0002c903002b94c6 ports 1 "MT25408 ConnectX Mellanox Technologies"

root@lumen:~# ibping 0002c903002b94c6  
Pong from ampere.(none) (Lid 2): time 0.057 ms
Pong from ampere.(none) (Lid 2): time 0.073 ms
（中略）
Pong from ampere.(none) (Lid 2): time 0.080 ms
Pong from ampere.(none) (Lid 2): time 0.075 ms
^C
--- ampere.(none) (Lid 2) ibping statistics ---
28 packets transmitted, 28 received, 0% packet loss, time 27905 ms
rtt min/avg/max = 0.057/0.075/0.081 ms
```

### qperf
どちらにもqperfを入れる
```sh 
apt install qperf
```

サーバー側でqperfを立ち上げる
```sh
qperf
```

クライアント側でオプションを指定する
```sh
root@ampere:~# qperf -vv -t 10  10.0.0.1  tcp_bw tcp_lat
tcp_bw:
    bw                =      1.7 GB/sec
    msg_rate          =       26 K/sec
    msg_size          =       64 KiB (65,536)
    time              =       10 sec
    timeout           =        5 sec
    send_cost         =      956 ms/GB
    recv_cost         =     1.36 sec/GB
    send_cpus_used    =      163 % cpus
    send_cpus_user    =      1.5 % cpus
    send_cpus_intr    =     95.1 % cpus
    send_cpus_kernel  =     66.1 % cpus
    send_cpus_iowait  =      0.1 % cpus
    send_real_time    =       10 sec
    send_cpu_time     =     16.3 sec
    send_bytes        =       17 GB
    send_msgs         =  259,717 
    recv_cpus_used    =      231 % cpus
    recv_cpus_user    =      2.3 % cpus
    recv_cpus_intr    =      136 % cpus
    recv_cpus_kernel  =     92.1 % cpus
    recv_cpus_iowait  =      0.1 % cpus
    recv_real_time    =       10 sec
    recv_cpu_time     =     23.1 sec
    recv_bytes        =       17 GB
    recv_msgs         =  259,676 
tcp_lat:
    latency          =     23.4 us
    msg_rate         =     42.7 K/sec
    msg_size         =        1 bytes
    time             =       10 sec
    timeout          =        5 sec
    loc_cpus_used    =     41.4 % cpus
    loc_cpus_user    =      1.9 % cpus
    loc_cpus_intr    =     18.6 % cpus
    loc_cpus_kernel  =     20.9 % cpus
    loc_real_time    =       10 sec
    loc_cpu_time     =     4.14 sec
    loc_send_bytes   =      213 KB
    loc_recv_bytes   =      213 KB
    loc_send_msgs    =  213,395 
    loc_recv_msgs    =  213,394 
    rem_cpus_used    =     41.1 % cpus
    rem_cpus_user    =      2.3 % cpus
    rem_cpus_intr    =     19.3 % cpus
    rem_cpus_kernel  =     19.5 % cpus
    rem_real_time    =       10 sec
    rem_cpu_time     =     4.11 sec
    rem_send_bytes   =      213 KB
    rem_recv_bytes   =      213 KB
    rem_send_msgs    =  213,395 
    rem_recv_msgs    =  213,395 
root@ampere:~# 
root@ampere:~# qperf -vv -t 120 10.0.0.1 rc_rdma_read_bw
rc_rdma_read_bw:
    bw                =  3.51 GB/sec
    msg_rate          =  53.5 K/sec
    msg_size          =    64 KiB (65,536)
    mtu_size          =     2 KiB (2,048)
    time              =   120 sec
    timeout           =     5 sec
    send_cost         =    23 ms/GB
    recv_cost         =   180 ms/GB
    send_cpus_used    =  8.07 % cpus
    send_cpus_user    =  1.67 % cpus
    send_cpus_intr    =  0.01 % cpus
    send_cpus_kernel  =  2.68 % cpus
    send_cpus_iowait  =  3.71 % cpus
    send_real_time    =   120 sec
    send_cpu_time     =   9.7 sec
    send_bytes        =   421 GB
    send_msgs         =  6.42 million
    recv_cpus_used    =  63.1 % cpus
    recv_cpus_user    =    17 % cpus
    recv_cpus_intr    =  16.8 % cpus
    recv_cpus_kernel  =    29 % cpus
    recv_cpus_iowait  =  0.34 % cpus
    recv_real_time    =   120 sec
    recv_cpu_time     =  75.7 sec
    recv_bytes        =   421 GB
    recv_msgs         =  6.42 million
    recv_max_cqe      =    21 

```


### iperf/iperf3
iperf3の場合、コマンドはiperf3
iperf2の場合、コマンドはiperfに置き換えて読むこと。そこまで詳しくは掘らない

#### サーバー側
```sh
root@lumen:~# iperf -s 
------------------------------------------------------------
Server listening on TCP port 5001
TCP window size:  128 KByte (default)
------------------------------------------------------------
[  4] local 10.0.0.1 port 5001 connected with 10.0.0.2 port 49692 (peer 2.1.5)
[  5] local 10.0.0.1 port 5001 connected with 10.0.0.2 port 49706 (peer 2.1.5)
[  6] local 10.0.0.1 port 5001 connected with 10.0.0.2 port 49708 (peer 2.1.5)
[ 11] local 10.0.0.1 port 5001 connected with 10.0.0.2 port 49720 (peer 2.1.5)
[ 14] local 10.0.0.1 port 5001 connected with 10.0.0.2 port 49728 (peer 2.1.5)
[ 16] local 10.0.0.1 port 5001 connected with 10.0.0.2 port 49738 (peer 2.1.5)
[  7] local 10.0.0.1 port 5001 connected with 10.0.0.2 port 49712 (peer 2.1.5)
[ 12] local 10.0.0.1 port 5001 connected with 10.0.0.2 port 49724 (peer 2.1.5)
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0-10.0 sec  1.76 GBytes  1.51 Gbits/sec
[  5]  0.0-10.0 sec  1.62 GBytes  1.39 Gbits/sec
[  6]  0.0-10.0 sec  1.56 GBytes  1.34 Gbits/sec
[ 11]  0.0-10.0 sec  1.69 GBytes  1.45 Gbits/sec
[ 14]  0.0-10.0 sec  1.70 GBytes  1.46 Gbits/sec
[ 16]  0.0-10.0 sec  1.66 GBytes  1.42 Gbits/sec
[  7]  0.0-10.0 sec  1.48 GBytes  1.27 Gbits/sec
[ 12]  0.0-10.0 sec  1.93 GBytes  1.66 Gbits/sec
[SUM]  0.0-10.0 sec  13.4 GBytes  11.5 Gbits/sec

```

#### クライアント側
```sh
root@ampere:~# iperf -c 10.0.0.1 -P 8
[  2] local 10.0.0.2 port 49692 connected with 10.0.0.1 port 5001
------------------------------------------------------------
Client connecting to 10.0.0.1, TCP port 5001
TCP window size: 85.0 KByte (default)
------------------------------------------------------------
[  4] local 10.0.0.2 port 49720 connected with 10.0.0.1 port 5001
[  8] local 10.0.0.2 port 49724 connected with 10.0.0.1 port 5001
[  7] local 10.0.0.2 port 49738 connected with 10.0.0.1 port 5001
[  3] local 10.0.0.2 port 49706 connected with 10.0.0.1 port 5001
[  6] local 10.0.0.2 port 49708 connected with 10.0.0.1 port 5001
[  1] local 10.0.0.2 port 49712 connected with 10.0.0.1 port 5001
[  5] local 10.0.0.2 port 49728 connected with 10.0.0.1 port 5001
[ ID] Interval       Transfer     Bandwidth
[  1] 0.0000-10.0221 sec  1.48 GBytes  1.27 Gbits/sec
[  5] 0.0000-10.0220 sec  1.70 GBytes  1.45 Gbits/sec
[  2] 0.0000-10.0218 sec  1.76 GBytes  1.50 Gbits/sec
[  4] 0.0000-10.0218 sec  1.69 GBytes  1.45 Gbits/sec
[  6] 0.0000-10.0219 sec  1.56 GBytes  1.33 Gbits/sec
[  3] 0.0000-10.0218 sec  1.62 GBytes  1.38 Gbits/sec
[  7] 0.0000-10.0221 sec  1.66 GBytes  1.42 Gbits/sec
[  8] 0.0000-10.0220 sec  1.93 GBytes  1.66 Gbits/sec
[SUM] 0.0000-10.0102 sec  13.4 GBytes  11.5 Gbits/sec
[ CT] final connect times (min/avg/max/stdev) = 0.153/0.279/0.383/0.075 ms (tot/err) = 8/0
```