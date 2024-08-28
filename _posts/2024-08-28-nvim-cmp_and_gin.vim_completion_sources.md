---
title: Neovim nvim-cmp の補完ソースプラグインと gin.vim の action 補完ソースを書いた話
date: 2024-08-28
modified_date: 2024-08-28 09:17:00+09:00
tags: Vim
toc: true
---

この記事は[Vim 駅伝]2024年8月28日の記事です。  
前回(2024年8月26日)の記事は [staticWagomU](https://wagomu.me/)さんの「[statusline隠すとかっこよくなることに気づいてしまった...](https://wagomu.me/blog/2024_08_26-vim-ekiden/ "statusline隠すとかっこよくなることに気づいてしまった... - 輪ごむの空き箱")」でした。  
次回の予定はは明後日(8月30日)になります。

-----

## はじめに
こんにちは、 teramako です。
諸事情により時間ができたので Neovim に挑戦し始め、かれこれ１ヶ月くらいが経ちました。

Vim 自体は昔から使っていたのですが、ほぼ無プラグイン (シンタックスハイライト系と [vim-surround](https://github.com/tpope/vim-surround) 程度) で過ごしていましたした。
Neovim ではプラグインをどんどん使っていこうと心機一転、試行錯誤を繰り返しています。
その様子を [vim-jp Slack] で呟いていたら声を掛けられて [Vim 駅伝]の記事にすることになりました。

ちなみに、Neovim 初心者、lua も初めて、 Vim の関数も分からん、と分からないことだらけから始めています。

[vim-jp Slack]: https://vim-jp.org/docs/chat.html "vim-jp » vim-jpのチャットルームについて"
[Vim 駅伝]: https://vim-jp.org/ekiden/ "Vim 駅伝"


## nvim-cmp(cmp-cmdline) と gin.vim

登場するプラグインは以下の３つ。

- [gin.vim] : git 操作を行うプラグイン
- [nvim-cmp] : 自動補完を行うプラグイン
  - [cmp-cmdline] : nvim-cmp の追加プラグインで、コマンドラインの自動補完を行う

gin.vim については以下の記事を参考にしてください。

- [gin.vimでgitの差分を快適に閲覧する | Atusy's blog](https://blog.atusy.net/2023/11/29/gin-diff/)
- [git操作プラグインを探しているならgin.vimはどう？ - 輪ごむの空き箱](https://wagomu.me/blog/2023-12-12-vim-adventcalendar/)
- [gin.vimで捗るgitのログ改竄 (instant fixup) | Atusy's blog](https://blog.atusy.net/2024/03/15/instant-fixup-with-gin-vim/)

中でも３つ目で紹介されている instant fixup に魅力を感じて導入しました。
あと、これまで git をコマンドラインで使用してきていたため、その感覚のまま使えそうだったのも決め手の一つです。

<figure><img src="/img/2024-08-28/gin-1.png" alt="画面キャプチャ"/>
<figcaption>git statusや git log そのままの見た目</figcaption></figure>

で、この gin.vim には `:GinStatus` 上でのファイル行、`:GinBranch` 上でのブランチ名行、 `:GinLog` 上でのコミット行で各種アクションを実行する機能があります。
アクションは多数あるのですが、コマンドライン上に `action: ` のプロンプトが出て、補完を効かせつつ選択することができます。

<figure><img src="/img/2024-08-28/gin-actions.png" alt="画面キャプチャ">
<figcaption>gin.vimのアクション補完</figcaption></figure>

## cmp-cmdline と gin.vim の組み合わせ問題
ところが、`cmp-cmdline` を導入したところ、この補完が効かなくなってしまいました。

`cmp-cmdline` は独自に補完機能を実現させていて、Vimネイティブの補完機能を使用していないためと思われます。
このことは [REAME](https://github.com/hrsh7th/nvim-cmp/blob/main/README.md) のサンプル設定にあるコメントに書かれています。

> ```lua
>   -- Use buffer source for `/` and `?` (if you enabled `native_menu`, this won't work anymore).
>   cmp.setup.cmdline({ '/', '?' }, {
>     -- 略
>   })
>
>   -- Use cmdline & path source for ':' (if you enabled `native_menu`, this won't work anymore).
>   cmp.setup.cmdline(':', {
>     -- 略
>   })
>   ```

バグではなく、仕様ですので仕方ありません。

## プラグイン(cmp-cmdline-prompt) の自作

しかし、簡単には諦められません。
gin.vim のアクション補完は重要です。
万事うまくやる方法はなくとも自身の環境だけで良いからと方法を探りました。
そして、最終的には割と汎用的に使える方法を見つけ、[nvim-cmp] 用の補完ソースのプラグインを書くに至りました。

- [GitHub - teramako/cmp-cmdline-prompt.nvim: nvim-cmp source for vim's command line `input()` prompt](https://github.com/teramako/cmp-cmdline-prompt.nvim)

主要なファイルは2つだけ（しかもその内の一つは1行しか書かれていない）の非常に小さなプラグインです。

```lua
-- プラグインマネージャー等で, nvim-cmp と cmp-cmdline-prompt を追加する
local cmp = require('cmp') -- nvim-cmp のインスタンスを得る
cmp.setup({ ... }) -- グローバルな設定
cmp.cmdline.setup('@', { -- `input()` 関数からの入力プロンプト時の設定
    sources = cmp.config.sources({
        { name = 'cmdline-prompt' },  -- 補完ソースを指定
    },
})
```

上記を `init.lua` 等に記載すれば最低限は動作すると思います。

<figure><img src="/img/2024-08-28/cmp-cmdline-prompt-1.png" alt="画面キャプチャ">
<figcaption>cmp-cmdline-prompt.nvim を使用して補完リストを出している画面</figcaption></figure>

## 補足

作成したプラグインと設定について、少し解説というか補足をします。

元々やりたかった [gin.vim] の「`action: `」プロンプト対応ですが、
調査していくとこのプロンプトは Vim の [`input()`] 関数を使用していることが分かりました。

作成したプラグインは [`input()`] 関数によるプロンプトに対応するものになります。

### `input()` と補完
[`input()`] 関数はコマンドライン上でプロンプトを出し、その入力値を受け取れる Vim に元々備わっている関数です。
[gin.vim] もこの関数を使用して「`action: `」プロンプトを出しています。

また、[`input()`] 関数では第三引数に補完設定があり、ここで `file`(ファイル名およびディレクトリ補完) や `dir`(ディレクトリ名補完) といった汎用的なものや独自のユーザー定義の補完を設定することができます。(補完タイプについては [:command-completion] を参照)
この補完ですが [`getcmdcompltype()`] で今現在の補完タイプが得られ、[`getcompletion()`] 関数を使うと補完リストを得ることができます。
(この辺りは [vim-jp Slack] にて Hibiki さん、atusy さん、kuu さんに大変お世話になりました。ありがとうございます。)

```vim
getcompletion(getcmdline(), getcmdcompltype())
"             ^ 入力文字列  ^ 補完タイプ
" 補完結果の文字列のリストが返ってくる
```

素晴らしいことに [gin.vim] のようなユーザー定義の独自の補完設定であってもリストを得ることできます。
これで得られた補完リストを [nvim-cmp] に渡してあげるのが、このプラグインの概要であり全てです。

### nvim-cmp `cmp.cmdline-setup()` の第一引数
さて、 [nvim-cmp] 側のコマンドライン補完について、サンプルの設定で `cmp.cmdline.setup('@'` と `'@'` を指定しました。
> ```lua
> cmp.cmdline.setup('@', { -- `input()` 関数からの入力プロンプト時の設定
>     sources = cmp.config.sources({
>         { name = 'cmdline-prompt' },  -- 補完ソースを指定
>     },
> })
> ```
これは何か。

コマンドラインには幾つか種類があり、
[nvim-cmp] のコマンドラインの補完設定となる `cmp.cmdline.setup()` の第一引数には、その種類を指定することになります。
README にも載っていますが、代表的なのは通常の Ex コマンドの `':'` や前方検索の `'/'` と 後方検索の `'?'` です。しかし、これら以外にも以下のようなものがあります。
(`:help getcmdtype()`)

| 種類 | 説明                                       |
|:----:|:-------------------------------------------|
| `:`  | 通常のexコマンド                           |
| `>`  | デバッグモードコマンド debug-mode          |
| `/`  | 前方検索コマンド                           |
| `?`  | 後方検索コマンド                           |
| `@`  | [`input()`] コマンド                       |
| `-`  | `:insert` または `:append` コマンド        |
| `=`  | `i_CTRL-R_=`                               |

 (https://vim-jp.org/vimdoc-ja/builtin.html#getcmdtype() より)

当初は、プロンプト文字を指定するのかと思い、 `cmp.cmdline.setup('action: ',` なんて記載をしてみたりと勘違いしたことをしました ^^;

`input()` 時の補完を行いたいため `@` を指定することになります。

[:command-completion]: https://vim-jp.org/vimdoc-ja/map.html#:command-completion "map - Vim日本語ドキュメント"
[`input()`]: https://vim-jp.org/vimdoc-ja/builtin.html#input() "builtin - Vim日本語ドキュメント"
[`getcmdcompltype()`]: https://vim-jp.org/vimdoc-ja/builtin.html#getcmdcompltype() "builtin - Vim日本語ドキュメント"
[`getcompletion()`]: https://vim-jp.org/vimdoc-ja/builtin.html#getcompletion() "builtin - Vim日本語ドキュメント"

### 補完の除外設定

汎用的に作ったプラグインですが、場合によっては別のプラグインを使うなどの理由で除外したくなるかもしれません。

例えば、「ファイルやディレクトリパスの補完には [cmp-path] を使うから要らない」等です。

[cmp-path]: https://github.com/hrsh7th/cmp-path "hrsh7th/cmp-path: nvim-cmp source for path"

そんな場合に備えて(…というか僕が除外したくなったのですが)、補完タイプ名などから除外することができます。

#### 単純な除外
`option.excludes` に除外したい補完タイプ名を記載します。
(補完タイプ名一覧は [:command-completion] を参照)

例外として、`custom,{func}` や `customlist,{func}` のユーザー定義の補完は "," 前の `custom` や `customlist` になります。
```lua
cmp.cmdline.setup('@', {
    sources = cmp.config.sources({
        {
            name = 'cmdline-prompt',
            option = {
                -- 除外したい補完タイプのリスト
                excludes = { 'file', 'dir' }
            }
        },
        { name = 'path' }
    },
})
```

#### 複雑な除外
`option.excludes` は関数にすることもでき、より細かな制御が可能です。
`true` (truthy) を返すと、補完を取りやめます。
```lua
cmp.cmdline.setup('@', {
    sources = cmp.config.sources({
        {
            name = 'cmdline-prompt',
            option = {
                ---@type fun(context: cmd.Context, completion_type: string, custom_function: string) : boolean
                excludes = function(context, completion_type, custom_function)
                    if completion_type == 'custom' or completion_type == 'customlist' then
                        if custom_function:match(...) then
                            return true
                        end
                    elseif context.filetype == 'gin-status' then
                        return true
                    end
                    return false
                end
            }
        }
    },
})
```
関数には幾つか引数として情報が渡されます。
1. [nvim-cmp] の補完コンテキスト ([context.lua](https://github.com/hrsh7th/nvim-cmp/blob/main/lua/cmp/context.lua) の `@class cmp.Context` 辺りを参照)
2. 補完タイプ名
3. ユーザー関数名 (補完タイプ名が `custom`, `customlist` 時以外は空文字)

### 補完の種類、ハイライト

`option` には他にも `option.kinds` で補完の種類(`Kind`)とハイライトも設定できます。
[lspkind.nvim] 等も併用しつつ見た目をカスタマイズするのに役立つかもしれません。

[lspkind.nvim]: https://github.com/onsails/lspkind.nvim "onsails/lspkind.nvim: vscode-like pictograms for neovim lsp completion items"

```lua
cmp.cmdline.setup('@', {
    sources = cmp.config.sources({
        {
            name = 'cmdline-prompt',
            option = {
                kinds = {
                    file   = cmp.lsp.CompletionItemKind.File,
                    buffer = {
                        kind = cmp.lsp.CompletionItemKind.File,
                        hl_group = 'CmpItemKindFolder'
                    }
                }
            }
        }
    },
})
```
補完タイプ名をキーに、種類を指定したり、`hl_group` に直接ハイライトグループを指定できます。

## 僕の設定

参考までに、僕の今現在の設定を抜粋して載せておきます。

```lua
local cmp = require('cmp')
local lspkind = require('lspkind')
lspkind.setup({ preset = 'codicons' })
-- cmp.setup({ .. 略 .. })
-- cmp.setup.cmdline({ '/', '?' }, { .. 略 .. })
-- cmp.setup.cmdline(':', { .. 略 .. })
require('cmp-gin-action');
-- See: cmp.lsp.CompletionItemKind
-- See: :help getcmdtype()
cmp.setup.cmdline('@', { -- vim.fn.input() 時の補完
    mapping = cmp.mapping.preset.cmdline(),
    sources = cmp.config.sources({
        {
            name = 'cmdline-prompt',
            ---@type prompt.Option
            option = {
                -- gin-action で補完するので customlist の補完は除外する
                excludes = { 'customlist' },
                -- 今のところ必要ないけど……補完タイプに応じて lspkind で装飾するために
                kinds = {
                    buffer      = cmp.lsp.CompletionItemKind.File,
                    dir         = cmp.lsp.CompletionItemKind.Folder,
                    file        = cmp.lsp.CompletionItemKind.File,
                    packadd     = cmp.lsp.CompletionItemKind.Module,
                    runtime     = cmp.lsp.CompletionItemKind.Folder,
                    scriptnames = cmp.lsp.CompletionItemKind.File,
                }
            }
        },
        { name = 'gin-action' }
    }),
    formatting = {
        fields = { 'kind', 'abbr', 'menu' },
        format = function(entry, vim_item)
            local item = entry:get_completion_item()
            if entry.source.name == 'cmdline-prompt' then
                vim_item.kind = cmp.lsp.CompletionItemKind[item.kind]
                local kind = lspkind.cmp_format({ mode = 'symbol_text' })(entry, vim_item)
                local strings = vim.split(kind.kind, '%s', { trimempty = true })
                kind.kind = ' ' .. (strings[1] or '')
                kind.menu = ' (' .. (item.data.completion_type or '') .. ')'
                kind.menu_hl_group = kind.kind_hl_group
                return kind
            elseif entry.source.name == 'gin-action' then
                vim_item.kind = ' ' -- gitマークを付けよう
                return vim_item
            else
                return vim_item
            end
        end
    },
    sorting = {
        comparators = { cmp.config.compare.order }
    },
    window = {
        completion = {
            col_offset = 6, -- "action: " の8文字分 - アイコン2文字
        },
    },
})
```

せっかく作ったプラグインですが、実はこれを使って [gin.vim] のアクション補完をさせていません。
別途、個人の `~/.config/nvim/` 配下に専用の補完ソース ([lua/cmp-gin-action/init.lua](https://github.com/teramako/dotfiles/blob/main/nvim/lua/cmp-gin-action/init.lua)) を作ってそちらで補完させています。(これをやりたくて除外設定をできるようにしました)

gin.vim 専用ということもあって情報量が多くなるためです。

<figure><img src="/img/2024-08-28/cmp-cmdline-prompt-2.png" alt="画面キャプチャ">
<figcaption>gin.vim用補完を使用して補完リストを出している画面</figcaption></figure>

以上、自作したプラグインの紹介でした。

----

## おまけ

せっかくですので、上記プラグインやコードを書くに至るまでに調べたこと、やったことを書いておきます。

### 分かったこと

調査していく中で分かったこと：

- Vim の `:help` は優秀
- プラグインの `:help` も優秀
- 「設定させていただきありがとうございます」

以下、試行錯誤の詳細。

### gin.vim のプロンプトをどうやって出しているか

- `action: ` プロンプトには [`input()`] を使用してプロンプト入力および補完をさせている
  - vim ネイティブな機能なので「特殊なこと」はしていない

これが結論なのですが、これに至るまで。

[gin.vim] は denoops.vim を使って裏で Deno (TypeScript) が動いています。

- [GitHub - vim-denops/denops.vim: 🐜 An ecosystem of Vim/Neovim which allows developers to write cross-platform plugins in Deno](https://github.com/vim-denops/denops.vim)
- [GitHub - vim-denops/deno-denops-std: 📚 Standard module for denops.vim](https://github.com/vim-denops/deno-denops-std/)

Deno は2年前ほどに [nyancat](https://github.com/teramako/nyancat.deno) を書いてみて遊んだ程度です。まぁだいだい JavaScript だし読むだけなら何とかなるだろうと読み始めます。

……ちょっと甘かったです。async/await の連続で、かつ、vim/neovim を動かすためのライブラリですので随所に Vim script も出てきます。
Deno も TypeScript も Vim script も詳しくない僕には辛かったです。

何も分からないけど、プロンプトの文字列 `action: ` の文字列は gin.vim 上に絶対載っているはずだ、という当たりを付けます。
雑に `git grep "action: "` で検索することから始めます。

```console
$ git grep -n "action: "
denops/gin/action/core.ts:152:    prompt: "action: ",
```

素晴らしい。
`   prompt: "action: ",` がまさにそれっぽいですね。対象のファイルを覗いてみます。
TypeScript に詳しいわけではないけど、雰囲気で読んでいきます。

> ```typescript
> async function doChoice(
>   denops: Denops,
>   bufnr: number,
>   range: Range,
> ): Promise<void> {
>   const cs = await list(denops, bufnr);
>   const name = await helper.input(denops, {
>     prompt: "action: ",
>     completion: (
>       arglead: string,
>       _cmdline: string,
>       _cursorpos: number,
>     ) => {
>       return Promise.resolve(
>         cs.filter((c) => c.name.startsWith(arglead)).map((c) => c.name),
>       );
>     },
>   });
>   if (name == null) {
>     return;
>   }
>   await fn.setbufvar(denops, bufnr, "denops_action_previous", name);
>   await call(denops, name, range);
> }
> ```
> https://github.com/lambdalisue/vim-gin/blob/0c0c47672fa1d23b230156f07ae6cb92144582a8/denops/gin/action/core.ts#L145-L168

どうやら、さらに `helper.input()` が入力プロンプトを司っているようです。
ファイル先頭にある `import * as helper from "jsr:@denops/std@^7.0.0/helper";` で `helper` をインポートしているみたいなので、 `denoops/std` 側を見に行きます。

見に行きたいのですが、ローカルの何処にあるのか分かりませんでした。
仕方ないので、Web で検索して GitHub 上のコードで読みました。

> ```typescript
> export async function input(
>   denops: Denops,
>   options: InputOptions = {},
> ): Promise<string | null> {
>   const suffix = await ensurePrerequisites(denops);
>   // (snip)
>   return denops.call(
>     `DenopsStdHelperInput_${suffix}`,
>     options.prompt ?? "",
>     options.text ?? "",
>     completion,
>     options.inputsave ?? false,
>   ) as Promise<string | null>;
> }
> ```
> https://github.com/vim-denops/deno-denops-std/blob/dae0f5e58f50af893070b1bdb2a52a0af10961c8/helper/input.ts#L215-L254

さらに `ensurePrerequisites()` も見てみます。

> ```vim
>     function! s:input_${suffix}(prompt, text, completion) abort
>       let options = {
>           \\ 'prompt': a:prompt,
>           \\ 'default': a:text,
>           \\ 'cancelreturn': s:escape_token_${suffix},
>           \\}
>       if a:completion isnot# v:null
>         let options['completion'] = a:completion
>       endif
>       try
>         let result = input(options)
>       catch /^Vim:Interrupt$/
>         return v:null
>       endtry
>       if result ==# s:escape_token_${suffix}
>         return v:null
>       endif
>       return result
>     endfunction
> ```
> https://github.com/vim-denops/deno-denops-std/blob/dae0f5e58f50af893070b1bdb2a52a0af10961c8/helper/input.ts#L51-L69

何やら Vim script を生成して実行させている様子。
ここで、 `input(options)` の箇所が Neovim 上に出てくるプロンプトの正体だろう。

`:help input()` で調べます。(Web 上のドキュメント: [input() - Builtin - Neovim docs](https://neovim.io/doc/user/builtin.html#input() "Builtin - Neovim docs"))
> ```
> input({prompt} [, {text} [, {completion}]])                            *input()*
>
> input({opts})
> 		The result is a String, which is whatever the user typed on
> 		the command-line.  The {prompt} argument is either a prompt
> 		string, or a blank string (for no prompt).  A '\n' can be used
> 		in the prompt to start a new line.
> 略
> ```

ということで、vim ネイティブに備えられている関数でプロンプトを出し、かつ、補完も出来るようになっていると分かりました。

TypeScript を読み始めた時には denops で特殊なことをしているのでは？ だとすると [nvim-cmp] で補完を出せないのも不思議はないなぁなんて思っていましたが、Vimネイティブな機能でした。
生意気なことを考えていてゴメンナサイ。


### 試行錯誤 その１

Vimネイティブな機能なら、nvim-cmp, cmp-cmdline を無効化するか、ネイティブな補完を呼び出せばいけるのでは？

#### nvim-cmp, cmp-cmdline の無効化すれば……？
```lua
cmp.setup.cmdline(':', {
    -- 略
    enabled = function()
        -- if ... then
        return false
    end
})
```
とすると条件に応じて一時的な無効化でできるっぽいのでやってみよう。

……ダメでした。
nvim-cmpでの補完試行 -> enable ではないので停止。となるだけで、ネイティブに引き渡すわけではなさそう

#### ネイティブな補完を呼び出せば……？
やり方が分かりませんでした。
Vim力不足によりいったん諦め。

### 試行錯誤 その２
そういえば、`cmp.setup.cmdline({ '/', '?' }, ...)` とか `cmp.setup.cmdline(':', ...)` の第一引数って、コマンドラインの先頭文字ではないだろうか。
`action:` のプロンプト時のみに限定できれば何かできるのでは？
```lua
cmd.setup.cmdline('action:', {
    -- 略
})
```
と書いてみるも、呼び出される気配なし。

### nvim-cmp, cmp-cmdline の Issue 調査
Vimネイティブ機能が使えないってことは、issue に何か載っているのでは？

#### nvim-cmp 側
- [Support completion in `vim.fn.input` · Issue #1690 · hrsh7th/nvim-cmp · GitHub](https://github.com/hrsh7th/nvim-cmp/issues/1690)

1. お、そのものがあるじゃないか。
2. あ、闇の Shougo さんだ、こんにちは～
3. んー、特に何も載ってないなあ

#### cmp-cmdline 側
- [does not complete vim.ui.input · Issue #16 · hrsh7th/cmp-cmdline · GitHub](https://github.com/hrsh7th/cmp-cmdline/issues/16)


1. > you can do the following:
   > ```lua
   > cmp.setup.cmdline("@", {
   >   sources = cmp.config.sources({
   >      { name = "path" },
   >      { name = "cmdline" },
   >  }),
   > })
   > ```
   > `:h getcmdtype()`, look at `@`
2. お？ `@` とは？
3. 指示どおり `:help getcmdtyp()` で調べる。なるほど、`@` が `input()` を示すのか
   ```
   getcmdtype()                                                      *getcmdtype()*
         Return the current command-line type. Possible return values
         are:
             :	normal Ex command
             >	debug mode command |debug-mode|
             /	forward search command
             ?	backward search command
             @	|input()| command
             -	|:insert| or |:append| command
             =	|i_CTRL-R_=|
   ```
4. そういえば、nvim-cmp の `:help` を引いてないな。見てみよう
5. `:help cmp.setup.cmdline` に載っているじゃん！
   ```
   *cmp.setup.cmdline* (cmdtype: string, config: cmp.ConfigSchema)
     Setup cmdline configuration for the specific type of command.
     See |getcmdtype()|.
     NOTE: nvim-cmp does not support the `=` command type.
   ```
6. `cmp.setup.cmdline('action:', ...)` なんて頓珍漢なことをしてゴメンナサイ
7. とはいえ、この Issue はディレクトリの補完をしたいなら `path` のソースを追加すれば良いよねって話だな。
8. そもそも gin.vim 用の補完ソースが……

と、この辺りで自力で補完ソースを作れば良いことに気付きました。


### gin.vim のアクション一覧はどうやって取る？

調査方針を変更し、補完ソースを自作できないか探っていきます。
まずは補完させたい値の一覧を取得できなければ話になりません。
が、実は最初の gin.vim の調査時にある程度目星が付いていました。
プロンプトを出すところで、`const cs = await list(denops, bufnr);` のコードがあったのを覚えていたのです。
ここから辿れば良さそうです。

> ```typescript
> async function doChoice(
>   denops: Denops,
>   bufnr: number,
>   range: Range,
> ): Promise<void> {
>   const cs = await list(denops, bufnr);    // ←ココ
>   const name = await helper.input(denops, {
>     prompt: "action: ",
>     completion: (
>       arglead: string,
>       _cmdline: string,
>       _cursorpos: number,
>     ) => {
>       return Promise.resolve(
>         cs.filter((c) => c.name.startsWith(arglead)).map((c) => c.name),
>       );
>     },
>   });
>   if (name == null) {
>     return;
>   }
>   await fn.setbufvar(denops, bufnr, "denops_action_previous", name);
>   await call(denops, name, range);
> }
> ```
> https://github.com/lambdalisue/vim-gin/blob/0c0c47672fa1d23b230156f07ae6cb92144582a8/denops/gin/action/core.ts#L145-L168

`list(denoops, bufnr)` 関数を見てみます。
> ```typescript
> export async function list(
>   denops: Denops,
>   bufnr: number,
> ): Promise<Action[]> {
>   let cs: Action[] = [];
>   await buffer.ensure(denops, bufnr, async () => {
>     const ms = await mapping.list(denops, "<Plug>(gin-action-", { mode: "n" });
>     cs = ms.flatMap((map) => {
>       const m = map.lhs.match(/^<Plug>\(gin-action-(.*)\)$/);
>       if (!m) {
>         return [];
>       }
>       return [{
>         lhs: map.lhs,
>         rhs: map.rhs,
>         name: m[1],
>       }];
>     });
>   });
>   return cs;
> }
> ```
> https://github.com/lambdalisue/vim-gin/blob/0c0c47672fa1d23b230156f07ae6cb92144582a8/denops/gin/action/core.ts#L50-L70

`mapping.list(denops, "<Plug>(gin-action-", { mode: "n" });` とあり、`mapping.list()` がどいういうものか分かりませんが、モード `n` の `<Plug>(gin-action-` が絡んでいる事が分かります。
さらに、`map.lhs.match(/^<Plug>\(gin-action-(.*)\)$/)` でアクション名を抽出しているようです。
`buffer` や `bufnr` もあることから対象のバッファ固有のものであるのも想像できます。

Vim で `mapping` といえば、map のことでしょうし、`{ mode: "n" }` は Normal モードのことでしょう。
map 一覧を取得する関数は絶対にあるはずだと、`:help` で検索します。

ここで cmp-cmdline が有効だと、各種キーワードを fuzzy に検索/補完してくれます。
これには大変助かりました。
やはり僕には cmp-cmdline が必要です。

<figure><img src="/img/2024-08-28/nvim_help_completion.png" alt="画面キャプチャ">
<figcaption>:help keymap 検索</figcaption></figure>

> ```vim
> nvim_buf_get_keymap({buffer}, {mode})                  *nvim_buf_get_keymap()*
>     Gets a list of buffer-local |mapping| definitions.
>
>     Parameters: ~
>       • {buffer}  Buffer handle, or 0 for current buffer
>       • {mode}    Mode short-name ("n", "i", "v", ...)
>
>     Return: ~
>         Array of |maparg()|-like dictionaries describing mappings. The
>         "buffer" key holds the associated buffer handle.
>
> ```

実際にこの関数を使って情報を取れるか確認してみます。
まずは `:lua vim.print(vim.api.nvim_buf_get_keymap(0, 'n'))` で出力したのですが、情報量が多すぎました。
必要とする `lhs` の情報、どうせならアクション名の抽出もしてみることにします。

ワンラインで書くのもツライと思ったので、ファイルに書いて実行させる方法を試しました。
（[nvim-lua-guide-ja/README.ja.md at master · willelz/nvim-lua-guide-ja · GitHub](https://github.com/willelz/nvim-lua-guide-ja/blob/master/README.ja.md) のガイドには大変助けられました。ありがとうございます。）

```lua
local nmaps = vim.api.nvim_buf_get_keymap(0, 'n')
for _, nmap in ipairs(nmaps) do
    local m = string.match(nmap.lhs, '^<Plug>%(gin%-action%-(%S+)%)')
    if m then
        print(m)
    end
end
```

テキトウなところに書いて保存し、 `:luafile` で読み込み＆実行します。

この7行を書くのに小一時間…。
`ipairs()` を忘れて for 文が動かず…、
`string.match()` の~~正規表現~~(正確には「Lua Pattern」というそうです)の書き方が分からず…。
「luaなら分かルワ」なんてクソみたいなダジャレを言っていた過去の自分を殴りたかったです。

悪戦苦闘したものの、なんとかアクション名の一覧抽出、および、それを lua から出来る確証を得られました。

```
yank:path:orig
yank:path
stash:keep-index
stash
unstage:intent-to-add
unstage
stage:intent-to-add
stage
(略)
```

ここまでできれば、補完ソースの書き方が分かれば行けそうです。

### 補完ソースの書き方

Issue 調査時にプラグインのヘルプを見ていて `:help cmp-develop` があることには気付いていましたので、まずはこれを見ます。
> ```markdown
> Develop                                                            *cmp-develop*
>
> Creating a custom source~
>
> NOTE:
>   1. The `complete` method is required. Others can be omitted.
>   2. The `callback` function must always be called.
>   3. You can use only `require('cmp')` in custom source.
>   4. If the LSP spec was changed, nvim-cmp may implement it without any announcement (potentially introducing breaking changes).
>   5. You should read ./lua/cmp/types and https://microsoft.github.io/language-server-protocol/specifications/specification-current.
>   6. Please add your source to the list of sources in the Wiki (https://github.com/hrsh7th/nvim-cmp/wiki/List-of-sources)
>   and if you publish it on GitHub, add the `nvim-cmp` topic so users can find it more easily.
>
> Here is an example on how to create a custom source:
> ```

また、サンプルコードも付いています。
> ```lua
> local source = {}
>
> -- 略
>
> ---Invoke completion (required).
> ---@param params cmp.SourceCompletionApiParams
> ---@param callback fun(response: lsp.CompletionResponse|nil)
> function source:complete(params, callback)
>   callback({
>     { label = 'January' },
>     { label = 'February' },
>     { label = 'March' },
>     { label = 'April' },
>     { label = 'May' },
>     { label = 'June' },
>     { label = 'July' },
>     { label = 'August' },
>     { label = 'September' },
>     { label = 'October' },
>     { label = 'November' },
>     { label = 'December' },
>   })
> end
>
> -- 略
>
> ---Register your source to nvim-cmp.
> require('cmp').register_source('month', source)
> ```

以下の2点が最低限必要なことが分かります。
- `require('cmp').register_source(...)` で補完ソースを登録すること
- 補完ソースは、`complete(_, callback)` 関数を定義して、`callback` 関数に補完アイテムのリストを渡すこと

あとは、`callback` に渡す補完アイテムについて、型定義が書かれています。
> ```lua
>   ---@param callback fun(response: lsp.CompletionResponse|nil)
> ```

深堀すると https://github.com/hrsh7th/nvim-cmp/blob/main/lua/cmp/types/lsp.lua に型定義情報が載っていました。
> ```lua
> ---@class lsp.CompletionList
> ---@field public isIncomplete boolean
> ---@field public itemDefaults? lsp.internal.CompletionItemDefaults
> ---@field public items lsp.CompletionItem[]
>
> ---@alias lsp.CompletionResponse lsp.CompletionList|lsp.CompletionItem[]
>
> -- 略
>
> ---@class lsp.CompletionItem
> ---@field public label string
> ---@field public labelDetails? lsp.CompletionItemLabelDetails
> ---@field public kind? lsp.CompletionItemKind
> ---@field public tags? lsp.CompletionItemTag[]
> ---@field public detail? string
> ---@field public documentation? lsp.MarkupContent|string
> ---@field public deprecated? boolean
> ---@field public preselect? boolean
> ---@field public sortText? string
> ---@field public filterText? string
> ---@field public insertText? string
> ---@field public insertTextFormat? lsp.InsertTextFormat
> ---@field public insertTextMode? lsp.InsertTextMode
> ---@field public textEdit? lsp.TextEdit|lsp.InsertReplaceTextEdit
> ---@field public textEditText? string
> ---@field public additionalTextEdits? lsp.TextEdit[]
> ---@field public commitCharacters? string[]
> ---@field public command? lsp.Command
> ---@field public data? any
> ---@field public cmp? lsp.internal.CmpCompletionExtension
> ```
> https://github.com/hrsh7th/nvim-cmp/blob/ae644feb7b67bf1ce4260c231d1d4300b19c6f30/lua/cmp/types/lsp.lua

### 試し書き
今までを踏まえて、最小限で書いて試してみます。

#### その 1
```lua
local cmp = require('cmp');
cmp.register_source('unko', {
    complete = function(_, _, callback)
        callback({
            { label = '💩' },
            { label = '💩💩' },
            { label = 'unkown' },
        })
    end
})
cmp.setup.cmdline('@', {
    mapping = cmp.mapping.preset.cmdline(),
    sources = cmp.config.sources({
        { name = 'unko' }
    }),
})
```

![画面キャプチャ](/img/2024-08-28/test-1_unko_completion.png "Neovimで実行した結果-1")
💩ｷﾀ━━━━(ﾟ∀ﾟ)━━━━!!

#### その 2

ちょっとまじめになります。

```lua
local cmp = require('cmp');
cmp.register_source('gin-action', {
    complete = function(_, _, callback)
        local items = {}
        for _, nmap in ipairs(vim.api.nvim_buf_get_keymap(0, 'n')) do
            local action = string.match(nmap.lhs, '<Plug>%(gin%-action%-(%S+)%)')
            if action then
                table.insert(items, { label = action })
            end
        end
        callback(items)
    end
})
cmp.setup.cmdline('@', {
    mapping = cmp.mapping.preset.cmdline(),
    sources = cmp.config.sources({
        { name = 'gin-action' }
    }),
})
```

![画面キャプチャ](/img/2024-08-28/test-2_gin-cation_completion.png "Neovimで実行した結果-2")
ｷﾀｷﾀｷﾀｷﾀ━━━━(ﾟ∀ﾟ)━━━━!!

### gin.vim 用コードの完成

大枠はできました。あとは細かな調整です。
いくつか気付くことがあります。

- 補完ポップアップの位置が左端にあり、入力位置と合っていない
- リストがソートされていない（実際にはファジーマッチによるランク付けでソートされている）

位置調整については、`:help cmp-config.window.completion.col_offset` に載っています。
> ```markdown
> .                                       *cmp-config.window.completion.col_offset*
> window.completion.col_offset~
>   `number`
>   Offsets the completion window relative to the cursor.
> ```

ソートについても、`:help cmp-config.sorting.comparators` で出てきます。
> ```markdown
> .                                               *cmp-config.sorting.comparators*
> sorting.comparators~
>   `(fun(entry1: cmp.Entry, entry2: cmp.Entry): boolean | nil)[]`
>   The function to customize the sorting behavior.
>   You can use built-in comparators via `cmp.config.compare.*`.
> ```
関数を書くか、ビルトインのものを指定できるようです。
[lua/cmp/config/comare.lua](https://github.com/hrsh7th/nvim-cmp/blob/main/lua/cmp/config/compare.lua) に各種関数があるので見て、 `sort_text` を見つけます。
コードを見ると、補完アイテムにソートキーとなる `sortText` をいれる必要がありそうです。

```lua
local cmp = require('cmp')
cmp.setup({
    -- 省略
})
cmp.setup.cmdline({ '/', '?' }, {
    -- 省略
})
cmp.setup.cmdline(':', {
    -- 省略
})
-- gin.vim の action: 用補完
cmp.register_source('gin-action', {
    is_available = function() -- filetype がgin-* の時のみ有効に
        local ft = vim.opt_local.filetype:get()
        if string.match(ft, '^gin%-') then
            return true
        end
        return false
    end,
    complete = function(_, _, callback)
        local items = {}
        -- nmap の lhs が '<Plug>(gin-action*)' のものを抽出
        -- see: https://github.com/lambdalisue/vim-gin/blob/main/denops/gin/action/core.ts#L50-L70
        for _, nmap in ipairs(vim.api.nvim_buf_get_keymap(0, 'n')) do
            local action = string.match(nmap.lhs, '<Plug>%(gin%-action%-(%S+)%)')
            if action then
                table.insert(items, {
                    label = action,
                    sortText = action,
                })
            end
        end
        callback(items)
    end
})
cmp.setup.cmdline('@', { -- vim.fn.input() 時の補完
    mapping = cmp.mapping.preset.cmdline(),
    sources = cmp.config.sources({
        { name = 'gin-action' }
    }),
    sorting = {
        comparators = { cmp.config.compare.sort_text }
    },
    window = {
        completion = {
            col_offset = 8, -- "action: " の8文字分
        },
    },
})
```

もう少し改造してアイコン(emoji)を付けて遊んだり、ドキュメントが出るようにしたりして完成です。

<figure><img src="/img/2024-08-28/gin-actions-completion.png" alt="画面キャプチャ">
<figcaption>改造後のgin actionの補完をしている様子</figcaption></figure>

### 汎用化に向けてさらに調査

ほぼ完成したのですが、[gin.vim] 専用です。もう少し発展させたいところです。

> \#### ネイティブな補完を呼び出せば……？  
> やり方が分かりませんでした。
> Vim力不足によりいったん諦め。

の調査に戻ります。

が、僕だけで調査には限界を感じ、[vim-jp Slack] へ助けを求めます。

結果。

> ```vim
> cmap <F7> <C-\>eTest()<CR>
> function Test()
>   return getcompletion(getcmdline(), getcmdcomptype())
> endfunction
> ```
> で、
> ```vim
> call input("prompt", "./", "file")
> ```
> プロンプトを出し `<F7>` すると、ファイルが出てｷﾀ━━━━(ﾟ∀ﾟ)━━━━!!
>
> _―― 試行錯誤中の呟きより_

(上のコードには誤りがあり、`getcmdcomptype()` ではなく `getcmdcomp**l**type()` が正解です)

> gin.vim のアクション一覧も出た。  
> 勝てる！  
> 汎用的な補完が書けそうな気がしてきた。
>
> _―― 試行錯誤中の呟きより_

<figure><img src="/img/2024-08-28/dir-completion.gif" alt="動画">
<figcaption>テスト - 1: `input()` の `dir` 補完</figcaption></figure>
<figure><img src="/img/2024-08-28/file-completion.gif" alt="動画">
<figcaption>テスト - 2: `input()` の `file` 補完</figcaption></figure>

### 汎用プラグインの作成

汎用的なものが作れそうなので、せっかくなら皆が使えるようにしようとプラグイン作成に乗り出します。

幸い、[List of sources · hrsh7th/nvim-cmp Wiki](https://github.com/hrsh7th/nvim-cmp/wiki/List-of-sources) に参考になるものがたくさんあるので幾つか見ていきます。

- `lua/*/init.lua` に補完ソースとなるコードを書く
- `after/plugin/*.lua` に補完ソースを登録するコードを書く

の2点を掴んだら、後は書くだけです。

### gin.vim 用コードとの両立

汎用的に作ったおかげで対応範囲が広がりましたが、生成された補完リストに物足りなさを感じ始めます。
これまでに作った gin.vim 専用コードとの両立ができないか考え、除外設定を設けることにします。


と、こんな感じでできあがったプラグインです。

## 最後に

大事なことなので、もう一度

- Vim の `:help` は大事
- プラグインの `:help` も大事
- 「設定させていただきありがとうございます」
- [vim-jp Slack] 楽しい

[nvim-cmp]: https://github.com/hrsh7th/nvim-cmp "GitHub - hrsh7th/nvim-cmp: A completion plugin for neovim coded in Lua."
[gin.vim]: https://github.com/lambdalisue/vim-gin "GitHub - lambdalisue/vim-gin: 🥃 Gin makes you drunk on Git"
[cmp-cmdline]: https://github.com/hrsh7th/cmp-cmdline "GitHub - hrsh7th/cmp-cmdline: nvim-cmp source for vim's cmdline"
