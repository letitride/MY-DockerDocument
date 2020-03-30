# 3. 実用的なコンテナの構築とデプロイ

## 永続化データをどう扱うか

### Data Volume

ホスト側のディスクをDocker側へマウントする
- ホスト側に指定のディレクトリが存在しない場合は作成される
- ホスト側ディレクトリの指定は絶対パスで指定
- ビルドしたコンテナの成果物を共有するには、コンテナ起動後、コンテナ側でマウントポイントにファイルを配置する
  - 例えばCMDやENTRYPOINTでコンテナ起動時にファイルを配置する
```
$ docker container run -v [ホスト側ディレクトリ]:[コンテナ側ディレクトリ] リポジトリ名 cp -r /app コンテナ側ディレクトリ
```

### Data Volumeコンテナ

ステートを持つコンテナに対して、共有可能なディレクトリを別のDockerコンテナが提供する方法。
ステートを持つコンテナ -> Volumeコンテナ -> ホスト とVolumeコンテナが間に入る

#### MySQLのデータをData Volumeコンテナに保持する

Volumeコンテナのイメージ
```
FROM busybox

VOLUME /var/lib/mysql

CMD ["bin/true"]
```
```bash
$ docker image build -t example/mysql-data:latest .
```
コンテナを作成しておく
```
$ docker container run -d --name mysql-data example/mysql-data:latest
```

MySQLコンテナの起動
- ```--rm``` はコンテナ停止時に自動でコンテナを削除
- ```volumes-from```でVoluemコンテナを指定。Volumeコンテナ側の```VOLUME /var/lib/mysql```が共有される
```
$ docker container run -d --rm --name mysql \
  -e "MYSQL_ALLOW_EMPTY_PASSWORD=yes" \
  -e "MYSQL_DATABASE=volume_test" \
  -e "MYSQL_USER=example" \
  -e "MYSQL_PASSWORD=example" \
  --volumes-from mysql-data \
  mysql:5.7
```
データ投入
```
$ docker exec -it mysql mysql -u root -p volume_test
Enter password: 
...
mysql> create table user (
  id int primary key auto_increment,
  name varchar(255)
) engine=InnoDB default charset=utf8mb4 collate utf8mb4_unicode_ci;
Query OK, 0 rows affected (0.05 sec)

mysql> insert into user(name)values('ghiyo'), ('docker'), ('Solomon Hykes');
Query OK, 3 rows affected (0.01 sec)
Records: 3  Duplicates: 0  Warnings: 0
```

Mysqlコンテナの停止 ```--rm```オプションで停止と同時にコンテナは破棄される
```
$ docker stop mysql
```

再度、コンテナの作成
```
$ docker container run -d --rm --name mysql \
  -e "MYSQL_ALLOW_EMPTY_PASSWORD=yes" \
  -e "MYSQL_DATABASE=volume_test" \
  -e "MYSQL_USER=example" \
  -e "MYSQL_PASSWORD=example" \
  --volumes-from mysql-data \
  mysql:5.7
```
登録データの確認
```
$ docker exec -it mysql mysql -u root -p volume_test
Enter password: 
...
mysql> select * from user;
+----+---------------+
| id | name          |
+----+---------------+
|  1 | ghiyo         |
|  2 | docker        |
|  3 | Solomon Hykes |
+----+---------------+
3 rows in set (0.00 sec)
```

#### データのエクスポートとリストア

- mysql-dataのVolumeを新しいbusyboxにマウント
- ホストとbusyboxでdata volume
- busybox起動時にマウントポイントにアーカイブを作成
- ホストのマウントされたVolumeに書き込まれる

```
$ docker run --rm -v [ホストファイルのディレクトリ]:/tmp \
  --volumes-from mysql-data \
  busybox \
  tar cvfz /tmp/mysql-backup.tag.gz /var/lib/mysql
```