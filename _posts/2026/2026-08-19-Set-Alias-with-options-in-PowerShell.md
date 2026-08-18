---
title: PowerShell でオプション付きエイリアスを設定する
date: 2026-08-19 02:00:00+09:00
tags: PowerShell
---

はっきり言って PowerShell のエイリアスは使いにくい。
コマンド・オプション付きのエイリアスを作れないところが特に。

```powershell
Set-Alias ls 'ls --color -F'
```
とかができない。

ということで、多少制限があるものの可能にするものを作った。

- https://github.com/teramako/PSApplicationAlias

## 使用方法

プロファイルに dot-source でスクリプトを読み込む形で使う。

> ```powershell
> . $PSScriptRoot\AppAlias.ps1
> ```

読み込むと、`Set-ApplicationAlias` というコマンド（関数）が使えるようになる。

> ```powershell
> # Remove built-in alias (AllScope) before redefining
> Remove-Item Alias:ls, Alias:cat
>
> Set-ApplicationAlias cat D:\Program\Git\usr\bin\cat.exe
>
> # If the command is a path that contains spaces, set it in the format `& "Command Name" ...`.
> Set-ApplicationAlias ls '& "C:\Program Files\coreutils\bin\ls.exe" --color -F'
> ```

定義済みのエイリアス削除して(※)、`Set-ApplicationAlias` でエイリアス設定をする。

※: PowerShell の組み込み alias（`ls`, `cat`, `dir` など）は AllScope で定義されていて、対話セッション中に Set-Alias で上書きできない。PROFILE 内で Remove-Item しておく必要がある。

## 内部動作

`Set-ApplicationAlias` 内部では PowerShell 本体の `Set-Alias` を呼んでエイリアス定義をしているが、Description プロパティに JSON 形式でコマンドパスと引数リストを埋め込んでいる。
そして、実行先の `Invoke-AliasApplication` という(dot-source で読み込んだスクリプト内で定義している)関数を固定的に設置している。

```powershell
Set-Alias -Name $Name -Value Invoke-AliasApplication -Description $json -Scope Global
```

```console
> Get-Alias -Definition Invoke-AliasApplication | select CommandType,DisplayName,Description

CommandType DisplayName                    Description
----------- -----------                    -----------
      Alias cat -> Invoke-AliasApplication {"Command":"D:\\Program\\Git\\usr\\bin\\cat.exe","Args":{}}
      Alias ls -> Invoke-AliasApplication  {"Command":"C:\\Program Files\\coreutils\\bin\\ls.exe","Args":["--color","-F"]}

```

※: `Get-ApplicationAlias` で上記と同等の結果が得られるようなコマンドもあるよ

`cat` や `ls` を実行すると、`Invoke-AliasApplication` が呼ばれる。

`Invoke-AliasApplication` 内では、実行元となったエイリアス名を取得し、そのエイリアスのDescriptionプロパティからコマンドパスと引数リストを得る。
あとは、パイプライン入力値と共にコマンドに引数を付けて実行する。

コード自体は、割と単純だ。

```powershell
function Invoke-AliasApplication {
    begin {
        $aliasName = $MyInvocation.InvocationName
        if ($aliasName -eq $MyInvocation.MyCommand.Name) {
            throw "Don't call this function directly."
        }

        $alias = Get-Alias $aliasName
        $info = $alias.Description | ConvertFrom-Json
        $cmdName = $info.Command
        $cmdArgs = $info.Args

        if (-not $cmdName) {
            throw ("Description of '{0}' may be incorrect: {1}" -f $aliasName, $alias.Description);
        }
    }
    end {
        $input | & $cmdName @cmdArgs @args
    }
}
```


ポイントは、実行元となったエイリアス名の取得するところ。
```powershell
$aliasName = $MyInvocation.InvocationName
```

元々、`$MyInvocation.InvocationName` で Unix系の `$0` に似ているものが取れて面白いことができそうだと目を付けていたのが今回役に立った。

## 制限: バイナリ入出力不可

- バイナリデータが欲しい時には、この方法でのエイリアス設定はしない方が良い。

まず、PowerShell 本体の動きの話から。

通常の Cmdlet や関数のパイプライン入出力は、その型のオブジェクトになる。
対して、`外部コマンド1 | 外部コマンド2` のような外部コマンドのパイプ間や、`外部コマンド > file` のような外部コマンドのリダイレクトでは、外部コマンド同士の世界に閉じられるためパイプライン入出力がバイナリ・ストリーム直で行われるようになっている。
そして、外部コマンドと PowerShell のCmdletや関数、式が混在すると、バイナリ・ストリームは文字列(`string[]`)化されて PowerShell とのやり取りが行われる。

で、エイリアス設定すると PowerShell の関数でラップされるので、`外部コマンド1 | 外部コマンド2` のように見えてもバイナリ・ストリームでの入出力が行われず、文字列での入出力になってしまう。

文字コードや改行コードが変わったりするため、以下のような違いが出てしまう。
```console
> Get-Command cat

CommandType     Name                                               Version    Source
-----------     ----                                               -------    ------
Alias           cat -> Invoke-AliasApplication

> sha1sum .\shift_jis.txt
46f0ea63e867da29af52457ea9e465af03dbb60b  ./shift_jis.txt
> D:\Program\Git\usr\bin\cat.exe .\shift_jis.txt | sha1sum
46f0ea63e867da29af52457ea9e465af03dbb60b  -
> cat .\shift_jis.txt | sha1sum
471399aeda50586bdc982b558bfde1da642c0fb3  -
```
`cat` を ApplicationAlias でラップすると PowerShell の関数として実行されるため、 **外部コマンド間のバイナリストリームではなく、文字列化されたパイプラインが流れる**。その結果、文字コード変換や改行変換が入り、ハッシュ値が変わってしまう。


## 制限: Cmdletには使えない

- 外部コマンドのエイリアス設定にしか使えない

Cmdlet のパラメーターは `string[]` な配列ではなく、ディクショナリ(`Hashtable`) で設定する必要があったり、パラメーター値も文字列とは限らないなど、複雑なので今回は外部コマンドのみに限定した。

Cmdlet の場合のパラメーター値設定は `$PSDefaultParameterValues` ([about_Parameters_Default_Values - PowerShell | Microsoft Learn](https://learn.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/about/about_parameters_default_values?view=powershell-7.6)) を使った方が良いだろうし。

