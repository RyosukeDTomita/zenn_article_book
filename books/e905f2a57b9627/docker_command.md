# Dockerコマンドの疑問に思ったところを調べた

## imageとcontainer
>
> [dockerサブコマンドの導入](https://qiita.com/zembutsu/items/6e1ad18f0d548ce6c266)

- docker container: コンテナ管理のコマンド
- docker image: イメージ管理コマンド

> [!NOTE]
> Dockerにはコンテナとイメージという概念がある。コンテナはサービスとも言い換えられ，docker psやdocker container lsで表示できる。
> コンテナのhostnameはdockerのCONTAINER IDである。

---

## run，build，create
>
> [run，up，build，createの違い](https://prograshi.com/platform/docker/difference-among-docker-compose-run-up-build-and-create/)

|コマンド     |処理内容                          |
|-------------|----------------------------------|
|docker run   |imageとcontainerを作成して起動する|
|docker build |imageの作成                       |
|docker create|containerを作成                   |

### create

- imageがリモートにあって必要ない時などはcreateを使う?
- createはコンテナを作るだけ。起動するにはstartが必用。

### run

- -d: バックグラウンドでコンテナを起動する。例えば，nginxを起動する時に-dをつけるとプロセスがバックグラウンドになるためそのまま次のコマンドを打てる。
- --name: nameを指定することでdocker execする時に指定するのをhashの代わりにcontainerの名前を指定できるようになる。

---

## exec vs start

### exec

- execは起動しているコンテナに対してコマンドを実行させる。
- docker execはコマンドの実行が終わり，停止したコンテナに対しては使えない。 --> 必用なら`tail -f /dev/null等で`コンテナが止まらないようにする。

### start

- startは単に停止しているコンテナを起動するだけ --> runで起動したコンテナを再開したりするのに使う。
- Dockerfile等で起動コマンドが決まっているならstartで起動できる。

## その他

### docker history

- キャッシュを確認できる。

```shell
docker history flask-app:latest
```

### inspect

- 詳細が見れる。

```shell
docker image inspect <イメージ名>
docker container inspect <コンテナ名>
docker volume inspect <ボリューム名>
docker network inspect <ネットワーク名>
```

### バックグラウンドで起動したコンテナの出力を確認する

- container logsを使う。

```shell
docker container logs <コンテナ名>
```

- Docker DesktopのContainersタブからも出力は確認できそうだが，2023/01/22現在ではすべてのコンテナが表示されていない。 --> Docker Desktopを起動してからdocker buildすると出てきた。

### docker cp

- 作成したコンテナからファイルやディレクトリを取得できる。

```shell
docker cp react-app-container:/usr/share/nginx/html build
```

---

## dockerコマンドの主要オプション

### -t，-i

- -tは疑似TTYを割り当てる。具体的にはターミナルのプロンプトが表示されるようになる。
- iは標準入力を開くことでコマンドを打ち込めるようにする。

> docker createの時点(コンテナ作成時点)で-itを指定していないと起動した(startした)コンテナがすぐに落ちてしまう。

```shell
docker exec -i d9c0e79eb672ea0b8019513b54b8e4badd985876b3afe9b68867c275b9798770 "/bin/bash" # コマンドは打てるが結果が綺麗に返ってこない。
ls
bin
boot
dev
etc
home

docker exec -t d9c0e79eb672ea0b8019513b54b8e4badd985876b3afe9b68867c275b9798770 "/bin/bash" # ターミナルのプロンプトは表示される
root@d9c0e79eb672:/# ls # コマンドの結果はdocker内の標準出力に表示されるためコマンドの結果は表示されない

docker exec d9c0e79eb672ea0b8019513b54b8e4badd985876b3afe9b68867c275b9798770 "/bin/bash" # docker内でbashが起動してexecは終了する。
docker exec -ti d9c0e79eb672ea0b8019513b54b8e4badd985876b3afe9b68867c275b9798770 "/bin/bash" # sshで入ったような感覚でコマンドが実行できる。
```

### docker create時の-itオプションの指定について検証

```shell
# オプション指定なしでcreate
docker create fileserver
fdf0cf295bfab35738197af9551996eb55fc62a07346021dc251636a827be3fd

docker start fdf0cf295bfab357
fdf0cf295bfab357

docker exec -it fdf0cf295bfab357 "/bin/bash" # コンテナが起動していないとエラーがでる。
Error response from daemon: Container fdf0cf295bfab35738197af9551996eb55fc62a07346021dc251636a827be3fd is not running

# -tのみ指定
docker create -t fileserver
4fb51e63044964443b47011ae85af89ed9832aafc5cde801e0c3dda26c7fc5cb

docker start 4fb51e63044964
4fb51e63044964

docker exec -it 4fb51e63044964 "/bin/bash" # 普通にコマンド打てる
root@4fb51e630449:/# ls

# -iのみで起動
docker create -i fileserver
4389ce1be7dfcc76dff1019ba0fc80d947bb0b2735b7641a70455553863358ed

docker start 4389ce1be7dfcc
4389ce1be7dfcc

docker exec -it 4389ce1be7dfcc "/bin/bash" # 普通にコマンド打てる
root@4389ce1be7df:/# ls
```
