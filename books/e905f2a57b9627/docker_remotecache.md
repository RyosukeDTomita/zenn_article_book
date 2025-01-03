---
title: "ベスプラ④/リモートキャッシュを使う"
---

## リモートキャッシュとは

DockerのBuildkitにはリモートのキャッシュを使う機能がある。

[公式ドキュメント](https://github.com/moby/buildkit/tree/master#cache)によると，4種類のキャッシュが使用できる
> inline: embed the cache into the image, and push them to the registry together
> registry: push the image and the cache separately
> local: export to a local directory
> gha: export to GitHub Actions cache
> In most case you want to use the inline cache exporter. However, note that the inline cache exporter only supports min cache mode. To enable max cache mode, push the image and the cache separately by using registry cache exporter.
> inline and registry exporters both store the cache in the registry. For importing the cache, type=registry is sufficient for both, as specifying the cache format is not necessary.

今回はinlineキャッシュを使う例を紹介する。

```shell
# buildxでリモートキャッシュを使ってbuildし，imageとキャッシュをpushする
docker buildx build --push -t ghcr.io/ryosukedtomita/devsecops-demo-aws-ecs --cache-to type=inline --cache-from type=inline .
```

![GitHub Container Registry](/images/dockerbook/packages.png)

```shell
# リモートキャッシュを使ってbuildだけを行う
docker buildx build \
  --cache-from=ghcr.io/ryosukedtomita/devsecops-demo-aws-ecs \
  --cache-to=type=inline \
  -t myapp .
```

- docker composeを使う場合のcompose.yamlの設定例

```
services:
  react-app:
    build:
      context: ./
      dockerfile: Dockerfile
      cache_from:
        - ghcr.io/ryosukedtomita/devsecops-demo-aws-ecs
        - type=inline
      cache_to:
        - ghcr.io/ryosukedtomita/devsecops-demo-aws-ecs
        - type=inline
    image: react-app:latest
    container_name: react-app-container
    ports:
      - 80:8080 # localport:dockerport
```
