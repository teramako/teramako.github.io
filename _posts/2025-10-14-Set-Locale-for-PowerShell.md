---
title: PowerShell のロケール設定
date: 2025-10-14T22:30:00+09:00
tags: PowerShell
---

Windows 標準で入っている PowerShell 5.1 ではなく、PowerShell 7 の話。

PowerShell モジュールを開発していてヘルプを英語で出したり日本語に切り替えたりしたくなった。

ヘルプの言語環境は `UICulture` が関わってくる。この値を設定し直せれば切り替え可能。

## 結論

- `[System.Threading.Thread]::CurrentThread.CurrentUICulture` を設定せよ。

## LANG 環境変数
```bash
LANG=ja_JP.UTF-8 pwsh
```
LANG=環境変数は有効に働くみたい。
ただし、pwsh プロセスの再起動が必要。

## `$PSUICulture`
PowerShell の自動変数に、`$PSCulture` と `$PSUICulture` がある。
しかし、この変数は読取専用であり、ここに新たな値を設定することができない。

```console
PS> $PSCulture = "ja-JP"
WriteError: Cannot overwrite variable PSCulture because it is read-only or constant.
```

## `[cultureinfo]::CurrentUICulture`
こちらは設定可能で、有効に働いた。

```powershell
[cultureinfo]::CurrentUICulture = "ja-JP"

Import-Module MyModule -Force
Get-Help Test-Cmd
```

既に読み込んでいるモジュールは再読み込みさせないと切り替わらない。

あと、`$PSUICulture` に反映されない点がなんだか気持ち悪い。

## `[System.Threading.Thread]::CurrentThread.CurrentUICulture`
現行スレッドのオブジェクトにも UICulture を設定できるようだったのでやってみたら、当りだった。

```console
PS> [System.Threading.Thread]::CurrentThread.CurrentUICulture = "ja-JP"
PS> $PSUICulture
ja-JP

PS> Import-Module MyModule -Force
PS> Get-Help Test-Cmd
```
