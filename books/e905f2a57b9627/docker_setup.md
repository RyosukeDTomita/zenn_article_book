---
title: "環境構築/main"
---

# Docker 環境構築

## 動作確認済み環境

- Ubuntu 20.04
- WSL2 Ubuntu(22.04)

## install

- [公式ドキュメントをみてインストールする](https://matsuand.github.io/docs.docker.jp.onthefly/engine/install/ubuntu/)
- sudo 以外で docker コマンドを実行できるようにする。

```shell
sudo usermod -aG docker ${USER}
su - ${USER}
id -nG #dockerが含まれていることを確認
```

---

## uninstall

> [docker を全て消す](https://arkgame.com/2022/05/14/post-308016/)
リンクが消えたときのために概要を書いておく。

```shell
dpkg -l | grep -i docker # dockerに関連するパッケージを検索。
# パッケージの削除
sudo apt-get purge -y docker-engine docker docker.io docker-ce docker-ce-cli 
sudo apt-get autoremove -y --purge docker-engine docker docker.io docker-ce
# すべてのdocker関連ファイルを削除する
sudo rm -rf /var/lib/docker /etc/docker
sudo groupdel docker
sudo rm -rf /var/run/docker.sock
sudo rm -rf /usr/local/bin/docker-compose
sudo rm -rf /etc/docker
sudo rm -rf ~/.docker
```

---

## docker proxy 設定

企業等では社外への通信はプロキシを通す必要があるので必要なら設定する。

> [参考](https://qiita.com/dkoide/items/ca1f4549dc426eaf3735)

- 環境変数
- ~/.docker/config.json

```json
{
  "proxies": {
    "default": {
      "httpProxy": "http://tomita:password@localhost:81",
      "httpsProxy": "http://tomita:password@localhost:81"
    }
  }
}
```

- /etc/systemd/system/docker.service.d/override.conf

```
[Service]
Environment = 'http_proxy=http://tomita:password@localhost:81' 'https_proxy=http://tomita:password@localhost:81
```

> [!NOTE]
> `sudo systemctl edit docker`するとうまく行かなかったので`mkdir /etc/systemd/system/docker.service.d`した後に vim で設定した。

- dockerサービスを再起動

```shell
sudo systemctl restart docker
```

場合によっては追加でコマンドを実行するように表示がでることがあるので標準出力の指示に従う。

- 接続チェック

```shell
sudo docker info # 設定できているかチェック proxy設定がでてくればOK
```

---

## Docker Desktop

[./docker_desktop.md](./docker_desktop.md)を参照。

---

## VSCodeをDockerコンテナ内で開く

[./devcontaienr.md](./devcontainer.md)を参照。
