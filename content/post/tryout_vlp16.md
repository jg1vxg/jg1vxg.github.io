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

LiDAR 入門として大変有名な VLP-16 を購入したので、物理配線を行って起動してみたという話

<!--more-->

# はじめに

- n 番煎じのネタですが、それでも良ければ
- LiDAR を買う～動作確認まで、の素人の試行錯誤をつらつらと
  - 記事に間違いなどあれば教えていただけると助かります
- VLP-16 改め LiDAR PUCK らしいですが VLP-16 と書きます
- 試す場合は自己責任でどうぞ（いつもの
- author は ROS などについてど素人

# Introduction

Twitter を眺めていたら Interface の 2023 年 1 月号に関するツイートが流れてきて、面白そうだったので購入。
https://interface.cqpub.co.jp/magazine/202301/
![interface](https://blog.akirakko.com/post/tryout_vlp16_materials/9a59cde8-5a52-9738-7042-96502114db75.png)

読んでいるうちに実機を触りたくなり、Twitter をポチポチしているうちに

- Velodyne というのが取っ付きやすそうである
  - VLP-16 はつくばチャレンジ？なるもので良く用いられているらしい
  - ネットに情報もよく出ている
- Ebay にて動作確認済みが売られている
- $200~300USD 程度である

という情報を仕入れていざ鎌倉。

# 購入

Ebay で適当な動作確認済みと謳っている品を落札。落札から 3 週間程度で届きました。

![vlp-16-overview.jpg](https://blog.akirakko.com/post/tryout_vlp16_materials/vlp-16-overall.png)

どんなものが来るのやらと戦々恐々としていましたが、けっこう綺麗なものが届いて安堵しました。

# 配線（もといジャンクションボックス自作）

買った LiDAR の配線は長さ 20cm 程度で 8 ピンコネクタがついていました。切るのは忍びなかったので、ジャンクションボックス的なのを自作することに（と言っても、適当に繋いでボックスに突っ込んだだけ）。

## 電源

LiDAR は手持ちで扱いたかったので、モバイルバッテリーから給電できるように USB PD のトリガーケーブルをポチりました。
https://www.amazon.co.jp/gp/product/B08JVJT9P7

![trigger-cable](https://blog.akirakko.com/post/tryout_vlp16_materials/type-c-trigger-cable-amazon.png)

この PD トリガーケーブルは DCφ2.1 らしいので、それに合うコネクタを適当に買ってきてガッチャンコします。

## LAN コネクタ

そこらへんに転がっている LAN ケーブルをぶった切ってくっつけました。

## LiDAR との接続

LiDAR から出てるコネクタは M12 の 8 ピンらしく、片側メスのアセンブリ済みがモノタロウに転がっていたのでこちらを購入。
https://www.monotaro.com/p/6404/6965/

![m12-cable](https://blog.akirakko.com/post/tryout_vlp16_materials/m12-code-cable-monotaro.png)

ピンアサインは oxts のサイトに乗っていた図を頼りにして作成。買ったメスケーブルとテスターを両手に頑張ります。
ネット上ではピンアサインが複数あるように見えますが、私が見つけた限りは M12 コネクタのオスかメスの違いでした。oxts のサイトに乗っている（下記写真の）ピンアサインは Male M12 なので、メス側のピンアサインはちょうど鏡のようになっているはずです。

![m12-pin](https://blog.akirakko.com/post/tryout_vlp16_materials/vlp-16-oxts-pin.png)

モノタロウで買った M12 メスケーブルは、色とピン番号がわかりません。ので、テスターで何色が何番に来ているかを調べ、LiDAR 側 M12 オスコネクタとの配線を考えます。

![m12-table](https://blog.akirakko.com/post/tryout_vlp16_materials/vlp-16-oxts-pin-table.png)

### LAN（M12 オス・端子 1~3 と 6 番）

Ether と書いてあるので、LAN ケーブルと接続するのでしょう。
VLP-16 は 100BASE-T なので、LAN ケーブルに使われる 4 対 8 本のうち、2 対 4 本だけ接続すれば良いわけです。
https://www.infraexpert.com/study/ethernet3.html

![rj45-pin](https://blog.akirakko.com/post/tryout_vlp16_materials/rj45-pin-excellent-fig.png)

B タイプの LAN ケーブルを切って使う場合、LAN（RJ45）コネクタの 1,2,3,6 番を接続すれば良いことが分かります。
あとは図の通り、VLP-16 の TX+を RJ45 の RX+へ、VLP-16 の TX-を RJ45 の RX-へ......と繋げばよいはずです。

### Power（端子 4 番）

VLP-16 の電源の＋側です。
私は PD トリガーケーブルを利用したので、端子のセンターをつなぎました。

### GPS（端子 5 と 8 番）

GPS のシリアルと PPS を VLP-16 に接続できるようです。
私は利用しなかったので、接続なし NC です。

### Ground(端子 7 番）

電源のー側、GPS を繋いだ場合は GPS の Ground もここにつなぎましょう。
私は電源のマイナス側だけ接続しました。

# It works!

![aaa](https://blog.akirakko.com/post/tryout_vlp16_materials/rviz-lidar-itworks.png)

# Reference & SpecialThanks

- [はじめての Velodyne
  ](https://qiita.com/TakanoTaiga/items/989a6407ed1ae2d1a97d)
- [RaspberryPi OS で VLP-16 を ROS2 で動かす（RaspberryPi4）](https://ar-ray.hatenablog.com/entry/2023/01/14/135723)
- [Velodyne VLP-32C を ros2 humble で動かした話](https://zenn.dev/naokitakahashi2/articles/6e6a9f730cce1d)
- [Hardware Integration with Velodyne LiDAR
  ](https://support.oxts.com/hc/en-us/articles/360017123620-Hardware-Integration-with-Velodyne-LiDAR-)
- [Ethernet LAN - Straight Cable / Cross Cable
  ](https://www.infraexpert.com/study/ethernet3.html)

---

この記事は Qiita から転記しました。
https://qiita.com/vxgatrtl/items/9a0ca35267ed63cecd6a
