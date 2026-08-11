---
title: PowerShell で実行結果を自動的に変数に収める
date: 2026-08-07 22:10:00+09:00
tags: PowerShell
---

昔、どこかで見たTips。

```powershell
$PSDefaultParameterValues["Out-Default:OutVariable"] = "LAST"
```

上記コードを `$PROFILE` に仕込んでおくと、コマンド実行結果が `$LAST` 変数に入る。

```powershell
PS1> Get-ChildItem

    Directory: ....

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d----          2022/05/24    17:12                *********
...

PS1> $LAST

    Directory: ....

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d----          2022/05/24    17:12                *********
...

PS1>
```

直前のコマンド実行結果を再度確認したい時や、加工したい場合に便利。

## 仕組み

### Out-Default コマンドレット

- [Out-Default]

PowerShell はコマンドラインをAST解析して、各コマンドをまとめたパイプラインにまとめる。
このパイプラインの最後のコマンドがFormat系(`Format-List`や`Format-Table`)でない場合に、暗黙的に`Out-Default`を末尾に追加する動きをしている。

- `Get-ChildItem` → `Get-ChildItem | Out-Default`
- `Get-ChildItem | Select-Object ...` → `Get-ChildItem | Select-Object ... | Out-Default`
- `Get-ChildItem | Format-List ...` → `Get-ChildItem | Format-List ...` (末尾が`Format-*` なので`Out-Default` は追加されない)

### `-OutVariable` パラメーター

- [about_CommonpPrameters]

また、すべてのコマンドレットには、出力結果を引数に指定した変数名の変数に収める `-OutVariable` というパラメーターを指定できる。
パイプラインに出力するオブジェクトを出力と同時に変数に収める。

変数は `ArrayList` 型で、出力が単一であっても配列になる点に注意。

### `$PSDefaultParameterValues` 変数

- [about_Parameters_Default_Values]

`$PSDefaultParameterValues` のディクショナリのキーに `<コマンド名>:<パラメーター名>`、値にそのパラメーターの値を入れるとパラメーターの規定値を設定できる。

---

これらを組み合わせる。
```powershell
$PSDefaultParameterValues["Out-Default:OutVariable"] = "LAST"
```

すると、`Get-ChildItem ...` は `Get-ChildItem | Out-Default -OutVariable LAST` に変換される。

## 使い道

### フィルタリングする

```powershell
PS1> Get-ChildItem
...
PS1> $LAST | Where-Object Length -gt 1MB
```

### 直前の結果をログに残す

```powershell
PS1> Get-ChildItem
...
PS1> $LAST | ConvertTo-Json | Set-Content file-list.json
```


[Out-Default]: https://learn.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/out-default?view=powershell-7.6 "Out-Default (Microsoft.PowerShell.Core) - PowerShell | Microsoft Learn"
[about_Parameters_Default_Values]: https://learn.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/about/about_parameters_default_values?view=powershell-7.6 "about_Parameters_Default_Values - PowerShell | Microsoft Learn"
[about_CommonpPrameters]: https://learn.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/about/about_commonparameters?view=powershell-7.6#-outvariable "-OutVariable - 共通パラメーターについて - PowerShell | Microsoft Learn"
