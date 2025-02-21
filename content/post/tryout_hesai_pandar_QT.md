+++
author = "akirakko"
title = "Hesai PandarQTを買って動かしてみた"
date = "2025-02-21"
description = "Hesai PandarQTを購入・接続した備忘録"
tags = [
    "LiDAR",
    "PandarQT",
    "Hesai",
    "ROS2",
]
noindex = false
+++


## はじめに

以前VLP-16を買って遊んでたが、

- Line数がもっと多いLiDARも使ってみたい
- 室内で使うには視野角が上下方向の視野角が狭い

というモチベーションでeBayをグルグルしたところ、Hesai Pandar QT64が安かったので購入。
諸々込で3万円台だったので、お買い得だったのではないだろうか？

## 構成

- LiDAR本体
  - Hesai PandarQT (64Line)
- InterfaceBox
  - QT64との通信は100BASE-T1という2線式Etherらしく、VLP-16の100BASE-Tと異なる
- PC
  - ゲーミングノート

## PC側設定

- Ubuntu 22.04
- ROS2 Humble導入済み
- CUDA 導入済み
- ノートPCのIPアドレスをLiDARと同じセグメントとなるよう設定済み。
- NTP(ntp.nict.jp)設定済み

### PTPサーバー構築

LiDAR単体で使う分には時刻同期は必要ない。
が、IMUやカメラなどとセンサフュージョンする際には、LiDARを含めてシステム全体の時刻を正しく設定する必要がある。
QT64は時刻同期にPTPというプロトコルを用いるので、まずはじめにLAN内PTPサーバーを設置する。

本来であればGPSなどをベースクロックとした、グランドマスタークロックを基準として時刻同期するのが良いと思う。
今回は簡単お手軽にノートPCを簡易PTPサーバー化する。

```bash
# apt install ptp4l
# ptp4l -i eth3 -m -S
```

ターミナルを閉じるとptpサーバーも止まるので、必要であればdaemon化すること。
eth3はNICを指定する。
NICによってはハードウェアタイムスタンプが利用できたりするが、今回はこれでよしする。

`ptp4l`を実行した状態でPCとLiDARを同じにLANへ接続すると、LiDARが自動的にPTPサーバーを探して時刻同期を始める。
LiDARが正しく同期されているかの確認としては、設定画面のPTPステータスで判断できる。
Lockedは正しく同期している状態、Freerunは時刻同期されていない状態、Trackingは同期を試みている状態を指すらしい。

### LiDARのドライバー導入

HesaiLidar_ROS_2.0のREADME通りに進める。

```sh
sudo apt-get update
sudo apt-get install -y libboost-all-dev libyaml-cpp-dev

mkdir -p ./colcon_ws/src
cd ./HesaiLidar_ROS_2.0/src
git clone --recurse-submodules https://github.com/HesaiTechnology/HesaiLidar_ROS_2.0.git

cd ..

colcon build --symlink-install
. install/local_setup.bash

```

CUDAを入れていると、自動で判定して有効にしてくれる。
`CUDA Found. CUDA Support is turned On.`というメッセージが表示される。

## PCAPを再生するとき

pcapを再生するときは設定を変更する必要がある。
これもREADMEに記載があるが、`config.yaml`の中のpcap_path、correction_file_path、firetimes_pathを指定する必要がある。

```yaml
lidar:
- driver:              
    udp_port: 2368                  
    ptc_port: 9347              
    device_ip_address: 192.168.1.201          
    pcap_path: "<The PCAP file path>"                  
    correction_file_path: "<The correction file path>" 
    firetimes_path: "<Your firetime file path>"      

```

correction_fileとfiretimesはHesaiLidar_SDK_2.0から持ってくる。
PandarQT (64Line)の場合、次の2ファイルを用意して、config.yamlで指定すると良さそう（間違ってたら教えて下さい）。

- ![correction_file](https://github.com/HesaiTechnology/HesaiLidar_SDK_2.0/blob/master/correction/angle_correction/PandarQT_Angle%20Correction%20File.csv)
- ![firetime](https://github.com/HesaiTechnology/HesaiLidar_SDK_2.0/blob/master/correction/firetime_correction/PandarQT_Firetime%20Correction%20File.csv)

### 参考文献

- <https://www.abudorilab.com/entry/2024/07/16/225906>
- <https://github.com/HesaiTechnology/HesaiLidar_ROS_2.0>
