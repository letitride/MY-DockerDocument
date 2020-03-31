# 4. Swarmによる実践的なアプリケーション構築

## Swarm Nodeのセットアップ

前章で定義したものを使用する。

```
$ docker-compose up -d
$ docker exec -it manager docker swarm init
$ docker exec -it worker01 docker swarm join --token SWMTKN-1-111akawk50svbnpsevfpolwin5p5ny6jd63vxyrpx8dyzig8il-97m8nwz866nsbf21w4579cjf5 manager:2377
$ docker exec -it worker02 docker swarm join --token SWMTKN-1-111akawk50svbnpsevfpolwin5p5ny6jd63vxyrpx8dyzig8il-97m8nwz866nsbf21w4579cjf5 manager:2377
$ docker exec -it worker03 docker swarm join --token SWMTKN-1-111akawk50svbnpsevfpolwin5p5ny6jd63vxyrpx8dyzig8il-97m8nwz866nsbf21w4579cjf5 manager:2377
```

networkの作成
```
$ docker exec -it manager docker network create --driver=overlay --attachable todoapp
```