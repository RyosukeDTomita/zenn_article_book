---
title: "Dockerのベストプラクティス"
---

# Dockerのベストプラクティス

## イメージの選び方

### ベースコンテナイメージの種類

- 適切なベースイメージを選ぶことでイメージを軽くするのが重要。

> [サイボウズの研修資料](https://speakerdeck.com/cybozuinsideout/docker2020?slide=31)このあたりみておくと良い。

### イメージタグ

- slim: 必要最小限のパッケージのみを含むことを示している。
- bullseye: Debianの安定版リリースに基づく。
- alpine: Alpine Linux

> [!NOTE]
> おすすめはdebian-slim説。alpineは使い勝手が良くない。

### イメージの探せるサイト

- [Docker Hub](https://hub.docker.com/)
- [Amazon ECR Public Gallery](https://gallery.ecr.aws/)
- [Google Container Registry](https://console.cloud.google.com/marketplace/product/google-cloud-platform/container-registry?hl=ja&project=geolocator-339315)
- Azure Container Registry

---

## イメージを軽くするテクニック

### レイヤーとは

- レイヤー: ファイルシステムの変更セットであり，Dockerコンテナ内のファイルシステムがDockerイメージのビルド中にどのように変更されたのかを表すもの。
- Dockerはレイヤーを積み重ねることでイメージを作成する。
- RUNやCOPYを実行するたびにread-onlyなレイヤーが積み重なっていきイメージサイズが大きくなる。 --> RUNをひとつにまとめることで作成されるレイヤーを少なくし，イメージを軽くできる。

### multe-stage build

- ビルド用のコンテナで作成した実行ファイルを実行用コンテナにコピーして実行する。
- ビルドに使用した中間イメージは最終的なイメージへ保存されないため，イメージサイズが小さくなる。

> [マルチステージビルドについて調べる](https://zenn.dev/masaruxstudy/articles/d85f6c1af3bf65)

```
# ビルド環境
FROM node:20 as build
WORKDIR /app
COPY . .
RUN npm install && npm run build


# プロダクション環境
FROM public.ecr.aws/eks-distro-build-tooling/eks-distro-minimal-base-nginx:latest-al23
COPY --from=build /app/build /usr/share/nginx/html # asで指定した名前の中間イメージからビルドしたアプリを取得
COPY nginx.conf /etc/nginx/conf.d/default.conf

# rootユーザ以外でサービスを起動するために最低限の権限を付与
USER root
RUN mkdir -p /var/log/nginx && chown -R nginx:nginx /var/log/nginx; touch /run/nginx.pid && chown -R nginx:nginx /run/nginx.pid
EXPOSE 80
USER nginx
CMD ["nginx", "-g", "daemon off;"]
```

### 不要ファイルを減らす

- 不要ファイルの削除

```
RUN apt updatte && apt install -y nginx && rm -rf /var/lib/lists
```

- インストール時に推奨パッケージを入れない

```
RUN apt update && apt install -y nginx --no-install-recommends
```

### diveを使って無駄ファイルを探す

- [dive](https://github.com/wagoodman/dive)
- イメージのレイヤごとに確認できる。
- Docker Desktopを使ったほうがわかりやすいかも

```shell
export DIVE_VERSION=$(curl -sL "https://api.github.com/repos/wagoodman/dive/releases/latest" | grep '"tag_name":' | sed -E 's/.*"v([^"]+)".*/\1/')
curl -OL https://github.com/wagoodman/dive/releases/download/v${DIVE_VERSION}/dive_${DIVE_VERSION}_linux_amd64.deb
sudo apt install ./dive_${DIVE_VERSION}_linux_amd64.deb
```

```shell
dive react-app:latest
```

### キャッシュを意識する

- Dockerfile内の命令は前のレイヤーからの変更を表現するため，各レイヤーはキャッシュされる。
- COPY等でファイルやディレクトリに変更があるとそれ以前のキャシュが無効になるので，COPYを一番下に書くことで無駄にキャッシュの再作成を防ぐことができる。
- pythonとかならrequirements.txtだけ先にCOPYしてソースは全体は一番最後にしてもいいかも。

### .dockerignoreを利用する

- dockerビルド時に対象とするディレクトリに含まれるものすべてをメモリ空間に読み込んでしまう。 --> 無駄なファイル.dockerignoreに記述して読み込まないようにする。

******

## Dockerfileを書く時のtips

### デフォルトシェルをbashにする

```
SHELL ["/bin/bash", "-c"]
```

### バージョン情報

- インストールするアプリ等はlatest等を使わずにバージョンを指定してやる。
- イメージ自体のバージョンもDockerfileのハッシュとかにしてもいいかも。ECRにpushするときにはcommit hashとかにしてた。

> [!NOTE]
> バージョンを書いておくことでキャッシュの有効活用にもつながる。aptでバージョンなしでインストールしていると最新ライブラリが出るたびにそれ以前のレイヤーのキャッシュが使えなくなる。

### 見やすい書き方を工夫する

```
RUN apt update && apt install -y \
  bzr \
  cvs
```

- ヒアドキュメントを使う方法。&&とか使うとうまくいかないかも

> [stage-1 4/4] RUN <<EOF (mkdir -p /var/log/nginx...):
> 0.239 /bin/sh: -c: line 2: syntax error near unexpected token `&&'
> 0.239 /bin/sh: -c: line 2:`  && chown -R nginx:nginx /var/log/nginx'

```
RUN <<EOF
mkdir -p /var/log/nginx
chown -R nginx:nginx /var/log/nginx
touch /run/nginx.pid
chown -R nginx:nginx /run/nginx.pid
EOF
```

### escape

- Dockerfileのデフォルトのエスケープ文字は\だが，これはWindowsのディレクトリの区切り文字と被るのでWindowsを使う場合には変更したほうが良い。
- Dockerfileの冒頭に以下のように記述する。

```
# escape=`
```

### LABEL

- 誰が作成/管理しているか，イメージの作成日時等を記載する。
- 組織やプロジェクトにおけるイメージを管理するのに使うためにDockerfileの冒頭に記載する。

```
LABEL vendor=hoge\
date="2023-12-5"\
com.example.version.is-production=""
```

### CMDとENTORYPOINT

- 組み合わせるとデフォルトの動作と引数を渡したときの動作を記述できる。
- CMDにデフォルト引数を書き，ENTRYPOINTにコマンドの固定部分を書く。

> [ENTORYPOINTは必ず実行，CMDはデフォルト引数](https://pocketstudio.net/2020/01/31/cmd-and-entrypoint/)

```
ENTRYPOINT ["ping", "-c", "3"]
CMD ["1.1.1.1"] # デフォルトでは1.1.1.1にpingするがコンテナ実行時に引数を渡されたらそこにpingする。
```

```shell
docker container run myping:latest # 1.1.1.1にping
docker container run myping:latest 8.8.8.8
```

### ARGとENV

#### ARG
>
> [Dockerfile ARG入門](https://qiita.com/nacika_ins/items/cf8ceb20711bd077f770)
Dockerfile内でのみ使用できる変数。

```
ARG PYTHON_VERSION="3.9.6"
RUN pyenv install ${PYTHON_VERSION}
```

- docker-compose.yml内だとargsで定義できる。

```
version: '3'

services:
  pytest-env:
    build:
      args:
        PYTHON_VERSION: "3.9.6"
      context: ./
      dockerfile: Dockerfile
      image: pytest-env:latest
```

```
# Dockerfileで再定義する。
ARG PYTHON_VERSION
```

#### ENV

- ENTRYPOINTやCMDにも使える。
- [ENTRYPOINTやARGの使用](https://qiita.com/hokutoasari/items/9043ed26402d6860d0a5)によってdocker-composeからENVIRONMENTからの上書きや代入がbuild時にはできない。run時には上書きされる。
- マルチステージビルドをしても両方で有効っぽい。

```
version: '3'

services:
  pytest-env:
    build:
      context: ./
      dockerfile: Dockerfile
      environment:
        PYTHON_VERSION: "3.9.6"
      image: pytest-env:latest
```

### VOLUMESを使う

- ローカルとdockerでディレクトリの共有ができるのでソースを変更するたびにビルドし直さなくて良くなる。
- イメージ作成後にやる。
- [docker-compose.yml](./docker-compose.md)で書いたほうが楽なのでこっちを参照されたし。
- volumeの共有には.dockerignoreは効かない。
- デバックでの使用にとどめたほうがいいかも。
- ソースを変更してもサービスを再起動するまでは変更が反映されない可能性がある。

#### ローカルディレクトリをマウントする

```shell
docker run -it --mount type=bind,source=./,target=/usr/local/app flask-app "/bin/bash"
```

> [!WARNING]
> volumeの共有はコンテナに対して行われるため，`docker compose up`でビルドしてコンテナを起動している状態で`docker run -it イメージ名 "/bin/bash"`を実行しても共有フォルダは見られない(別のサービスを立ち上げてbashを起動している扱いになるみたい。)
> docker exec -it コンテナ名 "/bin/bash"なら見れる。

> [!NOTE]
> volumesの共有とDockerfileのCOPYは併用可能である。
> DockerfileのCOPYで配置したソースをvolumeの共有を使ってローカルから編集した場合にはきちんとソースが変更されていた。

#### volumeを作成して使う

```shell
docker volume create test-volume
docker run -it --volume test-volume:/var/hoge flask-app "/bin/bash"
# docker内部からファイルを編集してみる
echo "test" > /var/hoge/test

# ローカルから見る。
docker volume inspect test-volume
[
    {
        "CreatedAt": "2024-01-21T03:07:09+09:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/test/_data",
        "Name": "test",
        "Options": null,
        "Scope": "local"
    }
]
sudo vim /var/lib/docker/volumes/test_data
```

### データの永続化

- Volume機能を使うのが良さそう。compose.yamlを使うと楽。

### daemonを使用しないでサービスを起動

```
CMD ["nginx", "-g", "daemon off"]
```

### docker-entrypoint.sh

- サービス起動コマンドが長い場合にはdocker-entrypoint.shに記載してENTRYPOINTに記載するのが主流みたい。

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

## Reference

- [コンテナを自動的に開始](https://docs.docker.jp/config/container/start-containers-automatically.html)
- [DockerFileのサンプル](https://github.com/RyosukeDTomita/apache2_docker_test)
