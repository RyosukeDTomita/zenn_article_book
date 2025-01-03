---
title: "環境構築"
---

## 動作確認済み環境

- Ubuntu 20.04
- WSL2 Ubuntu(22.04)
多分他の環境でも動くと思うが，確認していないので注意。

## install and setup

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

### 設定箇所

- 環境変数に設定

```shell
export http_proxy=http://sigma:password@localhost:81
```

- ~/.docker/config.json

```json
{
  "proxies": {
    "default": {
      "httpProxy": "http://sigma:password@localhost:81",
      "httpsProxy": "http://sigma:password@localhost:81"
    }
  }
}
```

- /etc/systemd/system/docker.service.d/override.conf

```
[Service]
Environment = 'http_proxy=http://tomita:password@localhost:81' 'https_proxy=http://tomita:password@localhost:81
```

:::message
`sudo systemctl edit docker`を使ってoverride.confの設定をすることも可能そうだが，自分はうまくいかなかったので`mkdir /etc/systemd/system/docker.service.d`した後に手動でoverride.confを作成して記述した。
:::

### 設定反映

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

- [Ubuntu に Docker Desktop をインストール](https://docs.docker.jp/desktop/install/ubuntu.html)を見ながら deb パッケージをインストールするだけ。
- gpg の設定をやっておくとよい。

```shell
gpg --generate-key
pass init <my pass key>
```

---
