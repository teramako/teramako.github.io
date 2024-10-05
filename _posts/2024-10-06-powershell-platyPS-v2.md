---
title: PowerShell platyPS v2 を試す
date: 2024-10-06 00:30:00+09:00
tags: PowerShell
---

[platyPS] というものがある。
PowerShell モジュールのヘルプ生成ツールだ。
詳細は [PlatyPS を使用して XML ベースのヘルプを作成する - PowerShell | Microsoft Learn](https://learn.microsoft.com/ja-jp/powershell/utility-modules/platyps/create-help-using-platyps?view=ps-modules) を参照。

[platyPS]: https://github.com/PowerShell/platyPS "PowerShell/platyPS: Write PowerShell External Help in Markdown"

## v2 ブランチ
現在のリリース版は v0.14.2 (https://www.powershellgallery.com/packages/platyPS/) だが、いろいろ不満があったので修正を試みようと clone してたら [`v2` ブランチ](https://github.com/PowerShell/platyPS/tree/v2) が作られていることに気付いた。

`v2` ブランチに切り替えてビルドしてみると、今までとは全然違うものになっていた。

### 違い

- 全て C# によるバイナリのコマンドになった
  - 今まではPowerShellスクリプトで作られた関数群だった
  - パフォーマンスが改善されている
- Markdown ファイルのパースに [Markdig] を使用するようになった
  - [Markdig] は PowerShell 7 にも同梱されているもの
  - 今までは独自にパースするコードが入っていた
- コマンド体系が変わった
  - コマンド名からして別物になっている

[Markdig]: https://github.com/xoofx/markdig

### コマンド体系

今までの流れは、以下のような感じだった。

1. Markdown のドキュメントを生成する
2. 生成された Markdown ドキュメントを編集
3. ヘルプファイルを作成

v2 でも大きな方針は変わらない。
が、物事の中心にあるは、`CommandHelp` というオブジェクトになっている。

整理のために入出力の関係図のようなものを作ってみたが、中心になっているのが `CommandHelp` オブジェクトであることが良く分かる。

- `input --> |command| output` の図

![](/img/2024-10-06/mermaid-diagram-2024-10-06-003501.svg)
