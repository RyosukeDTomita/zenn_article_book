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

今回はregistryキャッシュを使う例を紹介する。
:::message
registryにしたのはtype=maxをサポートしているため中間imageのキャッシュも使用できるから
:::

```shell
docker buildx build --cache-from=type=registry,ref=ghcr.io/ryosukedtomita/devsecops-demo-aws-ecs,mode=max --cache-to=type=registry,ref=ghcr.io/ryosukedtomita/devsecops-demo-aws-ecs,mode=max -t react-app .
```

![GitHub Container Registry](/images/dockerbook/packages.png)

```shell
# リモートキャッシュを使ってbuildだけを行う
docker buildx build \
  --cache-from=type=registry,ref=ghcr.io/ryosukedtomita/devsecops-demo-aws-ecs,mode=max \
  --cache-to=type=registry,mode=max,ref=ghcr.io/ryosukedtomita/devsecops-demo-aws-ecs \
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
        - ref=ghcr.io/ryosukedtomita/devsecops-demo-aws-ecs,type=registry,mode=max
      cache_to:
        - ref=ghcr.io/ryosukedtomita/devsecops-demo-aws-ecs,type=registry,mode=max
    image: react-app:latest
    container_name: react-app-container
    ports:
      - 80:8080 # localport:dockerport
```
