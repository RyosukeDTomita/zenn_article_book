# コンテナセキュリティ

- コンテナブレイクアウトやコンテナエスケープを防ぐために`root`でコンテナを起動しないことが大切。
- COPY命令で持ち込んだファイルにはクレデンシャルを含めない。途中で削除してもイメージに残ってしまう。
- distroless(Google提供のシェルすら含まれていない軽いイメージ)を使うことを視野に入れる。

> distrolessでもdebugと書いてあるやつはshellがあるので便利。

- ビルドし直す時には脆弱性のあるキャッシュが使われないように--no-cacheを使う。
- 信頼できるイメージ(Docker Officlial Image，Verified Publisher(Dockerによって検証)，Sponsord OSS)

## docker scout

- 指定したイメージやアーカイブをスキャンして脆弱性の有無を調べる。

```shell
docker scout quickview react-app:latest
docker scout cves react-app:latest
```

---

## イメージからクレデンシャルを抜き出す

- イメージから環境変数を取得

```shell
docker build -t myimage:latest -f- . <<EOF
FROM busybox
ENV AWS_ACCESS_KEY_ID=hogehoge
COPY secrets.txt /etc/secrets.txt
RUN rm /etc/secrets.txt
EOF

docker history myimage:latest | grep AWS_ACCESS_KEY_ID
<missing>      35 seconds ago   ENV AWS_ACCESS_KEY_ID=hogehoge                  0B        buildkit.dockerfile.v0
```

- イメージから途中で消したファイルを取得

```shell
echo "secret" > secrets.txt

docker build -t myimage:latest -f- . <<EOF
FROM busybox
ENV AWS_ACCESS_KEY_ID=hogehoge
COPY secrets.txt /etc/secrets.txt
RUN rm /etc/secrets.txt
EOF

docker save mayimage -o myimage.tar # イメージを保存

tar -tf myimage.tar | fgrep layer.tar # layerを抽出
5044846d4c5c84cb56580205b7e20cf4564d7b18d3e17a8f5cc3289d3e3b5b7b/layer.tar
5a3228fc3aa3e38eddcfe971c5abe08ec64ccd1414a33c38d6977bdbe88b9c51/layer.tar
92d557fbcdb88593a1f056f462d71a22750c1c24276052f0545ce4470dc516e3/layer.tar

tar xfO myimage.tar 5044846d4c5c84cb56580205b7e20cf4564d7b18d3e17a8f5cc3289d3e3b5b7b/layer.tar
etc/0000755000000000000000000000000014536044433010025 5ustar0000000000000000etc/secrets.txt0000664000000000000000000000001014536043242012224 0ustar0000000000000000secrets /etc/secrets.txtがありそうなのがわかる。

tar xfO myimage.tar 5044846d4c5c84cb56580205b7e20cf4564d7b18d3e17a8f5cc3289d3e3b5b7b/layer.tar | tar xfO - etc/secrets.txt # ファイルの中身を確認
secrets
```

```shell
# ファイル名がわかっていればワンライナーでいけそう
for layer in $(tar -tf myimage.tar | fgrep layer.tar); do tar xfO myimage.tar $layer;done | tar xfO - etc/secrets.txt
secrets
```
