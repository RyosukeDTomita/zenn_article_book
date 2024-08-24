---
title: "夏休みの宿題でPort Scannerを自作してみた②~開発環境準備編"
emoji: "👀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["portscanner", "python", "rye", "network", "nmap", "夏休み", "自作"]
published: true
---

## これまでのお話

- 夏だし，Port Scannerを自作してみたい。
- [技術選定編](https://zenn.dev/sigma_tom/articles/3de59d1f44aa7e)では，Port Scannerを自作するにあたってどのような技術を使うかを考えた。
- 今回は，Port Scannerの開発環境を整えつつ，全体設計を考えていきます。

---

## 開発環境を整える

とりあえず，ミニマムでやることを洗い出しました。

- GitHubのリポジトリを作る
- pythonのプロジェクトを作る
- Dev ContainerとDockerが動くような環境を整える(VSCodeのExtensionsとかもこのへんでやる)
  - Dockerfileの準備
  - .devcontainer/devcontainer.jsonの準備
  - dotfilesリポジトリの準備

### GitHubのリポジトリを作る

自分用の[template repository](https://github.com/RyosukeDTomita/template_repository_all)を以前作ったのでこれを雛形にしてリポジトリを作成しました。

テンプレートリポジトリを使うと自分が共通してよく使うファイルとかをあらかじめ用意しておけるので，リポジトリ作成時が時短できます。

### pythonのプロジェクトを作る

ryeは割と何でもできる優秀な子だったので，ryeでプロジェクトを作成しました。
[ryeの公式ドキュメント](https://rye.astral.sh/guide/basics/)にしたがって，プロジェクトを作成しました。
デフォルトでPythonのバージョンは3.12.4という結構新しいものがryeにおすすめされたので，そのまま使うことにしました。

```bash
cd my_port_scanner
rye init my_port_scanner
```

概ねこんな感じのディレクトリ構成になりました。

```
.
├── .git
├── .gitignore
├── .python-version
├── README.md
├── pyproject.toml
└── src
    └── my_portscanner
        └── __init__.py
```

今回は単体テストも書く予定なので，`tests`ディレクトリも作成しておきました。テストの実行やbuildもryeでできるらしいのでryeってすごい!

```bash
cd src/
mkdir tests
touch tests/__init__.py
```

### Dev ContainerとDockerが動くような環境を整える

- VSCodeにDev Container拡張機能をインストールしました。このあたりは[夏休みの自由研究: Dev Containerによる開発環境コンテナ化](https://zenn.dev/sigma_tom/articles/7ee1915d5c414b)を読んでみてください。

- Dockerfileを作成するにあたって，[niktoのDockerfile](https://github.com/sullo/nikto/blob/master/Dockerfile)の書き方を参考にしつつ，Dev Containerが使いやすいマルチステージビルドにしました。

誤算だったのは，ryeが仮想環境作成機能をオフにすることができなかったことです。自分はPython自体の仮想環境を使うのはあまり好きではないので，いつもDockerでPythonのバージョンを指定して使っているのですが，ryeを使うとvenvを使うことを強いられてしまいました。ryeのコンセプト的にはryeで全部できるようにしたいのかなと思うので優先度が低い機能なのかもしれませんが，poetryができるのでryeの今後に期待です。

で，結果的にこんな感じのDockerfileになりました。ヘビーなので読み飛ばして大丈夫です。

```Dockerfile
# Dev Container
FROM debian:bookworm-20240812 AS devcontainer

ARG PYTHON_VERSION=3.12.4

WORKDIR /app
COPY ../ .

# aqua install
RUN <<EOF
apt-get update -y
apt-get install -y --no-install-recommends wget ca-certificates
wget -q https://github.com/aquaproj/aqua/releases/download/v2.30.0/aqua_linux_amd64.tar.gz
rm -rf /usr/local/bin/aqua && tar -C /usr/local/bin/ -xzf aqua_linux_amd64.tar.gz
rm aqua_linux_amd64.tar.gz
rm -rf /var/lib/lists
EOF

# install packages and some tools.
# NOTE: rye is installed by aqua.
RUN <<EOF
aqua install
EOF

# build
RUN <<EOF
PATH=$PATH":$(aqua root-dir)/bin"
rye pin ${PYTHON_VERSION}
rye sync
rye build
EOF


FROM python:3.12.4-slim-bullseye AS run
WORKDIR /app

ARG VERSION="0.1.2"
LABEL version="${VERSION}" \
      author="RyosukeDTomita" \
      docker_compose_build="docker buildx bake" \
      docker_build="docker buildx build . -t my_portscanner" \
      docker_compose_run="docker compose run my_portscanner_app localhost" \
      docker_run="docker run my_portscanner localhost"

# create execution user with sudo
ARG USER_NAME="sigma"
RUN <<EOF
apt-get update -y
apt-get install -y --no-install-recommends sudo
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
RUN <<EOF
python3 -m pip install /app/dist/my_portscanner-${VERSION}-py3-none-any.whl
EOF

USER ${USER_NAME}
ENTRYPOINT ["sudo", "my_portscanner"]
```

このDockerfileを使うことでDev Containerを使って開発しつつ，buildしたイメージをDockerで使うことができます。

```shell
docker buildx build . -t my_portscanner
docker run my_portscanner localhost -sS -p 80
```

---

## 実装の範囲を決める

最初のゴールは小さめのほうがモチベーションが持続するので

- Connect Scan，SYN Scanを使えるようにする
  - オプションで切り替え可能にする
- 宛先とport番号を引数で指定して実行できるようにする
  - FQDNの場合は名前解決する

の2つを目指すことにしました

---

## TO BE CONTINUED

次回は具体的にPort Scannerを作成していきますする[**実装編**](https://zenn.dev/sigma_tom/articles/464f2e43420e0c)をお送りします。
好ご期待ください。
