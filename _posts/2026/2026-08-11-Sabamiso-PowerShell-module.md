---
title: Sabamiso.psm - PowerShell用外部コマンド補完モジュール
date: 2026-08-11 19:00:00+09:00
modified_date: 2026-08-11 20:30:00+09:00
tags: PowerShell
toc: true
---

去年(2025年)末に [PowerShell で外部コマンドの補完] で PowerShell 7.6 から [Register-ArgumentCompleter] に `-NativeFallback` のパラメーターがされてコマンドライン補完のコードを包括的に行えるようになったことを書いた。

Powershell モジュールとして動くものができているので、ずいぶんを遅くなってしまったが、その報告。

- [Sabamiso.psm] ([PowerShell Gallery | Sabamiso.psm](https://www.powershellgallery.com/packages/Sabamiso.psm/))
- [Sabamiso.completions] ([PowerShell Gallery | Sabamiso.completions](https://www.powershellgallery.com/packages/Sabamiso.completions/))

以前は "NativeCommandCompleter.psm" というド直球な名前だったが改名した。
また、補完エンジンとなるコアのモジュールである [Sabamiso.psm] と、各コマンドの補完コードを集めた [Sabamiso.completions] に分離している。

## [Sabamiso.psm]

補完エンジンと補完コードの登録を行うコマンド群を提供するモジュールである。

<img src="https://raw.githubusercontent.com/teramako/Sabamiso.psm/refs/heads/main/docs/imgs/Sabamiso_x512.png" width="256" align="right">

大枠の仕組みは [PowerShell で外部コマンドの補完] に書いた時と変わっていない。

1. 補完開始
2. コマンドライン解析
3. コマンド指定がある
   - PowerShell コマンド(Cmdlet)の補完 (PowerShell本体の仕事)
   - コマンド名指定で補完コードが登録されている(`Register-ArgumentCompleter -CommandName ...` ) → その補完(登録されたスクリプト実行)
   - 上記2つに該当しない (← **本 Sabamiso.psm の仕事**)
     - 実行コマンド名を取得、コマンド名の補完定義が登録済み(キャッシュ済み)か確認
     - 登録済み
       - 該当の補完定義から補完を開始
     - 未登録
       - 環境変数 `PS_COMPLETE_PATH` にある各ディレクトリ(環境変数 `PS_COMPLETE_PATH` は未設定なら、`<$PROFILE のあるディレクトリ>/completions` が設定される)から
         `実行コマンド名.ps1` のスクリプトを探す。
       - 見つかったスクリプトを実行。
       - 再度補完定義が登録されたか確認。(登録されていれば、「登録済み」へ)
   - 上記を通して補完候補がない
     - ファイル・ディレクトリ名補完 (PowerShell本体の仕事)

上記のような仕組みで動いている。

bash, zsh, fish 等のUnix系シェルの `/usr/share/*/completions/`, `/etc/*/completions/` を真似していると言えば分かりやすいだろうか。

### 遅延読み込み
上記のような仕組みで補完発動時に動的に読むため、PowerShell起動時に各コマンド用の補完コードを読み込む必要がなくなり、起動時間が長くなる要因を削減できる。

また、後述するけれど `posh-git` のような既存の補完モジュールを再利用するかたちでの遅延読み込みも可能。

### 補完候補出力(menu-complete)には、その説明文が表示される

fish の補完で候補と共に説明が出てくるのが好きで、強くインスパイアしている。

![Complete ls parameters](/img/2026-08-11/complete-ls-parameters.png)

### Windows対応

Windows コマンドで `/` 始まりのパラメーターも補完できる。

![Complete whoami parameters](/img/2026-08-11/complete-whoami-on-windows.png)

また、上図のようにロケールに合わせてローカライズされた説明文も出せるようになっている。

### 特殊な形態のコマンド

少し工夫が必要だが、 `dd` コマンドのような `-` から始まらないような特殊なパラメーターにも対応できる。
![Complete dd parameters](/img/2026-08-11/complete-dd-parameters.png)


## [Sabamiso.completions]

私自身が使うために、かつ、サンプルとして定義している補完コード群。


## 補完コードのスクリプト

実際の補完コードがどんな風になるかを書いておこう。

私自身が使うために書いているコードも参考に: https://github.com/teramako/Sabamiso.completions/tree/main/completions

補完コードの役割は **「補完定義」を登録** することだ。
そのコマンドが [Register-NativeCompleter](https://github.com/teramako/Sabamiso.psm/blob/main/docs/Sabamiso.psm/Register-NativeCompleter.md) になる。

一度登録されると、解除されない限りそのスクリプトは実行されないことに注意。

```powershell
Register-NativeCompleter -Name "<コマンド名>" `
                         -Description "<コマンドの説明>" `
                         -Aliases @(別名) `
                         -Parameters @(パラメーター定義) `
                         -SubCommands @(サブコマンド定義) `
                         -Arguments @(コマンド引数定義) `
                         -Style 'スタイル' `
                         -NoFileCompletions
```
このコマンドで、定義と登録ができる。
定義のみを作る [New-CommandCompleter](https://github.com/teramako/Sabamiso.psm/blob/main/docs/Sabamiso.psm/New-CommandCompleter.md) もあるが、こちらは SubCommands のところで説明する。

- `-Name`: コマンド名。必須
- `-Description`: そのコマンドの説明。オプショナル
- `-Aliases`: 主にサブコマンド用。(`tmux list-sessions` = `tmux ls` のような別名を定義するもの)
- `-Parameters`: `-i` や `--ignore-case` 等のオプション定義群
- `-SubCommands`: `git branch ...` の `branch` 等のサブコマンド群
- `-Arguments`: コマンド引数の補完定義
- `-NoFileCompletions`: 引数の補完時、ファイルパスの補完を無効にする
- `-Style`: コマンドのスタイル (`GNU`(デフォルト), `Unix`, `Windows`) 
  - `GNU`: GNU系コマンドのスタイル。
  - `Unix`: `GNU`とほぼ同じだが、`--<ロングオプション>=<値>` の `=`区切り指定はできない
  - `Windows`: オプション接頭辞が `/` になり、引数指定も `/<オプション名>:<値>` と `:` 区切りになる

### Parameters

パラメーターの定義は [New-ParamCompleter](https://github.com/teramako/Sabamiso.psm/blob/main/docs/Sabamiso.psm/New-ParamCompleter.md) で行う。

```powershell
New-ParamCompleter -ShortName @(ショートオプション名) `
                   -StandardName @(通常オプション名) `
                   -LongName @(ロングオプション名) `
                   -Description "説明" `
                   -Arguments <引数補完定義> `
                   -Style <スタイル>
```

- `-ShortName`: Unix系コマンドのショートオプション。
  - 一文字であること。
  - `ls -lF` の `lF` のように複数組わせられる。
  - 引数を必要する場合、 `head -n10` のように、`<ショートオプション><値>` の入力が可能
  - 規定の接頭辞は `-`。
- `-StandardName`: 旧スタイルのUnix系コマンド(`find` コマンドとか)、または、Windowsによくあるオプション。
  - 規定の接頭辞は `-`。
- `-LongName`: いわゆるGNUロングオプション
  - 規定の接頭辞は `--`。
  - 引数を必要とする場合、 `--file=...`, `--file ...` と `=` または ` ` の区切りが可能
    - `grep --color=<when>` のような引数が無い場合にフラグとして作用するものの場合は、`=` のみが区切り
- `-Arguments`: 引数の定義
  - 省略すると、フラグのパラメーターとして作用する
  - 引数値をファイルパスなどの典型的な型、静的な候補リスト、動的にPowerShellスクリプトから補完する設定が可能
- `-Style`: パラメーター固有のスタイル
  - 基本的には親コマンド定義の`Style`を引き継ぐが、特殊な事情がある場合に指定。通常は使わない。

#### 例

`grep` コマンドから一部抜粋。
```powershell
Register-NativeCompleter -Name grep -Parameters @(
    New-ParamCompleter -ShortName i -LongName ignore-case -Description 'Ignore case'
    New-ParamCompleter -ShortName f -LongName file -Description 'Use patterns from a file' `
                       -Arguments @{ Name = 'FILE'; Type = 'File' }
    New-ParamCompleter -LongName color `
                       -Arguments @{ Name = 'WHEN'; Nargs = '?'; Candidates = "auto","always","never" }
)
```

### SubCommands

サブコマンドの定義は、通常のコマンド定義と同じ [New-CommandCompleter](https://github.com/teramako/Sabamiso.psm/blob/main/docs/Sabamiso.psm/New-CommandCompleter.md) で行う。

```powershell
New-CommandCompleter -Name "<サブコマンド名>" `
                     -Description "<コマンドの説明>" `
                     -Aliases @(別名) `
                     -Parameters @(パラメーター定義) `
                     -SubCommands @(サブコマンド定義) `
                     -Arguments @(コマンド引数定義) `
                     -Style 'スタイル' `
                     -NoFileCompletions
```
パラメーターは `Register-NativeCompleter` と同じ。

#### 例

```powershell
Register-NativeCompleter -Name git -SubCommands @(
    New-CommandCompleter -Name log -Parameters @(
        New-ParamCompleter -ShortName g -LongName graph
        New-ParamCompleter -ShortName p -LongName patch
        New-ParamCompleter -ShortName n -LongName max-count -Arguments @{ Name = 'NUM' }
        New-ParamCompleter -LongName 'stat'
        New-ParamCompleter -LongName 'since','after' -Arguments @{ Name = 'date' }
        New-ParamCompleter -LongName 'until','before' -Arguments @{ Name = 'date' }
        New-ParamCompleter -LongName 'author','committer' -Arguments @{ Name = 'pattern' }
    )
    New-CommandCompleter -Name add -Parameters @(
        New-ParamCompleter -ShortName n -LongName dry-run
        New-ParamCompleter -ShortName v -LongName verbose
        New-ParamCompleter -ShortName f -LongName force
        New-ParamCompleter -ShortName p -LongName patch
        New-ParamCompleter -ShortName A -LongName all
    )
)
```

### Arguments

引数の補完定義(`Register-ArgumentCompleter` や `New-ParamCompleter` の `-Arguments` パラメーターの記述方法)は、やり方が2通りある。

- [New-ArgumentCompleter](https://github.com/teramako/Sabamiso.psm/blob/main/docs/Sabamiso.psm/New-ArgumentCompleter.md) コマンドを使用する
- ただのハッシュテーブル(`@{ ...}`)を使用する

どちらを使用しても良いが、コマンドを使用すると LanguageServer の支援を受けやすいが、冗長になりがちだ。
個人的な使い分けは、`-Arguments` に直接書く時はハッシュテーブルで簡潔に。使いまわすために変数に入れる時はコマンドとしている。



定義だが、補完候補を生成しないものと生成するものに3種類の合計4種類ある。

#### 補完候補を生成しないもの

ただ引数が必要だと示すだけのもの。
特に引数が必須のオプションは `-Arguments` の指定を必須としているため、`@{ Name = 'value名' }` を各機会は多い。

```powershell
New-ParamCompleter ... -Arguments (New-ArgumentCompleter -Name NUM)
# or
New-ParamCompleter ... -Arguments @{ Name = 'NUM' }
```

#### 補完候補生成: ファイル/ディレクトリやコマンド補完

ファイル/ディレクトリのパスやコマンドを補完させるもの。`-Type` パラメーターを指定する。

```powershell
New-ParamCompleter ... -Arguments (New-ArgumentCompleter -Name file -Type File)
New-ParamCompleter ... -Arguments (New-ArgumentCompleter -Name dir -Type Directory)
New-ParamCompleter ... -Arguments (New-ArgumentCompleter -Name cmd -Type Command)
# or
New-ParamCompleter ... -Arguments @{ Name = 'file'; Type = 'File' }
New-ParamCompleter ... -Arguments @{ Name = 'dir'; Type = 'Directory' }
New-ParamCompleter ... -Arguments @{ Name = 'cmd'; Type = 'Command' }
```

他にも `DelegatingCommand` という特殊なものがある。(`sudo` や `time` のように引数がコマンドで、それ以降はそのコマンドの補完が必要となるケース用)


#### 補完候補生成: 静的なリスト

事前に候補が分かっている場合は、`-Candidates` パラメーターに列挙する。

```powershell
New-ParamCompleter ... -Arguments (New-ArgumentCompleter -Name color -Candidates 'always','auto','never')
# or
New-ParamCompleter ... -Arguments @{ Name = 'color'; Candidates = 'always','auto','never' }
```

#### 補完候補生成: 動的なスクリプト

PowerShell スクリプト(ScriptBlock) で補完候補を生成には `-Script` パラメーターを使用する。
```powershell
New-ParamCompleter ... -Arguments (
    New-ArgumentCompleter -Name GROUP -Script {
        param([string] $wordToComplete)
        Get-Content '/etc/group' | ForEach-Object {
            if ($_ -match '^([^:]+):') {
                $group = $Matches[1]
                if ($group -like "$wordToComplete*") { $group }
            }
        }
    }
)
# or
New-ParamCompleter ... -Arguments @{
    Name = 'color';
    Script = {
        param([string] $wordToComplete)
        Get-Content '/etc/group' | ForEach-Object {
            if ($_ -match '^([^:]+):') {
                $group = $Matches[1]
                if ($group -like "$wordToComplete*") { $group }
            }
        }
    }
}
```

### ローカライズ

https://github.com/teramako/Sabamiso.completions/tree/main/completions を見ると分かると思うが、`-Description` に指定する説明文を変数化している。

```powershell
$msg = data { ConvertFrom-StringData @'
    key1 = text1
    key2 = text2
    # ...
'@ }
Import-LocalizedData -BindingVariable localizedMessages -ErrorAction SilentlyContinue;
foreach ($key in $localizedMessages.Keys) { $msg[$key] = $localizedMessages[$key] }
```

説明文のローカライズを可能とするためだ。

`completions/ja` 等のロケール・ディレクトリを作って `コマンド名.psd1` を入れておくと読み込まれて、`$msg`ディクショナリの各値が上書きされる。

参考: https://github.com/teramako/Sabamiso.completions/blob/main/completions/ja/whoami.psd1


### 既存のコマンド補完を再利用する

https://github.com/teramako/Sabamiso.psm/blob/main/README.md#example-3-use-posh-gits-completion に記載しているが、
補完用のモジュールが既にあってそれを再利用することも可能だ。
しかも、必要になった時に読み込まれる遅延読み込みのおまけつきで。

例えば、git コマンドの補完は `posh-git` があり、これを使用したい場合、以下のようなコードを `<プロファイル・ディレクトリ>/completions/git.ps1` に記述することで実現できる。
プロファイル内で `posh-git` を無駄に読み込む必要もなくなる。

```powershell
<#
.SYNOPSIS
    Regsiter `git` command completer with `posh-git`
.DESCRIPTION
    This script will be loaded by `Sabamiso.psm` poershell module.
.LINK
    dahlbyk/posh-git: A PowerShell environment for Git
    https://github.com/dahlbyk/posh-git
#>
param($wordToComplete, $commandAst, $cursorPosition)
Import-Module posh-git

# Reset the variable in the global scope
$global:GitPromptScriptBlock = $GitPromptScriptBlock

# The first time, generate the completion list manually
TabExpansion2 -inputScript $commandAst.ToString().PadRight($cursorPosition) `
              -cursorColumn $cursorPosition `
    | Select-Object -ExpandProperty CompletionMatches
```

`Import-Module posh-git` でモジュールを読み込む。すると、`posh-git` 側で `Register-ArgumentCompleter -CommandName git ...` の補完が登録される。
(次回以降は)`Register-ArgumentCompleter -CommandName git ...`で登録された補完スクリプトが呼ばれることになる。
初回は `TabExpansion2` 関数を手動で呼んで補完させる。


コマンド自身が補完候補を返すような、たとえば `dotnet complete` のようなものの場合、`<プロファイル・ディレクトリ>/completions/dotnet.ps1` に以下のようなコードを書く。

```powershell
<#
.SYNOPSIS
    Regsiter `dotnet` command completer
.DESCRIPTION
    This script will be loaded by `Sabamiso.psm` poershell module.
.LINK
    How to enable tab completion for the .NET CLI
    https://learn.microsoft.com/en-us/dotnet/core/tools/enable-tab-autocomplete
#>
param($wordToComplete, $commandAst, $cursorPosition)

Register-ArgumentCompleter -Native -CommandName dotnet -ScriptBlock {
    param($wordToComplete, $commandAst, $cursorPosition)
    dotnet complete --position $cursorPosition $commandAst.ToString() | ForEach-Object {
        [System.Management.Automation.CompletionResult]::new($_, $_, 'ParameterValue', $_)
    }
}
# The first time, generate the completion list manually
TabExpansion2 -inputScript $commandAst.ToString().PadRight($cursorPosition) `
              -cursorColumn $cursorPosition `
    | Select-Object -ExpandProperty CompletionMatches
```

[PowerShell で外部コマンドの補完]: /2025/12/16/PowerShell-NativeCommandCompletion
[Register-ArgumentCompleter]: https://learn.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/register-argumentcompleter?view=powershell-7.6 "Register-ArgumentCompleter (Microsoft.PowerShell.Core) - PowerShell | Microsoft Learn"
[Sabamiso.psm]: https://github.com/teramako/Sabamiso.psm "teramako/Sabamiso.psm: PowerShell module for complete native command parameters and arguments"
[Sabamiso.completions]: https://github.com/teramako/Sabamiso.completions "teramako/Sabamiso.completions: Completios definitions for Sabamiso.psm"
