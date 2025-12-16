---
title: PowerShell で外部コマンドの補完
date: 2025-12-16 10:57:00+09:00
tags: PowerShell
---

## TL;DR
PowerShell で外部コマンドの補完をするモジュールを開発中だよ

- [teramako/NativeCommandCompleter.psm]

----

なんだかんだ不満がありつつも、Linux(WSL)上で PowerShell を触っている今日この頃。

ただの文字列ではなくオブジェクトとして扱えることや、.NET の資産を使えることが大きい。
ただ、PowerShell 内のみの Cmdlet だけでは窮屈で外部コマンドを使用したいことが多々ある。というか外部コマンドでないとできないことが多い。
そうなるとPowerShellは無力だ。

出力はただの文字列だし、エラーハンドリングは面倒だし、コマンドラインの補完が全く効かないし。

そう、コマンドライン補完だ。

Linux の bash, zsh, fish 等々のシェルは外部コマンドの補完ができる。
この補完のおかげでコマンドライン生活ができるのだ。

PowerShell だってその点は分かっている。
[Register-ArgumentCompleter] を用いて外部コマンドの補完スクリプトの登録が可能だ。

でも、これ、`$PROFILE` 等に一つ一つ丁寧に各コマンドの補完スクリプトを登録する必要がある。

Linux上で外部コマンドが幾つあると思っているんだ。

## PowerShell 7.6.0-preview.5
そんな不満を持っていたところ、[PowerShell 7.6.0-preview.5] がリリースされた。

> Add the parameter `Register-ArgumentCompleter -NativeFallback` to support registering a cover-all completer for native commands ([#25230](https://github.com/PowerShell/PowerShell/pull/25230))

なにやら面白そうなのが追加されているじゃないですか。

## Register-ArgumentCompleter -NativeFallback

簡単に言うと、全コマンドで補完時に `Register-ArgumentCompleter` に与えたコードを通すことができるようになった。

これまでは、
`Register-ArgumentCompleter -CommandName <string[]> -ScriptBlock <scriptblock> [-Native] [<CommonParameters>]` で、外部コマンドの補完コードを定義していたが、`-CommandName` コマンド名の指定が必要だった。

これが、`Register-ArgumentCompleter -ScriptBlock <scriptblock> [-NativeFallback] [<CommonParameters>]` と、コマンド指定不要でコード登録ができるようになった。

1. 補完開始
2. コマンドライン解析
3. コマンド指定がある
   - PowerShell コマンド(Cmdlet) → Cmdletの補完(PowerShell内部コード)
   - コマンド名指定で補完コードが登録されている → その補完(登録されたスクリプト実行)
   - **NEW**: 上記2つに該当しない → 登録されたスクリプト実行(フォールバック的に)
   - 上記を通して補完候補がない → ファイル・ディレクトリ名補完

といった感じ。


### 補完コードの動的読み込み

Linux のシェル ―― bash, zsh, fish 等では、 `/etc/bash_completion.d` 等に各コマンドの補完コードが入っていて、必要に応じて読み込まれる。

これらと同じ事ができるよね？

早速試したが、…できた。

## モジュール開発

ということで、開発中である。

- [teramako/NativeCommandCompleter.psm]

fish の補完が気に入っているので、強くインスパイアされた補完リスト出力になっている。

組み込みで補完可能なコマンドの定義は以下にある

- https://github.com/teramako/NativeCommandCompleter.psm/tree/main/completions

メインはLinuxコマンドの補完だが、Windowsのコマンドも補完できるようにクロスプラットフォームに気を使っている（つもり）。
おかげでWindows用やLinux用(一部macOS/BSD用もあり)がごちゃまぜ状態…。

モジュールと補完定義は別リポジトリにすべきだったと少し後悔している。
C# プロジェクトだったのに、ご覧のありさまだ。

![languages](/img/2025-12-16/github-languages.png)

[teramako/NativeCommandCompleter.psm]: https://github.com/teramako/NativeCommandCompleter.psm
[Register-ArgumentCompleter]: https://learn.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/register-argumentcompleter
[PowerShell 7.6.0-preview.5]: https://github.com/PowerShell/PowerShell/releases/tag/v7.6.0-preview.5