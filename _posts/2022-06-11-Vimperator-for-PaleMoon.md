---
title: Vimperator for Pale Moon v31.0
date: 2022-06-11 02:00+09:00
---

[Pale Moon][PaleMoon]が[Firefoxの旧アドオンの仕組みに対応している][impress]と聞いて。

[Pale Moon用にフォークしているGitHubリポジトリ][vimperator-pm]があったのでclone + Make + インストールしてみた。

インストール自体はできてしまい、おぉ！！と思ったものの、
上記リポジトリは更新が6年前で終わっているし、いろいろ動かなかった。

ということで、エラーコンソールとにらめっこしつつ、フォークして動くようにしたよ。

- [https://github.com/teramako/vimperator-pm](https://github.com/teramako/vimperator-pm)

## Build & Install
以下でビルド
```console
$ cd vimperator-pm
$ (cd vimperator/; make xpi)
Building XPI...
Including en-US documentation
Including ja documentation
Replacing UUID, VERSION, DATE and UPDATEURL tags
Packaging up vimperator-3.15.0.xpi
SUCCESS: ../downloads/vimperator-3.15.0.xpi

```
downloadディレクトリに xpi ファイルが作成されるので、ドラッグ＆ドロップでインストールできるよ。

（↑↑普段、こんな丁寧な解説は書かないんだけど、自分が久々すぎて戸惑いながらやったのでｗ）


## 雑感
７～８年ぶりくらいにJavaScriptとかXULとかXPCOMを触った。
このあたりはもう完全に<ruby>古<rt>いにしえ</rt></ruby>の技術だね。
JavaScriptは現役とはいえ、Vimperatorの中のJavaScriptは昔のMozilla JSの方言が大量に入っていて今のEMCAScript 6thとは別物と言ってよい。

例えば、こんなコードがわりと頻繁に出てくる。
```javascript
{
    __iterator__: function() {
        for (let [k, v] in Iterator(obj)) {
            // do something
            yield v;
        }
    }
}
```

- `__iterator__`は特殊なメソッドでECMAScriptでいうところの `*[Symbol.iterator]() {}`
- Mozilla JSでは`*`をつけてジェネレータであることを宣言しなくても良い
- `Iterartor(obj)` で `obj` の各キー、および値の配列を返すイテレータを生成する 
- `for-in` で回す！ （`for-of`ではない）

等々

昔は先進的だったのだが……。


[PaleMoon]: https://www.palemoon.org/
[vimperator-pm]: https://github.com/casial/vimperator-pm
[impress]: https://forest.watch.impress.co.jp/docs/news/1409899.html "Firefox派生ブラウザー「Pale Moon」がレガシーアドオンのサポートを復活！ v31.0.0が公開 - 窓の杜"

