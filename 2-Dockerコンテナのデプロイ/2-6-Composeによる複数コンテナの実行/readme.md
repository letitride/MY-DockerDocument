# 2 Dockerコンテナのデプロイ

## 2.6 Composeによる複数コンテナの実行

### Jenkinsの作成・実行

docker-compose.yml
```
version: "3"
services: 
  master:
    container_name: master
    image: jenkins/jenkins:lts
    ports:
      - 8080:8080
    volumes: 
      - ./jenkins_home:/var/jenkins_home
```
- ```container_name``` 作成されるコンテナ名を指定できる
- ```volumes``` ホストファイルをコンテナファイルとしてマウントする

### Jenkinsの初期設定

http://localhost:8080/ に接続し、初期設定を行う。
初期パスワードはマウントボリューム、./jenkins_home/secrets/initialAdminPasswordで確認できる
設定値はコンテナからマウントボリュームを通してホスト(./jenkins_home)に保存される

### JenkinsコンテナのSSH keyを作成

jenkinsをmaster - slave構成にする為、masterコンテナのSSHキーを作成する。
masterコンテナへ接続 パスフレーズは設定しない。
```
$ docker exec -it master ssh-keygen -t rsa  -C ""
```

一旦コンテナを削除
```
$ docker-compose down
```

### slaveのプロビジョニングを追加

```
version: "3"
services: 
  master:
    container_name: master
    image: jenkins/jenkins:lts
    ports:
      - 8080:8080
    volumes: 
      - ./jenkins_home:/var/jenkins_home
    links:
      - slave01
  slave01:
    container_name: slave01
    image: jenkinsci/ssh-slave
    environment: 
      - JENKINS_SLAVE_SSH_PUBKEY=<./jenkins_home/.ssh/id_rsa.pub>
```
- ```links``` masterコンテナからslave01への接続はslave01というリテラルで名前解決できる。

### slaveノードの追加

コンテナの作成
```
$ docker-compose up -d
```

http://localhost:8080に接続し、[Jenkinsの管理] -> [ノードの管理] -> [新規ノード作成]を選択
- ノード名:"slave01"
- Permanent Agent: チェック
- リモートFSルート: /home/jenkins
- 起動方法: SSH経由でUnixマシンのスレーブエージェントを起動
  - ホスト: slave01 (```link:```定義したエイリアス)
  - 認証情報: [追加] -> Jenkins
    - 種類: SSH ユーザ名と秘密鍵 
    - ユーザ名: jenkins
    - 秘密鍵: 直接入力 <./jenkins_home/.ssh/id_rsa>
  -  	Host Key Verification Strategy; Non verifying Verification Strategy
  - 高度な設定
    - Javaのパス: /usr/local/openjdk-8/bin/java