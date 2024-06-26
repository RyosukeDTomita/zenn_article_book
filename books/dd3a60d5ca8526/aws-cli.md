---
title: "初期設定/AWS-CLIについて"
---

# AWS Command Line Interface

## 概要とインストール

- AWS Management Consoleで提供される機能と同等の機能を実装することができる。
- [install](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html#cliv2-linux-install)

```shell
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

---

## 初期設定

### IAMユーザの作成

[参考](https://udemy.benesse.co.jp/development/system/aws-cli.html)

- Programmatic access: プログラムによるアクセスをチェックする。
- Add user to Group --> Create group --> Filter policies --> AWS managed-job function
- csvでID等をダウンロード
- [cliの設定](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-quickstart.html)

```shell
aws configure
# regionはデフォルトは最も近い場所，outputはjsonである。
# ~/.aws/configにoutputとregionの設定
# ~/.aws/credentials # aws_secret_access_keyとaws_access_key_idを設定
```

```
# ~/.aws/config
[default]
region = ap-northeast-1
output = json
```

```
# ~/.aws/credentials
[default]
aws_access_key_id = AKxxxxxxxxxxxxxxxxxx
aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

使えるかテストする

```shell
aws sts get-caller-identity
```

### tab補完

[コマンド補完](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-completion.html)
.bashrcに追加する。

```sh
complete -C '/usr/local/bin/aws_completer' aws
```

---

## AWS-CLIでスイッチロールする

1. GUIでスイッチロールした先でaws-cli用のアカウントを作成する。
2. ~/.aws/configと~/.aws/credentialsに追記する

```
# ~/.aws/config
[profile super-admin-config]
region = ap-northeast-1
role_arn = arn:aws:iam::スイッチロール前のアカウントid:role/AdministratorRole
mfa_serial = arn:aws:iam::スイッチロール後のid:mfa/tomita(スイッチロール前のアカウントのiamユーザ名)
source_profile = super-admin # ~/.aws/credentialsのプロファイル名を指定
```

```
# ~/.aws/credentials
[super-admin]
aws_access_key_id = AKxxxxxxxxxxxxxxxxxx
aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

スイッチロール側のアカウントになっていることを確認する。

```shell
aws sts get-caller-identity --profile super-admin
```

---

## 主なコマンド

### --profile

~/.aws/credentialsを編集することでプロファイルを作成でき，これを指定できる。

```
[default]
aws_access_key_id =
aws_secret_access_key =
[test]
aws_access_key_id =
aws_secret_access_key =
```

### --region

リージョンを指定する。

### --output

- コマンドの出力形式を変更できる。

### --query

- aws cliで出力されるJSON形式の標準出力をJMESPathの使用に準拠してフィルタリングできる(必要な項目だけスライスできるイメージ)。

```sh
aws --query Users[0].[UserName, Path , UserId]
```

### --fileter

- queryと同じようなコマンドだが，サーバーサイドで処理が行われる。
- すべてのコマンドが使えるわけではない。

### サブコマンド

#### help

#### wait

- 特定の条件がみたされるまでAWS CLIを待機させる。
