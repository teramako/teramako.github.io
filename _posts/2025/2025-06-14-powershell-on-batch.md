---
title: bat ファイルだけど中身は PowerShell なスクリプト
date: 2025-06-14T21:00:00+09:00
tags: PowerShell
---

単体 PowerShell スクリプト (.ps1) なファイルの実行は少々面倒だ。
エクスプローラーからダブルクリックしても実行できない（メモ帳で開かれる）し、
`ExecutionPolicy` の設定が未設定または不適格で実行が阻害されたりする。

実行用のバッチファイルを用意する手もあるけど、単一ファイルの方が嬉しいこともある。

ということで、なんか方法がないかなと vim-jp Slack で訊いたら教えてくれた。

- [ヘッダー追加する vim plugin (denops.vim で！)](https://zenn.dev/yukimemi/articles/2021-03-08-dps-ahdr-denops-vim)

> ```batch
> @set __SCRIPTPATH=%~f0&@powershell -NoProfile -ExecutionPolicy ByPass -InputFormat None "$s=[scriptblock]::create((gc -enc utf8 -li \"%~f0\"|?{$_.readcount -gt 2})-join\"`n\");&$s" %*
> @exit /b %errorlevel%
> Write-Host "Hello world !"
> ```

少々複雑だけど、やっていることは割と単純。

ファイル内の3行目以降を抽出した `ScriptBlock` を生成して実行している。

## ちょっと工夫

通常のPowerShellスクリプトだと `$PSScriptRoot` で実行スクリプトが入っているディレクトリを得られるが、上記だとファイル実行にならないために `$PSScriptRoot` が空になる。
実行ディレクトリが得られると嬉しいことが多いので、少し修正。

```batch
@set __SCRIPTPATH=%~f0
@set __SCRIPTDIR=%~dp0
@powershell -NoProfile -ExecutionPolicy ByPass "$s=[ScriptBlock]::Create((gc -enc utf8 -li \"%~f0\"|select -skip 4) -join \"`n\");&$s" %*
@exit /b %ErrorLevel%

Write-Host $env:__SCRIPTPATH
Write-Host $env:__SCRIPTDIR
```


## 昔の……

bat ファイルだけど中身は JScript、なんてのができたわけだけど、
この PowerShell 版として使えそうだ。

```batch
@if(0)==(0) ECHO OFF
CScript //NoLogo //E:JScript "%~f0" %*
GOTO :EOF
@end

Wscript.Echo("Hello")
```

