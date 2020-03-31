# 3. 実用的なコンテナの構築とデプロイ

## 3-5. コンテナ配置戦略

### Docker Swarm

複数のホスト上でコンテナをオーケストレーションする。ホストを跨いだコンテナ間の通信やコンテナの配置を任せる
学習では"複数のホスト"を用意するのは困難なので、Docker in Doker(ホスト)として学習する

Docker in Doker(dind)のホストとなるDockerコンテナは以下の通り。
- worker01〜03: アプリケーションコンテナが動作するホスト。3台用意する
- manager: Swarmクラスタを制御するホスト。workerへ適切にコンテナを配備する
- registry: Swarmとは関係ないがdindの場合、worker, manage内から取得するDockerイメージのリポジトリとなるコンテナが必要。DockerHubなどで代替可能。

各コンテナの定義
docker-compose.yml

- ```privileged: true```: 特権モードでの実行。コンテナ内でsystemctlが実行できる。dindで必要。
- ```expose```: 公開ポートの指定
- ```command: "--insecure-registry registry:5000"``` docker pullは通常、httpsで行う為、httpでもpull可能にする

```
version: "3"
services: 
  registry:
    container_name: registry
    image: registry:2.6
    ports: 
      - 5000:5000
    volumes: 
      - "./registry-data:/var/lib/registry"
  manager:
    container_name: manager
    image: docker:18.05.0-ce-dind
    privileged: true
    tty: true
    ports: 
      - 8000:80
      - 9000:9000
    depends_on: 
      - registry
    expose: 
      - 3375
    command: "--insecure-registry registry:5000"
    volumes: 
      - "./stack:/stack"
  worker01: 
    container_name: worker01
    image: docker:18.05.0-ce-dind
    privileged: true
    tty: true
    depends_on: 
      - registry
      - manager
    expose: 
      - 7946
      - 7946/udp
      - 4789/udp
    command: "--insecure-registry registry:5000"
  worker02: 
    container_name: worker02
    image: docker:18.05.0-ce-dind
    privileged: true
    tty: true
    depends_on: 
      - registry
      - manager
    expose: 
      - 7946
      - 7946/udp
      - 4789/udp
    command: "--insecure-registry registry:5000"
  worker03: 
    container_name: worker03
    image: docker:18.05.0-ce-dind
    privileged: true
    tty: true
    depends_on: 
      - registry
      - manager
    expose: 
      - 7946
      - 7946/udp
      - 4789/udp
    command: "--insecure-registry registry:5000"
```
```
$ docker-compose up -d
CONTAINER ID        IMAGE                    COMMAND                  CREATED              STATUS              PORTS                                                              NAMES
a4864a38d48e        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   7 seconds ago        Up 5 seconds        2375/tcp, 4789/udp, 7946/tcp, 7946/udp                             worker02
a36ba6ea1463        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   7 seconds ago        Up 5 seconds        2375/tcp, 4789/udp, 7946/tcp, 7946/udp                             worker03
83ec47899a9d        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   7 seconds ago        Up 5 seconds        2375/tcp, 4789/udp, 7946/tcp, 7946/udp                             worker01
b170282d6fb6        docker:18.05.0-ce-dind   "dockerd-entrypoint.…"   About a minute ago   Up 6 seconds        2375/tcp, 3375/tcp, 0.0.0.0:9000->9000/tcp, 0.0.0.0:8000->80/tcp   manager
92d3304fe792        registry:2.6             "/entrypoint.sh /etc…"   About a minute ago   Up About a minute   0.0.0.0:5000->5000/tcp                                             registry
```

### Swarm managerの設定

```
$ docker swarm init
```
```
$ docker exec -it manager docker swarm init
Swarm initialized: current node (w1miys4j65ljo4zuasmvsww89) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-4nugaxoeqcjr688slsfgioyiducuzxmscnq8d84opl8o62dw9b-7w8ql9lge6vq1g3qiq2j4rtst 192.168.80.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
workerをクラスタと登録する際、ここで表示されたtokenが必要となる

### Swarmクラスタにworkerホストを登録する
```
$ docker swarm join --token [token] [managerHost:port]
```

worker01〜03までをjoinする
```
$ docker exec -it worker01 docker swarm join --token SWMTKN-1-4nugaxoeqcjr688slsfgioyiducuzxmscnq8d84opl8o62dw9b-7w8ql9lge6vq1g3qiq2j4rtst manager:2377

$ docker exec -it worker03 docker swarm join --token SWMTKN-1-4nugaxoeqcjr688slsfgioyiducuzxmscnq8d84opl8o62dw9b-7w8ql9lge6vq1g3qiq2j4rtst manager:2377

