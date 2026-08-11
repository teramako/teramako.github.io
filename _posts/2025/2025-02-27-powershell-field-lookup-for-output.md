---
title: PowerShell フィールドの値出力に ToString() が呼ばれるとは限らない
date: 2025-02-27 00:51:00+09:00
tags: PowerShell
toc: true
---

## 問題.1
以下のPowerShellコードはどのような出力になるか？

```powershell
class DataBase {
    [int] $Id;
    [string] $Name;
    [string] ToString() {
        return "{0}:{1}" -f $this.Id, $this.Name
    }
}

$dict = [ordered]@{
    Data = [DataBase]@{ Id = 1; Name = "A" };
    List = [DataBase]@{ Id = 2; Name = "B" }, [DataBase]@{ Id = 3; Name = "C" };
}
$dict
```

### 答え
```
Name                           Value
----                           -----
Data                           1:A
List                           {2:B, 3:C}
```

それぞれ、`ToString()` された結果が Value 列に出る。
配列(というかEnumerableなオブジェクトの場合)は `,`区切りで `ToString()` した結果を `{`,`}` で囲まれて出力される。

## 問題.2
問題.1 の `DataBase` クラスを継承するクラスを作って出力させた場合は？
```powershell
class ExData : DataBase {
}

$dict2 = [ordered]@{
    Data = [ExData]@{ Id = 1; Name = "A" };
    List = [ExData]@{ Id = 2; Name = "B" }, [ExData]@{ Id = 3; Name = "C" };
}
$dict2
```

### 答え

```
Name                           Value
----                           -----
Data                           1:A
List                           {B, C}
```

Data は想定通り、`ToString` された結果。
しかし、List の配列の場合は異なり、`Name` プロパティが抽出されているように見える。

## 何故？

結論から言うと、特殊な条件下では特定のプロパティ名に合致したプロパティの値を出している。

- 対象のオブジェクトが Enumerable であること
- その中身のオブジェクトの型に直接 `ToString()` メソッドが宣言されていないこと
- **特定パターンに合致する名前のプロパティ**を持っていること

より詳細な条件は PowerShell のコードを読み切れていないので上記は完全ではないと思うけど、だいたい合っているはず。

### 『特定パターンに合致する名前のプロパティ』について

https://github.com/PowerShell/PowerShell/blob/c64a636d9bdcae3d00aab551685d87b48cc35e30/src/System.Management.Automation/FormatAndOutput/common/Utilities/MshObjectUtil.cs#L61-L65
```csharp
        internal static PSPropertyExpression GetDisplayNameExpression(PSObject target, PSPropertyExpressionFactory expressionFactory)
        {
            // first try to get the expression from the object (types.ps1xml data)
            PSPropertyExpression expressionFromObject = GetDefaultNameExpression(target);
            if (expressionFromObject != null)
            {
                return expressionFromObject;
            }

            // we failed the default display name, let's try some well known names
            // trying to get something potentially useful
            string[] knownPatterns = new string[] {
                "name", "id", "key", "*key", "*name", "*id",
            };

            // 省略
        }
```
該当する PowerShell 内部のコードは上記になる。

`name`, `id`, `key`, `*key`, `*name`, `*id` の順でプロパティ名を検索(Case-Insensitive, `*` はワイルドカード)して行き、最初に見つかったプロパティとなる。

[問題.2](#section-2) では、 `Name`, `Id` プロパティを持っていて、最初に見つかった `Name` プロパティを抽出しているわけだ。

### 抽出プロパティの指定
上記 C# コードを見ると、`knownPatterns` での検索の前に、さらに特定のプロパティ名を算出している。

> // first try to get the expression from the object (types.ps1xml data)

の部分だ。

PowerShell では、DotNet オブジェクトを `types.ps1xml` で拡張して特殊な情報を付与したり、プロパティやメソッドを追加できるようになっている。
その `types.ps1xml` で抽出するプロパティ名を指定できる。

※: `types.ps1xml` の概要については、 [about_Types.ps1xml] 辺りを参照

[about_Types.ps1xml]: https://learn.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/about/about_types.ps1xml "about_Types.ps1xml - PowerShell | Microsoft Learn"

以下のような感じだ。
```xml
<?xml version="1.0" encoding="utf-8"?>
<Types xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/PowerShell/PowerShell/master/src/Schemas/Types.xsd">
  <Type>
    <Name>DataBase</Name><!-- クラスの名前(名前空間付きで) -->
    <Members>
      <MemberSet>
        <Name>PSStandardMembers</Name><!-- (たぶん)固定値 -->
        <Members>
          <NoteProperty>
            <Name>DefaultDisplayProperty</Name><!-- 固定値 -->
            <Value>Id</Value><!-- 抽出させるプロパティ名 -->
          </NoteProperty>
        </Members>
      </MemberSet>
    </Members>
  </Type>
</Types>
```

ちなみに、存在しないプロパティ名を記載すると、抽出するプロパティが無いと見なされてフォールバック的に `ToString()` が呼ばれる。
プロパティの抽出が気に入らず `ToString()` させたい場合はこれで回避することも可能だと思う。
（副作用があるかもしれないので、おススメはしない）


#### 応用
抽出させたいプロパティだけでは不足がある等でカスタマイズしたい場合、以下のように `ScriptProperty` で Getter を追加して、それを指し示すと良さそう。

というか、やってみたら出来た 😄

`IsHidden` 属性を付けて隠しプロパティにしておくとより良い。

```xml
<Types>
  <Type>
    <Name>DataBase</Name>
    <Members>
      <MemberSet>
        <Name>PSStandardMembers</Name>
        <Members>
          <NoteProperty>
            <Name>DefaultDisplayProperty</Name>
            <Value>__displayProperty__</Value><!-- 抽出させるプロパティ名 -->
          </NoteProperty>
        </Members>
      </MemberSet>
      <ScriptProperty IsHidden="true"><!-- 隠しGetterを追加 -->
        <Name>__displayProperty__</Name>
        <GetScriptBlock>"[{0}] {1}" -f $this.Id, $this.Name</GetScriptBlock>
      </ScriptProperty>
    </Members>
  </Type>
</Types>
```

## 最後に
PowerShell 難しい……
