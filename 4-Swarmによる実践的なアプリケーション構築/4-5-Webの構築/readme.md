# 4. Swarmによる実践的なアプリケーション構築

## 4-5. Webの構築

イメージ定義の取得
```
$ git clone https://github.com/gihyodocker/todoweb
```

```
$ docker image build -t ch04/todoweb:latest .
$ docker image tag ch04/todoweb:latest localhost:5000/ch04/todoweb:latest

```

$ cp todonginx/etc/nginx/conf.d/public.conf.tmpl todonginx/etc/nginx/conf.d/nuxt.conf.tmpl

locationディレクティブを追加

todonginx/etc/nginx/conf.d/nuxt.conf.tmpl
```
location /_nuxt/ {
    alias /var/www/_nuxt/$1;
    {{ if var "LOG_STDOUT" }}
    access_log  /dev/stdout json;
    error_log   /dev/stderr;
    {{ else }}
    access_log  /var/log/nginx/backend_access.log json;
    error_log   /var/log/nginx/backend_error.log;
    {{ end }}
}
```
```
cp todonginx/Dockerfile todonginx/Dockerfile-nuxt
```
Dockerfile-nuxt
```
```
ENTRYPOINT [ \
  "render", \
      "/etc/nginx/nginx.conf", \
      "--", \
  "render", \
      "/etc/nginx/conf.d/upstream.conf", \
      "--", \
  "render", \
      "/etc/nginx/conf.d/nuxt.conf", \
      "--" \
]
```
$ docker image build -f Dockerfile-nuxt -t ch04/nginx-nuxt:latest .
$ docker image tag ch04/nginx-nuxt:latest localhost:5000/ch04/nginx-nuxt:latest
$ docker image push localhost:5000/ch04/nginx-nuxt:latest
```

stack/todo-frontend.yml
```
version: "3"
services:
  nginx:
    image: registry:5000/ch04/nginx-nuxt:latest 
    deploy:
      replicas: 2
      placement:
        constraints: [node.role != manager]
    depends_on:
      - web
    environment:
      SERVICE_PORTS: 80
      WORKER_PROCESSES: 2
      WORKER_CONNECTIONS: 1024
      KEEPALIVE_TIMEOUT: 65
      GZIP: "on"
      BACKEND_HOST: todo_frontend_web:3000
      BACKEND_MAX_FAILES: 3
      BACKEND_FAIL_TIMEOUT: 10s
      SERVER_PORT: 80
      SERVER_NAME: localhost
      LOG_STDOUT: "true"
    networks:
      - todoapp
    volumes:
      - assets:/var/www/_nuxt

  web:
    image: registry:5000/ch04/todoweb:latest
    deploy:
      replicas: 1
      placement:
        constraints: [node.role != manager]
    environment:
      TODO_API_URL: http://todo_app_nginx:80 #書籍の誤植あり
    networks:
      - todoapp
    volumes:
      - assets:/todoweb/.nuxt/dist

networks:
  todoapp:
    external: true

volumes:
  assets:
    driver: local
```
```
$ docker exec -it manager docker stack deploy -c /stack/todo-frontend.yml todo_frontend
```

### haproxy

stack/todo-ingress.yml
```
version: "3"
services:
  haproxy:
    image: dockercloud/haproxy
    networks:
      - todoapp
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager
    ports:
      - 80:80
      - 1936:1936
networks:
  todoapp:
    external: true
```
```
$ docker exec -it manager docker stack deploy -c /stack/todo-ingress.yml todo_ingress
```

## ブラウザからアクセス

http://localhost:8000/