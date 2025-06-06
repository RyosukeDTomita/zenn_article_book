---
title: "テスト駆動開発"
---

## テスト駆動開発の定義

> 1. 網羅したいテストシナリオのリスト（テストリスト）を書く
> 2. テストリストの中から「ひとつだけ」選び出し、実際に、具体的で、実行可能なテストコードに翻訳し、テストが失敗することを確認する
> 3. プロダクトコードを変更し、いま書いたテスト（と、それまでに書いたすべてのテスト）を成功させる（その過程で気づいたことはテストリストに追加する）
> 4. 必要に応じてリファクタリングを行い、実装の設計を改善する
> 5. テストリストが空になるまでステップ2に戻って繰り返す [^1]

> TDDは分析技法であり，実際には開発のすべてのアクティビティを構造化する技法なのだ。[^2]

## TDDを実施する時のコツ

[^2]

- 実装したい機能1つにつき，テストを1つ書いてみる。
- 一度にテストをいくつも書かない。一つのテストを通すことに集中する。
- 通っていないテストがある状態で新しいテストを作らない
- 手が動きにくい時には，仮実装を取り入れる。ベタ書きで開始するなど。
- テストがうまくかけない時には三角測量を使ってみる。いくつかassertを用意してみることで，実装のイメージがつかみやすくなる。
- この場合どうなるんだろう?と疑念が浮かんだらテストのリストに追加して実装してみる。
- コードを書く前にどこに行くべきかがわかるまではコードを書き始めない。
- アサートファースト: テストを書く際にassertを最初に書く。
- テストで使用するデータに意味をもたせる。1でも2でもいいなら1を使う。
- 最後に書いたテストが失敗する状態で離席すると，中断前の状態を思い出しやすい。
- 15分から30分で一周できるようにする。

---

## Reference

- [^1]: [翻訳 テスト駆動開発の定義](https://t-wada.hatenablog.jp/entry/canon-tdd-by-kent-beck)
- [^2]: [テスト駆動開発 Kent Bech](https://shop.ohmsha.co.jp/shopdetail/000000004967/)
