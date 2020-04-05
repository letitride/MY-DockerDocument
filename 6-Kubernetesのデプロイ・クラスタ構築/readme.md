# 6. Kubernetesのデプロイ・クラスタ構築

GCEの環境設定
```
$ gcloud auth login
$ gcloud config set project [project-id]
$ gcloud config set compute/zone asia-northeast1-a
```

kubernetes clusterの作成
```
$ gcloud container clusters create gihyo --machine-type=n1-standard-1 --num-nodes=3
ERROR: (gcloud.container.clusters.create) ResponseError: code=403, message=Kubernetes Engine API is not enabled for this project. Please ensure it is enabled in Google Cloud Console and try again: visit https://console.cloud.google.com/apis/api/container.googleapis.com/overview?project=[project-ID] to do so.
```
と出るのでコンソールに提示されたURLにアクセスし、Kubernetes Engine APIを有効にする。

ノードの作成
```
$ gcloud container clusters create gihyo --machine-type=n1-standard-1 --num-nodes=3
$ gcloud container clusters describe gihyo
```
kubectlに作成したclusterのcredentialを渡す
```
$ gcloud container clusters get-credentials gihyo
Fetching cluster endpoint and auth data.
kubeconfig entry generated for gihyo.
```

kubectlが扱うクラスタの確認
```
$ kubectl  config get-contexts
CURRENT   NAME                                            CLUSTER                                         AUTHINFO                                        NAMESPACE
          docker-desktop                                  docker-desktop                                  docker-desktop                                  
          docker-for-desktop                              docker-desktop                                  docker-desktop                                  
*         gke_gihyo-kube-273101_asia-northeast1-a_gihyo   gke_gihyo-kube-273101_asia-northeast1-a_gihyo   gke_gihyo-kube-273101_asia-northeast1-a_gihyo   
```
クラスタを切り替える場合は以下で切り替える。
```
$ kubectl config use-context [CONTEXT_NAME]
```

kubectlでクラスタのノードを確認
```
$ kubectl get nodes
NAME                                   STATUS   ROLES    AGE   VERSION
gke-gihyo-default-pool-b5dd99ac-0n11   Ready    <none>   23h   v1.14.10-gke.27
gke-gihyo-default-pool-b5dd99ac-705q   Ready    <none>   23h   v1.14.10-gke.27
gke-gihyo-default-pool-b5dd99ac-hccl   Ready    <none>   23h   v1.14.10-gke.27
```

## Master Slave構成のMySQLをGKE上に構築する

### PersistentVolumeとPersistentVolumeClaim

PersistentVolumeはプラットフォームに応じたストレージの実体。PersistentVolumeとPersistentVolumeClaimは実体を抽象化した設定・アクセスインターフェース。

PersistentVolumeClaimの設定例は以下。
- ```ReadWriteOnce```: 1つのノードから読み書きできる
- ```resouces.requests.storage```: 確保する容量
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ssd
  resources:
    requests:
      storage: 4Gi
```

### StorageClass

プラットフォームに応じた使用するストレージを定義する。```PersistentVolumeClaim```に記述した```storageClassName: ssd```の定義がこれ。

storage-class-ssd.yml
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
  labels:
    kubernetes.io/cluster-service: "true"
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```
```
$ kubectl apply -f storage-class-ssd.yml 
storageclass.storage.k8s.io/ssd created
```

### StatefulSet

DeploymentはステートレスなPodに対して複製・管理を容易に行えたが、ステートフルなPodに対してはStatefulSetで管理を行う。Podの識別子を管理しPodが再作成されてもストレージなど同じPodに紐づける。

mysql-master.yml & mysql-slave.ymlを作成

```
$ kubectl apply -f mysql-master.yml 
service/mysql-master unchanged
statefulset.apps/mysql-master created
$ kubectl apply -f mysql-slave.yml 
service/mysql-slave created
statefulset.apps/mysql-slave created
```

初期データの登録
```
$ kubectl exec -it mysql-master-0 init-data.sh
```
```
$ kubectl exec -it mysql-slave-0 bash
root@mysql-slave-0:/# mysql -u root -pgihyo tododb -e "SHOW TABLES;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------------+
| Tables_in_tododb |
+------------------+
| todo             |
+------------------+
```

## TODO APIをGKE上に構築する

todo-api.yml
```
$ kubectl apply -f todo-api.yml 
service/todoapi unchanged
deployment.apps/todoapi created
```

## TODO WebアプリケーションをGKE上に構築する

todo-web.yml

nginxコンテナとwebコンテナで共通のvolume、assetsをマウントする

- /var/www/_nuxt @ nginx
- /dist @ web

このままではassetsは空のvolumeなので、webコンテナ起動時に/distにデータをコピーする
```
lifecycle:
  postStart: 
    exec:
      command:
        - cp
        - -R
        - /todoweb/.nuxt/dist
        - /
```

```
$ kubectl apply -f todo-web.yml 
service/todoweb created
deployment.apps/todoweb created
```

## IngressでWebアプリケーションを公開する

ingress.yml
```
$ kubectl apply -f ingress.yml 
ingress.extensions/ingress created
```
```
$ kubectl get ingress
NAME      HOSTS   ADDRESS         PORTS   AGE
ingress   *       34.107.195.96   80      3m22s
```