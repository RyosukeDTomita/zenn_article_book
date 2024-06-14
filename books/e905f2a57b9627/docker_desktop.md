---
title: "環境構築/DockerDesktop"
---

# Docker Desktop

## 環境構築

- [Ubuntu に Docker Desktop をインストール](https://docs.docker.jp/desktop/install/ubuntu.html)を見ながら deb パッケージをインストールするだけ。
- gpg の設定をやっておくとよい。

```shell
gpg --generate-key
pass init my pass key
```

---

## 機能

- docker scout: Early Accessで無料なだけかも?イメージスキャンができる。

## Extensions

- Aqua Trivy
- Dive in: 不要なファイルとかを探せる。

### VSCode

- Docker: Docker DesktopをVSCodeから使ってビルドとかできる。

---

## メモ

- 一旦設定のGeneral --> Start Docker Desktop when you log inをオンにしておく。Docker Desktopが起動していない時に作ったリソースが反映されないっぽい。
