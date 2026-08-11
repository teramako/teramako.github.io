---
title: sed -iでシンボリックリンクを編集するとリンクが消える件
date: 2022-01-07 21:30:00+09:00
tags: Linux
---

```console
$ echo "foo file" > foo
$ ln -s foo bar
$ ls -l
total 4.0K
lrwxr-xr-x 1 teramako staff 3  1  7 21:26 bar -> foo
-rw-r--r-- 1 teramako staff 9  1  7 21:29 foo
```
bar ファイルを sed -i で書き換える

```console
$ sed -i 's/foo/bar/' bar
$ ls -l
total 8.0K
-rw-r--r-- 1 teramako staff 9  1  7 21:29 bar
-rw-r--r-- 1 teramako staff 9  1  7 21:29 foo
$ cat bar
bar file
```
すると、bar がシンボリックリンクではなく実ファイルになってしまう。

何故？ということで、GNU sed のソースを読んでみた。
ちなみに、僕はC言語分からん。完全に雰囲気で読んだ。

## とりあえず

```shell
git clone git://git.sv.gnu.org/sed
```

## -i オプションの解析部分(sed.c)
まずは、`sed.c` のコマンドラインオプションを解析しているところから。

```c
int
main (int argc, char **  argv)
{
  // ...
  while ((opt = getopt_long (argc, argv, SHORTOPTS, longopts, NULL)) != EOF)
    {
      switch (opt)
        {
          // ...
          case 'i':
              // ...
              if (optarg == NULL)
              /* use no backups */
              in_place_extension = xstrdup("*");
              
              else if (strchr (optarg, '*') != NULL)
                in_place_extension = xstrdup (optarg)

              else
              {
                  in_place_extension = XCALLOC (strlen (optarg) + 2, char);
                  in_place_extension[0] = '*';
                  strcpy (in_place_extension + 1, optarg)
              }

              break;
        }
    }
}
```
ふむ、ふむ、よく分からんが、`in_place_extension`変数がバックアップファイルとなる名前の元っぽい。
`in_place_extension`を使用している箇所を追いかける。

## sed変換結果出力先(execute.c)
```c
static void 
open_next_file (const char *name, struct input *input)
{
  // ...
  if (in_place_extension)
    {
      // ...
      output_file.fp = ck_mkstemp (&input->out_file_name, tmpdir, "sed",
                                   write_mode);
      // ...
    }
  // ...
}
```
どうやら、`open_next_file` は出力先のファイルポインタを定めているようだ。
で、-i 設定時は `ck_mkstemp`を呼んでいて、雰囲気的に一時ファイルが出力先になるようだ。

## 後処理(execute.c)
```c
static void
closedown (struct input *input)
{
  // ...
  if (in_place_extension && output_file.fp != NULL)
    {
      // ...
      if (strcmp (in_place_extension, "*") != 0)
        {
          char *backup_file_name = get_backup_file_name (target_name);
          ck_rename (target_name, backup_file_name, input->out_file_name);
          free (backup_file_name);
        }

      ck_rename (input->out_file_name, target_name, input->out_file_name);
      // ...
    }
}
```
`closedown`では、開いたファイルの後処理系を担っているようで、-i 時には以下の処理を行っているようだ。
1. バックアップファイル名(-iオプションの引数)が指定されている場合、元ファイルをバックアップファイル名にリネーム
2. `open_next_file`で作った一時ファイルを、元ファイルの名前にリネーム

## 結論

端折りながらもダラダラと書いたが、 総合すると、
1. `-i`のin-placeなsedでは、一時ファイル(`mkostemp`)に変換した結果を書き出す
2. バックアップが必要なら、元ファイルを`rename`
3. 一時ファイルを元ファイルへ`rename`
   
当初、僕はファイルを読み書きモードで開いて書き換えているのでは？と想像していたのだが、蓋を開けてみたら全然違う動きだった。

`rename`、つまり、mvコマンドで上書きしているようなものなので、シンボリックリンクだったものが普通のファイルになってしまう理由も理解でき、満足。

