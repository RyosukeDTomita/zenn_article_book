---
title: "ベスプラ③/Cache mountでbuildの高速化"
---

## Cache mountとは

Dockerのbuildkitにはローカルのディレクトリをマウントしてキャッシュとして使う機能がある。
前提として，Dockerのbuildkitが有効になっている必要がある。

### TypeScriptの場合

```
FROM node:20 AS build
WORKDIR /app

# キャッシュを利用してnpm installを高速化
COPY ./package.json ./
RUN --mount=type=cache,target=/root/.npm npm install
```
