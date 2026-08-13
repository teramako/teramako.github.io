---
title: Sashimi - PowerShellモジュール
date: 2026-08-13 23:00:00+09:00
tags: PowerShell
toc: true
---

先日は[Sabamiso.psm - PowerShell用外部コマンド補完モジュール](https://teramako.github.io/2026/08/11/Sabamiso-PowerShell-module.html)の紹介をした。
今日は [Sashimi] というPowerShellモジュールの紹介をするよ。
バージョン 3.0.0 をリリースし、当初計画してやろうと思ったことをすべてやったので、記念に。

<img src="https://raw.githubusercontent.com/teramako/Sashimi/refs/heads/main/docs/img/Sashimi_512.png" alt="Sashimi logo" width="256" align="right">

[Sashimi] は外部コマンドの入出力をバイナリ(`byte[]`)で行うことにフォーカスしたコマンド群だ。

## 動機

### PowerShell の外部コマンド実行における問題点

PowerShell には内部にCmdletと呼ばれるコマンドや関数が用意されていて、基本的にはそれらを使用してコードを書くことになる。
他のシェルで言えばビルトイン・コマンドを多数抱えているようなものだ。
これらを使用している限り、PowerShellの特徴であるオブジェクト・ベースのパイプライン入出力が可能になる。

とはいえ、外部コマンドも必要になるし、実行も当然できるようになっている。
できるのだが、入出力に一定の制限が掛かる。
パイプライン入出力は **文字列** になる制限だ。

正確には全てが文字列になるわけではなく、`外部コマンド1 | 外部コマンド2` と外部コマンドが連結されている場合、この間はバイナリで入出力が行われるのだがPowerShell上からは見えない内部に隠ぺいされた動きなので、全て文字列になると思って良い。

多くの場合はこれで問題ない。
だが、文字列では困る場合もある。

- バイナリデータそのものが欲しい
- 入出力で扱われる文字コードが、文字列化時のエンコーディングと合っていない

1番目の「バイナリデータそのものが欲しい」はまぁ稀だろうけど、2番目のは割と遭遇したことがあるのではないだろうか？

### 解決策

いちおう解決策はある。

外部コマンドを実行して、その出力が文字列化される時のエンコーディングをあらかじめ変更しておくことだ。
`[Console]::OutputEncoding` をコマンドが出力するエンコーディングに変えることで対応が可能となる。

```powershell
try {
    $currentEncoding = [Console]::OutputEncoding
    [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
    $result = externalCommand # ←外部コマンド
} finally {
    [Console]::OutputEncoding = $currentEncoding
}
```

一応ね、これで解決はするんだけどさ、毎回こんな面倒なことやってられない。

というのが開発のきっかけ。

## "Sashimi" について

プロジェクト名/モジュール名はあっさりと決まった。

- プロジェクト名には「食べ物」とか「料理」から引っ張ってくるのが個人的な流行
- 文字列ではなく、バイナリデータ（**生データ**）を扱う

⇒ 刺身！

## インストール

PowerShell Gallery で公開している。

**PowerShell 7.6** 以上が必要な点に注意！

- [PowerShell Gallery | Sashimi 3.0.0](https://www.powershellgallery.com/packages/Sashimi/3.0.0)

```powershell
Install-PSResource -Name Sashimi
```

## コマンド紹介

### Invoke-RawCommand (alias: raw)
主題のコマンド。

外部コマンドを実行し、標準入力/標準出力/標準エラー を `byte[]` で受け取るコマンド。

- ドキュメント: https://github.com/teramako/Sashimi/blob/main/docs/Sashimi/en-US/Invoke-RawCommand.md

```powershell
PS1> Invoke-RawCommand echo abc
97
98
99
10
PS1> "うんこ" | Invoke-RawCommand cat
227
129
134
227
130
147
227
129
147
```

生データを扱うことが主題としてあるので、基本的には `byte[]` が出力されて上記のような結果になる。

さすがにこれでは非常に扱い難いｗ

ということで、`-AsString` パラメーター（alias: `-s`）と `-Encoding` パラメーター（alias: `-e`）がある。

```powershell
PS1> Invoke-RawCommand -s echo abc
abc
PS1> "うんこ" | Invoke-RawCommand -s cat
うんこ
```

ちなみに、`-AsString` による文字列化だが、 **行単位** で出力する仕様。

`-Encoding` はデフォルト(未指定)では UTF-8 になる。Shift_JIS出力されるものを扱いたい場合は `-Encoding` を指定する。

```powershell
PS1> raw -s .\test-shift_jis.bat
�����Shift_JIS����
PS1> raw -s -e shift_jis .\test-shift_jis.bat
これはShift_JISだよ
```

#### 標準エラー出力

`-AsString` は基本的に標準出力に作用するパラメーターだった。
対して、標準エラーは、`-AsString` の有無に関わらず文字列化する。

標準出力はバイナリになる可能性があるが、標準エラーは基本的にコンソールに印字されるべきものであり文字列だろう。という前提。

#### コマンド引数を正しく与える

以下を実行した時、どうなるか？

```powershell
"abc`n" | raw -s cat -e
```

答え:
エラーになる。
```output
Invoke-RawCommand: Missing an argument for parameter 'Encoding'. Specify a parameter of type 'System.String' and try again.
```
`-e` が `Invoke-RawCommand` のパラメーターと解釈されてしまうためだ。

正しく cat コマンドに渡すには:

- `"abc" | raw -s cat '-e'` と引用符で囲って文字列であると明示する
- `"abc" | raw -s cat -- -e` と `--` を付ける

少々面倒だが、我慢してもらうしかない。

#### ScriptBlockモード

上記のような課題と、`raw ... | raw ... | raw ...` と毎度 `raw` を付けるのが面倒。
ということで ScriptBlock モードを用意している。

```powershell
raw -s { "abc`n" | cat -e | base64 }
```

`{ ... }` にPowerShellスクリプトを書くことができる。
(PowerShell 的に由緒正しいコードで、`ScriptBlock` という匿名関数に近いもの)

実はブロック内を AST 解析をして、外部コマンド部分と判断できた部分を書き換えて実行しており、
ただの1ステートメントに限らず複数のステートメントを書けるし、`if` や `foreach` 文を書くこともできる。変数も読み書きできる。

また、外部コマンドがパイプで連続している場合には、`-AsString`(`-s`)を末尾のみに適用させる工夫もしている。

よって、`curl` で画像(バイナリデータ)を取ってきて、`img2sixel` でSixelに変換し、文字列として出力するようなことができる。

```powershell
$imgUrl = 'https://raw.githubusercontent.com/teramako/Sashimi/refs/heads/main/docs/img/Sashimi_512.png'
raw -s { curl $imgUrl | img2sixel -w256 }
```

![curl and img2sixel](/img/2026-08-13/curl-and-img2sixel.png)

### ConvertTo-RawString (alias: b2a)

`byte[]` → `string` を行う補助コマンド。 ("binary to ascii" 略して "b2a")

- ドキュメント: https://github.com/teramako/Sashimi/blob/main/docs/Sashimi/en-US/ConvertTo-RawString.md

`iconv -t utf8 -f ...` と思ってくれて良い。

元々 `Invoke-RawCommand` に `-AsString` パラメーターがない時のもので、
`raw ... | raw ... | b2a -e shift_jis` みたいな使い方をして最終的に文字列を得ることを目的としていた。`-AsString` がある今は、このコマンドの存在意義は薄れた。

ただ、`ConvertTo-RawString` には `-Raw` パラメーターがある。
`Invoke-RawCommand` の `-AsString` と `ConvertTo-RawString` (`-Raw`なし)は行単位で出力し、改行コードは除去されるので、これでは不都合がある場合には使うことがあるだろう。

```powershell
raw cat test.json | b2a -Raw | ConvertFrom-Json
```

### ConvertFrom-RawString (alias: a2b)

`string` → `byte[]` を行う補助コマンド。 ("ascii to binary" 略して "a2b")

- ドキュメント: https://github.com/teramako/Sashimi/blob/main/docs/Sashimi/en-US/ConvertFrom-RawString.md

`iconv -f utf8 -t ...` と思ってくれて良い。

`Invoke-RawCommand` の標準入力に `byte[]` のみを受け付けていた時のもので、
`"うんこ" | a2b | raw ...` みたいな使い方を目的としていた。

### Out-RawFile (alias: rawout)

`byte[]` → ファイル を行う補助コマンド。

- ドキュメント: https://github.com/teramako/Sashimi/blob/main/docs/Sashimi/en-US/Out-RawFile.md

PowerShellのリダイレクトや `Out-File` って、`byte[]` データをバイナリとして書き出してくれないので作った。

### Show-HexDump (alias: hexd)

Unix系コマンドで言えば、`hexdump` コマンドにあたる。

- ドキュメント: https://github.com/teramako/Sashimi/blob/main/docs/Sashimi/en-US/Show-HexDump.md

元々はこれ単体で https://github.com/teramako/HexDump/ プロジェクトとして作っていたのだが、[Sashimi] と非常に相性が良いので取り込んだ。

マルチバイトの表示には結構こだわって作っていて絵文字のようなサロゲートペアとなるような文字も奇麗に出せる。

#### サンプル画像
[旧HexDumpプロジェクト](https://github.com/teramako/HexDump/)時に撮ったものなので若干古いけど、

- ![](https://github.com/user-attachments/assets/01182654-a0ce-49bb-a51b-5a6b64094bb9)
- ![](https://github.com/user-attachments/assets/29a41c77-fda2-461a-ba1e-710716e0e63f)
- ![](https://github.com/user-attachments/assets/3168ba4e-4ee5-4df3-be4e-5ddefb0d8e8d)

カラーリングも行える

- ![](https://github.com/user-attachments/assets/bfa54c04-a758-4ee0-9dca-a5884a799d58)
- ![](https://github.com/user-attachments/assets/464e23b4-6f24-45d5-80c2-ee9a3d7b5361)
- ![](https://github.com/user-attachments/assets/ba7c7a7b-95c1-4eb3-9995-a3b2b75f8b42)


### Test-RawCommand (alias: raw?)

- ドキュメント: https://github.com/teramako/Sashimi/blob/main/docs/Sashimi/en-US/Test-RawCommand.md

`Invoke-RawCommand` と内部でやっていることはほぼ同じだが、外部コマンドを実行したプロセスの終了コードによって真偽値（終了コードが 0 なら `$true`, それ以外は `$false`）を返すことを目的としている。

出力結果ではなく、終了コードによる条件分岐をしたい場面での使用を想定している。

標準出力、標準エラーは Information レコードとして出力をするので、後々結果を取り出すことも可能だ。

```powershell
if (raw? grep pattern path/to/text -InformationVariable info) {
    $info.Where({$_.Tags -in "stdout"}).Message | Write-Output
} else {
    $info.Where({$_.Tags -in "stderr"}).Message | Write-Error
}
```

[Sashimi]: https://github.com/teramako/Sashimi/ "teramako/Sashimi: Raw process I/O for PowerShell — execute external commands and handle stdin/stdout/stderr as byte streams."
