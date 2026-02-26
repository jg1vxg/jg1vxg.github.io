+++
title = "NEC IXでCisco Meraki APのEthernet over GREを終端する"
date = "2026-02-26"
tags = ["NEC IX", "Meraki", "EoGRE", "GRE", "L2VPN"]
description = "Cisco APのEoGREの終端をNEC IXでやってみた話"
noindex = false
+++

APにVLANをたくさん収容したかったが、フロアスイッチの設定コストが高く、QinQやVlanIDをたくさん利用することが難しい環境だった。そこで、手元にあったNEC IXを用いてEoGREを終端してみることにした。
Cisco Meraki APのEoGRE（Ethernet over GRE）機能を使い、Wi-FiクライアントのL2トラフィックをNEC IXで終端・ブリッジする構成となっている。
CW9172 + NEC IX3110の組み合わせでは500Mbpsぐらい流したところでCPU律速となった。特にハードルがなければASR終端やVLANを引っ張るほうが無難。


<!--more-->

## 概要

既存ネットワーク上のVLAN 212およびVLAN 427を、Meraki APを経由してWi-Fiクライアントまで延伸したい。APは2台で、管理用サブネット172.21.19.0/26内にDHCPでIPアドレスが割り当てられる。延伸先のサブネットにはDHCPサーバが設置済みのため、NEC IX側ではNATやDHCPの処理は行わず、純粋なL2ブリッジとして動作させる。

Meraki Dashboard上の設定パスは `Wireless > Configure > Access control > Client IP and VLAN` で、`Tunneled` → `Ethernet over GRE` を選択してコンセントレーター (=NEC IX) のIPとGRE Keyを入力する。


## ネットワーク構成

```
 既存ネットワーク (Trunk: VLAN212, VLAN427)
    │
  GE1 (Tagged)
 ┌──┴──────────────────────────┐
 │        NEC IX3110           │
 │  GE0.0: 10.0.0.11/24    │
 │  GE1.1: VLAN212 (bg 212)   │
 │  GE1.2: VLAN427 (bg 427)   │
 │  Tunnel0: AP1×V212 (bg 212)│
 │  Tunnel1: AP1×V427 (bg 427)│
 │  Tunnel2: AP2×V212 (bg 212)│
 │  Tunnel3: AP2×V427 (bg 427)│
 └──┬──────────────────────────┘
  GE0 (Untagged: MGMT)
    │
    ├── Meraki AP1 (10.0.0.12)
    └── Meraki AP2 (10.0.0.13)
```

GE0をMGMTサブネット兼GREトンネルのSourceとし、GE1を上流スイッチとのTrunkに使用する。分離する意味はあんまりなかった。

### GRE KeyとVLAN IDの対応

| VLAN ID | GRE Key | bridge-group | Tunnel (AP1) | Tunnel (AP2) |
|---------|---------|-------------|-------------|-------------|
| 212     | 212     | 212         | Tunnel0.0   | Tunnel2.0   |
| 427     | 427     | 427         | Tunnel1.0   | Tunnel3.0   |

GRE Key = VLAN IDとしてみた

### AP側IPアドレスの扱い

AP 2台はMGMTサブネットのDHCPで動的にIPを取得する。NEC IXのGREトンネルは `tunnel destination` にIPアドレスの指定したため、DHCPサーバ側でMACアドレスベースの固定割当を行い、AP1を10.0.0.12、AP2を10.0.0.13に固定した。
AP側不定でも出来るのかは試してない。


### コンフィグ

```
bridge irb enable

interface GigaEthernet0.0
  ip address 10.0.0.11/24
  no shutdown

interface GigaEthernet1.0
  no ip address
  no shutdown

interface GigaEthernet1.1
  encapsulation dot1q 212 tpid 8100
  auto-connect
  no ip address
  bridge-group 212
  no shutdown

interface GigaEthernet1.2
  encapsulation dot1q 427 tpid 8100
  auto-connect
  no ip address
  bridge-group 427
  no shutdown

interface Tunnel0.0
  tunnel mode gre ip
  tunnel destination 10.0.0.12
  tunnel source GigaEthernet0.0
  tunnel key 212
  no ip address
  bridge-group 212
  no shutdown

interface Tunnel1.0
  tunnel mode gre ip
  tunnel destination 10.0.0.12
  tunnel source GigaEthernet0.0
  tunnel key 427
  no ip address
  bridge-group 427
  no shutdown

interface Tunnel2.0
  tunnel mode gre ip
  tunnel destination 10.0.0.13
  tunnel source GigaEthernet0.0
  tunnel key 212
  no ip address
  bridge-group 212
  no shutdown

interface Tunnel3.0
  tunnel mode gre ip
  tunnel destination 10.0.0.13
  tunnel source GigaEthernet0.0
  tunnel key 427
  no ip address
  bridge-group 427
  no shutdown

ip route default 10.0.0.1
ip ufs-cache enable
```

この設定だとAP数・VLAN数で無限に設定が増えていくので、もう少しコンパクトになると嬉しい。今回はまあヨシとする。


## Meraki Dashboard側の設定

SSIDごとに以下を設定する。

- `Wireless > Configure > Access Control > Client IP and VLAN` → `External DHCP server assigned`
- `Tunneled` → `Ethernet over GRE: tunnel data to a concentrator`
- Concentrator: `10.0.0.11`（IX3110のGE0.0）
- GRE Key: SSID-Aは `212`、SSID-Bは `427`

## 結果
NEC IX3110を終端としてスピテスをすると、約500Mbpsで頭打ちになる。CPU使用率は50%に張り付き、1コアを使い切っている様子。この構成、緊急避難としては良いかもだが、GREがHWサポートされた終端を用意するのが良いですね。

## 参考

- [Meraki: EoGRE Concentration for SSIDs](https://documentation.meraki.com/General_Administration/Service_Providers_-_SPs/EoGRE_Concentration_for_SSIDs)
- [NEC IX FAQ: Ethernet over IP機能](https://jpn.nec.com/univerge/ix/faq/etherip.html)
- [NEC IX FAQ: ブリッジ機能](https://jpn.nec.com/univerge/ix/faq/bridge.html)
- [NXP MPC8572E Product Page](https://www.nxp.com/products/MPC8572E)
- [UNIVERGE IX3315 製品情報](https://jpn.nec.com/univerge/ix/Info/ix3315.html)
