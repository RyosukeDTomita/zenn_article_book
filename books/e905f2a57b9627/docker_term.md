---
title: "Docker関連用語"
---

## imageとcontainer

- image: Dockerfileによって作成される環境のスナップショット。 `docker image ls`で確認できる。
- container: imageをもとに作成されるソフトウェアの実行環境。`docker container ls`で確認できる。

---

## dockerコマンドの違い

### imageに関連するコマンド(pull, build)

- `docker pull`: Docker Hubからイメージを取得する。
- `docker build`: Dockerfileからイメージを作成する。

### containerに関連するコマンド(create, start, run, exec)

- `docker create`: containerを作るだけ。起動はしない。
- `docker start`: 停止しているcontainerを起動する。
- `docker exec`: 起動しているcontainerに接続してコマンドを実行する
- `docker run`: containerを新たに作成して起動する。

### デタッチドモードvsフォアグランドモード

- デタッチドモード: バックグラウンドでコンテナを実行するモード。`docker run`の`-d`オプションで指定する。
  - 例えばウェブサーバーを立ち上げた時にデタッチドモードならばログが画面にでないため，そのまま別のコマンドを実行できる。
  - `docker run`の`-a`オプションと`-d`オプションはコンフリクトを起こすため同時に使えない。
- フォアグラウンドモード: コンテナをを実行するモード。`docker run`の`-a`オプションで標準入力と標準出力をつなげることができる。

> [デタッチモードの違い](https://qiita.com/leomaro7/items/a96b62659ab676933f64)

```shell
docker run -it -a stdout -a stdin fileserver "/bin/bash" # フォアグラウンドモードではコマンドが実行できる
```

### 旧コマンドと新コマンド

> [新旧Dockerコマンドの一覧](https://www.infra-linux.com/menu-docker2/old-new-commands/)

古いコマンドと長いコマンドが存在し，現状はどちらも使える。
e.g. `docker run`と`docker container run`，`docker ps`と`docker container ls`。

---

## docker-compose

- Dockerfileは単独のDockerイメージを作るための設定ファイル
- docker-composeは複数のDockerイメージを組み合わせてアプリケーションを実行することができる。設定ファイルは`compose.yaml`が使われる。
  - `compose.yaml`はvolumeの設定や開くポートなど通常`docker run`で指定するオプションをyamlファイルに事前に記述しておくことができるため，長いコマンドを覚えなくて済むのが個人的には一番嬉しいところ。

:::message
正確には`compose.yaml`以外のファイル名を使うこともできるが，公式の推奨は`compose.yaml`である。
> The default path for a Compose file is compose.yaml (preferred) or compose.yml that is placed in the working directory. Compose also supports docker-compose.yaml and docker-compose.yml for backwards compatibility of earlier versions. If both files exist, Compose prefers the canonical compose.yaml.
>
> 引用元: [dockerdocs](https://docs.docker.com/compose/intro/compose-application-model/#the-compose-file)
:::

---

## BuildKit

BuildKitはDockerのビルドエンジンであり，Dockerのビルドを高速化するための機能

### BuildKitを有効にする方法

- `DOCKER_BUILDKIT=1`を環境変数を設定する
- 実行時に指定する`DOCKER_BUILDKIT=1 docker build .`
- buildxというbuildkitを使うためのDocker CLI拡張機能を使う
  - `docker buildx build .`
  - `docker buildx bake`(docker composeの場合)

---

## Dev Container

> The Visual Studio Code Dev Containers extension lets you use a container as a full-featured development environment. It allows you to open any folder inside (or mounted into) a container and take advantage of Visual Studio Code's full feature set.
> [Dev Container](https://code.visualstudio.com/docs/devcontainers/containers)

VSCodeの拡張機能であるDev Containerを使うことで，container内でVSCodeを使うことができる機能。
Dev Containerを使用して開発環境をコンテナ化することで，以下のメリットがある。

- `devcontainer.json`に記述したVSCodeのExtensions(特にフォーマッタやlinter)，ツールが自動でインストールされ，チームで開発環境を統一することができる。
- Dev Containerを使う時のみツールはインストールできるので，Dockerfileを汚さずに開発環境でほしいツールを使うことができる(e.g. 疎通確認のためにpingやcurlが一時的にほしいみたいな)。
- 新規メンバーの環境構築にかかる工数の削減。

---

## Docker Desktop

DockerをGUIで操作するためのツール。Docker Desktopを使うことで，Dockerの操作が簡単になる。

### 便利なExtensions例

- Aqua Trivy: イメージスキャンによりコンテナの脆弱性を検出する。
- Dive in: 不要なファイルを探すことでイメージサイズの削減に役立つ

## 参考: busybox

> [busyboxについて](https://www.infra-linux.com/menu-docker3/busybox/)

- Linuxでよく使われるコマンド群が入っている軽量なツールボックス。コンテナや組み込みなどでよく使われることが多いため紹介しておく。
- バイナリは1つで，コマンドはハードリンクを貼ることで実行できるようになっている。

```shell
docker run -it busybox "/bin/sh" # bashは使えなかった
find / -inum $(ls -i /bin/busybox | cut -d " " -f3) | head -n 10 # -inumは同じinode(ファイルのメタデータが記載されている)を表示するとすべてのコマンドが表示された。 --> すべて同じファイルである。
/bin/less
/bin/false
/bin/basename
/bin/ftpget
/bin/makedevs
/bin/busybox
/bin/sh
/bin/ttysize
/bin/kbd_mode
/bin/bootchartd


# すべてのコマンドのハッシュが同じなのでハードリンクであることがわかる。
md5sum /bin/busybox; for i in $(ls /bin);do md5sum "/bin/${i}"; done
 | head -n 5
65046c52895d2ce9c9db7059c7b8e1cf  /bin/busybox
65046c52895d2ce9c9db7059c7b8e1cf  /bin/[
65046c52895d2ce9c9db7059c7b8e1cf  /bin/[[
65046c52895d2ce9c9db7059c7b8e1cf  /bin/acpid
65046c52895d2ce9c9db7059c7b8e1cf  /bin/add-shell
65046c52895d2ce9c9db7059c7b8e1cf  /bin/addgroup
```

---
