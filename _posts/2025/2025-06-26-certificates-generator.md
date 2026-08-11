---
title: TLS (SSL) 証明書生成 Makefile
date: 2025-06-26T22:00:00+09:00
---

- https://gist.github.com/teramako/9352b4ab5860204b7d9f66119b33060b

TLS(SSL) 証明書を作成する Makefile を作った。

- CA, サーバー証明書、クライアント証明書の生成が可能
- ECDSA な秘密鍵生成にも対応
- PKCS12 フォーマットへ出力が可能
- x509 v3 拡張を使用
  - サーバー証明書には SAN 値の設定が可能

<script src="https://gist.github.com/teramako/9352b4ab5860204b7d9f66119b33060b.js"></script>

