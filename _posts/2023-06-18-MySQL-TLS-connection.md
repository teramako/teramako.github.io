---
title: MySQL TLS(SSL)通信をする
date: 2023-06-18
tags: MySQL
mermaid: true
---

MySQLの暗号化通信として、`openssl` コマンドで証明書を作る話。

## 問題点
[mysql_ssl_rsa_setup] コマンドを使えば簡単に証明書を作ることができる。
しかし、作成されるサーバー証明書の Subject 値が <code>/CN=MySQL_Server_<var>version</var>_Auto_Generated_Server_Certificate</code> といった値になってしまい、`x509` のサーバー証明書の検証でエラーになってしまう。
使用できるのは、`--ssl-mode=VERIFY_CA` のCA証明書の検証までで、`--ssl-mode=VERIFY_IDENTIFY` が通らない。

## 証明書作成
ということで、横着せずに手動で `openssl` コマンドを叩いて、鍵や証明書を作成していく。

### CA証明書作成

まずは、CA証明書から作る。

1. 秘密鍵の作成
    ```console
    $ openssl genrsa -out ca-key.pem 2048
    Generating RSA private key, 2048 bit long modulus
    .............+++
    ..................+++
    e is 65537 (0x10001)
    ```
2. <ruby>CSR<rp>(</rp><rt>証明書 署名リクエスト</rt><rp>)</rp></ruby>(Certificate Signing Request)作成
    ```console
    $ openssl req -new -out ca.csr -key ca-key.pem -subj "/CN=MySQL CA" -nodes -out ca.csr
    ```
    - `-subj` に指定するSubject値は、MySQLユーザーのSSL設定の`REQUIRE ISSUER ...`に影響してくる。
      指定したい場合はメモっておくと良い。
3. CA用X509v3拡張ファイル作成
    ```bash
    cat <<EOF > ca.csx
    basicConstraints = critical, CA:TRUE
    keyUsage = cRLSign, keyCertSign
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid:always, issuer
    EOF
    ```
    - 各項目の意味は `man x509v3_config` でmanマニュアルを見るか、 [x509v3_config] を見ると良い。
4. CA証明書作成
    ```console
    $ openssl x509 -req -days $((365 * 10)) -signkey ca-key.pem -in ca.csr -extfile ca.csx -out ca.pem
    Signature ok
    subject=/CN=MySQL CA
    Getting Private key
    ```

### サーバー証明書作成

作成したCA証明書を基にサーバー証明書を作っていく。

1. 秘密鍵作成
    ```console
    $ openssl genrsa -out server-key.pem 2048
    Generating RSA private key, 2048 bit long modulus
    ........................................................................................................................................+++
    ...............+++
    e is 65537 (0x10001)
    ```
2. サーバー証明書用のCSR作成
    ```console
    $ openssl req -new -key server-key.pem -subj "/C=JP/ST=Tokyo/O=teramako/CN=MySQL Server Certificate" -nodes -out server.csr
    ```
    - Subjectの<ruby>CN<rp>(</rp><rt>Common Name</rt><rp>)</rp></ruby>には、サーバーのホスト名を入れると良いが、後続の<ruby>SAN<rp>(</rp><rt>Subject Alternative Name</rt><rp>)</rp></ruby>で設定するのでテキトウに
