---
title: "Docker関連用語"
---

# docker関連用語

## 基本用語でわかりにくいもの

- attach: 接続すること。
- tty: 利用者が入力した文字を別の機器に送信したり，別の機器から受信した文字情報を利用者に提示したりする機能を持った端末やソフトウェアのこと。

## busybox

- Linuxでよく使われるコマンド群が入っている。バイナリは1つで，コマンドはハードリンクが貼られている。

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

## デタッチドvsフォアグランド

### デタッチドモード

- デタッチドモード: バックグラウンドでコマンドを実行するモード。docker runの-aオプションとはコンフリクトを起こし，コマンドの実行もできない。
- 例えばウェブサーバーを立ち上げた時にデタッチドモードならばログが画面にでない。

> [デタッチモードの違い](https://qiita.com/leomaro7/items/a96b62659ab676933f64)

```shell
docker run -it -a stdout -a stdin fileserver "/bin/bash" # フォアグラウンドモードではコマンドが実行できる
```

---

## Dockerfileとdocker-compose.yml
>
> [Dockerfileとdocker-composeを利用すると何が嬉しいのか](https://qiita.com/sugurutakahashi12345/items/0b1ceb92c9240aacca02)

- Dockerfileは単独のDockerイメージを作るための設定ファイル
- docker-compose.ymlは複数のDockerイメージを組み合わせてアプリケーションを実行するための設定ファイル

---

## BuildKit

> [BuildKitのメリット等](https://qiita.com/tatsurou313/items/ad86da1bb9e8e570b6fa)

- ビルダーインスタンスとは: BuiliKitでイメージを構築する際に使用する内部コンポーネントである。
- ビルド時にビルダーインスタンスは自動で起動するらしい。

> [!NOTE]
> [ビルダーインスタンス](https://qiita.com/shoji-kai/items/503187773e4cd94ff17d#%E3%83%93%E3%83%AB%E3%83%80%E3%83%BC%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9)によると自分で作成が必用な場合があるっぽい。

### BuildKitを使う方法

- `DOCKER_BUILDKIT=1`を環境変数を設定する or `DOCKER_BUILDKIT=1 docker build .`
- `docker buildx build .`を使う。
- [docker-compose.ymlを使いたい場合](./docker-compose.md)には`docker buildx bake`を使うことも可能。

---

## メモ

- docker container lsとdocker psは一緒
- docker ps -aででてくるのは停止しているサービスである。
- docker buildはイメージのビルドをおこなうため RUNは実行される。
- CMDはサービス起動時に実行される。
- docker imagesとdocker image lsも同じ
- CMDの実行が終わったコンテナに対してexecできない(-dをつけていても関係ない)。runは可能
- CMDの実行を止めないためにtail -f /dev/nullするなど
- docker compose up -dのデタッチは出力が止まるかどうかに関係していてコンテナの停止とは関係がない。
