+++
author = "akirakko"
title = "Kibanaで市町村データを可視化した話"
date = "2025-02-21"
description = "Kibana/ESを使って、市町村の形にそってヒートマップを書いた話"
tags = [
    "Kibana",
    "Elasticsearch",
]
noindex = false
+++

Elasticsearch に入っているデータを元に、Kibana で可視化する機会があった。
値を点・円の大きさで可視化する knowledge はあったが、今回は市町村の形ごとに色を変えるヒートマップとして表現した。
地形データを変えれば都道府県や任意の区分けで色塗りできると思う。

<!--more-->

## 完成図

以下の例は、東京都における年度別・市町村レベル別の人口をヒートマップで可視化したもの。
市町村の形で縁取られたカラーマップとなっていることが確認できる。

![result-map](https://blog.akirakko.com/post/visualization_esdata_in_map/result-map.png)

## 事前準備

### データ投入

今回は可視化に使うデータ（可視化したい情報）を事前に ES へ投入しておく。
例として市町村の人口を可視化してみる。

- 市町村毎の人口：https://www.toukei.metro.tokyo.lg.jp/juukiy/jy-index.htm

### 地形データ準備

- 地形データ：https://github.com/smartnews-smri/japan-topography
  - この例ではローポリゴンな境界線を使ったが、10 倍細かくメッシュを取られた地形データも公開されている

## 手順

1. ES へデータを投入する
2. ES の Maps ➜ Create ➜ Import GeoJson ➜ 地形データの GeoJson を読み込ませる
3. Joins で、「人口と自治体コードの index」と「地形データ」を結合する
   ![joins](https://blog.akirakko.com/post/visualization_esdata_in_map/joins.png)
4. LayerStyle で色や見栄えを設定する
   ![layerstyle](https://blog.akirakko.com/post/visualization_esdata_in_map/layerstyle.png)
5. it works!

## 参考文献

- 加工済み市町村ポリゴンデータ　https://github.com/smartnews-smri/japan-topography.git
- 市区町村データ：国土交通省 国土数値情報（行政区域） https://nlftp.mlit.go.jp/ksj/
- https://qiita.com/terukizm/items/46b531b7fcc7a5f6f436
- https://www.elastic.co/jp/blog/creating-region-map-with-japan-prefectures
- https://dev.classmethod.jp/articles/kibana-region-map-custom-geojson/
