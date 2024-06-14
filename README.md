# Zenn Article and Zenn book

![un license](https://img.shields.io/github/license/RyosukeDTomita/zenn_article_book)

## INDEX

- [ABOUT](#about)
- [LICENSE](#license)
- [PREPARING](#preparing)
- [HOW TO USE](#how-to-use)

---

## ABOUT

- [Zenn Articles](https://zenn.dev/sigma_tom)

- [Zenn Books](https://zenn.dev/sigma_tom?tab=books)

---

## LICENSE

---

[FIXME](./LICENSE)

---

## PREPARING

> [公式ドキュメントの通りにやるだけ](https://zenn.dev/zenn/articles/install-zenn-cli)

- Zennのアカウントから**GitHubからのデプロイ**を有効化する。

```shell
# install
npm init --yes
npm install zenn-cli

# セットアップ
npx zenn init
```

---

## HOW TO USE

### 記事と本

> [記事と本の作り方](https://zenn.dev/zenn/articles/zenn-cli-guide)

#### 記事を書く

- articles/配下に記事を置く。

```shell
cd ~/aws_my_knowledge/
npx zenn new:article
```

- 作成されたファイル内に`published: true`にし，git pushすると公開される。

- 公開前にプレビューする

```shell
npx zenn preview
```

- 満足できたら，`git push`し，zenn側で公開設定を行うと記事が公開される。

#### 本を書く

- books/配下に本を作るためのディレクトリに置く。

### 画像を使う

> [画像の使い方](https://zenn.dev/zenn/articles/deploy-github-images)

- images/配下に画像を置く。
- 参照するときには/から書く。

```
![hoge](/images/test.png)
```

> [!WARNING]
> ../images/test.pngような書き方をしない。

---
