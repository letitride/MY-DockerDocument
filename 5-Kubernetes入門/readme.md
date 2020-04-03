# 5. Kubernetes入門

kubernetesがstarting...から起動しない場合、
```
$ rm -rf ~/Library/Group\ Containers/group.com.docker/pki/
$ rm -rf ~/.kube
```
とした後、DockerのpreferencesからDockerのRestartとReset kubernetes clusterを行う。
プログレスが周りっぱなしになっている場合、プログレス上で右クリックしたりすると治まる場合がある。

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.8.3/src/deploy/recommended/kubernetes-dashboard.yaml
```
```
$ kubectl get pod --namespace=kube-system -l k8s-app=kubernetes-dashboard
```
```
$ kubectl proxy
```
```
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login
```
signinを求められるがskipできる

## KubernetesクラスタとNode

NodeとはDockerが動作するホストを指す。通常はDockerをインストールしたサーバマシンであったり、EC2のインスタンスであったりする。

Kubernetesクラスタは各Nodeを管理下におき、クラス全体を管理するNode、1台以上のMasterNodeとWorkerNode群で構築される。

クラスタに参加しているNode一覧を確認
```
$ kubectl get nodes
NAME             STATUS   ROLES    AGE    VERSION
docker-desktop   Ready    master   4h8m   v1.15.5
```

## Namespace

Kubernetesクラスタ内に仮想的なクラスタを配置できる懸念。
```
$ kubectl get namespace
NAME              STATUS   AGE
default           Active   4h10m
docker            Active   4h9m
kube-node-lease   Active   4h10m
kube-public       Active   4h10m
kube-system       Active   4h10m
```

## Pod

コンテナの集合体。たとえばリバースプロキシとアプリケーションをセットとしたpodを定義したりする。

Podでまとめられたコンテナ群はPod毎に必ず同一Node上に配置される。

Podを複製して同じPodを別Nodeに配置することができる。

単一Node環境以外では原則MasterNodeにアプリケーションPodがデプロイされることはない。

Pod単位でIPアドレスを持つ。よってPod内のコンテナは同一IPアドレスを持つことになる。portの衝突に注意。

よってPodはネットワークサーバー、中のコンテナはアプリケーションデーモンのような関係になる。

### Podの作成とデプロイ

Nginxをリバースプロキシとしたechoサーバを作成する
 - ```metadata```: Podに対する属性値を指定

```
apiVersion: v1
kind: Pod
metadata:
  name: simple-echo
spec:
  containers:
  - name: nginx
    image: gihyodocker/nginx-proxy:latest
    env:
    - name: BACKEND_HOST
      value: localhost:8080
    ports:
    - containerPort: 80
  - name: echo
    image: gihyodocker/echo:latest
    ports:
    - containerPort: 8080
```
```
$ kubectl apply -f simple-pod.yml 
pod/simple-echo created
```

### Podの一覧を取得

```
$ kubectl get pod
NAME          READY   STATUS    RESTARTS   AGE
simple-echo   2/2     Running   0          9m50s
```

### Pod内のコンテナを操作する

```
$ kubectl exec -it [pod名] sh -c [コンテナ名]
```
```
$ kubectl exec -it simple-echo sh -c nginx
```
```
$ kubectl logs -f simple-echo -c echo
```

### Podの削除

```
$ kubectl delete pod simple-echo
```

## ReplicaSet

Podの定義及び複製のルールを記述した単位。

### ReplicaSetの作成とデプロイ

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo
  labels:
    app: echo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx-proxy:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:latest
        ports:
        - containerPort: 8080
```
```
$ kubectl apply -f simple-replicaset.yml
```
```
$ kubectl get pod
NAME         READY   STATUS    RESTARTS   AGE
echo-rdgjd   2/2     Running   0          52s
echo-rf8m2   2/2     Running   0          52s
echo-wl9ht   2/2     Running   0          52s
```

### ReplicaSetの削除

```
$ kubectl delete -f simple-replicaset.yml 
```

## Deployment

ReplicaSetを作成する。ほぼReplicaSetと同様の定義だが、作成したReplicaSetの世代管理が可能となる為、Deploymentを通してReplicaSetを作成することが一般的。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  labels:
    app: echo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx-proxy:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:latest
        ports:
        - containerPort: 8080
```
```
$ kubectl apply -f simple-deployment.yml --record
```
```
$ kubectl get pod,replicaset,deployment --selector app=echo
NAME                        READY   STATUS    RESTARTS   AGE
pod/echo-79dcfc9f84-kgcn9   2/2     Running   0          2m28s
pod/echo-79dcfc9f84-kz89n   2/2     Running   0          2m28s
pod/echo-79dcfc9f84-zx4cm   2/2     Running   0          2m28s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.extensions/echo-79dcfc9f84   3         3         3       2m28s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/echo   3/3     3            3           2m29s
```

### リビジョンの確認

```
$ kubectl rollout history deployment echo
deployment.extensions/echo 
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple-deployment.yml --record=true
```

### ReplicaSetのライフサイクル

Pod数の更新 = リビジョンは更新されない

コンテナイ定義を更新 = リビジョンが更新される
```
$ kubectl get pod --selector app=echo
NAME                    READY   STATUS              RESTARTS   AGE
echo-677fb7c47d-dgdwq   2/2     Running             0          25s
echo-677fb7c47d-p8tqb   0/2     ContainerCreating   0          11s
echo-79dcfc9f84-kz89n   2/2     Running             0          9m42s
echo-79dcfc9f84-zx4cm   2/2     Terminating         0          9m42s

