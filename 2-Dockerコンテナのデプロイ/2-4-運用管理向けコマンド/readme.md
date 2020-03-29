# 2 Dockerコンテナのデプロイ

## 2.4 運用管理向けコマンド

### prune 破棄

#### 停止したコンテナの一括削除

```
$ docker container prune
```

#### コンテナ化されていないイメージの一括削除

```
$ docker image prune
```

#### 未使用Dockerリソースの一括削除

```
$ docker system prune
```

#### コンテナの利用状況の取得
```
$ docker container stats
```