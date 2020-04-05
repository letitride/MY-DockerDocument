# 7. Kubernetesの発展的な利用

## Job

1つ以上のPodを起動し、指定した処理を実行する。処理完了したPodはクラスタ内で削除されず保持される。

simple-job.yml
```
apiVersion: batch/v1
kind: Job
metadata:
  name: pingpong
  labels:
    app: pingpong
spec:
  parallelism: 3
  template:
    metadata:
      labels:
        app: pingpong
    spec:
      containers:
        - name: pingpong
          image: gihyodocker/alpine:bash
          command: ["/bin/bash"]
          args: 
            - "-c"
            - |
              echo [`date`] ping!
              sleep 10
              echo [`date`] pong!
      restartPolicy: Never
```
```
$ kubectl apply -f simple-job.yml 
job.batch/pingpong created
$ kubectl logs -f -l app=pingpong
[Sun Apr 5 02:13:28 UTC 2020] ping!
[Sun Apr 5 02:13:38 UTC 2020] pong!
[Sun Apr 5 02:13:17 UTC 2020] ping!
[Sun Apr 5 02:13:27 UTC 2020] pong!
[Sun Apr 5 02:13:17 UTC 2020] ping!
[Sun Apr 5 02:13:27 UTC 2020] pong!
```

## CronJon

jobは一度だけの実行だったが、スケジューリングして処理を実行できる。

Podはスケジュールに応じて作成され続ける。

simple-cronjob.yml
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: pingpong
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: pingpong
        spec:
          containers:
            - name: pingpong
              image: gihyodocker/alpine:bash
              command: ["/bin/bash"]
              args: 
                - "-c"
                - |
                  echo [`date`] ping!
                  sleep 10
                  echo [`date`] pong!
          restartPolicy: Never
```


## Secret

認証情報などの情報をbase64エンコードした値でKubernetesリソースとして登録できる。

登録したリソースはvolumeにマウントしたり、valueFromKeyから参照して各コンテナから扱うことができる。

```
$ echo "your_username:$(openssl passwd -quiet -crypt your_password)"|base64
eW91cl91c2VybmFtZTpmSXYxTUpUNXRGYXlNCg==
```

nginx-secret.yml
```
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret
type: Opaque
data:
  .htpasswd: eW91cl91c2VybmFtZTpmSXYxTUpUNXRGYXlNCg==
```
```
$ kubectl apply -f nginx-secret.yml
```

### volumeMount

作成したsecretは以下のようにvolumeにマウントできる
```
volumes:
  name: nginx-secret-volume
  secret:
    secretName: nginx-secret
```
あとは、コンテナにvolumeマウントすればよい
```
...略
containers:
  volumeMounts:
    - mountPath: pathto
      name: nginx-secret-volume
      readOnly: true
volumes:
  name: nginx-secret-volume
  secret:
    secretName: nginx-secret
```

### valueFrom

valueFromの場合はvalueFrom.secretKeyRefにmameとdataに定義したkeyを指定する。

```
...中略
env:
  - name: SECRET_KEY
    valueFrom: 
      secretKeyRef:
        name: nginx-secret
        key: .htpasswd
```

## ユーザ管理とRole-Based Access Control

### ClusterRole

クラスタ全体で有効な許可する操作群の定義。後ほどClusterRoleを与えるServiceAccountと紐づける

pod-reader-role.yml
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

### ClusterRoleBinding

ClusterRoleとServiceAccountを紐づける

pod-reader-account.yml
```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pod-read-binding
subjects:
  - kind: ServiceAccount
    name: gihyo-user
    namespace: default
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount

所謂ユーザ。コマンドから作成できる。secretに認証トークンが保存されている。

```
$ kubectl create serviceaccount gihyo-user
$ kubectl get serviceaccount  gihyo-user -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2020-04-05T03:14:54Z"
  name: gihyo-user
  namespace: default
  resourceVersion: "623243"
  selfLink: /api/v1/namespaces/default/serviceaccounts/gihyo-user
  uid: 9ee1aef2-76eb-11ea-a8d5-42010a920005
secrets:
- name: gihyo-user-token-tnk4m
```

認証トークンの確認 ```data.token```が認証トークン。base64エンコードされているので使用時にはデコードする必要がある。
```
$ kubectl get secret  gihyo-user-token-tnk4m -o yaml
$ echo '<token>' | bade64 -D
```

### 作成したユーザの使用

kubernetesクラスタのcontextの確認
```
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://kubernetes.docker.internal:6443
  name: docker-desktop
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://35.221.116.206
  name: gke_gihyo-kube-273101_asia-northeast1-a_gihyo
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-for-desktop
- context:
    cluster: gke_gihyo-kube-273101_asia-northeast1-a_gihyo
    user: gke_gihyo-kube-273101_asia-northeast1-a_gihyo
  name: gke_gihyo-kube-273101_asia-northeast1-a_gihyo
current-context: gke_gihyo-kube-273101_asia-northeast1-a_gihyo
kind: Config
preferences: {}
users:
- name: docker-desktop
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: gke_gihyo-kube-273101_asia-northeast1-a_gihyo
  user:
    auth-provider:
      config:
        access-token: ****
      name: gcp
```

