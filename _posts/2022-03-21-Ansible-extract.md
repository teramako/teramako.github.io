---
title: Ansible extractフィルターを完全に理解した
date: 2022-03-21 23:40:00+09:00
tags: Ansible
---

Ansible のフィルターに `extract` がある。
今までイマイチ使い方を理解してなかったが、ソースを見たりしてようやく理解に達した。

`extract` の使い方は [Extracting values from containers][extract] に載っている。
が、ほぼ `map` との組み合わせを前提に作られているのため、単体としての動きが分かり難い。

## 単体で使う
`map` と組み合わせずに使うとどうなるか。

<!-- {% raw %} -->
```yaml
- hosts: localhost
  vars:
    container:
      a: AAA
      b: BBB

  tasks:
    - debug:
        msg: '{{ "a" | extract(container) }}'
```
<!-- {% endraw %} -->

```
TASK [debug] *************
ok: [localhost] => {
    "msg": "AAA"
}
```
`container` からフィルターとして与えたキー(`"a"`)の値を取り出す動作となる。

ただ、これって、{% raw %}`{{ container.a }}` や `{{ container['a'] }}`{% endraw %} と変わらず、何の意味がなるの？ってなってしまう。

## mapと組み合わせる

ところが、`map`との組み合わせると有用性が出てくる。
フィルターとして与えたキーのリストから、それぞれの値を取り出すことができるようになる。

<!-- {% raw %} -->
```yaml
tasks:
  - debug:
      msg: '{{ ["a","b"] | map("extract", container) | list }}'
```
<!-- {% endraw %} -->

```
TASK [debug] *************
ok: [localhost] => {
    "msg": [
        "AAA",
        "BBB"
    ]
}
```

### 各ホストのファクト値をまとめて取り出す
各ホストで実行したタスクの結果を最後にまとめて出力させてみる。
<!-- {% raw %} -->
```yaml
- hosts: host1, host2
  tasks:
    - command: whoami
      register: whoami_result
        
  post_tasks:
    - debug:
        msg: |-
          {{ ansible_play_hosts |
              map("extract", hostvars, ["whoami_result", "stdout"]) |
                list }}'
      run_once: true
```
<!-- {% endraw %} -->

各playで実行したホストのリストは`ansible_play_hosts`にある。また、`hostvars`から各ホスト名をキーにしたdictにファクト情報がある。
`hostavars[ホスト名].whoami_result.stdout` を取り出すには、上記のような記述になる。

全てのホストでタスクが成功していたら、次のタスクを実行。みたいのに使えそうである。

## json_query の方が楽かも^^;

ここまで書いておいてなんだけど、`json_query`を使ったほうが楽かもしれない。

<!-- {% raw %} -->
```yaml
    - debug:
        msg: |-
          {{ hostvars | json_query("*.whoami_result.stdout") }}
      run_once: true

```
<!-- {% endraw %} -->


[extract]: https://docs.ansible.com/ansible/2.9/user_guide/playbooks_filters.html#extracting-values-from-containers "Extracting values from containers - Filtrers - Ansible Documentation"
