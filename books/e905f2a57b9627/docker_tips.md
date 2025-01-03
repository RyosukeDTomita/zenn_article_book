---
title: "Dockerのちょっと役立つコマンド"
---

## Build Cacheを消す

溜まってくるとディスクを消費するので定期的に消すと良い。

```shell
docker builder prune
```

---

## 古いリソースをまとめて消す

- まとめてプロセスを止める。

```shell
docker stop $(docker ps -a -q)
```

- まとめてイメージ削除

```shell
docker rmi $(docker images -q) -f
```

## inspect

- 詳細が見れる。

```shell
docker image inspect <イメージ名>
docker container inspect <コンテナ名>
docker volume inspect <ボリューム名>
docker network inspect <ネットワーク名>
```

### バックグラウンドで起動したコンテナの出力を確認する

- container logsを使う。

```shell
docker container logs <コンテナ名>
```

- Docker DesktopのContainersタブからも出力は確認できそうだが，2023/01/22現在ではすべてのコンテナが表示されていない。 --> Docker Desktopを起動してからdocker buildすると出てきた。

### docker cp

- 作成したコンテナからファイルやディレクトリを取得できる。

```shell
docker cp react-app-container:/usr/share/nginx/html build
```

---

### ヒアドキュメントを使う

複数行のRUNを書くときに見やすくなる

```
# ヒアドキュメントを使う書き方
RUN <<EOF bash -ex
mkdir -p /var/log/nginx
chown -R nginx:nginx /var/log/nginx
touch /run/nginx.pid
chown -R nginx:nginx /run/nginx.pid
EOF
```

:::message
`bash -ex`をつけることでエラーが発生した際に途中でビルドを止めたり，実行するコマンドを表示してくれるようになる。
詳しくは[ヒアドキュメントの使い方](https://zenn.dev/sigma_tom/articles/d7fe76cd063320)を参照
:::

:::message alert
&&とヒアドキュメントは同時に使えない。

```
[stage-1 4/4] RUN <<EOF (mkdir -p /var/log/nginx...):
0.239 /bin/sh: -c: line 2: syntax error near unexpected token `&&'
0.239 /bin/sh: -c: line 2:`  && chown -R nginx:nginx /var/log/nginx'
```

:::

### escape

- Dockerfileのデフォルトのエスケープ文字は\だが，これはWindowsのディレクトリの区切り文字と被るのでWindowsを使う場合には変更したほうが良い。
- Dockerfileの冒頭に以下のように記述する。

```
# escape=`
```

### LABEL

- 誰が作成/管理しているか，イメージの作成日時等を記載する。
- 組織やプロジェクトにおけるイメージを管理するのに使うためにDockerfileの冒頭に記載する。

---

## かっこいいテクニック

### urlからDockerfileを使う

- githubのurlの場合はそのままだとtxtファイルじゃないのでrawボタンを押してtxtに戻したときのurl(raw.githubusercontent.com)を使う。
- COPY等はうまくいかない時がある。

```shell
docker build https://raw.githubusercontent.com/RyosukeDTomita/react-roulette-frontend/master/Dockerfile
------
 > [stage-1 3/4] COPY nginx.conf /etc/nginx/conf.d/default.conf:
------
context:11
--------------------
   9 |     FROM public.ecr.aws/eks-distro-build-tooling/eks-distro-minimal-base-nginx:latest-al23
  10 |     COPY --from=build /app/build /usr/share/nginx/html
  11 | >>> COPY nginx.conf /etc/nginx/conf.d/default.conf
  12 |     
  13 |     
--------------------
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref 336cfdc9-80d8-4a0b-8eeb-ab7bbd24de39::zv5zv3x0mmzjn5w83izjcb6ev: "/nginx.conf": not found
```

### 標準入力からDockerfileをつくる

```shell
docker build -t myimage:latest -f- . <<EOF
FROM busybox
RUN echo "test" > sample.txt && cat sample.txt
EOF
```

---

## エラーログ

### failed to solve: failed to compute cache key: failed to calculate checksum of ref

- COPYしようとしたファイルパスが間違っていた。

### docker runに失敗するコンテナにshellアクセスしたい時

> [docker runに失敗するコンテナを起動する](https://qiita.com/sigma_devsecops/items/30089aa363c9ac6707be)

現在のentrypointを上書きすることでコンテナを起動できる。

```shell
docker run -it --rm --entrypoint /bin/bash イメージ名 -c "sleep 99999999"
```

---