ユーザのセット
```
$ kubectl config set-credentials gihyo-user --token=***
```

コンテキストの作成 コンテキストはユーザ名と接続するクラスタからなる定義

cluster=gke_gihyo-kube-273101_asia-northeast1-a_gihyoに接続するuser=gihyo-userをcontext名gihyo-userとする。
```
$ kubectl config set-context gihyo-user --cluster=gke_gihyo-kube-273101_asia-northeast1-a_gihyo --user=gihyo-user
```

以下が```$ kubectl config show```で作成されたことが確認できる
```
- context:
    cluster: gke_gihyo-kube-273101_asia-northeast1-a_gihyo
    user: gihyo-user
  name: gihyo-user
```

contextの切り替え
```
$ kubectl config use-context gihyo-user
```

podへのreadはできるが...
```
$ kubectl get pod
NAME                       READY   STATUS      RESTARTS   AGE
mysql-master-0             1/1     Running     0          24h
mysql-slave-0              1/1     Running     0          24h
mysql-slave-1              1/1     Running     0          24h
pingpong-fzp4c             0/1     Completed   0          83m
pingpong-mcqqg             0/1     Completed   0          83m
pingpong-zvx49             0/1     Completed   0          83m
todoapi-5467bf5d79-gwlfw   2/2     Running     0          143m
todoapi-5467bf5d79-pxk4b   2/2     Running     0          143m
todoweb-6df88ccc69-922vm   2/2     Running     0          123m
todoweb-6df88ccc69-bl6k4   2/2     Running     0          123m
```
他へのリソースにはアクセスできない
```
$ kubectl get deployment
Error from server (Forbidden): deployments.extensions is forbidden: User "system:serviceaccount:default:gihyo-user" cannot list resource "deployments" in API group "extensions" in the namespace "default"
```

### ServiceAccountユーザのRoleに応じたPodを作成する

ServiceAccountとPodを紐づけることによってServiceAccountに応じたRoleをPodに紐づけることができる。

以下のpodからは kubectl get pod が実行できる。
```
kind: Pod
spec:
  serviceAccountName: gihyo-pod-reader
  containers:
    ...略
```

## Helm

デプロイ先に依存する設定値を管理できる仕組み。

ローカルにCLIとKubernetes Cluster上に独立したmamespaceでリソースマネージャーが乗っかる。

### Chart

各Kubernetesのリソースtemplate群。HemlはCharを通してKubernetes上にデプロイする。

#### デフォルトvalueとカスタムvalue

上記で示した設定値を管理する仕組み。ユーザはカスタムvalueファイルを用意して、環境ごとの設定値をchart install時に注入する。尚、カスタムvalueに記載のない設定値はデフォルトvalueの設定値が使用される
```
$ helm install -f <カスタムvalueファイル> --name <リリース名> <Chartリポジトリ/Chart名>
```

### Chartのローカリリポジトリを使用する

作成したコンテナをデプロイするKubernetesリソースをローカルのChartリポジトリで管理する。

ローカルリポジトリは、CLIをインストールしたホストに作成されている。

ローカルリポジトリの起動
```
$ helm server &
```

Chartの雛形を作成する
```
$ helm create <chart名>
```

雛形内の```{{.Values.}}```と部分がvalueファイルの適用部分となる。

```
{.Values.nginx.image.repository}}
```
```
nginx:
  image:
    repository: gihyodocker/nginx:latest
```
のようなマッパーになる。

#### Chartのパッケージング

Chart.yml
```
apiVersion: v1
description : A Helm chart for Kubernetes
name: echo
version: 0.1.0
```
```
$ helm package echo
```

#### 作成したチャートのインストール

環境に応じたカスタムvalueファイルを用意。
```
$ helm install -f <カスタムファイル> --name echo local/echo
```

### Chartリポジトリの公開と取り込み

#### 公開

Chartリポジトリは /stable/index.yaml の定義があればrepositoryとして認識できる。

```
$ git init
$ mkdir stable
$ helm create example
$ helm package example
$ cd ../
$ git add .
$ git commit ...
```
としてChartリポジトリをgit管理してgitリモートリポジトリにプッシュすればよい。

#### 取り込み

```
$ helm repo add <リポジトリ名> <リポジトリURL>
```
github上のChartコードを取り込む場合はgithubページ等で/stableにhttpアクセスできれば良い。


## コンテナのヘルスチェック

### livenessProbe

コンテナが動いているか？の確認。

```
conteiners:
  - name: echo
    livenessProbe:
      exec:
        command:
          - <command>
          - <option>
      initialDelaySeconds: 3
      periodSeconds: 5
```

### readinessProbe

コンテナがServiceのリクエストを受け取れるか？の確認。
```
conteiners:
  - name: echo
    readinessProbe:
      httpGet:
        path: /
        port: 8080
      timeSeconds: 3
      initialDelaySeconds: 3
    ports:
      - containerPort: 8080
```