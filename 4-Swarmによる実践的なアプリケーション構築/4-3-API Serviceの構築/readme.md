# 4. Swarmによる実践的なアプリケーション構築

## 4-3. API Serviceの構築

アプリケーションのダウンロード
```
$ git clone https://github.com/gihyodocker/todoapi
```

### API Serviceのコンテナ定義

go buildした成果物をCMDで実行する
```
RUN cd /go/src/github.com/gihyodocker/todoapi && go build -o bin/todoapi cmd/main.go
RUN cd /go/src/github.com/gihyodocker/todoapi && cp bin/todoapi /usr/local/bin/
CMD ["todoapi"]
```

### imageのbuild & push

```
$ docker image build -t ch04/todoapi:latest .
$ docker image tag ch04/todoapi:latest localhost:5000/ch04/todoapi:latest
$ docker image push localhost:5000/ch04/todoapi:latest
```

### swarm stackの作成

```
version: "3"
services: 
  api:
    image: registry:5000/ch04/todoapi:latest
    deploy:
      replicas: 2
    environment:
      TODO_BIND: ":8080"
      TODO_MASTER_URL: "gihyo:gihyo@tcp(todo_mysql_master:3306)/tododb?parseTime=true"
      TODO_SLAVE_URL:  "gihyo:gihyo@tcp(todo_mysql_slave:3306)/tododb?parseTime=true"
    networks:
      - todoapp
networks:
  todoapp:
    external: true
```
```
$ docker exec -it manager docker stack deploy -c /stack/todo-app.yml todo_app
```