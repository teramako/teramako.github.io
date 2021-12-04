---
title: Ansibleタスクの記述順序
date: 2021-12-04 21:30:00+09:00
tags: ansible
---

## 今まで

今まで、Ansibleのタスクの記述順序は、以下のような順序で書いていた。
1. name
2. action
   - +引数
3. register
4. when
5. loop
6. その他

とにかく、最初に名前で次にアクションだった。

{% raw -%}
```yaml
- name: a task
  debug:
    msg:
     - "foo: {{ item.foo }}"
     - "bar: {{ item.bar }}"
  when: item.enabled
  loop: "{{ list }}"
  tags: [never, debug]
```
{% endraw -%}

↑↑のようなタスクくらいなら全く問題ないが、少し複雑なものを書き始めて1タスクが長くなると、忘れた頃に `when` や `loop` が出てくる。
特に、`block` で複数タスクをまとめたりすると、本当に忘れた頃に `when` が出てくる。
慌てて最初から読み直すことになり手間だ。

また、blockでは、blockに掛かる `when` なのか、block中のタスクの`when`なのか分からなくなることが出てきた。

## これから

ということで、記述順序を変えてみることにした。
`loop`と`when`をactionより先に書くのだ。

1. name
   - ここは変わらない。まずはタスク名
2. loop
   - 次にループ系。
   - ループに使用する変数`item`が後続で出てくる可能性を示すために先に書く
3. when
   - `when` はそもそものタスクを実行するかどうかに関わってくる。よって、actionより先に記述
   - また、loopの`item`変数を使うこともあるので、loopの後に記述する
4. action
   - +引数
5. register
6. その他

{% raw -%}
```yaml
- name: a task
  loop: "{{ list }}"
  when: item.enabled
  debug:
    msg:
     - "foo: {{ item.foo }}"
     - "bar: {{ item.bar }}"
  tags: [never, debug]

- name: block tasks
  when: ...
  block:
    - name: Copy files to remote server
      loop: ...
      when: ...
      copy:
        src: ...
        dest: ...

  always:
    - name: ...
      action: ...
```
{% endraw -%}

`loop`や`when`以外のキーワードはどうするか様子見中。

