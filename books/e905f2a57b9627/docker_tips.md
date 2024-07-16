---
title: "Dockerのちょっと役立つコマンド"
---

# Docker tips

## Build Cacheを消す

溜まってくるとディスクを消費するので定期的に消すと良い。

```shell
docker builder prune
```

## 古いリソースの削除する

- まとめてプロセスを止める。

```shell
docker stop $(docker ps -a -q)
```

- まとめてイメージ削除

```shell
docker rmi $(docker images -q) -f
```

## docker runに失敗するコンテナにshellアクセスする

> [docker runに失敗するコンテナを起動する](https://qiita.com/sigma_devsecops/items/30089aa363c9ac6707be)

現在のentrypointを上書きすることでコンテナを起動できる。

```shell
docker run -it --rm --entrypoint /bin/bash イメージ名 -c "sleep 99999999"
```

---

## デバックのヒント

- [デバックノウハウ](https://zenn.dev/suzuki_hoge/books/2022-03-docker-practice-8ae36c33424b59/viewer/3-9-debug-know-how)

---

## エラーログ

### failed to solve: failed to compute cache key: failed to calculate checksum of ref

- COPYしようとしたファイルパスが間違っていた。

---

## メモ

- alpineの代わりに[debian-slim](https://x.com/shibu_jp/status/1791634887071400034)を使おう
