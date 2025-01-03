---
title: "ベスプラ⑤/security対策"
---

## コンテナセキュリティの基本方針

- コンテナブレイクアウトやコンテナエスケープを防ぐために`root`でコンテナを起動しないことが大切。
- COPY命令で持ち込んだファイルにはクレデンシャルを含めない。途中で削除してもイメージに残ってしまう。
  - .dockerignoreで除外するなど
- distroless(Google提供のシェルすら含まれていない軽いイメージ)を使うことを視野に入れる。

> distrolessでもdebugと書いてあるやつはshellがあるので便利。

- ビルドし直す時には脆弱性のあるキャッシュが使われないように--no-cacheを使う。

## 信頼できるイメージをベースに使う

公式が出しているイメージをベースにする。

- Docker Officlial Image，Verified Publisher(Dockerによって検証)
- Sponsord OSSなど

---

## rootユーザ以外でコンテナを起動する

最終的サービスを実行するユーザが非rootユーザであれば途中はrootユーザを使っても問題ない。

### イメージに存在する非rootユーザを使う

イメージによってはデフォルトで非rootユーザが存在しているのでこれを使う。

```shell
FROM node:20
WORKDIR /app
USER root
RUN apt update && apt install -y nginx --no-install-recommends

# defaultの非rootユーザを使用する
USER nobody
ENTRYPOINT ["nginx", "-g", "daemon off;"]
```

### 自前で非rootユーザを作成する

以下はsudo権限が必要だったのでsudo使用可能なユーザを作成する例

```shell
FROM python:3.12.4-slim-bullseye AS run
WORKDIR /app


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
adduser --ingroup ${USER_NAME} --gecos "my_portscanner user" --shell /bin/bash --no-create-home --disabled-password ${USER_NAME}
echo 'using sudo'
usermod -aG sudo ${USER_NAME}
echo "${USER_NAME} ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
rm -rf /var/lib/lists
EOF

COPY --from=devcontainer --chown=${USER_NAME}:${USER_NAME} ["/app/dist/my_portscanner-${VERSION}-py3-none-any.whl", "/app/dist/my_portscanner-${VERSION}-py3-none-any.whl"]

# install app
RUN python3 -m pip install /app/dist/my_portscanner-${VERSION}-py3-none-any.whl

USER ${USER_NAME}
ENTRYPOINT ["sudo", "my_portscanner"]
```

### imageの中でクレデンシャルを扱う

#### NG例1: ENVを使ってクレデンシャルを渡してしまう

- DockerfileのENVにクレデンシャルを指定した場合，`docker history`で参照可能

```shell
docker build -t myimage:latest -f- . <<EOF
FROM busybox
ENV AWS_ACCESS_KEY_ID=hogehoge
EOF

docker history myimage:latest | grep AWS_ACCESS_KEY_ID
<missing>      35 seconds ago   ENV AWS_ACCESS_KEY_ID=hogehoge                  0B        buildkit.dockerfile.v0
```

#### NG例2: imageにCOPYでクレデンシャルファイルを渡してしまう

```shell
echo "secret" > secrets.txt

docker build -t myimage:latest -f- . <<EOF
FROM busybox
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

#### 対策例

- クレデンシャルが誤って混入しないように.dockerignoreを使う
- multi-stage buildを使って中間imageでのみクレデンシャルを扱う --> 最終imageにはクレデンシャルが残らない
- buildxの`--secret`を使ってマウントする

```shell
docker buildx build -t myimage:latest --secret id=aws_access_key,src=./.env -f- . <<EOF
FROM busybox
RUN --mount=type=secret,id=aws_access_key cat /run/secrets/aws_access_key
EOF
```

---

## Dockerのセキュリティツール

### docker scout

- 指定したイメージやアーカイブをスキャンして脆弱性の有無を調べられるツール。

```shell
docker scout quickview react-app:latest
docker scout cves react-app:latest
```

### trivy

- コンテナスキャンやクレデンシャルスキャンが可能

```shell
trivy image --exit-code 1 --vuln-type os --ignorefile .trivyignore --no-progress --format table -o container-scanning-report.txt --severity CRITICAL,HIGH <image名>:<tag>
```

---
