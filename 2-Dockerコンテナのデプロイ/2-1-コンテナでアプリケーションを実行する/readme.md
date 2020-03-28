# 2 Dockerコンテナのデプロイ

## 2.1 コンテナでアプリケーションを実行する

イメージの取得
```
$ docker image pull gihyodocker/echo:latest
```

取得したイメージからコンテナを実行する
```
$ docker container run -t -p 9000:8080 gihyodocker/echo:latest
```

Dockerfileからimageの作成 ```-t``` でイメージ名を指定

```
docker image build -t イメージ名[:タグ名] Dockerfileのパス
```
```
$ docker image build -t example/echo:latest .
```

### CMDとENTRYPOINT

Dockerfileに記述するCMDは```$ docker container run```実行時に上書きすることができる。この場合、echo yayがCMDへ渡す引数となる。
```
$ docker container run イメージ名 echo yay
```

ENTRYPOINTがDockerfileに記述された場合、CMDはENTRYPOINTに指定されたコマンドへ渡す引数となる。この場合、```$ docker container run```実行時にtailコマンドへ渡す引数が変更可能となる。
```
ENTRYPOINT ["tail"]
CMD ["-f", "access.log"]
```

### コンテナ実行時のオプション

- -t コンテナ出力をホストターミナルへ転送
- -p ポートフォワード ```-p portFromHost:portFromDocker``` HOSTポートを指定しない場合は空いているポートをバインディングする
- -d バックグラウンドで実行