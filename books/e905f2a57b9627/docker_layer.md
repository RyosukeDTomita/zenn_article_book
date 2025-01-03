---
title: "ベスプラ②/レイヤを意識する"
---


## レイヤーとは

> Docker は構築命令の順番に従い、構築を構成します。Dockerfile 内の各命令は、イメージレイヤに大ざっぱに相当します。
> 構築を実行するとき、 ビルダ は以前に構築したレイヤを再利用しようとします。イメージのレイヤが変更されていない場合は、ビルダは 構築キャッシュ からキャッシュを取り出します。最後の構築からレイヤに変更がある場合は、対象レイヤと以降のレイヤをすべて再構築されます。
> [Docker-docs-ja レイヤ](https://docs.docker.jp/build/guide/layers.html)

![layers](/images/dockerbook/layers.png)

- Dockerfileの中での命令一つにつき，イメージレイヤーが作成され，それが積み重なっていくことでイメージが作成される。
- Dockerのbuild時にイメージレイヤーがキャッシュされており，レイヤ単位で変更の有無を確認してキャッシュを使用するか再度ビルドするかを判断している。
- つまり，Dockerfileの書き方を工夫することでレイヤキャッシュをうまく使うことができ，ビルド時間の短縮やイメージサイズの削減が可能となる。

---

## レイヤーを意識したDockerfileの書き方

### multi-stage buildを使う

> マルチステージビルドを行うには、Dockerfile 内にFROM行を複数記述します。 各FROM命令のベースイメージは、それぞれに異なるものとなり、各命令から新しいビルドステージが開始されます。 イメージ内に生成された内容を選び出して、一方から他方にコピーすることができます。 そして最終イメージに含めたくない内容は、放っておくことができます。
> [マルチステージビルドの利用](https://matsuand.github.io/docs.docker.jp.onthefly/develop/develop-images/multistage-build/#use-multi-stage-builds)

Dockerfileを見るのが一番わかりやすいと思うのでサンプルを貼っておく。

マルチステージビルドを使うことで，一つ上のFROMで作成したイメージをベースにして，新たなイメージを作成することができる。

```
# ビルド環境
FROM node:20 as build
WORKDIR /app
COPY . .
RUN npm install && npm run build


# プロダクション環境
FROM build as production
EXPOSE 80
CMD ["node", "server.js"]
```

これにより，build環境で作成したレイヤーはproduction環境には含まれないため，最終的なimageのレイヤ数が少なくなり，イメージサイズが小さくなる。

また，一つ上のFROMで作成したイメージから成果物(build/など)をコピーして，新たなイメージを作成することも可能である。これはdistrolessイメージを使う際に特に役立つ。distrolessイメージはにはshellなどが含まれいていないため，コンテナ内で開発やデバックをするには不向きだが，上位イメージで開発した後にmulti-stage buildを使ってdistrolessイメージに成果物をコピーすることで，最終的にはdistrolessイメージを使い，イメージサイズを小さくすることができる。

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

### RUN命令を適切にまとめる

純粋にDockerfileの命令の数を減らせばレイヤの数が減り，ビルド時間が短縮される。

```
# 悪い例
RUN apt-get install -y python3
RUN apt-get install -y python3-pip
RUN apt-get install -y python3-dev
RUN rm -rf /var/lib/apt/lists/*
```

```
# 良い例
RUN apt-get install -y python3 python3-pip python3-dev && \
    rm -rf /var/lib/apt/lists/*
```

だが，単純にまとめれば良いというわけではない。RUNをまとめすぎることで逆にキャッシュが効かなくなり，ビルド時間がかかることもある。
以下は，ryeというパッケージ管理ツールをインストールしてビルドする例である。

```
# 悪い例
RUN apt update && apt install -y python3 python3-pip python3-dev && pip install rye \
    rm -rf /var/lib/apt/lists/* && rye build
```

悪い例ではpythonのソースコードが変更されるたびにapt updateから実行されることになる。

そこで，以下の良い例のようにrye buildを別のRUN命令にすることで，ライブラリのインストール部分の処理はキャッシュされるようになるため，rye buildの部分だけが再実行される。

```
# 良い例
RUN apt update && apt install -y python3 python3-pip python3-dev && pip install rye  && \
    rm -rf /var/lib/apt/lists/*
RUN rye build
```

前述したmulti-stage buildを使用する場合，中間イメージのレイヤ数を気にする必要はないため，すべてのRUNをまとめることを優先するよりも，開発中にキャッシュが効くように工夫した方が良いと思う。
自分は以下のような基準でRUNをまとめるようにしている。

- 失敗しやすい処理
- 時間のかかる処理
- 変更が多い(キャッシュが効かない)処理
- 処理の単位でわける(ライブラリのインストール，アプリケーションのビルドなど)

### 変更の少ないファイルを先にCOPYする

悪い例ではCOPYをまとめているため，requirements.txtが変更されていない場合にも`pip install -r requirements.txt`がキャッシュされないため，毎回ライブラリのインストールが実行される。

```
# 悪い例
FROM python:3.12
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
RUN rye build
```

Dockerfileの中でCOPY命令を使う際，変更の少ないファイル(requirements.txtなど)を先にCOPYすることで，requirements.txtが変更されていない場合には`pip install -r requirements.txt`がキャッシュされるため，毎回ライブラリのインストールが実行されることがなくなる。

```
# 良い例
FROM python:3.12
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
RUN rye build
```
