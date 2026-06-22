---
title: "Haskellでは存在しない値をどう扱うか? 〜Maybeを使う〜"
emoji: "🤔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Haskell", "Java", "Maybe"]
published: true
---

:::message
この記事はQiitaとのクロス投稿です。
https://qiita.com/sigma_devsecops/items/4c9986b025a47e5bded8
:::

## はじめに: `Maybe`を過小評価していた

Haskellで競技プログラミングをやっていると[`Data.Maybe`](https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Data-Maybe.html)(以降`Maybe`と記載)を扱うことがある。

自分は積極的に`Maybe`を使うことは行っておらず、どちらかというと[チェック例外](https://e-words.jp/w/%E6%A4%9C%E6%9F%BB%E4%BE%8B%E5%A4%96.html)のようにコンパイルエラーがでるから`Just`と`Nothing`の処理をしている状態であったが、最近`Maybe`を使うことでコードが読みやすくなることに気がついた。

そのため、この記事では以前の自分のように雰囲気で`Maybe`を使っている人向けに、その良さを少しでも伝えることを目的としている。

### 環境

サンプルとして記載しているコードは以下の環境で動作検証を行っている

- 2026年6月2日時点での[AtCoder](https://atcoder.jp/)環境
- 自分の手元の環境
  - OS: Ubuntu 24.04 LTS
  - GHC: 9.6.7
  - OpenJDK: 21.0.10 2026-01-20

---

## Javaでは存在しない値をどう扱うか

Haskellの`Maybe`について語る前に比較対象として、Javaでの存在しない値の扱いについて書いておく。

### `null`を使う

JLSの`null`に関する記述を引用する。

> 3.10.8 The Null Literal
>
> The null type has one value, the null reference, represented by the null literal null, which is formed from ASCII characters.
> NullLiteral:
> null
>
> A null literal is always of the null type (§4.1). [^1]

また、`null`型そのものについては次のようにも述べられている。

> There is also a special null type, the type of the expression null (§3.10.8, §15.8.1), which has no name.
>
> Because the null type has no name, it is impossible to declare a variable of the null type or to cast to the null type.
>
> The null reference is the only possible value of an expression of null type. [^2]

上記を要約すると、以下のようになる。

- `null`型とはただ一つ、`null`参照(=`null`リテラル)だけを値空間に持つ、名前を持たない特別な型である
- `null`型には名前がないため、`null`型の変数を宣言したり、`null`型へのキャストはできない

    ```java
    public class NullTypeDemo {
        public static void main(String[] args) {
            // null x1 = null; // null型の変数は作れない error: not a statement
            // Object x2 = (null) "hello"; // null型へのキャストはできない error: ';' expected

            // nullリテラルは任意の型の変数に代入できる。
            String s = null;
            // null型からのキャスト(null参照を他の参照型へキャスト)は可能
            String s2 = (String) null;
            System.out.println(s2); // nullと出力される
        }
    }
    ```

Javaの`null`には`NullPointerException`まわりにつらい部分が詰まっている。

#### `null`つらいポイント①: コンパイル時に`null`チェックができない

前述した通り、`null`リテラルは任意の型に代入することができるが、`null`に対してメソッドを呼び出してもコンパイル時に検知できず、実行時に`NullPointerException`が上がる。

```java:NullPo.java
public class NullPo {
    public static void main(String[] args) {
        String word = null;
        System.out.println(word.length()); // nullに対してメソッド呼び出し -> NullPointerException
    }
}
```

```shell
javac NullPo.java # コンパイルエラーなし
java NullPo
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "String.length()" because "<local1>" is null
        at NullPo.main(NullPo.java:4)
```

#### `null`つらいポイント②: どこから`null`が入ったのかわかりにくい

`null`リテラルがどこで混入されたのかもわかりにくいという問題点もある。

以下の簡易的なサンプルコードの場合、`getWord()`メソッド経由で`null`が入ったのか、直接`word`に`null`が代入されたのかがエラーメッセージからわからない。

```java:NullPo.java
// getWordからnullが返された例
public class NullPo {
    public static void main(String[] args) {
        // String word = null;
        String word = getWord();
        System.out.println(word.length()); // nullに対してメソッド呼び出し -> NullPointerException
    }
    public static String getWord() {
      return null; // メソッドが失敗してnullが返されたとイメージしてほしい
    }
}
```

```java:NullPo.java
// nullで値が上書きされた例
public class NullPo {
    public static void main(String[] args) {
        // String word = getWord();
        String word = null;
        System.out.println(word.length()); // nullに対してメソッド呼び出し -> NullPointerException
    }
    public static String getWord() {
      return null; // メソッドが失敗してnullが返されたとイメージしてほしい
    }
}
```

```shell
# 実行時エラーは同じ
java NullPo.java
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "String.length()" because "<local1>" is null
        at NullPo.main(NullPo.java:5)
```

### `Optional`を使って多少`null`のつらい部分を解決する

このように、`null`の抱える問題を解決するため、Javaには`Optional`が存在する。

> A container object which may or may not contain a non-null value. If a value is present, isPresent() returns true. If no value is present, the object is considered empty and isPresent() returns false. [^3]

`Optional`を使うことで、プログラマはその値が存在しないかもしれないことを知ることができる。

以下の例は`null`を返すレガシーAPIを`Optional`でラップした例だ。

```java
import java.util.Optional;

public class Opt {
    // nullを返すレガシーAPIをそのまま使いたくない!
    static String legacyFind(String id) {
        return id.equals("1") ? "Alice" : null;
    }
    // Optionalでラップ
    static Optional<String> find(String id) {
        return Optional.ofNullable(legacyFind(id));
    }
    public static void main(String[] args) {
        // String user1 = find("1"); // これはコンパイルエラー incompatible types
        Optional<String> user1 = find("1"); // Optional[Alice]
        System.out.println(user1.get()); // Alice
    }
}
```

しかし、`Optional`にもいくつか抜け穴が存在する。

まずは、`Optional`自体に`null`を再代入することが可能である点だ。
この場合も、コンパイルエラーでは検知できず、実行時に`NullPointerException`が発生してしまう。

```java
import java.util.Optional;

public class Opt {
    // nullを返すレガシーAPIをそのまま使いたくない!
    static String legacyFind(String id) {
        return id.equals("1") ? "Alice" : null;
    }
    // Optionalでラップ
    static Optional<String> find(String id) {
        return Optional.ofNullable(legacyFind(id));
    }
    public static void main(String[] args) {
        Optional<String> user1 = find("1"); // Optional[Alice]
        user1 = null; // nullが代入できてしまう
        System.out.println(user1.get()); // 実行時にNullPointerExceptionが発生する
    }
}
```

これを防ぐためには、なるべく変数は再代入不可の`final`で宣言するくらいしかなく、根本的な解決はない。

```java
final Optional<String> user1 = find("1");
```

次に[`Optional.get()`](https://download.java.net/java/early_access/loom/docs/api/java.base/java/util/Optional.html#get())は中身が空でないことの検証をコンパイラが強制しないことだ。

以下のコードを実行すると、実行時エラー`NoSuchElementException`が発生する。

```java
import java.util.Optional;

public class Opt {
    // nullを返すレガシーAPIをそのまま使いたくない!
    static String legacyFind(String id) {
        return id.equals("1") ? "Alice" : null;
    }
    // Optionalでラップ
    static Optional<String> find(String id) {
        return Optional.ofNullable(legacyFind(id));
    }
    public static void main(String[] args) {
        Optional<String> user9 = find("9"); // Optional.empty
        // 値がemptyかどうかのチェックをコンパイラに強制してほしいが強制されない
        System.out.println(user9.get());
    }
}
```

以下のように、`isPresent()`を使えば実行時に値の存在をチェックすることはできるが、チェックはコンパイラによって強制されていないため、プログラマの努力に頼った解決策となってしまう。

```java
import java.util.Optional;

public class Opt {
    // nullを返すレガシーAPIをそのまま使いたくない!
    static String legacyFind(String id) {
        return id.equals("1") ? "Alice" : null;
    }
    // Optionalでラップ
    static Optional<String> find(String id) {
        return Optional.ofNullable(legacyFind(id));
    }

    public static void main(String[] args) {
        Optional<String> user9 = find("9"); // Optional.empty
        // 値の存在をチェックするように改善
        if (user9.isPresent()) {
            System.out.println(user9.get()); // user9は空なので、このブロックは実行されない
        }
    }
}

```

---

## Haskellの場合

### `undefined`

この記事のタイトルにもある`Maybe`について語る前に、Javaの`null`に近い概念として`undefined`を紹介しておく。

`undefined`は、どんな型にもなれる多相型(polymorphic type)の値であり、評価されたときに例外を投げる[^4]。

```haskell
undefined :: HasCallStack => a 
```

`undefined`は、あらゆる型になれるという点で`null`同様、あらゆる場所に混入可能であり、実行時エラーの原因となりうる。

しかし、`null`よりはマシな存在と言える。

まず、Haskellは遅延評価を採用しているため、`undefined`がコード中に存在していても、評価されるまでは例外が上がらない。

また、`undefined`が表す状態は未定義の1つだけであるという点も`null`と異なる点である。

:::message
自分はいきなり、メインの操作を書くのが難しいときに`where`句などを使って部品を作る前にメインの操作を`undefined`にいったんすることでLinterの警告を抑止するのに使うことが多い[^5]
それ以外の場合には、後述する`Maybe`を使っている。
:::

```haskell
solve :: [Int] -> Int
solve xs = undefined
  where
    tmp = "何かしらの途中操作"
```

↓Linterの警告が抑止できる

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3718390/ac12462c-235c-4892-8fa6-0dd67a4a8f4d.png)

↓`undefined`で埋めてないとLinterが警告を出す

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3718390/e56410eb-00ce-4413-8ab5-a4eee55ff0b1.png)

### `Maybe`

登場までかなり長かったが、いよいよ`Maybe`について紹介する。

> The Maybe type encapsulates an optional value. A value of type Maybe a either contains a value of type a (represented as Just a), or it is empty (represented as Nothing). Using Maybe is a good way to deal with errors or exceptional cases without resorting to drastic measures such as error.
>
> The Maybe type is also a monad. It is a simple kind of error monad, where all errors are represented by Nothing. A richer error monad can be built using the Either type.[^6]

`Maybe a`型の値は、値がある場合には`Just a`を、値がない場合には`Nothing`で表される。
これを使うことで、例外的なケースをうまく処理できるようになる。

`Maybe`は`null`や`Optional`と異なり、コンパイラが値がない可能性の検証を強制する[^7][^8]。

[`find`](https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Data-List.html#v:find)は、関数が`True`を返す値が見つかれば`Just a`を、見つからなければ`Nothing`を返す関数である。

以下は値の検証をしない例で、コンパイルエラーになる。

```haskell
import Data.List (find)

main :: IO ()
main = do
  let xs = ['a'..'z']
  let c = find (=='h') xs
  print (c : "askell") -- 検証なし
```

```text
error: [GHC-83865]
    • Couldn't match type ‘Char’ with ‘Maybe Char’
      Expected: [Maybe Char]
        Actual: String
```

値をきちんと検証する例はこちら。

```haskell
import Data.List (find)

main :: IO ()
main = do
  let xs = ['a'..'z']
  let c = find (=='h') xs
  -- print (c : "askell")
  case c of
    Just a -> print (a : "askell")
    Nothing -> print "Nothing"
```

:::message
Rustをご存知の方は、Rustの[Option型](https://doc.rust-jp.rs/rust-by-example-ja/std/option.html)の大先輩がHaskellの`Maybe`と思えば良い。
:::

---

## 実装例: `Maybe`への書き換えで見通しがよくなる例

以下の問題の解答を実装したとき、`Maybe`を使うとロジックがきれいになることに気がついた。

https://atcoder.jp/contests/awc0001/tasks/awc0001_b

### `Maybe`を使わない例

この問題では、条件を満たす生徒がいない場合、-1を出力することになっている。
そのため、`foldl'`の初期値を-1にして、生徒が見つかったら更新を行うという形で実装している。

```haskell
{-# OPTIONS_GHC -Wunused-imports #-}

import Data.List (foldl')

solve :: Int -> Int -> [Int] -> Int
solve l r ps = fst $ foldl' go (-1, l - 1) (zip [1 ..] ps :: [(Int, Int)])
  where
    go :: (Int, Int) -> (Int, Int) -> (Int, Int)
    go acc@(_, p) acc'@(_, p')
      | p' > p && p' <= r = acc' -- 同じ点数の場合は出席番号が若いほうが優先される
      | otherwise = acc

main :: IO ()
main =
  interact $ \inputs ->
    let ls = lines inputs
        [n, l, r] = map read . words $ head ls :: [Int]
        ps = map read . words $ ls !! 1 :: [Int]
     in show (solve l r ps) ++ "\n"

```

### `Maybe`を使う例

先ほどの`Maybe`を使わない実装はシンプルでそれはそれでありだと思うが、コードを読む人の視点に立ったときに、問題文を読まないと`foldl'`の初期値が-1になっているのがなぜか読み取りづらい。

`Maybe`を使うことで、若干記述量は増えるが、-1が例外時の出力であることがコードから読み取れるようになる。

```haskell
{-# OPTIONS_GHC -Wunused-imports #-}

import Data.List (foldl')

-- Maybeを使うことで単純にデータの変換になってうれしい。
solve :: Int -> Int -> [Int] -> Int
solve l r ps =
  case foldl' go Nothing (zip [1 ..] ps :: [(Int, Int)]) of
    Nothing -> -1
    Just (i, _) -> i
  where
    go :: Maybe (Int, Int) -> (Int, Int) -> Maybe (Int, Int)
    go acc (i', p')
      | p' < l || r < p' = acc -- 範囲外は無視
      | otherwise = case acc of
          Nothing -> Just (i', p')
          Just (_, p)
            | p' > p -> Just (i', p') -- 真に大きいときだけ更新 → 同点は最小番号が残る
            | otherwise -> acc

main :: IO ()
main =
  interact $ \inputs ->
    let ls = lines inputs
        [n, l, r] = map read . words $ head ls :: [Int]
        ps = map read . words $ ls !! 1 :: [Int]
     in show (solve l r ps) ++ "\n"

```

---

## `Either`について

自分のHaskellで競技プログラミング以外ほとんどやっていないので、`Either`を使いたい場面に遭遇していないため、割愛。使いだしたらそのうち記事にする予定。

---

## まとめ

- Javaの`null`の弱点を補うために`Optional`が存在する。しかし、`Optional`は値の存在チェックをコンパイラによって強制することはできない。
- Haskellの`Maybe`を使って存在しない値を扱う。`Maybe`はコンパイラで値の存在チェックを強制する。
- `Maybe`を使うことで例外的状態を扱える。

---

## おまけ: `Maybe`を使った実装例を追加で掲載

https://atcoder.jp/contests/abc425/tasks/abc425_b

```haskell
{-# LANGUAGE CPP #-}
{-# OPTIONS_GHC -Wno-x-partial #-}
{-# OPTIONS_GHC -Wunused-imports #-}

import Data.List (find, permutations)

-- (1..N)の全順列から、条件を満たすものを最初に1つ見つける。
-- 条件: 各iについて Ai == -1 または Pi == Ai。
solve :: Int -> [Int] -> [String]
solve n as = case find valid (permutations [1 .. n]) of
  Nothing -> ["No"]
  Just p -> ["Yes", unwords (map show p)]
  where
    valid :: [Int] -> Bool
    valid p =
      and
        [ a == -1 || p_i == a
        | (p_i, a) <- zip p as
        ] -- andで全部Trueの時だけTrueを返す

main :: IO ()
main =
  interact $ \inputs ->
    let ls = lines inputs
        n = read (head ls) :: Int
        as = map read . words $ ls !! 1 :: [Int]
     in unlines (solve n as)

```

---

## Reference

[^1]: <https://docs.oracle.com/javase/specs/jls/se25/html/jls-3.html#jls-3.10.8>
[^2]: <https://docs.oracle.com/javase/specs/jls/se25/html/jls-4.html#jls-4.1>
[^3]: <https://download.java.net/java/early_access/loom/docs/api/java.base/java/util/Optional.html>
[^4]: <https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Prelude.html#v:undefined>
[^5]: <https://wiki.haskell.org/Undefined>
[^6]: <https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Data-Maybe.html>
[^7]: `-Wincomplete-patterns（-Wall）`依存で、デフォルトでは Justだけ書いても通る。
[^8]: [fromJust](https://hackage-content.haskell.org/package/base-4.22.0.0/docs/Data-Maybe.html#v:fromJust)を使うと`Nothing`の場合に実行時例外が発生する。
