---
title: 初期設定"
---

## IAMユーザのセットアップ

- 日常的にrootアカウントを利用することは避け，IAMユーザを作ってログインするようにする。
- マイページ --> アカウントからIAMユーザ/ロールによる請求情報へのアクセスを有効にする。
- AWSマネジメントコンソールへのアクセスを選択し，パスワードのリセットを解除。
- 一旦はAdministoratorAccessを付与しておく。
- **ログイン用のURL**が表示されるのでブックマークしておくと楽
    > [ログインURL](https://xxxxxxxxxxx.signin.aws.amazon.com/console/)

### Organizationsを作ってSSOサインインできるようにする

いわゆるおひとりさまOrganizationsというやつ。参考になりそうな記事を貼っておく

- [アクセスキーを使ったaws-cliはもうやめよう](https://qiita.com/s_moriyama/items/14b703cc0dfa91a6f464)
- [「おひとり様」AWS Organizationsを運用する](https://qiita.com/kyooooonaka/items/af3b36d5e946b3152021)

#### AWS CLIの設定

[AWS CLIの設定方法](./aws-cli.md)

---

## 請求周りの設定

### Billing Dashboard: 請求関連の情報を見るページ

- BIlling preferencesを選択 --> Invoice delivery preferencesからpdfの請求情報を受け取るにチェック
- Alert preferencesから無料枠を超えそうになるとアラートをオン。
- Cloud Watch請求アラートをオンにしておく --> いくらを超えたらアラートを上げる等ができるように鳴る。

### Cloud Watchの設定

- アラーム --> すべてのアラーム --> アラームの作成
- リージョンをバージニアに変更するとBilling，Total Estimated Chargeが選べるようになる。
- 一旦，10ドルを超えたらアラームが鳴るように設定する。
- 通知の設定からメールアドレスを入れるのを忘れずに

### 請求書の確認方法

- 請求書
- コストエクスプローラー: 条件で絞り込みできる。請求タグとしてなにかタグをつけておくと良い。 --> コスト分析を実施してレポートを生成するツール。当月の使用料金の確認や，次の月末の予想請求額の計算が可能。実際の支払額を見るものではない。
- Pricing Calculator: 詳細な条件で金額を見積もりできる。

---

## Cloud Trailで操作ログを残す

- ユーザの操作によるイベントに対してモニタリング。
- デフォルトで有効になっているが，保存期間は90日のみ。
- S3に操作ログを置くように設定すれば無期限で保存できるがS3の料金は必要。ログには料金はかからない。
- Multi-region trailはデフォルトでオンであり，どのリージョンでも情報が見れる。
- Trailを作成したらCloudWatch LogsをEnableにする。
- デフォルトでManagement eventsはAllになっているのですべての操作が保存される。
- データイベント: リソース上，またはリソース内で実行されたリソースオペレーションについての情報を得る(ex: S3バケットおよび，バケットないオブジェクトに対するGetObject等のAPIアクティビティ)。追加料金がかかる。
- Insight イベント: 管理アクティビティを分析して，異常なAPIアクティビティやエラー率を検出する。従量課金性。データイベントを使用していることが前提。

> [データイベントやinsightイベント](https://docs.aws.amazon.com/ja_jp/awscloudtrail/latest/userguide/cloudtrail-concepts.html#cloudtrail-concepts-data-events)

---

## AWS Config
>
> [参考](https://qiita.com/OPySPGcLYpJE0Tc/items/0da76225e763b2911f29)

- Cloud Trailと異なり，リソースの変更にたいしてモニタリング。
- コンソールから検索する時はConfigで
- Ruleはユーザが選択してそれを記録する。
- SettingsからRecordingsをオフにすると課金が止まる。
- **無料枠がないのでちょっと使っただけで0.2ドルほどお金がかかってしまうので注意**

```shell
# AWS Cli経由でAWS Configを削除する方法(作ってみて不要だった場合)

aws configservice describe-delivery-channels
{
    "DeliveryChannels": [
        {
            "name": "default",
            "s3BucketName": "config-bucket-177179343845"
        }
    ]
}

aws configservice delete-delivery-channel --delivery-channel-name default # nameを指定して削除
aws configservice describe-delivery-channels # 削除確認
{
    "DeliveryChannels": []
}
```

---

## CloudWatch

- CloudWatchの中にCloudWatch Logsの項目もある。
- Metricsから選択することでデフォルトで取得できる情報を見れる。 --> Dashboardに登録可能。
- デフォルトではユーザ(お客様)の管理範囲の情報は収集しない。
- EC2: CPUやハードウェア，ネットワークのステータスは収集されるがメモリやアプリケーションのステータスなどは収集されない。
- RDS: フルマネージドサービスなのでメモリの情報やディスクの使用量の情報も収集される。

### EC2にカスタムメトリクスを設定する

- [IAMロールの設定方法](https://qiita.com/t_okkan/items/9bec49fa5be76de4e5ef)を見て`CloudWatchAgentServerPolicy`，`AmazonEC2RoleforSSM`，`AmazonSSMReadOnlyAccess`を含んだロールを付与。

> SSM経由でインスタンスにアクセスしたいのでAmazonSSMFullAccessを付与。

- [カスタムメトリクスの設定](https://kun432.hatenablog.com/entry/ec2-custom-metrics-with-cloudwatch-agent)agentをインストールしてカスタムファイルを作成し，サービス起動して再起動する。

```shell
sudo yum install amazon-cloudwatch-agent collectd -y
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard # コンフィグを作る。
cp /opt/aws/amazon-cloudwatch-agent/bin/config.json /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json # 作成したjsonのパスはおかしいから移動する。
sudo systemctl enable --now amazon-cloudwatch-agent
```

- CloudWatchのDashBoard等に追加する。 --> デフォルトではCPU使用率は見れたがディスク容量やメモリ情報は見られなかったが見れるようになった。

> 表示が0-1にデフォルトはなっているのでグラフのタイプを変えたりすると直る。

### アプリケーションログの書き出し

- やり方は基本的にカスタムメトリクスの設定と同じ。
- コンフィグを作る手順で他に収集したいログがあるか聞かれた時にyesにして/var/log/httpd/access_logと/var/log/httpd/error_logを指定する。

> [参考](https://blog.serverworks.co.jp/cloud-watch-logs-apache-access-log-setting)

---

## AWSのサポートプランについて

- ベーシック: 技術サポートは稼働状況のヘルスチェックについてのサポートのみ。Trusted Advisorは一部。
- 開発者: 1アカウントのみ9-18時で技術サポートを使える。アーキテクチャガイダンス。
- ビジネスプラン: Trusted Advisorが全項目に。技術サポートを作成できるユーザが無制限で24時間に。サポートAPIの使用。アーキテクチャユースケース。サードパーティー製ソフトウェアのサポート。
- エンタープライズ: TAM(AWSのスペシャリスト)，コンシェルジュサポート(アカウントに対する問い合わせに迅速な対応をしてくれる)，ホワイトグローブケースルーティング。

---

## Amazon Inspector

- EC2，Lambda，ECR(フルマネージドDockerコンテナレジストリでイメージの保存と取得を行う)が対象でリソース変更時に自動的にスキャンしてリスクスコアを出してくれる。
- General settingsからスイッチ一つで起動と停止ができる。
- CVEとか出してくれる。

---

## Access Advisor

- アクセス可能なサービスと過去のアクセス履歴が見られる。
- IAMユーザ，IAMロール画面のタブにある。

---

## Access Analyser

- IAMのAccess reportsから作れる。
- 信頼ゾーン外からのアクセスを見れる。
- リージョンごとに作成可能

---

## Security Hub

- セキュリティのベストプラクティスのチェックを行う
- General Settingsから停止できる。

## Guard Duty

- 不審なアクティビティを検知する。
