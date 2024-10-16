---
title: WSL で git commit にGPG署名をする
date: 2024-10-16 15:00:00+09:00
tags: git
toc: true
---

Windows (WSL) で Neovim を使っている。

[gin.vim] を通して git 操作をしてるが、 Github の Verified 表記 (![](/img/2024-10-16/github-verified.png)) に憧れて署名を付ける設定を施した。  
しかし、Neovim からGPG 署名を付けようと設定したところ、`git commit` のパスフレーズ入力時にターミナルの表示がおかしくなる事象が発生。

![broken-terminal](/img/2024-10-16/broken-terminal.png)

詳しいことは分かっていないが、 GnuPG では [pinentry] というものを使ってパスフレーズ等の入力をやりとりしている模様。  
ターミナルのエスケープをゴリゴリやってそうで Vim などとは相性が悪そうだ。

MacOS なら pinentry-mac、Linux にも gt や gtk でGUIを出すプログラムがあるみたいだが、Windows は……？

Windows というか WSL で使えるのがないか探したところ [diablodale/pinentry-wsl-ps1] を発見。

## [diablodale/pinentry-wsl-ps1]

中身は一つの bash スクリプトで、ファイル名から推察できるが内部で Windows 側の PowerShell を呼び出している。  
このスクリプトを GnuPG が実行する pinetry プログラムとして指定するかたちになる。

## 設定

### スクリプト設定

1. テキトウなところに  
   `git clone https://github.com/diablodale/pinentry-wsl-ps1.git`
2. 実行権限の付与  
   `chmod ug=rx pinentry-wsl-ps1.sh`
3. パスが通ったところにシンボリックリンクを作成  
   `ln -s pinentry-wsl-ps1.sh ~/bin/pinetry-wsl-ps1.sh`
4. Windows `powershell.exe` にもパスを通す  
   `ln -s /mnt/c/Windows/SysWOW64/WindowsPowerShell/v1.0/powershell.exe ~/bin/powershell.exe`

#### 実行テスト

```console
$ echo "GETPIN" | pinentry-wsl-ps1.sh
OK Your orders please
```

GUIのフォームが出てくることを確認。  
![](/img/2024-10-16/GPG-key-prompt.png)

### GnuPG 設定

`~/.gnupg/gpg-agent.conf` を編集し（存在しないなら新規作成）以下を追加

```conf
pinentry-program ~/bin/pinentry-wsl-ps1.sh
```

あと、最後に `pkill gpg-agent` でGPGエージェントを停止させた。(本当に必要だったか不明)

[gin.vim]: https://github.com/lambdalisue/vim-gin "GitHub - lambdalisue/vim-gin: 🥃 Gin makes you drunk on Git"
[pinentry]: https://www.gnupg.org/software/pinentry/index.html
[diablodale/pinentry-wsl-ps1]: https://github.com/diablodale/pinentry-wsl-ps1/tree/master "diablodale/pinentry-wsl-ps1: GUI for GPG within Windows WSL for passwords, pinentry, etc."