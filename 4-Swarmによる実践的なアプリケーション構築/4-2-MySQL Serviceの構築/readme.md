# 4. Swarmによる実践的なアプリケーション構築

## 4-2. MySQL Serviceの構築

MySQLイメージ構築用リポジトリ
```
$ git clone https://github.com/gihyodocker/tododb
```

### etc/mysql/mysql.conf.d/mysqld.cnf

log_binのパスはmaster / slave 全サービス共通

server_idはコンテナ起動時にENTRYPOINTで指定したadd-server-id.shでサービス間で重複しない値をファイル末尾に追記する。
```
echo "server-id=$MYSQL_SERVER_ID" >> /etc/mysql/mysql.conf.d/mysqld.cnf
```

### レプリケーションを設定する

prepare.shにて定義

環境変数$MYSQL_MASTERが定義されている場合はマスターとみなし、処理をぬける

masterへの疎通確認
```
mysql -h $MYSQL_MASTER_HOST ...
```

masterに対してレプリケーションユーザと権限の作成

- slave サービスから実行
- IF NOT EXISTSで未作成時のみ実行
- $SOURCE_IPはslaveのネットワークアドレスを指定できる
```
mysql -h $MYSQL_MASTER_HOST -u root -p$MYSQL_ROOT_PASSWORD -e "CREATE USER IF NOT EXISTS '$MYSQL_REPL_USER'@'$SOURCE_IP' IDENTIFIED BY '$MYSQL_REPL_PASSWORD';"
mysql -h $MYSQL_MASTER_HOST -u root -p$MYSQL_ROOT_PASSWORD -e "GRANT REPLICATION SLAVE ON *.* TO '$MYSQL_REPL_USER'@'$SOURCE_IP';"
```

masterのbinlogポジションを取得
```
SHOW MASTER STATUS
```
slaveにbinlogポジションを指定
```
CHANGE MASTER TO ... MASTER_LOG_FILE='...', MASTER_LOG_POS=...
```

### ビルドとSwarmクラスタとしての利用

```
$ docker image build -t ch04/tododb:latest .
$ docker image tag ch04/tododb:latest localhost:5000/ch04/tododb:latest
$ docker image push localhost:5000/ch04/tododb:latest
```

### Swarm上でMaster/Slaveサービスを実行する

stack/todo-mysql.yml
```
version: "3"
services:
  master: 
    image: registry:5000/ch04/tododb:latest
    deploy:
      replicas: 1
      placement:
        constraints: [node.role != manager]
    environment:
      MYSQL_ROOT_PASSWORD: gihyo
      MYSQL_DATABASE: tododb
      MYSQL_USER: gihyo
      MYSQL_PASSWORD: gihyo
      MYSQL_MASTER: "true"
    networks:
      - todoapp
  slave:
    image: registry:5000/ch04/tododb:latest
    deploy:
      replicas: 2
      placement:
        constraints: [node.role != manager]
    depends_on:
      - master
    environment:
      MYSQL_MASTER_HOST: master
      MYSQL_ROOT_PASSWORD: gihyo
      MYSQL_DATABASE: tododb
      MYSQL_USER: gihyo
      MYSQL_PASSWORD: gihyo
      MYSQL_REPL_USER: repl
      MYSQL_REPL_PASSWORD: gihyo
    networks:
      - todoapp
networks:
  todoapp:
    external: true
```
```
$ docker exec -it manager docker stack deploy -c /stack/todo-mysql.yml todo_mysql
```

### MySQLコンテナを確認し、初期データを投入する

```--no-trunc```をつけてIDを切り捨て表示しないこと
```
$ docker exec -it manager docker service ps todo_mysql_master --no-trunc
ID                          NAME                  IMAGE                                                                                                      NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
rus2r8c3qpye56043mk1yi3g6   todo_mysql_master.1   registry:5000/ch04/tododb:latest@sha256:d0ec4cb6aa248befa33c6eab95d794cd434fd5c41b61d43e24b95a85b642dda6   a3a91ec86870        Running             Running 2 seconds ago
```
master dbへのシェルログイン
```
$ docker exec -it [NODE] docker exec -it [NAME].[ID] bash
```
```
$ docker exec -it a3a91ec86870 docker exec -it todo_mysql_master.1.rus2r8c3qpye56043mk1yi3g6 bash
```

データの投入と確認
```
$ docker exec -it a3a91ec86870 docker exec -it todo_mysql_master.1.rus2r8c3qpye56043mk1yi3g6 init-data.sh
```
```
$ docker exec -it a3a91ec86870 docker exec -it todo_mysql_master.1.rus2r8c3qpye56043mk1yi3g6 mysql -u gihyo -pgihyo tododb
mysql> select * from todo limit 1\G
*************************** 1. row ***************************
     id: 1
  title: MySQLのDockerイメージを作成する
content: MySQLのMaster、Slaveそれぞれで利用できるように、環境変数で役割を制御できるMySQLイメージを作成する
 status: DONE
created: 2020-03-31 14:51:52
updated: 2020-03-31 14:51:52
1 row in set (0.03 sec)
```

### レプリケーションの確認

```
$ docker exec -it manager docker service ps todo_mysql_slave --no-trunc
ID                          NAME                 IMAGE                                                                                                      NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
to2lzxtcq268q3j1ivw1mgh7v   todo_mysql_slave.1   registry:5000/ch04/tododb:latest@sha256:d0ec4cb6aa248befa33c6eab95d794cd434fd5c41b61d43e24b95a85b642dda6   f5f2f386dd80        Running             Running 7 minutes ago                       
0835ngntifq9zyqi7r09eo2pv   todo_mysql_slave.2   registry:5000/ch04/tododb:latest@sha256:d0ec4cb6aa248befa33c6eab95d794cd434fd5c41b61d43e24b95a85b642dda6   5188c673da69        Running             Running 7 minutes ago

$ docker exec -it f5f2f386dd80 docker exec -it todo_mysql_slave.1.to2lzxtcq268q3j1ivw1mgh7v mysql -u gihyo -pgihyo tododb
mysql> select * from todo limit 1\G;
*************************** 1. row ***************************
     id: 1
  title: MySQLのDockerイメージを作成する
content: MySQLのMaster、Slaveそれぞれで利用できるように、環境変数で役割を制御できるMySQLイメージを作成する
 status: DONE
created: 2020-03-31 14:51:52
updated: 2020-03-31 14:51:52
1 row in set (0.00 sec)
```
```
$ docker exec -it 5188c673da69 docker exec -it todo_mysql_slave.2.0835ngntifq9zyqi7r09eo2pv mysql -u gihyo -pgihyo tododb
mysql> select * from todo limit 1\G;
*************************** 1. row ***************************
     id: 1
  title: MySQLのDockerイメージを作成する
content: MySQLのMaster、Slaveそれぞれで利用できるように、環境変数で役割を制御できるMySQLイメージを作成する
 status: DONE
created: 2020-03-31 14:51:52
updated: 2020-03-31 14:51:52
1 row in set (0.00 sec)
```

