---
title: シェルスクリプトのテンプレート
date: 2022-05-02 21:00:00+09:00
tags: shell
---

最近シェルスクリプトの書き方が定まってきた。
僕の場合は、外部コマンドの羅列を順番に実行できれば十分で、プラスαとしてコマンド実行成否を出せれば良い。

## テンプレート
先に全体像。

```bash
#!/bin/bash
## 1. USAGE関数
USAGE() {
cat <<EOF
 $0 [-h] paramA paramB

 Parameter:
   1. paramA: 
   2. paramB:
   
 Options:
   -h : This message
EOF
}
## 2. シェルのオプション設定
set -u -o pipefail

## 3. オプション、パラメーター設定
while getopts :h OPT
do
    case "${OPT}" in
        h) USAGE; exit 1;;
    esac
done
shift $(( OPTIND - 1 ))

## 4. 関数定義
out() {
    local D=$(date +"%Y-%m-%d %H:%M:%S")
    echo "[${D}][$$] $*"
}
cmd() {
    out "exec $@"
    eval "$@"
    local RC=$?
    if (( RC == 0 )); then
        out "✅ Success"
    else
        out "🛑 Failed [RC=${RC}]"
    fi
    return ${RC}
}
taskA() {
    out -i "Start taskA"
    cmd_1
    local rc=$?
    if (( rc == 0 )); then
        out "✅ Success taskA"
    else
        out "🛑 Failed taskA [RC=${RC}]"
    fi
    return $rc
}
taskB() {
    # 略
}

## 5. メイン処理
out "Start"
{
    taskA &&
    taskB
}
RC=$?
if (( RC == 0 )); then
    out "🎉 Successfully completed."
else
    out "Failed [RC=${RC}]"
fi
exit ${RC}
```

## 1. USAGE関数を定義

まず、最初に`USAGE`関数を定義して、スクリプトを説明を入れる。
毎行`echo`するのは面倒なので、`cat`+ヒアドキュメントで雑に作る。 ぱっと見スッキリしていて良い。
```bash
USAGE() {
cat <<EOF
 $0 [-h] paramA paramB

 Parameter:
   1. paramA: 
   2. paramB:
   
 Options:
   -h : This message
EOF
}
```

## 2. シェル設定
`set -u -o pipefail` を設置する

- `-u`: 未定義変数への参照時にエラーにする
- `-o pipefail`: パイプ中の前コマンドのエラーを正しく拾う

`-e`を付けるのも定番らしいが、以下の理由で僕は付けないことが多い。
- コマンドによっては終了コード `0` 以外を正常として扱いたい場合があって、その場合制御が煩雑になる
- エラーになった場合にも後続処理を行いたい(一時ファイルの削除など)

## 3. スクリプトのオプションやパラメータ設定
`-h` のヘルプメッセージを出すオプションだけは作っておく。
後々、オプションを追加したくなることもあるし、とりあえず書いておくと追加が楽になる。
```bash
while getopts :h OPT
do
    case "${OPT}" in
        h) USAGE; exit 1;;
    esac
done
shift $(( OPTIND - 1 ))
```

## 4. 関数を定義する
### out 関数
特定の書式でメッセージ出力をする関数。
とりあえず、日時とPIDくらいは出すようにしておく。
```bash
out() {
    local D=$(date +"%Y-%m-%d %H:%M:%S")
    echo "[${D}][$$] $*"
}
```

一旦は標準出力のみ。後々必要になったら `tee -a`でファイル出力とかを追加する。

### 実行したいコマンドを関数でラップする
コマンド実行の成否で処理が分かれる単位で関数化する。

だいたい以下のような感じになる。
1. 開始メッセージ出力
2. コマンド実行
    - 終了コードを<var>rc</var>に格納しておく
3. 終了メッセージ出力
    - 絵文字を使って成否がぱっと見でわかるようにしておく
4. <var>rc</var>でリターン

```bash
taskA() {
    out -i "Start taskA"
    cmd_1  # ←実行したいコマンド
    local rc=$?
    if (( rc == 0 )); then
        out "✅ Success: taskA"
    else
        out "🛑 Failed: taskA [RC=${rc}]"
    fi
    return $rc
}
taskB() {
    # 略
}
```

### cmd 関数(Optional)
単純にコマンドを一つ実行したいだけのものの羅列になる場合は、以下のような `cmd`関数を用意する。
```bash
cmd() {
    out "exec $@"
    eval "$@"
    local rc=$?
    if (( rc == 0 )); then
        out "✅ Success"
    else
        out "🛑 Failed [RC=${rc}]"
    fi
    return ${rc}
}
```

## 5. メイン処理

上記、関数でラップしたものを `&&` で繋げて実行。
`{}`でグループ化すると全出力をログに書き出すとかの処理追加が楽。

各コマンドを関数でラップしておくのでメイン処理としては非常に単純な構造になる。

```bash
out "Start"
{
    taskA &&
    taskB
}
RC=$?
if (( RC == 0 )); then
    out "🎉 Successfully completed."
else
    out "Failed [RC=${RC}]"
fi
exit ${RC}
```
