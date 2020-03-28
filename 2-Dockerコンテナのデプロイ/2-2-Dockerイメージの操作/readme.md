# 2 Dockerコンテナのデプロイ

## 2.2 Dockerイメージの操作

### イメージビルド時のオプション

- -f Dockerfile名を指定
```
$ docker image build -f Dockerfile-custom -t custom:latest .
```
- --pull=true Dockerfileで指定したFROMイメージを常に取得してビルドする。latestタグなどはホストイメージ内に存在すれば取得しないがDockerHub側で更新されている場合など

### イメージのタグ付け

イメージに別のイメージ名、タグを設定する。ハッシュ値は同じイメージとなる。
```
$ docker image tag 元イメージ名:タグ 新イメージ名:タグ
```
```
$ docker image tag example/echo:latest newimage/echo:1.0
REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
example/echo              latest              218af0318a4f        33 minutes ago      803MB
newimage/echo             1.0                 218af0318a4f        33 minutes ago      803MB
```

### イメージのPush

イメージ名はDocker Hubの ```user名/****```とする。

```
$ docker image push イメージ名[:タグ]
```