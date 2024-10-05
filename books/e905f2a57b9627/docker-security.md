---
title: "Docker security"
---

## コンテナセキュリティの基本方針

- コンテナブレイクアウトやコンテナエスケープを防ぐために`root`でコンテナを起動しないことが大切。
- COPY命令で持ち込んだファイルにはクレデンシャルを含めない。途中で削除してもイメージに残ってしまう。
- distroless(Google提供のシェルすら含まれていない軽いイメージ)を使うことを視野に入れる。

> distrolessでもdebugと書いてあるやつはshellがあるので便利。

- ビルドし直す時には脆弱性のあるキャッシュが使われないように--no-cacheを使う。
- 信頼できるイメージ(Docker Officlial Image，Verified Publisher(Dockerによって検証)，Sponsord OSS)

## 一般ユーザでコンテナを起動する

### イメージに存在する非rootユーザを使う

```shell
FROM debian:bookworm-20240812 AS devcontainer

# defaultの非rootユーザを使用する
USER nobody
ENTRYPOINT ["ls"]
```

```shell
docker build -t test .
docker run test -l
total 60
lrwxrwxrwx   1 root root    7 Aug 12 00:00 bin -> usr/bin
drwxr-xr-x   2 root root 4096 Mar 29  2024 boot
drwxr-xr-x   5 root root  340 Sep 28 15:46 dev
drwxr-xr-x   1 root root 4096 Sep 28 15:46 etc
drwxr-xr-x   2 root root 4096 Mar 29  2024 home
...
```

### aptでversionを指定してインストールする

使用するimageに対してaptがサポートしているライブラリのversionが一つだけの場合でもversionを書いておくのがベター。

```shell
# 使用可能なライブラリの一覧を調べる
apt list openssl -a
Listing... Done
openssl/focal-updates,focal-security,now 1.1.1f-1ubuntu2.23 amd64 [installed]
openssl/focal 1.1.1f-1ubuntu2 amd64

openssl/focal-updates,focal-security 1.1.1f-1ubuntu2.23 i386
openssl/focal 1.1.1f-1ubuntu2 i386

# =でversionを指定する
apt-get install -y --no-install-recommends openssl=1.1.1f-1ubuntu2.23
```

### 自前でrootユーザを作成して使う

作成したユーザにsudo権限を与えている。
過去に作成した[my_portscannerのDockerfile](https://github.com/RyosukeDTomita/my_portscanner/blob/main/Dockerfile)から抜粋。

```
FROM python:3.12.4-slim-bullseye AS run
WORKDIR /app

ARG VERSION="0.2.0"

# install sudo
RUN <<EOF bash -ex
apt-get update -y
apt-get install -y --no-install-recommends sudo
EOF

ARG USER_NAME="sigma"

# create execution user with sudo
RUN <<EOF bash -ex
echo 'Creating ${USER_NAME} group.'
addgroup ${USER_NAME}
echo 'Creating ${USER_NAME} user.'
adduser --ingroup ${USER_NAME} --gecos "my_portscanner user" --shell /bin/bash --no-create-home --disabled-password ${USER_NAME} # ホームディレクトリを作らず，デフォルトshellをbashに設定し，passwordを設定しない
echo 'using sudo'
usermod -aG sudo ${USER_NAME}
echo "${USER_NAME} ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers # passwordなしでsudoを実行できるようにする
rm -rf /var/lib/lists
EOF

COPY --from=devcontainer --chown=${USER_NAME}:${USER_NAME} ["/app/dist/my_portscanner-${VERSION}-py3-none-any.whl", "/app/dist/my_portscanner-${VERSION}-py3-none-any.whl"] # 作成したユーザに所有権を移譲してcopy

# install app
RUN python3 -m pip install /app/dist/my_portscanner-${VERSION}-py3-none-any.whl

USER ${USER_NAME}
ENTRYPOINT ["sudo", "my_portscanner"]
```

---

## docker scout

- 指定したイメージやアーカイブをスキャンして脆弱性の有無を調べられるツール。

```shell
docker scout quickview react-app:latest
docker scout cves react-app:latest
```

---

## イメージからクレデンシャルを抜き出すハンズオン

- イメージから環境変数を取得

```shell
docker build -t myimage:latest -f- . <<EOF
FROM busybox
ENV AWS_ACCESS_KEY_ID=hogehoge
COPY secrets.txt /etc/secrets.txt
RUN rm /etc/secrets.txt
EOF

docker history myimage:latest | grep AWS_ACCESS_KEY_ID
<missing>      35 seconds ago   ENV AWS_ACCESS_KEY_ID=hogehoge                  0B        buildkit.dockerfile.v0
```

- イメージから途中で消したファイルを取得

```shell
echo "secret" > secrets.txt

docker build -t myimage:latest -f- . <<EOF
FROM busybox
ENV AWS_ACCESS_KEY_ID=hogehoge
COPY secrets.txt /etc/secrets.txt
RUN rm /etc/secrets.txt
EOF

docker save mayimage -o myimage.tar # イメージを保存

tar -tf myimage.tar | fgrep layer.tar # layerを抽出
5044846d4c5c84cb56580205b7e20cf4564d7b18d3e17a8f5cc3289d3e3b5b7b/layer.tar
5a3228fc3aa3e38eddcfe971c5abe08ec64ccd1414a33c38d6977bdbe88b9c51/layer.tar
92d557fbcdb88593a1f056f462d71a22750c1c24276052f0545ce4470dc516e3/layer.tar

tar xfO myimage.tar 5044846d4c5c84cb56580205b7e20cf4564d7b18d3e17a8f5cc3289d3e3b5b7b/layer.tar
etc/0000755000000000000000000000000014536044433010025 5ustar0000000000000000etc/secrets.txt0000664000000000000000000000001014536043242012224 0ustar0000000000000000secrets /etc/secrets.txtがありそうなのがわかる。

tar xfO myimage.tar 5044846d4c5c84cb56580205b7e20cf4564d7b18d3e17a8f5cc3289d3e3b5b7b/layer.tar | tar xfO - etc/secrets.txt # ファイルの中身を確認
secrets
```

```shell
# ファイル名がわかっていればワンライナーでいけそう
for layer in $(tar -tf myimage.tar | fgrep layer.tar); do tar xfO myimage.tar $layer;done | tar xfO - etc/secrets.txt
secrets
```