$ docker exec -it worker03 docker swarm join --token SWMTKN-1-4nugaxoeqcjr688slsfgioyiducuzxmscnq8d84opl8o62dw9b-7w8ql9lge6vq1g3qiq2j4rtst manager:2377
```

### Swarmクラスタの確認

```
$ docker node ls
```
```
$ docker exec -it manager docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
ne19o91r9yi6qpop99vigkses     83ec47899a9d        Ready               Active                                  18.05.0-ce
glcnp0vfm4yxg8z0i183o2v59     a36ba6ea1463        Ready               Active                                  18.05.0-ce
9svfzg1ciag63izuxzm3pjupr     a4864a38d48e        Ready               Active                                  18.05.0-ce
w1miys4j65ljo4zuasmvsww89 *   b170282d6fb6        Ready               Active              Leader              18.05.0-ce
```

### registryにDockerイメージをプッシュする

プライベートリポジトリを利用する際は予めpush先のレジストリURL(ここではregistry=localhost:5000)をタグ化しておく
```
$ docker image tag example/echo:latest localhost:5000/example/echo:latest
```
```
$ docker image push localhost:5000/example/echo:latest
```

### worker上でDockerイメージをプルする

```
$ docker exec -it worker01 docker image pull registry:5000/example/echo:latest
$ docker exec -it worker02 docker image pull registry:5000/example/echo:latest
$ docker exec -it worker03 docker image pull registry:5000/example/echo:latest
```

### Swarm Service

1つのイメージからなる単位。Swarmクラスタ内でそのイメージをどのように扱うかを定義。コンテナの起動まで行う。
Swarm manager内で定義する
- ```--publish [ホストPORT]:[コンテナPORT]``` port forwarding.
```
$ docker exec -it manager docker service create --replicas 1 --publish 8000:8000 --name echo registry:5000/example/echo:latest 
```
```
$ docker exec -it manager docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
ia76w2y3l2my        echo                replicated          1/1                 registry:5000/example/echo:latest   *:8000->8000/tcp
```
起動サービスの確認
```
$ docker exec -it manager docker service ps echo
ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
v1otjzn2x11b        echo.1              registry:5000/example/echo:latest   83ec47899a9d        Running             Running 5 minutes ago
```
サービスをスケールする
```
$ docker service scale [サービス名]=[コンテナ数]
```
```
$ docker exec -it manager docker service scale echo=6
```
serviceが異なるノード上で起動されたことが確認できる
```
$ docker exec -it manager docker service ps echo
ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
v1otjzn2x11b        echo.1              registry:5000/example/echo:latest   83ec47899a9d        Running             Running 8 minutes ago                            
ixs5akf888x9        echo.2              registry:5000/example/echo:latest   a4864a38d48e        Running             Running about a minute ago                       
phnt4b69cw9i        echo.3              registry:5000/example/echo:latest   a4864a38d48e        Running             Running about a minute ago                       
p6vbuioa9le5        echo.4              registry:5000/example/echo:latest   83ec47899a9d        Running             Running about a minute ago                       
5nb7jpq684n1        echo.5              registry:5000/example/echo:latest   b170282d6fb6        Running             Running 57 seconds ago                           
t6z8jra8f1g1        echo.6              registry:5000/example/echo:latest   a36ba6ea1463        Running             Running about a minute ago        
```
デプロイしたサービスの削除
```
$ docker service rm [サービス名]
```
```
$ docker exec -it manager docker service rm echo
```
### Stack

複数のserviceをグルーピングした単位。Swarmのdocker-compose版みたいなもの。
ネットワークは1stack内でのネットワークとなる為、複数のstack間でコンテナ間接続が必要になる場合は共通のoverlayネットワークを割り当てる必要がある

#### networkの作成

```
$ docker network create --driver=overlay --attachable [チャネル名]
```
```
$ docker exec -it manager docker network create --driver=overlay --attachable ch03
```

#### stack定義

NginxがリバースプロキシとなりechoコンテナのAPIを実行する。という定義とする。

- ```placement```: デプロイ先を定義
  - ```constraints: [node.role != manager]```: managerノード以外にデプロイ
- ```BACKEND_HOST```はgihyodocker/nginx-proxy固有の環境変数
- ```echo_api```はstackデプロイ時に指定するstack名がprefix_service名となる
- ```external: true```: 既存ネットワークの接続を許可

stack/ch03-webapi.yml
```
version: "3"
services:
  nginx:
    image: gihyodocker/nginx-proxy:latest
    deploy:
      replicas: 3
      placement:
        constraints: [node.role != manager]
    environment:
      BACKEND_HOST: echo_api:8080
    depends_on:
      - api
    networks:
      - ch03
  api:
    image: registry:5000/example/echo:latest
    deploy:
      replicas: 3
      placement:
        constraints: [node.role != manager]
    networks:
      - ch03
