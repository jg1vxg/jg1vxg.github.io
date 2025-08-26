+++
author = "akirakko"
title = "Memo：PoE Injector powered by USB-PD for Site Survey"
date = "2025-08-25"
description = "USB-PD to PoE"
tags = [
    "PoE",
    "USB-PD",
    "WiFi",
    "APoS",
]
noindex = false
+++

USB-PD to PoE

<!--more-->

## Disclaimer

Do your own risk

## Parts

- PoE Injector
  - <https://www.amazon.co.jp/dp/B07VD6B5T8>
- DCDC stepup converter (-> 48V)
  - <https://www.amazon.co.jp/dp/B0CPM2KRSC>
- USB PD Trigger (PD Decoy)
  - <https://www.amazon.co.jp/dp/B09WTQC5Q4>

## Connection

piece of cake

![pic1.png](https://blog.akirakko.com/post/memo-wifi-usb-pd-poe/pic1.png)

I successfully powered up both the UniFi U7 Pro and Cisco 9130 APs using an Anker 737 Power Bank.
According to my Type-C tester (KM003C) readings, power consumption hovers around 20W.
Anker 737 give you approximately 3-4 hours of runtime on the portable battery pack.