$ kubectl rollout history deployment echo
deployment.extensions/echo 
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple-deployment.yml --record=true
2         kubectl apply --filename=simple-deployment.yml --record=true
```

### ロールバックの実行

特定のリビジョンのプロファイル確認

```
$ kubectl rollout history deployment echo --revision=1
```

直前のrevisionに巻き戻す
```
$ kubectl rollout undo deployment echo
```

### Deploymentの削除

関連するReplicaSetとPodも削除される

```
$ kubectl delete -f simple-deployment.yml
```

## Service

Podに対してのネットワークインターフェースを提供する。このネットワークインターフェースはkubernetesクラスタ内からのアクセスに使用される。

ターゲットはラベルセレクタによって指定できる。

Label付のレプリカセット simple-replicaset-with-label.yml
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo-spring
  labels:
    app: echo
    release: spring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
      release: spring
  template:
    metadata:
      labels:
        app: echo
        release: spring
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx-proxy:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:latest
        ports:
        - containerPort: 8080
---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo-summer
  labels:
    app: echo
    release: summer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
      release: summer
  template:
    metadata:
      labels:
        app: echo
        release: summer
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx-proxy:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:latest
        ports:
        - containerPort: 8080
```
```
$ kubectl apply -f simple-replicaset-with-label.yml
$ kubectl get pod -l app=echo -l release=summer
NAME                READY   STATUS    RESTARTS   AGE
echo-summer-kmk98   2/2     Running   0          66s
```

### Servicdの作成

上記、release=summer labelを持つpodに対してアクセスできるServiceを定義する。

simple-service.yml
```
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector:
    app: echo
    release: summer
  ports:
    - name: http
      port: 80
```
```
$ kubectl apply -f simple-service.yml 
```

### Serviceへの接続確認

クラスタへデバッグコンテナをデプロイする
```
$ kubectl run -i --rm --tty debug --image=gihyodocker/fundamental:0.1.0 --restart=Never -- bash -il
debug:/# curl http://echo/
Hello Docker!!debug:/#
```
```
$ kubectl logs -f echo-summer-kmk98 -c echo
2020/04/02 23:44:55 start server
2020/04/02 23:56:54 received request
```

Serviceの名前解決は ```[Service名].[Namespace名]```となる。

サービス名:echo Namespace:defaultの場合は
```
$ curl http://echo.default
```
となる。また、同一Namespace内からのアクセスはNamespaceの指定を省略可能。よって、```http://echo```にてアクセスできる。

### NodePort Service

ノードのportと作成済みサービスのポートフォワードを定義できる。これによりノードからkubernetesクラスタへアクセスできる。

simple-nodeport-service.yml 
```
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  type: NodePort
  selector:
    app: echo
  ports:
    - name: http
      port: 80
```
```
$ kubectl apply -f simple-nodeport-service.yml 
service/echo configured
$ kubectl  get svc echo
NAME   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
echo   NodePort   10.108.46.126   <none>        80:30250/TCP   20m
$ curl http://localhost:30250
Hello Docker!!
```

## Ingress

NodePortはポートフォワード、L4の転送なので、L7、例えばURLのパスによって接続先を変更するといったようなことはできない。

上のレイヤでルーティングする場合、Ingressを使用する。

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.16.2/deploy/mandatory.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.16.2/deploy/provider/cloud-generic.yaml

$ kubectl -n ingress-nginx get service,pod
NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/default-http-backend   ClusterIP      10.106.106.205   <none>        80/TCP                       5m14s
service/ingress-nginx          LoadBalancer   10.103.142.72    localhost     80:31708/TCP,443:30478/TCP   4m55s

NAME                                            READY   STATUS    RESTARTS   AGE
pod/default-http-backend-6cdd6c64f8-cwq7g       1/1     Running   0          5m14s
pod/nginx-ingress-controller-675df7b6fd-dfzcr   1/1     Running   0          5m13s
```

### Ingressを通じたアクセス

```
$ kubectl apply -f simple-ingress.yml 
ingress.extensions/echo created
ichikawafumiyanoMacBook-Pro:5-Kubernetes入門 ichikawafumiya$ kubectl get ingress
NAME   HOSTS              ADDRESS   PORTS   AGE
echo   ch05.gihyo.local             80      12s
```
```
$ curl http://localhost -H 'Host: ch05.gihyo.local'
Hello Docker!!
```