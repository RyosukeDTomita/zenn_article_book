---
title: "Docker Networkの使い方"
---


## ネットワークを作る

```shell
docker network create work-network
```

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

:::message
docker-composeの場合は明示的にnetworkを指定しなくても同じyamlファイル内のサービスは同じネットワークに所属する。
名前解決したい際にはcompose.yamlのサービス名を指定する。

```
services:
  reverse_proxy_app:
    build:
      context: ./reverse_proxy
      dockerfile: Dockerfile
    image: lua-reverse-proxy:latest
    container_name: reverse_proxy_container
    ports:
      - 80:80 # localport:dockerport
  redis_app:
    build:
      context: ./redis
      dockerfile: Dockerfile
    image: redis-img:latest
    container_name: redis_container
    ports:
      - 6379:6379 # localport:dockerport
```

この構成の場合reverse_proxy_app内から`rediscli -h redis_app`のようにして名前解決して接続できる。
:::

---
