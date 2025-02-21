+++
author = "akirakko"
title = "Velodyne VLP-16を買って動かしてみた"
date = "2024-04-04"
description = "VLP-16を購入・接続した備忘録"
tags = [
    "LiDAR",
    "VLP-16",
    "Velodyne",
    "ROS2",
]
noindex = false
+++

LiDAR入門として大変有名なVLP-16を購入したので、物理配線を行って起動してみたという話
<!--more-->


# はじめに
- n番煎じのネタですが、それでも良ければ
- LiDARを買う～動作確認まで、の素人の試行錯誤をつらつらと
  - 記事に間違いなどあれば教えていただけると助かります
- VLP-16改めLiDAR PUCKらしいですがVLP-16と書きます
- 試す場合は自己責任でどうぞ（いつもの
- authorはROSなどについてど素人



# Introduction
Twitterを眺めていたらInterfaceの2023年1月号に関するツイートが流れてきて、面白そうだったので購入。
https://interface.cqpub.co.jp/magazine/202301/
![interface](https://blog.akirakko.com/post/tryout_vlp16_materials/9a59cde8-5a52-9738-7042-96502114db75.png)


読んでいるうちに実機を触りたくなり、Twitterをポチポチしているうちに
- Velodyneというのが取っ付きやすそうである
    - VLP-16はつくばチャレンジ？なるもので良く用いられているらしい
    - ネットに情報もよく出ている
- Ebayにて動作確認済みが売られている
- $200~300USD程度である

という情報を仕入れていざ鎌倉。


# 購入
Ebayで適当な動作確認済みと謳っている品を落札。落札から3週間程度で届きました。

![vlp-16-overview.jpg](https://blog.akirakko.com/post/tryout_vlp16_materials/vlp-16-overall.png)

どんなものが来るのやらと戦々恐々としていましたが、けっこう綺麗なものが届いて安堵しました。

# 配線（もといジャンクションボックス自作）
買ったLiDARの配線は長さ20cm程度で8ピンコネクタがついていました。切るのは忍びなかったので、ジャンクションボックス的なのを自作することに（と言っても、適当に繋いでボックスに突っ込んだだけ）。


## 電源
LiDARは手持ちで扱いたかったので、モバイルバッテリーから給電できるようにUSB PDのトリガーケーブルをポチりました。
https://www.amazon.co.jp/gp/product/B08JVJT9P7

![trigger-cable](https://blog.akirakko.com/post/tryout_vlp16_materials/type-c-trigger-cable-amazon.png)

このPDトリガーケーブルはDCφ2.1らしいので、それに合うコネクタを適当に買ってきてガッチャンコします。

## LANコネクタ
そこらへんに転がっているLANケーブルをぶった切ってくっつけました。

## LiDARとの接続
LiDARから出てるコネクタはM12の8ピンらしく、片側メスのアセンブリ済みがモノタロウに転がっていたのでこちらを購入。
https://www.monotaro.com/p/6404/6965/

![m12-cable](https://blog.akirakko.com/post/tryout_vlp16_materials/m12-code-cable-monotaro.png)

ピンアサインはoxtsのサイトに乗っていた図を頼りにして作成。買ったメスケーブルとテスターを両手に頑張ります。
ネット上ではピンアサインが複数あるように見えますが、私が見つけた限りはM12コネクタのオスかメスの違いでした。oxtsのサイトに乗っている（下記写真の）ピンアサインはMale M12なので、メス側のピンアサインはちょうど鏡のようになっているはずです。

![m12-pin](https://blog.akirakko.com/post/tryout_vlp16_materials/vlp-16-oxts-pin.png)

モノタロウで買ったM12メスケーブルは、色とピン番号がわかりません。ので、テスターで何色が何番に来ているかを調べ、LiDAR側M12オスコネクタとの配線を考えます。

![m12-table](https://blog.akirakko.com/post/tryout_vlp16_materials/vlp-16-oxts-pin-table.png)

### LAN（M12オス・端子1~3と6番）
Etherと書いてあるので、LANケーブルと接続するのでしょう。
VLP-16は100BASE-Tなので、LANケーブルに使われる4対8本のうち、2対4本だけ接続すれば良いわけです。
https://www.infraexpert.com/study/ethernet3.html

![rj45-pin](https://blog.akirakko.com/post/tryout_vlp16_materials/rj45-pin-excellent-fig.png)

BタイプのLANケーブルを切って使う場合、LAN（RJ45）コネクタの1,2,3,6番を接続すれば良いことが分かります。
あとは図の通り、VLP-16のTX+をRJ45のRX+へ、VLP-16のTX-をRJ45のRX-へ......と繋げばよいはずです。


### Power（端子4番）
VLP-16の電源の＋側です。
私はPDトリガーケーブルを利用したので、端子のセンターをつなぎました。

### GPS（端子5と8番）
GPSのシリアルとPPSをVLP-16に接続できるようです。
私は利用しなかったので、接続なしNCです。

### Ground(端子7番）
電源のー側、GPSを繋いだ場合はGPSのGroundもここにつなぎましょう。
私は電源のマイナス側だけ接続しました。



# It works!
![aaa](https://blog.akirakko.com/post/tryout_vlp16_materials/rviz-lidar-itworks.png)

# Reference & SpecialThanks
- [はじめてのVelodyne
](https://qiita.com/TakanoTaiga/items/989a6407ed1ae2d1a97d)
- [RaspberryPi OSでVLP-16をROS2で動かす（RaspberryPi4）](https://ar-ray.hatenablog.com/entry/2023/01/14/135723)
- [Velodyne VLP-32C を ros2 humble で動かした話](https://zenn.dev/naokitakahashi2/articles/6e6a9f730cce1d)
- [Hardware Integration with Velodyne LiDAR
](https://support.oxts.com/hc/en-us/articles/360017123620-Hardware-Integration-with-Velodyne-LiDAR-)
- [Ethernet LAN - Straight Cable / Cross Cable
](https://www.infraexpert.com/study/ethernet3.html)

***

この記事はQiitaから転記しました。
https://qiita.com/vxgatrtl/items/9a0ca35267ed63cecd6a
