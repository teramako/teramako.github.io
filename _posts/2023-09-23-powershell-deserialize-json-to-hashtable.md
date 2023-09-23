---
title: PowerShell v5.1 (Desktop) で JSON デシリアライズする
date: 2023-09-23 18:30:00+09:00
modified_date: 2023-09-23 18:37:00+09:00
tags: PowerShell
mermaid: true
---

最近 PowerShell モジュールを書いているが、REST API から JSON を取得するのに、[ちょっとしたユーティリティ][JsonDeserializer]を作った話。

使用対象環境に Windows標準のDesctop Edition v5.1 が含まれているが、ここで問題が発生した。

JSON文字列をオブジェクトに変換するのに `ConvertFrom-Json` を使用するが、APIから返ってくる値のキーに `name` と `NAME` のような大文字/小文字のみで区別されたものがあるのだ。
JSONとしては問題なくても、PowerShell は基本的に大文字/小文字を区別しないため、そこで齟齬が発生する。

```powershell
'{ "name": "a", "NAME": "A" }' | ConvertFrom-Json
```
```
ConvertFrom-Json : JSON 文字列を変換できません。この文字列から変換されたディクショナリでキー 'unko' と 'UNKO' が重複します。
発生場所 行:1 文字:36
+ '{ "unko": "💩", "UNKO": "💩" }' | ConvertFrom-Json
+                                    ~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (:) [ConvertFrom-Json]、InvalidOperationException
    + FullyQualifiedErrorId : DuplicateKeysInJsonString,Microsoft.PowerShell.Commands.ConvertFromJsonCommand

```

PowerShell Core v7 系を使用している場合は、[`ConvertFrom-Json`][ConvertFrom-Json-v7] に `-AsHashtable` パラメーターがあり、これで問題を解決できる。
オブジェクトへの変換結果を `PSCustomObject` でなく `Hashtable`(`psobject`) にして得ることができるのだ。

しかし、残念ながら Windows標準のPowerShell Desktop v5.1 では使えない。
仕方ないので自作した。
「自作」と言っても、ただ単に .NET のライブラリを使用しただけだが。

- [teramako/JsonDeserializer: Deserialize JSON string to Hashtable on PowerShell v5.1 Desktop][JsonDeserializer]

## 基本

[JavaScriptSerializer Class (System.Web.Script.Serialization) | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.web.script.serialization.javascriptserializer?view=netframework-4.8.1) を使用する。
```powershell
Add-Type -AssemblyName System.Web.Extensions
$JsonSerializer = [System.Web.Script.Serialization.JavaScriptSerializer]::new()
$obj = $JsonSerializer.DeserializeObject($str)
```

ここで返ってくる値は、`System.Collections.Generic.Dictionary[string, object]` か `object[]` だ。
PowerShell から少し扱いにくいし、Core v7系の`ConvertFrom-Json -AsHashtable` と結果が異なってしまう。

`Hashtable` へ変換する必要がある。

## Hashtableへ変換

単純には `$JsonSerializer.Deserialize($str, [hashtable])` または、キャストで `[hashtable]$JsonSerializer.DeserializeObject($str)` する手もある。
しかし、内部にさらにオブジェクトがある場合、そのオブジェクトは未変換のままだ。
再帰的に `Hashtable` へ変えていく必要がある。

```powershell
function ConvertFrom-Dictionary ([IDictionary] $dict) {
    $resultHash = [ordered]@{}
    foreach ($entry in $dict.GetEnumerator()) {
        if ($entry.Value -is [array]) {
            $resultHash.Add($entry.Key, (ConvertFrom-Array $entry.Value))
        } elseif ($entry.Value -is [IDictionary]) {
            $resultHash.Add($entry.Key, (ConvertFrom-Dictionary $entry.Value))
        } else {
            $resultHash.Add($entry.Key, $entry.Value)
        }
    }
    return $resultHash
}
function ConvertFrom-Array ([array] $array) {
    $length = $array.Count
    $resultArray = New-Object Object[] -ArgumentList $length
    for ($index = 0; $index -lt $length; $index++) {
        $item = $array[$index]
        if ($item -is [array]) {
            $resultArray[$index] = ConvertFrom-Array $item
        } elseif ($item -is [IDictionary]) {
            $resultArray[$index] = ConvertFrom-Dictionary $item
        } else {
            $resultArray[$index] = $item
        }
    }
    return $resultArray
}

$obj = $JsonSerializer.DeserializeObject($str.ToString())
if ($obj -is [array]) {
    return ConvertFrom-Array $obj
} elseif ($obj -is [IDictionary]) {
    return ConvertFrom-Dictionary $obj
}
```

## Core v7 では使用できない
Core v7には `ConvertFrom-Json -AsHashtable` があるため、上記を使う必要はない。
というか、 `Add-Type -AssemblyName System.Web.Extensions` で読み込んでいる `Sytem.Web.Extenions` が Core には存在しない。
エラーになってしまう。
代わりに、[JsonConvert Class](https://www.newtonsoft.com/json/help/html/T_Newtonsoft_Json_JsonConvert.htm) があるようだ。

ともかく、Desktop Edition, Core Edition 両方で使用したい場合は、条件分岐を入れておいた方が良さそう。
```powershell
if ($PSEdition -eq "Desktop") {
    # Windows Desktop Edition
    # ...
} else {
    # Core Edition
    # ...
}
```

## 余談

Desktop Edition, Core Edition の `ConvertFrom-Json`、自作した `System.Web.Script.Serialization.JavaScriptSerializer` のコードすべてに言えることだが、不正な JSON 文字列であってもデシリアライズできる。

```console
PS> @"
> { invalid_1: "invalid", "invalid_2": 'invalid', 'invalid_3': "invalid" }
> "@ | ConvertFrom-Json

invalid_1 invalid_2 invalid_3
--------- --------- ---------
invalid   invalid   invalid

```
1. キーにあたる部分を `"` で括らない
2. キーを `'` で括る
3. 値を `'` で括る

すべて問題なく動いてしまう。
これは良いのか？

[JsonDeserializer]: https://github.com/teramako/JsonDeserializer "teramako/JsonDeserializer: Deserialize JSON string to Hashtable on PowerShell v5.1 Desktop"
[ConvertFrom-Json-v7]: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json?view=powershell-7.3 "ConvertFrom-Json (Microsoft.PowerShell.Utility) - PowerShell | Microsoft Learn"
