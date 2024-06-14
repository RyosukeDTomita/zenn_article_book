---
title: "docker composeについて"
---

# docker composeについて

## docker-composeとdocker compose

- docker composeがV2らしい。[docker-composは非推奨](https://www.konosumi.net/entry/2023/02/26/142508)
- 推奨ファイル名がdocker-compose.ymlからcompose.yamlに変更になったらしい。

---

## ファイル名

[公式ドキュメント](https://docs.docker.jp/compose/compose-file/index.html)によるとcompose.yamlが推奨。

## docker composeの主要なコマンド
>
> [docker composeの一覧](https://qiita.com/nikadon/items/995c5705ff1171f7484d)
>
### up

```shell
docker compose up # docker-compose.ymlに書かれているすべてのサービスをビルドして起動する。
```

> [!NOTE]
> `docker compose up`はイメージが存在する場合には再度のビルドは行わない。そのため，キャッシュを利用しつつ，イメージをビルドしたい際にはbuildが必用。
> `docker compose up --build`の様に実行するとイメージのビルドをしつつ，コンテナを起動してくれる。

### build

- BuildKitを使いたい場合にはbuildx bakeを使う。

```shell
docker buildx bake
```

### run

- 事前定義していないコマンドを実行するのに使う。

> [docker-compose.ymlにentrypointを記載](https://docs.docker.jp/v1.12/compose/compose-file.html#entrypoint)しているならrunで起動できそう?

- docker-compose.ymlのservice名を使ってコマンドを実行する。

- ENTRYPOINTに/bin/bash -cが指定されているので以下のようにコマンドが実行できる。

```
ENTRYPOINT ["/bin/bash", "-c"]
```

```shell
docker compose run pytest-env "pytest"
```

> [!NOTE]
> docker compose run時にはcompose.ymlとかDockerfileに記載したEXPOSEの設定は反映されない。

### start

- docker-compose.ymlやDockerfileのCMDやentrypointに記載のあるコマンドを使ってコンテナを起動したい時に使う。

### exec

- 起動しているコンテナに対してコマンドを実行するのに使う。
- docker-compose.ymlのservice名を使ってコマンドを実行する。

- ENTRYPOINTに/bin/bash -cがついていても起動しているコンテナに対するコマンドなので/bin/bash -cをつけてコマンドを実行する。

```shell
docker compose exec pytest-env /bin/bash -c "pytest"
```

---

## docker-compose.yml例

```yaml
version: '3'

services:
  app:
    build:
      context: ./
      dockerfile: Dockerfile
    image: flask-app # image名
    container_name: flask-container # docker execで指定するコンテナ名
    volumes:
        - ./:/usr/local/app
    ports:
      - 80:8000 # localport:dockerport # これかくとDockerfileのEXPOSE不要
    command: /usr/local/bin/gunicorn run:app -b 0.0.0.0:8000 --chdir /usr/local/app
```

---

## docker-compose.ymlとdocker network

- 同じdocker-compose.ymlに記載すると同じネットワークに所属する。
- docker-compose.ymlのサービス名(上の例の場合はapp)でアクセスできる。
- この名前解決には`127.0.0.11`が使われる。[ソース](https://dev.classmethod.jp/articles/docker-service-discovery/)

> [!NOTE]
> ローカル端末の127.0.0.1:80から127.0.0.1:8000にアクセスしようとした際にコンテナ内でlocalhost:8000を指定してもうまくいかないのでサービス名を使用する。

---

## memo

- docker-compose.ymlでportsを指定したらDockerfileのEXPOSEはいらないぽい。
- DockerfileにCMD書くよりもdocker-compose.ymlにcommandを書くほうが楽そう。
