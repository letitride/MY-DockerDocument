# 2 Dockerコンテナのデプロイ

## 2.5 Docker Composeでマルチコンテナを実行する

### コンテナ起動時のプロビジョニング定義

docker-compose.ymlに定義する
```
version: "3"
services:
  echo: 
    image: example/echo:latest
    ports:
      - 9000:8080
```
- ```echo``` 配置したディレクトリ名 + 指定した値がコンテナ名となる


### コンテナの作成・実行

docker-compose.ymlを配置したディレクトリ内で実行。2度実行しても新たなコンテナは作成されない
```
$ docker-compose up -d
```

### コンテナの停止・削除

コンテナ削除まで行われる。
```
$ docker-compose down
```

停止のみの場合は
```
$ docker-compose stop
```

再起動は
```
$ docker-compose start
```

### イメージをDockerfileからビルドしてコンテナを実行する

docker-compose.yml
```
version: "3"
services:
  echo: 
    build: .
    ports:
      - 9000:8080
```
- ```build```にDockerfileの配置先を記述。この場合```.```カレントに配置

ビルドのみ
```
$ docker-compose buid
```
常にビルドして実行する
```
$ docker-compose up -d --build
```