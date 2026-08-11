---
title: PowerShell types.ps1xml の GetScriptBlock に型を指定する
date: 2025-03-03T23:32:00+09:00
tags: PowerShell
---

PowerShell モジュール等で書く [types.ps1xml][about_Types.ps1xml] において、
既存のオブジェクトにプロパティを追加したい場合、以下のように `<ScriptProperty>` に `<GetScriptBlock>` を書く。

```xml
<?xml version="1.0" encoding="utf-8"?>
<Types xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/PowerShell/PowerShell/master/src/Schemas/Types.xsd">
  <Type>
    <Name>Namespace.TypeName</Name>
    <Members>
      <MemberSet>
        <Name>PSStandardMembers</Name>
        <Members>
          <PropertySet>
            <Name>DefaultDisplayPropertySet</Name>
            <ReferencedProperties>
              <Name>ExtendedProp</Name>
            </ReferencedProperties>
          </PropertySet>
        </Members>
      </MemberSet>
      <ScriptProperty>
        <Name>ExtendedProp</Name>
        <GetScriptBlock>$this.Name ...</GetScriptBlock>
      </ScriptProperty>
    </Members>
  </Type>
</Types>
```

しかし、このオブジェクトの `Get-Member` 等でプロパティの情報を見ると型が `System.Object` で表示される。

```console
PS1> $obj | Get-Member

   TypeName: Namespace.TypeName

Name          MemberType     Definition
----          ----------     ----------
...
ExtendedProp  ScriptProperty System.Object ExtendedProp {get=$this.Name ...}
...
```

これがちょっと気持ち悪い。
実際に `Object` で困ることはまずないと思うけど、きちんと型を指定したい！ ...という人がいるかもしれない。

## `OutputType` 属性を付ける
探った結果。

```xml
<Types>
  <Type>
    <Members>
      <ScriptProperty>
        <Name>ExtendedProp</Name>
        <GetScriptBlock>[OutputType([string])] param() $this.Name ...</GetScriptBlock>
      </ScriptProperty>
    </Members>
  </Type>
</Types>
```

`[OutputType("型名文字列")] param()` か `[OutputType([型名])] param()` を付けましょう。

ただし、これは本物の型ではないことに注意。ヒントみたいなものに過ぎない。
[about_Functions_OutputTypeAttribute] にも以下のように書かれている。

> OutputType 属性値は、ドキュメントのメモにすぎません。 関数コードから派生したり、実際の関数の出力と比較したりしていません。 そのため、値が不正確になる可能性があります。


[about_Types.ps1xml]: https://learn.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/about/about_types.ps1xml "about_Types.ps1xml - PowerShell | Microsoft Learn"
[about_Functions_OutputTypeAttribute]: https://learn.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/about/about_functions_outputtypeattribute "about_Functions_OutputTypeAttribute - PowerShell | Microsoft Learn"
