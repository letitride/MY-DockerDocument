# 2 Dockerコンテナのデプロイ

## 2.3 Dockerコンテナの操作

### コンテナの作成と実行

docker container run は新しいコンテナを作成する。
```
$ docker container run [options] イメージ名[:タグ] [コマンド] [コマンド引数]
```
停止したコンテナを再度実行するにはdocker restart または startを実行する
```
$ docker restart コンテナ名
```

#### 作成->起動時にコンテナのシェルに入る

コマンド引数を渡さないとCMDコマンドが優先されシェルに入れないケースがある
```
$  docker container run -it イメージ名[:タグ] bash
```
#### 作成時にコンテナ名を指定する
```
$ docker container run --name コンテナ名 イメージ名[:タグ]
```

### コンテナの停止

```
$ docker stop コンテナ名
```

### コンテナの破棄

```
$ docker rm コンテナ名
```

### 実行中コンテナの標準出力をキャプチャする

```
$ docker container logs -f コンテナ名
```

### 実行中コンテナ内でコマンドを実行する

```
$ docker exec [option] コンテナ名 実行するコマンド
```

よく使うのは実行中コンテナのシェルに入る操作
```
$ docker exec -it container_name bash
```

### 実行中コンテナ間やホスト間でファイルをコピーする

```
$ docker container cp [コピー元:]file [コピー先:]file
```
コンテナ間でコピー
```
$ docker container cp from_container:/readme.md to_container:/
```
コンテナからホストへコピー
```
$ docker container cp from_container:/readme.md /path/to/
```