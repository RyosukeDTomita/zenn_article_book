---
title: "docker compose"
---

## docker composeを使用するメリット

- Dockerfileは単独のDockerイメージを作るための設定ファイル
- docker-composeは複数のDockerイメージを組み合わせてアプリケーションを実行することができる。設定ファイルは`compose.yaml`が使われる。
  - `compose.yaml`はvolumeの設定や開くポートなど通常`docker run`で指定するオプションをyamlファイルに事前に記述しておくことができるため，長いコマンドを覚えなくて済むので単一イメージの場合でも使うと良い。

## docker-composeとdocker compose

docker composeがV2らしい。[docker-composは非推奨](https://www.konosumi.net/entry/2023/02/26/142508)

---

## yamlのファイル名

`compose.yaml`以外のファイル名を使うこともできるが，公式の推奨は`compose.yaml`である。
> The default path for a Compose file is compose.yaml (preferred) or compose.yml that is placed in the working directory. Compose also supports docker-compose.yaml and docker-compose.yml for backwards compatibility of earlier versions. If both files exist, Compose prefers the canonical compose.yaml.
>
> 引用元: [dockerdocs](https://docs.docker.com/compose/intro/compose-application-model/#the-compose-file)

---

## docker composeの最低限のコマンド

### up

```shell
docker compose up # docker-compose.ymlに書かれているすべてのサービスをビルドして起動する。
```

:::message alert
`docker compose up`はイメージが存在する場合には再度のビルドは行わない。
`docker compose up --build`の様に実行するとイメージのビルドをしつつ，コンテナを起動してくれる。
:::

### build,buildx

- BuildKitを使いたい場合にはbuildx bakeを使う。

```shell
docker buildx bake
```

---
