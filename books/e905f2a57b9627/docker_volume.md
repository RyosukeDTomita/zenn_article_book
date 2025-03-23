---
title: "volume mountについて"
---

## bind mountとどちらを使うべきか

volumen mountを使うべき

> 新しい Docker アプリケーションを開発する場合は、代わりに 名前付きボリューム の利用を検討してください。バインド マウントは Docker CLI コマンドを使って直接管理できません。
> [バインドマウントの使用](https://docs.docker.jp/storage/bind-mounts.html)

---

## volume mountについて

- ローカルとdockerでディレクトリの共有ができるのでソースを変更するたびにビルドし直さなくて良くなる。基本的には開発環境で使用する。
- volume mountには`.dockerignore`は効かない。
- volumes mountとDockerfileのCOPYは併用可能である。
 - DockerfileのCOPYで配置したソースをvolume mountし，ローカルから編集した場合にはコンテナ内のリソースが編集される。


### dockerコマンドでのマウント例

```shell
docker run -it --mount type=bind,source=./,target=/usr/local/app flask-app "/bin/bash"
```

:::message alert
volumeの共有は**コンテナ**に対して行われるため，上記のコマンド実行後に`docker run -it イメージ名 "/bin/bash"`等でvolume mountされたものを参照することはできない。これは`docker run`で立ち上げるコンテナが別だからである。
`docker exec -it コンテナ名 "/bin/bash"`であれば起動しているコンテナにつながるので参照可能。
:::

### compose.yamlでのマウント例

```yaml
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

