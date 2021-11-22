---
title: Ansible playbookで日付+タイムゾーンを扱う
date: 2021-11-23 01:30:00+09:00
tags: ansible
---

Ansible playbookを書こうとしてハマっていたのでメモ

WebサービスにアクセスしてJSONデータを取得し、条件に合ったものをSlack通知するAnsible Playbookを作ろうとした。

JSONから得られるデータは、以下のような感じ。

```json
{
  "items": [
    {
      "id": "xxxA",
      "title": "A アイテム",
      "createdAt": "2021-11-22T09:00:00-03:00"
    },
    {
      "id": "xxxB",
      "title": "B アイテム",
      "createdAt": "2021-11-21T09:00:00-03:00"
    }
  ]
}
```

これらから、例えば、作成日が24時間以内のものを抽出したい。さらに、出力は日本時間に直したい。

## 結論

以下のように実行環境とインプット、アウトプットが複雑だったことに起因するのだが、
- 実行環境は python2 でタイムゾーンがUTC
- 入力値のJSONのタイムスタンプにタイムゾーンがあり、実行環境と異なる
- 出力は、日本時間(+09:00)にしたい

これらをするために、かなり複雑になったが**全てUTC換算したUNIX時間で扱う**ことにした。

{% raw %}
```yaml
- name: extract new items
  set_fact:
    # UTC換算したUNIX時間でcreatedAtが24時間以内のものを抽出
    new_items: |
      {%- set utc_base = '1970-01-01' | to_datetime('%Y-%m-%d') -%}
      {%- set now_epoch = (now(utc=true) - utc_base).total_seconds() -%}
      {%- set items = [] -%}
      {%- for item in result.items -%}
        {%- set createdAt = (
              item.createdAt | regex_replace('-03:00$','') |
                to_datetime('%Y-%m-%dT%H:%M:%S') - utc_base
            ).total_seconds() + 3 * 60 * 60 -%}
        {%- if createdAt + 24 * 60 * 60 > now_epoch -%}
          {{ items.append(item) }}
        {%- endif -%}
      {%- endfor -%}
      {{ items }}

- name: Notify to slack
  loop: "{{ new_items }}"
  slack:
    token: xxxx
    # "<タイトル> Carated at <%Y-%m-%d %H:%M>" をメッセージにする
    # 実行環境(UTC)から日本時間に直して出力
    msg: |
      {%- set utc_base = '1970-01-01' | to_datetime('%Y-%m-%d') -%}
      {{ item.title }} Created at {{ 
          '%Y-%m-%d %H:%M' | strftime(
            (item.createdAt |
              regex_replace('-03:00$', '') |
              to_datetime('%Y-%m-%dT%H:%M:%S') - utc_base
            ).total_seconds() + (3 * 60 * 60) + (9 * 60 * 60)
          )
      }}
```
{% endraw %}

## ハマっていたこと

で、ハマっていたのは
- python2環境で、'%z'のタイムゾーンを扱えない
- UNIX時間への変換方法
- Slack通知する際には、日本時間(JST+09:00)に直したい

### datetime.strptime が '%z'(タイムゾーン)を扱えない

python2の datetime.strptime() （`to_datetime`フィルターに使用されるメソッド）だが、タイムゾーンを示す `%z` が扱えず、エラーになる。

よって、"2021-11-22T09:00:00-03:00" がそのままでは扱えず、`regex_replace`で除去する必要があった。

```
'2021-11-22T09:00:00-03:00' |
  regex_replace('-03:00$', '')  |
  to_datetime('%Y-%m-%dT%H:%M:%S')
```

### UNIX時間への変換
ローカル環境は python3 だったので、ちょっとハマった。

Python3なら `datetime.dateime.utcnow().timestamp()` と datetime オブジェクトの timestamp() メソッドでUNIX時間が出てくる。
しかし、Python2 には datetime.timestamp() メソッドがない。

どうするかというと、`(now - datetime(1970,1,1)).total_seconds()`のように1970年1月1日との差分から秒にする。

これは、[Filters - Ansible Documentation の"Other Useful Filters"](https://docs.ansible.com/ansible/2.9/user_guide/playbooks_filters.html#id8)にヒントが載っていたのだが、気付かずに自力でどうにかすることになった。

上記'%z'が扱えない部分と組み合わせて、UNIX時間に変換するには
```
(
    ('2021-11-22T09:00:00-03:00' |
      regex_replace('-3:00$', '') |
      to_datetime('%Y-%m-%dT%H:%M:%S'))
    - ('1970-01-01' | to_datetime('%Y-%m-%d'))
).total_seconds()
```

### タイムゾーンの扱い

上記でタイムゾーンを無視して計算していまっているのでUTCになっておらず、計算しにくい。
UTC換算してあげる。
今回は、タイムゾーンが `-03:00` と固定であることが分かっていて良かった。単純に3時間分進めてあげる。
そうでなかったらより面倒になっていたことだろう。

```
(
    ('2021-11-22T09:00:00-03:00' |
      regex_replace('-3:00$', '') |
      to_datetime('%Y-%m-%dT%H:%M:%S'))
    - ('1970-01-01' | to_datetime('%Y-%m-%d'))
).total_seconds()
+ (3 * 60 * 60)
```

これがUTC換算したUNIX時間である。

### 日本時間(+09:00)で出力
出力は `'%Y-%m-%d %H:%M' | strftime(epoch_time)` で行うが、日本時間(+09:00)にしたい。

ここで、`strftime` はpythonのtime.localtime()を使用するので、実行環境のタイムゾーンを踏まえた値になる。
実行環境が同じ日本時間なら考慮不要だが、今回はUTCなので`+ (9 * 60 * 60)` した値になる。

```
'%Y-%m-%d %H:%M' | strftime(
  (
    ('2021-11-22T09:00:00-03:00' |
        regex_replace('-3:00$', '') |
        to_datetime('%Y-%m-%dT%H:%M:%S'))
        - ('1970-01-01' | to_datetime('%Y-%m-%d'))
  ).total_seconds()
  + (3 * 60 * 60)
  + (9 + 60 * 60)
)
```
