---
title: Jekyll 用 docker-compose.yml
---

ローカルでjekyllを試すためにdocker-compose.ymlを書いた。
```yaml
version: '3'
services:
  app:
    image: jekyll/jekyll:pages
    command: jekyll serve --force_polling
    environment:
      - TZ=Asia/Tokyo
    volumes:
      - .:/srv/jekyll
    ports:
      - 4000:4000
```

あとは、`docker-compose up`しておいて、`http://localhost:4000` へアクセスすれば確認しながら投稿できる。
バックグラウンド実行したければ `docker-commpose up -d`で。
```shell_session
$ docker compose up
[+] Running 2/2
 ⠿ Network teramako_default  Created                                                  0.1s
 ⠿ Container teramako-app-1  Created                                                  0.5s
Attaching to teramako-app-1
teramako-app-1  | ruby 2.7.1p83 (2020-03-31 revision a0c7c23c9c) [x86_64-linux-musl]
teramako-app-1  | Configuration file: /srv/jekyll/_config.yml
teramako-app-1  |             Source: /srv/jekyll
teramako-app-1  |        Destination: /srv/jekyll/_site
teramako-app-1  |  Incremental build: disabled. Enable with --incremental
teramako-app-1  |       Generating...
teramako-app-1  |        Jekyll Feed: Generating feed for posts
teramako-app-1  |                     done in 0.504 seconds.
teramako-app-1  |  Auto-regeneration: enabled for '/srv/jekyll'
teramako-app-1  |     Server address: http://0.0.0.0:4000
teramako-app-1  |   Server running... press ctrl-c to stop.

```

