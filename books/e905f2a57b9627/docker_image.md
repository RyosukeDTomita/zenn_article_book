---
title: "ベスプラ①/imageサイズを小さくする"
---

## なぜimageサイズを小さくするのか

- imageサイズを小さくすることでビルド時間やデプロイ時間を短縮できる。
- クラウドサービスを使う場合イメージサイズが小さい方が課金額を抑えられることも。
  - [AWS ECS](https://aws.amazon.com/jp/ecr/pricing/)の場合はイメージ保存量に応じたストレージ量と外部へのデータ転送量によって課金額が変わる。

---

## イメージを小さくする方法

### 小さめのイメージを選択する

- alpineはかなりイメージサイズが小さいがライブラリのversion指定ができないなど，使い勝手が良くない場合があるため，個人的にはdebianのslimイメージを使うことをおすすめする。
- [debian](https://hub.docker.com/_/debian)の場合はslimイメージを選ぶだけで40MBほどのイメージサイズの削減になる。

```shell
docker images
REPOSITORY   TAG             IMAGE ID       CREATED        SIZE
debian       bookworm        11c49840db54   11 days ago    117MB
debian       bookworm-slim   802cc311ed7d   11 days ago    74.8MB
```

- distrolessイメージを使うとさらにイメージサイズを小さくできる。詳しくは後述のmulti-stage buildに記載する。

#### 参考: ベースイメージを探せるサイト

- [Docker Hub](https://hub.docker.com/)
- [Amazon ECR Public Gallery](https://gallery.ecr.aws/)
- [Google Container Registry](https://console.cloud.google.com/marketplace/product/google-cloud-platform/container-registry?hl=ja&project=geolocator-339315)

### .dockerignore

- .dockerignoreに書いたファイルやディレクトリはイメージに含まれないため，イメージサイズを小さくできる。
  - 特に.git/などは効果が大きいので絶対に書いておくべき
- 書き方は.gitignoreと同じ

```
# pythonの例
.gitignore
CODEOWNERS
.git/
.github/
Dockerfile
docker-compose.yml
pyproject_toml-0.0.10-py3-none-any.whl
```

### 不要ファイルを減らす，削除する

#### Dockerfileの書き方を工夫する

 ライブラリのインストール時のキャッシュや推奨パッケージを入れない書き方をする。

```
RUN apt install -y nginx --no-install-recommends && rm -rf /var/lib/lists
```

```
RUN pip install -r requirements.txt --no-cache-dir
```

#### diveを使って無駄ファイルを探す方法

- [dive](https://github.com/wagoodman/dive)からdebファイルを落としてきてaptでinstallするだけで使える。
- イメージのレイヤごとにファイルサイズを確認できるため，不要なファイルを探すのに適している。
- Docker DesktopのExtensionsでも使用できる。

```shell
dive <image-name>
```
