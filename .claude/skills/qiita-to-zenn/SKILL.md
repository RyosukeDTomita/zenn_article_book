---
name: qiita-to-zenn
description: Qiita記事URLからZennのクロス投稿記事(articles/<slug>.md)を生成する。Qiita API v2で本文Markdownを取得し、ZennのフロントマターとZenn固有記法へ変換して保存する。クロス投稿の案内ボックスとQiita原文へのリンクも自動挿入する。
when_to_use: ユーザーが「このQiita記事をZennにも登録」「クロス投稿の準備」「Qiita記事をZenn記事化」などQiita→Zennの移植を依頼したとき。Qiita記事のURL(https://qiita.com/<user>/items/<id>)が示されたら本skillを使う。
argument-hint: [qiita-url] [slug]
allowed-tools: Bash Read Write Edit
---

# Qiita → Zenn クロス投稿

Qiita記事をこのリポジトリのZenn記事(`articles/<slug>.md`)として生成する。

## 入力

- 第1引数: Qiita記事URL(`https://qiita.com/<user>/items/<item_id>`)。未指定なら直近の会話から特定するか、ユーザーに尋ねる。
- 第2引数(任意): Zennのslug。未指定なら内容から英小文字・数字・ハイフンの12〜50文字で命名する(例: `haskell-maybe-optional-null`)。

## 手順

### 1. Qiita APIで本文を取得

`item_id` をURL末尾から取り出し、Qiita API v2で取得する。

```bash
curl -s "https://qiita.com/api/v2/items/<item_id>" -o /tmp/qiita_item.json
python3 -c "import json;d=json.load(open('/tmp/qiita_item.json'));print(d['title']);print([t['name'] for t in d['tags']])"
python3 -c "import json;print(json.load(open('/tmp/qiita_item.json'))['body'])" > /tmp/qiita_body.md
```

`title` と `tags`(Zennの`topics`にする)と `body`(本文Markdown)を使う。

### 2. Zennフロントマターを付ける

```yaml
---
title: "<Qiitaのtitleそのまま>"
emoji: "<内容に合う絵文字を1つ>"
type: "tech"          # 技術記事。アイデア系なら idea
topics: [<Qiitaのtags配列をそのまま>]
published: true       # まず確認するなら false で下書き
---
```

その直後にクロス投稿の案内ボックスを置く(原文URLは引数のQiita URL)。

```
:::message
この記事はQiitaとのクロス投稿です。
<qiita-url>
:::
```

### 3. Qiita記法 → Zenn記法へ変換

`/tmp/qiita_body.md` を本文に流し込みつつ、以下を必ず変換する。

- **noteボックス**: Qiitaの `:::note info` / `:::note warn` → Zennの `:::message`、`:::note alert` → `:::message alert`。閉じる `:::` はそのまま。
- **ファイル名つきコードブロック**: Qiitaは `` ```ファイル名.ext ``(ファイル名のみ)だが、Zennは `言語が先`。`` ```ext:ファイル名.ext `` の形へ直さないとシンタックスハイライトが消える。
  - 例: `` ```NullPo.java `` → `` ```java:NullPo.java ``
  - 拡張子→言語の対応は内容から判断する(`.java`→`java`, `.hs`→`haskell`, `.py`→`python`, `.ts`→`typescript` など)。
  - 言語名のみのフェンス(`` ```java ``, `` ```haskell ``, `` ```shell ``, `` ```text ``)はそのままでよい。
- **水平線**: `---` の3文字を使う(`___` / `***` は使わない)。Qiita原文が `---` ならそのまま。
- **画像**: QiitaのS3画像URL(`qiita-image-store.s3...`)は外部URLとしてそのまま参照できる。Qiita側削除で画像が消えるリスクをユーザーに一言伝え、必要なら `images/` 配下へ取り込む。
- **AtCoder等のURL単独行**: ZennもURL単独行はカード展開されるのでそのまま。

### 4. 保存と報告

`articles/<slug>.md` に書き出す。最後に次を報告する。

- 作成パス
- 行った変換(noteボックス変換、コードフェンス修正の箇所数 など)
- `published` の値(下書きにしたい場合は `false` に変える旨)
- 画像を外部URL参照にしているリスク

## 注意

- このリポジトリは Zenn 連携リポジトリ。記事は必ず `articles/` 配下に置く。
- frontmatter以外の本文はQiita原文の意味を変えない(機械的な記法変換のみ)。
