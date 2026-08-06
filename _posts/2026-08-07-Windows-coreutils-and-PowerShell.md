---
title: Windows coreutils と PowerShell
date: 2026-08-07 00:03:00+09:00
tags: PowerShell
---

## core-utils

Unix系のコマンドが使えるようになる [core-utils] がMicrosoftからリリースされている。
(参考: [ニュースにもなった](https://forest.watch.impress.co.jp/docs/news/2113918.html "「Coreutils for Windows」が一般提供 ～Linuxなどの定番コマンドをWindowsでネイティブ実行 - 窓の杜"))

- リポジトリ: https://github.com/microsoft/coreutils


普通にインストールすると、`C:\Program Files\coreutils` にインストールされる。
`C:\Program Files\coreutils\bin` に各exeファイルが入っている。
各exeファイルすべてが 8MB 近くあるように見えるが、ハードリンクで構成されているため実態は小さい。

ちょっと面白いのは `coreutils-manager.exe` があって、このコマンドで使うコマンドの管理(ハードリンクの作成(enable)/削除(disable))ができる。

```console
> coreutils-manager disable ls
```
とすると `ls.exe` が消える。有効にするなら `coreutils-manager enable ls` とすれば良い。
`coreutils-manager status` で状態一覧も見られる。

まぁこれ自体は良い。
Microsoft自身がネイティブで動くコマンドを用意してくれて喜ばしいことだ。

## お行儀が悪い

ただ、かなりお行儀が悪いことをしてくれる。

バージョン 2026.6.16 時点での話だが、インストールすると勝手にPowerShellプロファイルにコードを仕込むのだ。以下のコードが追加される。
https://github.com/microsoft/coreutils/blob/a350b9cf5c9c178b1089902cde89275688862e32/src/pwsh-install-template.ps1

コマンドライン実行時に呼ばれる関数 `PSConsoleHostReadLine` を上書きして、coreutilsで有効になっているコマンド名のものがあったら、coreutilsのフルパスに変換してしまうようなことをしている。

PowerShell の動きとしてはコマンド部分をエイリアス、関数、コマンドレットから検索し、見つからなかったら PATH環境変数から外部コマンドを検索するが、フルパスに置き換えられるため、これらの検索は行われずにダイレクトに core-utils のコマンドが実行されることになる。
`ls` や `cat` など、PowerShell 自体がエイリアスとして `Get-ChildItem` や `Get-Content` になっていてもガン無視される。

### 問題点

- 何の注意喚起もない
- 訓練されたPowerShellユーザーは `ls` は `Get-ChildItem` のエイリアスになっていて、外部コマンドよりも優先度が高いことを知っている。 `Get-ChildItem` のつもりで `ls` を実行して混乱することになる。
- `ls` のパラメーターをタブ補完しようとすると、PowerShell本体は `ls` を `Get-ChildItem` と認識しているので core-utilsの `ls.exe` ではなく `Get-ChildItem` のパラメーターを補完するになり不整合が生じる

### インストール時と言ったが、それは嘘だ

正確には、冒頭で挙げた `coreutils-manager.exe` でコマンドの有効化/無効化を行ってもプロファイルの仕込みが走る。

### 無効化する術が分からない

インストール/アップデート時のみ気を付けておけば良いとは限らないことが分かったので、本格的に無効化する（書き換えが走っても実質スキップされるような）術を探った。

が、結論としては分からなかったorz

プロファイルには以下のようなヘッダー、フッターが付いている。
ここだけ残せば書き換えがコードの仕込みを抑制できるのでは？ → ダメだった。

```powershell
# DO NOT MODIFY -- coreutils -- 60b36fc6-2d59-49df-be51-28dd2f4c3c9a
# vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv
...
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
# DO NOT MODIFY -- coreutils -- 60b36fc6-2d59-49df-be51-28dd2f4c3c9a
```

途中の `function PSConsoleHostReadLine { ... }` を実質何もしないコードにしておく案は？ → ダメだった。
```powershell
function PSConsoleHostReadLine {
    param()
    return [Microsoft.PowerShell.PSConsoleReadLine]::ReadLine($host.Runspace, $ExecutionContext, $?)
}
```


### Issue になっている

- https://github.com/microsoft/coreutils/issues/161

まぁ当然のようにIssueに登録されている。

修正されるのを待つしかない。


[core-utils]: https://learn.microsoft.com/ja-jp/windows/core-utils/overview "Windowsの Coreutils の概要 | Microsoft Learn"