3. サーバー用X509v3拡張ファイル作成
    ```bash
    cat <<EOF > server.csx
    basicConstraints = CA:FALSE
    keyUsage = digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid, issuer
    subjectAltName = DNS:localhost, IP:127.0.0.1, DNS:mysql-source, IP:172.28.0.2
    EOF
    ```
    - <ruby>SAN<rp>(</rp><rt>Subject Alternative Name</rt><rp>)</rp></ruby>となる`subjectAltNameには、クライアントから接続先として使用されるであろうホスト名やIPアドレスを入れておく
        - ローカルホストからも使用できるように `localhost` や `127.0.0.1` を入れておくと良い
        - 複数サーバー（更新用と参照用(replica)など）の場合は、このSAN値を変えて証明書を作成していくと、同CA証明書ファイルでそれぞれにアクセスできる(はず)
    - 一応、`keyUsage`や`extendedKeyUsage`でサーバー証明書の用途であることを明示しておく。
4. サーバー証明書作成
    ```console
    $ openssl x509 -req -days $((365*10)) -CA ca.pem -CAkey ca-key.pem -CAcreateserial -in server.csr -extfile server.csx -out server-cert.pem
    Signature ok
    subject=/C=JP/ST=Tokyo/O=teramako/CN=MySQL Server Certificate
    Getting CA Private Key
    ```
    - 初回なので、`-CAcreateserial` オプションを指定してテキトウなシリアル番号の生成と付与を行う
        - CA証明書に `ca.pem` を指定しているので、同ディレクトリに `ca.srl` ファイルが作成される。
        - 以降、CA証明書を使用して新たな証明書を作成する場合は、`-CAserial ca.srl` オプションでシリアル番号が入ったファイルを指定すると良い。
5. 確認
    ```console
    $ openssl x509 -text -noout -in server-cert.pem
    Certificate:
        Data:
            Version: 3 (0x2)
            Serial Number:
                e0:b4:1c:0c:2c:e7:74:4a
        Signature Algorithm: sha256WithRSAEncryption
            Issuer: C=JP, ST=Tokyo, O=teramako, CN=MySQL CA
            Validity
                Not Before: Jun 18 11:34:49 2023 GMT
                Not After : Jun 15 11:34:49 2033 GMT
            Subject: C=JP, ST=Tokyo, O=teramako, CN=MySQL Server Certificate
            Subject Public Key Info:
                Public Key Algorithm: rsaEncryption
                    Public-Key: (2048 bit)
    (略)
                    Exponent: 65537 (0x10001)
            X509v3 extensions:
                X509v3 Subject Alternative Name:
                    DNS:localhost, IP Address:127.0.0.1, DNS:mysql-source, IP Address:172.28.0.2
        Signature Algorithm: sha256WithRSAEncryption
    (略)
    ```
    - X509v3 extensions の SAN 値に狙い通りのホスト名やIPアドレスが入っていることを確認する

### クライアント証明書

1. 秘密鍵作成
    ```console
    $ openssl genrsa -out client-key.pem 2048
    Generating RSA private key, 2048 bit long modulus
    ..........................................................................+++
    ......................................................................................+++
    e is 65537 (0x10001)
    ```
2. クライアント用CSR作成
    ```console
    $ openssl req -new -key client-key.pem -subj "/C=JP/ST=Tokyo/CN=MySQL Client Certificate" -nodes -out client.csr
    ```
    - `-subj` のSubjectはMySQLユーザーのSSL設定の`REQUIRE SUBJECT ...`に影響してくる。
      使用したい場合は、メモっておくと良い。
3. クライアント用X509v3拡張ファイル作成
    ```bash
    cat <<EOF > client.csx
    basicConstraints = CA:FALSE
    keyUsage = digitalSignature, keyAgreement
    extendedKeyUsage = clientAuth
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid, issuer
    EOF
    ```
    - 一応、`keyUsage`や`extendedKeyUsage`でクライアント証明書の用途であることを明示しておく。
4. 証明書作成
    ```console
    $ openssl x509 -req -days $((365*10)) -in client.csr -CA ca.pem -CAkey ca-key.pem -CAserial ca.srl -out client-cert.pem
    Signature ok
    subject=/C=JP/ST=Tokyo/CN=MySQL Client Certificate
    Getting CA Private Key
    ```

## テスト

### MySQLユーザー作成
テキトウにユーザーを作る。
```console
mysql@server> create user ssl_user1 identified by 'P@ssw0rd' require x509;
Query OK, 0 rows affected (0.00 sec)

mysql@server> select @@hostname,User,Host,ssl_type,x509_issuer,x509_subject from mysql.user where user = 'ssl_user1';
+--------------+-----------+------+----------+-------------+--------------+
| @@hostname   | User      | Host | ssl_type | x509_issuer | x509_subject |
+--------------+-----------+------+----------+-------------+--------------+
| mysql-source | ssl_user1 | %    | X509     |             |              |
+--------------+-----------+------+----------+-------------+--------------+
1 row in set (0.00 sec)
```

### 接続テスト

```console
$ mysql -u ssl_user1 -p"P@ssw0rd" -h mysql-source --ssl-ca ca.pem --ssl-cert client-cert.pem --ssl-key client-key.pem
(略)
mysql@client> \s
--------------
mysql  Ver 14.14 Distrib 5.7.41, for Linux (x86_64) using  EditLine wrapper

Connection id:          1260
Current database:
Current user:           ssl_user1@172.28.0.3
SSL:                    Cipher in use is DHE-RSA-AES128-GCM-SHA256
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server version:         5.7.41-log MySQL Community Server (GPL)
Protocol version:       10
Connection:             mysql-source via TCP/IP
Server characterset:    latin1
Db     characterset:    latin1
Client characterset:    latin1
Conn.  characterset:    latin1
TCP port:               3306
Uptime:                 1 hour 1 min 49 sec

Threads: 2  Questions: 2997  Slow queries: 0  Opens: 130  Flush tables: 1  Open tables: 123  Queries per second avg: 0.808
--------------

mysql>
```

SSL接続のセッションも確認してみる
```console
mysql@server> select s1.conn_id,s1.user,s1.db,s1.command,s1.state,s1.time,s2.* from sys.session s1 inner join sys.session_ssl_status s2 on s1.thd_id = s2.thread_id;
+---------+----------------------+------+---------+--------------+------+-----------+-------------+---------------------------+---------------------+
| conn_id | user                 | db   | command | state        | time | thread_id | ssl_version | ssl_cipher                | ssl_sessions_reused |
+---------+----------------------+------+---------+--------------+------+-----------+-------------+---------------------------+---------------------+
|    1260 | ssl_user1@172.28.0.3 | NULL | Sleep   | NULL         |  178 |      1286 | TLSv1.2     | DHE-RSA-AES128-GCM-SHA256 | 0                   |
|    1271 | root@localhost       | sys  | Query   | Sending data |    0 |      1297 |             |                           | 0                   |
+---------+----------------------+------+---------+--------------+------+-----------+-------------+---------------------------+---------------------+
2 rows in set (0.02 sec)
```


[x509v3_config]: https://www.openssl.org/docs/manmaster/man5/x509v3_config.html
[mysql_ssl_rsa_setup]: https://dev.mysql.com/doc/refman/8.0/en/mysql-ssl-rsa-setup.html "MySQL :: MySQL 8.0 Reference Manual :: 4.4.3 mysql_ssl_rsa_setup — Create SSL/RSA Files"
