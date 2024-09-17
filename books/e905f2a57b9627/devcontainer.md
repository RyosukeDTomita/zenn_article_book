---
title: "環境構築/devcontainerを使う"
---

# devcontainerの使い方(任意)

## ABOUT

- Dev ContainerとはDocker内でVSCodeを開くことで開発環境のコンテナ化を行うことができる機能である。本筋のDockerとは関係ないのでDockerをいじりたいだけの人は飛ばしてOK。

- [devcontainer-test](https://github.com/RyosukeDTomita/devcontainer-test)に試す用のリポジトリがある。


### What is the benefit of DevContainer

- DevContainerを使うと，Dockerコンテナ内でVSCodeを開ける
  - 起動しているコンテナでVSCodeを起動する --> `Attach to running container`
  - VSCodeからコンテナをビルドして起動させる。`Reopen in container` or `Rebuild and Reopen in container`
- VSCodeのExtensions等も一括で管理できるので環境構築が楽になる

---

## PREPARING

1. install Docker, VSCode
2. clone this repository
3. install [DevContainer](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)(VSCode Extensions)
4. In VSCode, from the command palette, select `Reopen in Container`

```shell
code ~/devcontainer-test
```

:::message
> コンテナの再ビルドを行いたい場合は，`Rebuild and Reopen in Container`を使う。
:::

---

## Dev Containerの機能一覧

### dotfilesを使って開発環境に必要な設定ファイルを持ち込む

- ~/.bashrc等をDev Containersに持ち込む機能。
- **VSCodeの個人用のsettings.json**に記載する。
- 設定は自分のGitHub上で管理している`dotfiles`リポジトリに合わせて変更すること。

```json
  // Dev Containersでdotfilesを使う。
  // NOTE: .gitconfigは何もしなくてもDev ContainersのHOMEディレクトリにコピーされる。
  "dotfiles.repository": "https://github.com/RyosukeDTomita/dotfiles.git",
  "dotfiles.targetPath": "~/dotfiles",
  "dotfiles.installCommand": "install.sh", // ~/dotfilesのrepository topからみたスクリプトのパスを指定する。
```

:::message
`dotfiles.installCommand`には~/dotfilesのrepository topから見たスクリプトのパスを指定する。--> dotfilesのGitHub Repositoryにinstall.shは配置が必要。
指定しない場合のデフォルトファイル名は以下が認識される。
- install.sh
- install
- bootstrap.sh
- setup.sh
- setup
:::

#### dotfiles TIPS

- Dev Containerの環境では，`REMOTE_CONTAINERS=true`が環境変数に定義されているのでこれを使ってDev Containerかどうかを判別できる。 --> dotfilesを使う時だけ有効にしたい設定とかdotfilesで書ける。

---

### Extensions関連

#### 拡張機能の設定ファイルをどこにおくか
- devcontainer.jsonと.vscode/settings.json両方に記載ができる。
- 基本は.vscode/settings.json配下で良いと思っているが一部コンテナ名等を使う場合にはdevcontainer.jsonに記載したほうが楽かも。

#### Extensionsを追する方法

例えば，[peacock](https://marketplace.visualstudio.com/items?itemName=johnpapa.vscode-peacock)をインストールしたい場合には，インストールコマンド`ext install johnpapa.vscode-peacock`にて記載される`johnpapa.vscode-peacock`をdevcontainer.jsonに記載すればよい。

#### DevContainerに自分だけが使用するExtensionsを持ち込む

- **個人用のsettings.json**に記載する。.vscode/settings.jsonではないので注意。

```json
  // Devcontainerの個人的に使うExtensionsを入れる。
  "dev.containers.defaultExtensions": [
    "formulahendry.auto-rename-tag",
  ],
}
```

---

### Dockerfileのマルチステージビルドを使ってDevContainer用の環境を作る

- Dockerfileを使う場合にはdevcontainer.jsonでtargetの指定ができる。
- 一方，compose.yamlを使う場合には別のcompose.yamlを準備する必要がある。

:::message
> compose.yamlの優先順位は.devcontainer/compose.yamlが優先される。
:::

```yaml
# for devcontainer
version: '3'

services:
  react-app:
    build:
      target: devcontainer // targetを指定
      context: ./
      dockerfile: Dockerfile
    image: react-img-devcontainer:latest
    container_name: react-container-devcontainer
    ports:
      - 80:8080 # localport:dockerport
```

```
# DevContainer
FROM node:20-bullseye AS devcontainer # targetに指定する値
WORKDIR /app
COPY . .
RUN cd react-default-app && npm install

# Build Image
FROM node:20-bullseye AS build
WORKDIR /app
COPY ./react-default-app .
RUN npm install && npm run build
```

:::message alert
- compose.yamlのサービス名をdevcontainer.jsonのserviceを同じにする必要がある。
:::

```yaml
# compose.yaml
version: '3'

services:
  react-app: # devcontainer.jsonに指定するサービス名
    build:
      target: devcontainer
      context: ./
      dockerfile: Dockerfile
    image: react-img-devcontainer:latest
    container_name: react-container-devcontainer
    ports:
      - 80:8080 # localport:dockerport
```

```json
{
  "name": "dev-container-test", // 任意の名前
  "dockerComposeFile": [
    "../compose.yaml",
    "compose.yaml"
  ],
  "service": "react-app", // compose.yamlのサービス名
```

> [参考1](https://github.com/microsoft/vscode-remote-release/issues/7810)
> [参考2](https://stackoverflow.com/questions/78421879/devcontainer-docker-compose-best-practice)

---

### 各種コマンドの使い方

```json
{
  "name": "dev-container-test",
  "dockerComposeFile": [
    "../compose.yaml",
    "compose.yaml"
  ],
  "service": "react-app",
  "workspaceFolder": "/app",
  "overrideCommand": true, // コンテナを起動したままにする DockerfileのCMDで永続するコマンドを実行しているなら不要
  // rvest.vscode-prettier-eslintに使うパッケージをinstall
  "postCreateCommand": "yarn add -D prettier@^3.1.0 eslint@^8.52.0 prettier-eslint@^16.1.2 @typescript-eslint/parser@^5.0.1 typescript@^4.4.4",
  "postStartCommand": ["echo", "Hello, DevContainer"], // DevContainer起動時
  "postAttachCommand": ["echo", "Hello, DevContainer"], // DevContainerに既存コンテナをattach時
  "initializeCommand": ["echo", "Starting DevContainer..."], // DevContainerのビルド前，実行前にローカルで実行されるコマンド
```

#### installスクリプトを使う

必要そうなパッケージをまとめた[install-pkg.sh](./install-pkg.sh)を作成し，postCreateCommandで実行する。

---

### ディレクトリのマウント

- ファイル単体でのマウントはできないぽい。設定ファイルとかは前述のdotfilesを使って共有する。

```json
{
  "name": "dev-container-test",
  "dockerComposeFile": [
    "../compose.yaml",
    "compose.yaml"
  ],
  "service": "react-app",
  "workspaceFolder": "/app",
  "overrideCommand": true,
  "mounts": [
    "source=~/.aws,target=/home/root/.aws,type=bind" // ~/.awsをマウントできる
  ],
```

