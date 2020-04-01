# 4. Swarmによる実践的なアプリケーション構築

## 4-4. Nginxの構築

イメージ定義の取得
```
$ git clone https://github.com/gihyodocker/todonginx
```

### バックエンドサーバへの振り分け

todonginx/etc/nginx/conf.d/upstream.conf.tmpl

ファイルはentrykitのテンプレートファイル。環境変数がbindできる
- ```server```: 宛先
```
upstream backend {
    server {{ var "BACKEND_HOST" }} max_fails={{ var "BACKEND_MAX_FAILS" | default "3" }} fail_timeout={{ var "BACKEND_FAIL_TIMEOUT" | default "10s" }};
}
```

### ルーティング

todonginx/etc/nginx/conf.d/public.conf.tmpl
- ```proxy_pass```: 転送先。upstreamを定義しているのでupstreamディレクティブbackendを指定
```
server {
    listen {{ var "SERVER_PORT" | default "80" }} default_server;
    server_name {{ var "SERVER_NAME" | default "localhost" }};
    charset utf-8;

    location / {
        proxy_pass http://backend;
        proxy_pass_request_headers on;
        proxy_set_header host $host;
        {{ if var "LOG_STDOUT" }}
        access_log  /dev/stdout json;
        error_log   /dev/stderr;
        {{ else }}
        access_log  /var/log/nginx/backend_access.log json;
        error_log   /var/log/nginx/backend_error.log;
        {{ end }}
        {{ if var "BASIC_AUTH_FILE" }}
        auth_basic "Restricted";
        auth_basic_user_file {{ var "BASIC_AUTH_FILE" }};
        {{ end }}
    }
}
```

### image のbuild & push

```
$ docker image build -t ch04/nginx:latest .
$ docker image tag ch04/nginx:latest localhost:5000/ch04/nginx:latest
$ docker image push localhost:5000/ch04/nginx
```

### apiのstackに参加

```
version: "3"
services:
  nginx:
    image: registry:5000/ch04/nginx:latest
    deploy:
      replicas: 2
      placement:
        constraints: [node.role != manager]
    depends_on:
      - api
    environment:
      WORKER_PROCESSES: 2
      WORKER_CONNECTIONS: 1024
      KEEPALIVE_TIMEOUT: 65
      GZIP: "on"
      BACKEND_HOST: api:8080
      BACKEND_MAX_FAILS: 3
      BACKEND_FAIL_TIMEOUT: 10s
      SERVER_PORT: 80
      SERVER_NAME: todo_app_nginx
      LOG_STDOUT: "true"
    networks:
      - todoapp
  api:
    image: registry:5000/ch04/todoapi:latest
    deploy:
      replicas: 2
      placement:
        constraints: [node.role != manager]
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
コンテナ起動時にapiの名前解決が出来ない場合があるが、時間がかかるケースがある為、しばらく待ってみてserviceがnunningしていればよい。