---
title: PowerShell 文字列の幅を得る
date: 2024-10-07 01:30:00+09:00
tags: PowerShell
---

バイト数ではなく、文字数でもなく、文字が半角で幾つあるかの話。
コンソール・アプリケーションを作る上で問題になることが多いと思う。

で、PowerShell において、日本語や Emoji を出力してもカラムがある程度正しく表示されるので凄いなと思っていた。
(Emoji の一部は崩れたりして完全ではないけど)

```console
PS1> [PSCustomObject]@{ カラム1 = "あいうえお"; col2 = "💩💩😀"; col3 = "value"  }

カラム1    col2   col3
-------    ----   ----
あいうえお 💩💩😀 value
```

## PowerShell で文字幅を得る

どうやっているんだろう？ と調べた結果、以下の方法で幅を得られると判明した。

```console
PS1> $Host.UI.RawUI.LengthInBufferCells("💩")
2
PS1> $Host.UI.RawUI.LengthInBufferCells("あいうえお")
10
```

### C# から

`Cmdlet` 実装クラスから使用可能

```csharp
class Command : Cmdlet
{
    void Method()
    {
        int cellWidth = CommandRuntime.Host?.UI.RawUI.LengthInBufferCells("aa")
                        ?? throw new NullReferenceException();
    }
}
```

## コード

内部の具体的な実装は以下のコードと思われる。

> https://github.com/PowerShell/PowerShell/blob/c191efe890c74e6227a803905d7eb342728484fc/src/Microsoft.PowerShell.ConsoleHost/host/msh/ConsoleControl.cs#L2785-L2806
>
> ```csharp
>         internal static int LengthInBufferCells(char c)
>         {
>             // The following is based on http://www.cl.cam.ac.uk/~mgk25/c/wcwidth.c
>             // which is derived from https://www.unicode.org/Public/UCD/latest/ucd/EastAsianWidth.txt
>             bool isWide = c >= 0x1100 &&
>                 (c <= 0x115f || /* Hangul Jamo init. consonants */
>                  c == 0x2329 || c == 0x232a ||
>                  ((uint)(c - 0x2e80) <= (0xa4cf - 0x2e80) &&
>                   c != 0x303f) || /* CJK ... Yi */
>                  ((uint)(c - 0xac00) <= (0xd7a3 - 0xac00)) || /* Hangul Syllables */
>                  ((uint)(c - 0xf900) <= (0xfaff - 0xf900)) || /* CJK Compatibility Ideographs */
>                  ((uint)(c - 0xfe10) <= (0xfe19 - 0xfe10)) || /* Vertical forms */
>                  ((uint)(c - 0xfe30) <= (0xfe6f - 0xfe30)) || /* CJK Compatibility Forms */
>                  ((uint)(c - 0xff00) <= (0xff60 - 0xff00)) || /* Fullwidth Forms */
>                  ((uint)(c - 0xffe0) <= (0xffe6 - 0xffe0)));
>
>             // We can ignore these ranges because .Net strings use surrogate pairs
>             // for this range and we do not handle surrogate pairs.
>             // (c >= 0x20000 && c <= 0x2fffd) ||
>             // (c >= 0x30000 && c <= 0x3fffd)
>             return 1 + (isWide ? 1 : 0);
>         }
> ```
