+++
author = "akirakko"
title = "PythonのExe化"
date = "2022-07-27"
description = "Pythonで作成したGUIアプリをExe化した話"
tags = [
    "Python",
    "Windows",
    "Tkinter",
]
noindex = false
+++

Python・Tkinterで業務アプリを作成する機会があり、配布のためにPython仮想環境ごとExe化したという話
<!--more-->


# Python exe化
 Pythonをexe化するとなにが嬉しいかって？？
 - コマンドが苦手なひとでも動かせる！
 - 環境構築が必要ない！exeファイルをコピペするだけで(๑•̀ㅂ•́)و✧


## via PyInstaller
`$ pip install pyinstaller`
`$ pyinstaller hoge.py --onefile`
とすると、配下のdistフォルダの中に生成される。簡単！
なお2-3分かかる模様

### 注意点
- 仮想環境(Pyenv、anaconda）にインストールされた**すべてのライブラリ**を纏めようとするのでファイルサイズがデカくなる。
    - 特に、numpyなどの数値計算ライブラリは問答無用で200MB超えたりする。numpyが依存するMKLがでかいらしい。
    - ので、プログラムはなるべくライブラリを使わず + 最終的なexe化は必要最小限の仮想環境を作って実施すると吉
    
- ファイルサイズがでかいと、問答無用で実行時間も長くなる。
    - Pythonネイティブだと3秒もかからないのに、60秒かかったりとか。

## via nuitka
Referer：
https://blog.tsukumijima.net/article/python-nuitka-usage/

```sh
pip install nuitka
nuitka --mingw64 --follow-imports --onefile (エントリーポイントにしたい Python ファイル).py 
```

### 注意点
- ビルドが遅い
    - Python→C++→exeと踏んでいるらしい。難読化かな（）
    - Pyinstallerの10倍はかかる


## 直面した課題

### exe化しても動かない！
- main関数を分ける。
    - ![main](https://blog.akirakko.com/post/exenize-python-code/image.png)
    - いままで、def main()の中身をそのままif __name__ =='__main__':に書いてたらno module named hogehogeだった
