---
title: "JetBrainsのIDEで\\を入力すると直前の文字が消えてしまう現象の対処法"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["IDE", "PhpStorm", "Goland", "IntelliJ", "IDEA"]
published: true
---

数ヶ月ほど前から Mac の PhpStorm や Goland で \ を入力すると直前の文字が消えてしまう現象に悩まされていましたが、対処法がわかったので共有します。

# 対処法

1. `Cmd + Shift + A` で Actions のダイアログを開く
1. registry と入力し、Registry を開く
   ![](https://storage.googleapis.com/zenn-user-upload/d7f0a859f485-20211205.png)
1. pressAndHold と入力し、 `ide.mac.pressAndHold.workaround` のチェックを外す
   ![](https://storage.googleapis.com/zenn-user-upload/7293777dd193-20211205.png)

# 対処法の見つけ方

[YouTrack](https://youtrack.jetbrains.com/issues) で backslash などで検索し関連しそうな Issue を見て回っていました。

今回の対処法が記載されていたのは [こちらの Issue](https://youtrack.jetbrains.com/issue/VIM-2352) でした。