networks:
  ch03:
    external: true
```

#### stackのデプロイ

```
$ docker stack deploy -c [スタックファイル] [スタック名]
```
```
$ docker exec -it manage docker stack deploy -c /stack/ch03-webapi.yml echo
```
デプロイされたstackの確認
```
$ docker exec -it manager docker stack ls
NAME                SERVICES
echo                2
```
stackによってデプロイされたコンテナの一覧
```
$ docker stack ps [stack名]
```
```
$ docker exec -it manager docker stack ps echo
ID                  NAME                IMAGE                               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
jfntuy35lilb        echo_api.1          registry:5000/example/echo:latest   83ec47899a9d        Running             Running 4 minutes ago                       
o4i22zcujzuh        echo_nginx.1        gihyodocker/nginx-proxy:latest      a4864a38d48e        Running             Running 4 minutes ago                       
kdyew0algbyi        echo_api.2          registry:5000/example/echo:latest   a4864a38d48e        Running             Running 4 minutes ago                       
oysyecvpve8r        echo_nginx.2        gihyodocker/nginx-proxy:latest      a36ba6ea1463        Running             Running 4 minutes ago                       
51nxd0dgrvzu        echo_api.3          registry:5000/example/echo:latest   a36ba6ea1463        Running             Running 4 minutes ago                       
qtx8kd4kkgmy        echo_nginx.3        gihyodocker/nginx-proxy:latest      83ec47899a9d        Running             Running 4 minutes ago    
```
stackによってデプロイされたサービスの一覧
```
$ docker stack services [stack名]
```
```
$ docker exec -it manager docker stack services echo
ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
ed5ntt3siic8        echo_api            replicated          3/3                 registry:5000/example/echo:latest   
zsl1kimled94        echo_nginx          replicated          3/3                 gihyodocker/nginx-proxy:latest    
```

#### visualizerでデプロイされたコンテナを可視化

- ```mode: global```: レプリカ数を指定せず全てのノードに1つづコンテナを配置
  - 後に続く```constraints```でmanagerのみに配置としている

stack/ch03-webapi.yml
```
version: "3"
services:
  app:
    image: dockersamples/visualizer
    ports:
      - "9000:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      mode: global
      placement:
        constraints: [node.role == manager]
```
visualizerのデプロイ
```
$ docker exec -it manager docker stack deploy -c /stack/visualizer.yml visualizer
```
managerにコンテナがdeployされていることがわかる
```
$ docker exec -it manager docker stack ps visualizer
ID                  NAME                                       IMAGE                             NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
02o6jm6qg7oi        visualizer_app.w1miys4j65ljo4zuasmvsww89   dockersamples/visualizer:latest   b170282d6fb6        Running             Starting 3 seconds ago 
```

managerコンテナは、ホストに9000:9000のポートフォワードを行っているので、ホストからlocalhost:9000で最終的にvisualizer_app:8080に接続される

### ServicesをSwarmクラスタ外から利用する

echoサービスに対してSwarmクラスタ外から利用可能にする

manageにHAProxyを立てる

stack/ch03-ingress.yml
```
version: "3"
services:
  haproxy:
    image: dockercloud/haproxy
    networks:
      - ch03
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      mode: global
      placement:
        constraints: [node.role == manager]
    ports:
      - 80:80
      - 1936:1936
networks:
  ch03:
    external: true
```
```
$ docker exec -it manager docker stack deploy -c /stack/ch03-ingress.yml ingress
```

nginx_echoサービスにSERVICE_PORTを指定。

manager上のhaproxyは同一ネットワーク上のSERVICE_PORTの定義があるすべてのサービスに対して負荷分散を行う。

stack/ch03-webapi.yml
```
version: "3"
services:
  nginx:
    image: gihyodocker/nginx-proxy:latest
    deploy:
      replicas: 3
      placement:
        constraints: [node.role != manager]
    environment:
      SERVICE_PORT: 80
      BACKEND_HOST: echo_api:8080
    depends_on:
      - api
    networks:
      - ch03
  api:
    image: registry:5000/example/echo:latest
    deploy:
      replicas: 3
      placement:
        constraints: [node.role != manager]
    networks:
      - ch03
networks:
  ch03:
    external: true
```
```
$ docker exec -it manager docker stack rm echo 
$ docker exec -it manager docker stack deploy -c /stack/ch03-webapi.yml echo
```

