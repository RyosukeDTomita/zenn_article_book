---
title: "「foldrで短絡できていませんでした」の反省札"
emoji: "🙇"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Haskell", "AtCoder", "競技プログラミング", "関数型プログラミング", "foldr"]
published: true
---

:::message
この記事はQiitaとのクロス投稿です。
https://qiita.com/sigma_devsecops/items/3234a53bb3d0e2d8ab8f
:::

## これは何?

ここ1ヶ月くらい、[Haskell](https://www.haskell.org/)で精進している。

下記の問題を[`foldr`](https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Data-List.html#v:foldr)を使ってACしたのだが、意図したものと異なる美しくないコードを実装してしまった。

[反省札](https://dic.pixiv.net/a/%E5%8F%8D%E7%9C%81%E6%9C%AD)として本記事を執筆し、今回の失敗の解像度を上げたいと思う。

https://atcoder.jp/contests/abc071/tasks/abc071_b

---

## そもそも短絡とは?

> Short-circuit folds
>
> Examples of short-circuit reduction include various boolean predicates that test whether some or all the elements of a structure satisfy a given condition. Because these don't necessarily consume the entire list, they typically employ foldr with an operator that is conditionally strict in its second argument. Once the termination condition is met the second argument (tail of the input structure) is ignored. No result is returned until that happens. [^1]

短絡は英語だとShort-circuitと言う。

Haskellには、畳み込み(fold)に使う関数が多数存在するが、[`foldr`](https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Data-List.html#v:foldr)は短絡ができる。

`foldl`などの短絡ができない畳み込みを使用する場合にはリストの要素を全て走査する必要がある。

しかし、`foldr`のような短絡ができる畳み込みの場合、リスト全てを走査する前に条件が満たされた場合、残りの走査は実施されない。

これは、Haskellが遅延評価という、値が使用される際に評価する評価方法を採用しているため、実現できている。

例として、[`all`](https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Data-List.html#v:all)という全てがTrueの時だけTrueを返す関数を使って説明する。
これは、簡略化して書くと以下のように書ける。

```haskell
all :: (a -> Bool) -> [a] -> Bool
all f xs = foldr (\x e -> f x && e) True xs
```

これを使い、リスト`[3,2,1]`がすべて4より大きいかを調査してみる。

```haskell
all (>4) [3,2,1]
```

これは以下のように展開される。

```
foldr (\x e -> (>4) x && e) True [3,2,1]
```

```
(\x e -> (>4) x && e) 3 (foldr (\x e -> (>4) x && e) True [2,1])
```

一番左のラムダ式を評価すると、

```
(>4) 3 && (foldr (\x e -> (>4) x && e) True [2,1])
```

つまり、このようになる。

```
False && (foldr (\x e -> (>4) x && e) True [2,1])
```

`&&`は遅延評価により、左側が`False`の場合、右側を評価せず、`False`を返す。
つまり、条件を満たさない要素が1つ見つかるまで評価は行われるが、1つ見つかって以降は評価が行われない。

これが短絡の強みである。

---

## 自分の良くない実装: `foldr`を使った短絡をしたつもりができていない

自分は、`foldr`を使うと短絡できることはなんとなく知っていたがうまく実装できていなかった。

以下のコードの何が良くないだろうか?

```haskell
{-# LANGUAGE MonoLocalBinds #-}
{-# OPTIONS_GHC -Wno-x-partial #-}
{-# OPTIONS_GHC -Wunused-imports #-}

import Control.Arrow ((>>>))
import Data.Set qualified as Set

solve :: String -> String
solve s = if null result then "None" else result
  where
    -- foldrを使い、一つ見つかったら残りは評価しない(短絡)したつもりでnullチェックは毎回走ってしまう。
    result = foldr (\c acc -> (if null acc && c `Set.notMember` sSet then c : acc else acc)) [] $ reverse ['a' .. 'z'] -- ['a'..]だとUnicode全域になってしまう
    sSet = Set.fromList $ init s

main :: IO ()
main =
  interact $
    solve >>> (++ "\n")

```

確かに、`Set.notMember`という$O(\log n)$の走査を`null`チェックを入れることで全リスト(全てのアルファベットの文字)に対して実施しないようにはなっている。
しかし、このコードでは全てのリストに対して`null`チェックが走ってしまっている。

うまく短絡することができれば、1つ見つかった以降は`null`チェックは評価しなくて良いのにである。

では、なぜこの実装だと短絡ができていないのだろうか。
この実装の良くないところは、`if null acc`の部分で`acc`の評価を強制しているところである。
`acc`を返すだけのような評価を強制しない操作であれば、問題ない。
`acc`の評価を強制する関数を正格な演算子と呼んだりする。

自分の過去記事で[^2]正格な演算子である`+`を使い、`foldr`でリストの合計を求める際にどのように畳み込みが行われるかを説明した。

```
-- 再掲
foldr (+) 0 [1,2,3,4]
→ 1 + foldr (+) 0 [2,3,4]
→ 1 + (2 + foldr (+) 0 [3,4])
→ 1 + (2 + (3 + foldr (+) 0 [4]))
→ 1 + (2 + (3 + (4 + foldr (+) 0 [])))
→ 1 + (2 + (3 + (4 + 0))) 再帰展開終了
→ 1 + (2 + (3 + 4)) unwindフェーズ開始。サンク1つ消費
→ 1 + (2 + 7) サンク1つ消費
→ 1 + 9 サンク1つ消費
→ 10 サンク1つ消費
```

`+`をあえて、`(\x acc -> x + acc)`のようにラムダ式に書き換えてみるとわかりやすい。

```
foldr (\x acc -> x + acc) 0 [1,2,3,4]
→ (\x acc -> x + acc) 1 (foldr (\x acc -> x + acc) 0 [2,3,4])
```
この場合、`x`は`1`、`acc`は`(foldr (\x acc -> x + acc) 0 [2,3,4])`となっているのがわかる。
つまり、正格な演算子を評価するためには`x`、`acc`の両方の評価が必要なのである。

そのため、先程の自分の解答は畳み込みに使用する関数が正格であるため、`foldl`同様に最後までリストを走査しており、`foldl`で書き換え可能な状態となってしまっている。
(なんなら、`foldr`バージョンは`reverse`しているので読みにくいし、計算コスト的にも無駄が多い)。

```haskell
-- foldlで書くとこうなる
result = foldl (\acc c -> (if null acc && c `Set.notMember` sSet then c : acc else acc)) [] ['a' .. 'z'] -- ['a'..]だとUnicode全域になってしまう

```

---

## 短絡するにはどうすればよかったか?

`foldr`の畳み込みに使用する関数を非正格にすればよい。
具体的には`acc`の評価が強制されない形にする。
以下の例```(\c acc -> if c `Set.notMember` sSet then [c] else acc)```では、条件判定に`acc`を使わず`c`だけで分岐している点がポイントである。`c`が`sSet`に存在しない時は`acc`を無視して`[c]`を返すため、残りの`foldr`(=`acc`)は評価されず短絡できる。

```
foldr (\c acc -> if c `Set.notMember` sSet then [c] else acc) "None" ['a' .. 'z']
-- `a`が`sSet`に存在しない場合
-> (\c acc -> if c `Set.notMember` sSet then [c] else acc) 'a' (foldr (\c acc -> if c `Set.notMember` sSet then [c] else acc) "None" ['b' .. 'z'])
-> ['a']
```

```haskell
{-# LANGUAGE MonoLocalBinds #-}
{-# OPTIONS_GHC -Wno-x-partial #-}
{-# OPTIONS_GHC -Wunused-imports #-}

import Control.Arrow ((>>>))
import Data.Set qualified as Set

solve :: String -> String
solve s = if null result then "None" else result
  where
    result = foldr (\c acc -> if c `Set.notMember` sSet then [c] else acc) "None" ['a' .. 'z'] -- ['a'..]だとUnicode全域になってしまう
    sSet = Set.fromList $ init s

main :: IO ()
main =
  interact $
    solve >>> (++ "\n")

```

今回解いた問題は、計算量がもともと少ない問題であったため、パフォーマンス改善はわずかであった(15ms -> 9ms)

[2026年7月追記](https://qiita.com/drafts/3234a53bb3d0e2d8ab8f/edit#2026%E5%B9%B47%E6%9C%88%E8%BF%BD%E8%A8%98)に別の問題でパフォーマンスについて追記した。

---

## おまけ

`foldr`で書いても良いが、同じことができるHaskellの便利な関数を紹介する。

[`find`](https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Data-List.html#v:find)は[^1]に記載のあるように短絡できる関数の一つなので、条件を満たす値を1つ見つけた段階で`Just a`を返して短絡する。

```haskell
{-# OPTIONS_GHC -Wno-x-partial #-}
{-# OPTIONS_GHC -Wunused-imports #-}

import Control.Arrow ((>>>))
import Data.List (find)
import Data.Set qualified as Set

-- Sに含まれない最小の英小文字を返す。全て含む場合はNone
solve :: String -> String
solve s = case find (`Set.notMember` sSet) ['a' .. 'z'] of
  Just c -> [c]
  Nothing -> "None"
  where
    sSet = Set.fromList s

main :: IO ()
main =
  interact $
    lines >>> head >>> solve >>> (++ "\n") -- 入力値側で改行を削除

```

---

## 2026年7月追記

自分は`foldr`や`find`のようなメソッドを使うほうが好みではあるため、[コメント](https://qiita.com/sigma_devsecops/items/3234a53bb3d0e2d8ab8f#comment-728d4ffcd7e87d3b3b75)いただいたような手で再帰を書かないように努力している。
だが、頑張っても畳み込みでは短絡できない問題もあるのでこういうのは諦めて手で再帰したほうが良さそう。

https://atcoder.jp/contests/abc121/tasks/abc121_c

```haskell
{-# LANGUAGE BangPatterns #-}
{-# LANGUAGE MonoLocalBinds #-}
{-# OPTIONS_GHC -Wno-x-partial #-}
{-# OPTIONS_GHC -Wunused-imports #-}

import Data.List (sortOn)


solve :: Int -> [[Int]] -> Int
solve m abList = fst $ go (0, 0) abTupleList
  where
    abTupleList = sortOn fst $ map (\[a, b] -> (a, b)) abList
    go :: (Int, Int) -> [(Int, Int)] -> (Int, Int)
    go (!result, !cnt) _
      | cnt == m = (result, cnt)
    go (!result, !cnt) ((a, b) : rest) =
      let n = min b (m - cnt)
       in go (result + (n * a), cnt + n) rest

main :: IO ()
main = interact $ \inputs ->
  let ls = lines inputs
      [n, m] = map read . words $ head ls :: [Int]
      abList = map (map read . words) $ drop 1 ls :: [[Int]]
   in show (solve m abList) ++ "\n"

```

<details><summary>`foldl`版のコード</summary>

```haskell
-- foldl'版。短絡せず全要素を処理してみる
{-# LANGUAGE BangPatterns #-}
{-# LANGUAGE MonoLocalBinds #-}
{-# OPTIONS_GHC -Wno-x-partial #-}
{-# OPTIONS_GHC -Wunused-imports #-}

import Data.List (foldl', sortOn)

solve :: Int -> [[Int]] -> Int
solve m abList = fst $ foldl' go (0, 0) abTupleList
  where
    abTupleList = sortOn fst $ map (\[a, b] -> (a, b)) abList
    go :: (Int, Int) -> (Int, Int) -> (Int, Int)
    go (result, cnt) (a, b) =
      let n = min b (m - cnt)
       in (result + (n * a), cnt + n)

main :: IO ()
main = interact $ \inputs ->
  let ls = lines inputs
      [n, m] = map read . words $ head ls :: [Int]
      abList = map (map read . words) $ drop 1 ls :: [[Int]]
   in show (solve m abList) ++ "\n"

```

</details>

と比較して633 ms -> 404 msと大幅な高速化に成功!
メモリ使用量も1/3はいかないものの大幅に削減されている。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3718390/2b92dca0-229e-47f9-9263-a2447e8b6987.png)
↑(真ん中はfoldrで短絡に失敗した実装の結果)

---

## Reference

[^1]: https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Data-Foldable.html#g:14
[^2]: [Haskellのfoldl、foldl'、foldrを比較してみた](https://qiita.com/sigma_devsecops/items/206874ce5130abe280da)
