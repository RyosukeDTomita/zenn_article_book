---
title: "Docker Networkの使い方"
---


# Docker Network

## ネットワークを作る

```shell
docker network create work-network
```

> [!NOTE]
> networkはイメージやコンテナ削除すると一緒に消える?

---

## ネットワーク間で通信してみる

- --network-alias dbでdocker network内で`db`(network-alias)で名前解決できる。
- 127.0.0.11で名前解決している。

```shell
docker container run --network work-network --network-alias db hands-on-4:1.0
```

```shell
docker container run --network work-network  hands-on-3:1.0 php /work/main.php

# dbにnmapしてみる。
nmap -sS -sV 172.21.0.2
PORT     STATE SERVICE VERSION
3306/tcp open  mysql   MySQL 8.2.0

# 名前解決してみる。
dig db
;; ANSWER SECTION:
db.			600	IN	A	172.21.0.2

;; Query time: 0 msec
;; SERVER: 127.0.0.11#53(127.0.0.11) (UDP)
;; WHEN: Thu Mar 07 12:45:26 UTC 2024
;; MSG SIZE  rcvd: 38

# ipでnmapしてもいける
nmap -sS -sV 172.21.0.2
PORT     STATE SERVICE VERSION
3306/tcp open  mysql   MySQL 8.2.0
MAC Address: 02:42:AC:15:00:02 (Unknown)
```

- docker-composeの場合でも同様で，compose.yamlに記載のあるサービス名を使って名前解決できる。

> [compose.yamlの例](https://github.com/RyosukeDTomita/lua-reverse-proxy-with-flask/blob/1729ae9aaf49528001dd3572a56c3d7d37d08625/compose.yaml#L4)
> [nginx.confで127.0.0.11を指定している](https://github.com/RyosukeDTomita/lua-reverse-proxy-with-flask/blob/1729ae9aaf49528001dd3572a56c3d7d37d08625/reverse_proxy/conf/nginx.conf#L40)
> [luaスクリプト中で指定するipの部分にcompose.yamlのサービス名を指定している](https://github.com/RyosukeDTomita/lua-reverse-proxy-with-flask/blob/1729ae9aaf49528001dd3572a56c3d7d37d08625/reverse_proxy/src/main.lua#L30)

---
