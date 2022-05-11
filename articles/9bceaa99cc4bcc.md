---
title: "GKE Autopilotの永続ボリュームに Filestoreを使ってみた"
emoji: "🗂"
type: "tech"
topics: ["GKE", "Kubernetes", "GCP"]
published: true
---

## はじめに

GKEの永続ボリュームとして Filestoreを使用しています。
導入時の背景とセットアップ方法について、説明していきます。
KubenetesやGCPの基本的な使い方を理解している方が読まれる事を想定して書いています。

### 導入の経緯

* ファイルへのアクセスを高速に行う
* 障害への対策として複数のゾーンにまたがるGKEクラスタを利用する必要性

**という要件・制約の元でファイルの保存先を決める必要がありました。**

ファイルのマウントではなくObjectストレージ(GCS)で
ファイル読み書きを行うことも検討しましたが、速度を考慮して
一時保存はFilestore、最終保存先としてGCSを使用することにしました。

また可用性向上のため、GKE[リージョン クラスタ](https://cloud.google.com/kubernetes-engine/docs/concepts/regional-clusters?hl=ja)をデプロイする必要がありました。
複数のNodeから読み書きを行うため、
[ディスクへのアクセス方法](https://kubernetes.io/ja/docs/concepts/storage/persistent-volumes/#%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%83%A2%E3%83%BC%E3%83%89
) を `ReadWriteMany` にしました。
(補足ですが、ゾーンクラスタの場合 [ComputeEngineの永続ディスク](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/gce-pd-csi-driver
)を利用することが可能で、費用を安く抑えることができます。)

[Compute Engine クライアントでのファイル共有のマウント](https://cloud.google.com/filestore/docs/mounting-fileshares?hl=ja)
を行うか Filestore を使ってマウントを行うかどちらかという選択肢の中で
費用は高くても高速でマネージドなFilestoreを使うことになりました。

### 実際にデプロイする

`gcloud`(GCPコンソールでも可能), `kubectl` のコマンドが必要になります。
基本的な使い方については解説しないです。

[AutopilotモードのGKE](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview)をデプロイしていきます。

#### GKEデプロイ

```bash
gcloud container \
--project ${YOUR_PROJECT} \
clusters create-auto "test-autopilot-cluster-1" \
--region "asia-northeast1" \
--release-channel "regular" \
--network ${YOUR_VPC_NETWORK} \
--subnetwork ${YOUR_SUB_NETWORK} \
--cluster-ipv4-cidr "/17" --services-ipv4-cidr "/22"
```

数分後作成したクラスタの情報が表示されるので、
`kubectl get nodes`等で接続の確認を行ってください。

#### Filestoreのデプロイ

次に Filestoreのデプロイを行なって行きます。**料金が高いので消し忘れには注意してください**

**`--network=name`　は作成したクラスタと同じにしてください。**
**同じVPC内に配置しないとマウントすることができません。**

```bash
# 最小が1TBなので、1TBを指定
gcloud filestore instances create test-filestore \
--project=${YOUR_PROJECT} \
--zone=asia-northeast1-b \
--file-share=capacity=1TB,name=test_shared \
--network=name=${YOUR_VPC_NETWORK}  \
--tier=STANDARD
```

数分後に作成されたインスタンス情報が表示されます。
作成後に

```bash
gcloud filestore instances list

> INSTANCE_NAME   LOCATION           TIER      CAPACITY_GB  FILE_SHARE_NAME  IP_ADDRESS ~
test-filestore  asia-northeast1-b  STANDARD  1024         test_shared      10.225.83.106 ~
```

を入力して IPアドレスをメモしておきます。

デプロイ直後のディレクトリのパーミッションは`root`権限になっているため
Filestore をフォーマッティングして行きます。

AutopilotのPodからの変更はできないので、GCEのインスタンスを立てます。

```bash
gcloud compute instances create temp-workload \
--zone=asia-northeast1-b \
--machine-type=f1-micro \
--network=sweeep-dev-vpc \
--subnet=sweeep-dev-vpc-subnet
```

GCEインスタンスの作成後
インスタンスへ sshを行い以下のように権限を変更します

```bash
# ssh
gcloud compute ssh temp-workload --zone=asia-northeast1-b
sudo apt-get update &&  sudo apt-get install nfs-common
# マウント
sudo mkdir -p /opt/shared && sudo mount $FilestoreIP:/test_shared /opt/shared
# 権限を変更
sudo chmod 766 /opt/shared
```

`$FilestoreIP`には先ほどコピーしたFilestoreのIPアドレスをを入力してください。
以上で Filestoreのデプロイは完了です。

#### 永続ボリュームのデプロイ

**次に永続ボリュームをデプロイして行きます。
[動的なプロビジョニングを行います](https://kubernetes.io/ja/docs/concepts/storage/persistent-volumes/#%E5%8B%95%E7%9A%84)
PersistentVolume(PV), PersistentVolumeClaim(PVC)のリソースを作成します。
PVとは 噛み砕くとクラスタに配置されるストレージそのもののようなイメージで、
PVCは ユーザが要求するストレージ(使用するサイズ等を指定できる)です。
NodeとPod(CPUやメモリを調節できる)のような関係で理解するとわかりやすいと思います。
[詳細はこちらを確認してください。](https://kubernetes.io/ja/docs/concepts/storage/persistent-volumes/#%E6%A6%82%E8%A6%81)

```yaml:persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
  labels:
      app: test-volume
spec:
  capacity: # Filestoreの容量
    storage: 1T
  accessModes: # アクセス方法
  - ReadWriteMany
  nfs:
    path: /test_shared
    server:  10.225.83.106 # コピーした FilestoreのIPを入力してください。
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-disk
spec:
  storageClassName: ""
  accessModes: # アクセス方法
    - ReadWriteMany
  resources:
    requests: # 使用するサイズ (pvc < pv)
      storage: 1T　
  selector:
    matchLabels:
      app: test-disk
```

上記のファイルをデプロイし、取得が確認できたら完了です。

```bash
kubectl apply -f persistent-volume.yaml
> persistentvolume/test-pv created
persistentvolumeclaim/test-disk created

kubectl get pv

> NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
test-pv   1T         RWX            Retain           Available                                   12s


> kubectl get pvc
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-disk   Pending                                                     12s

# STATUS は要求時に変更される
```

## 動作確認

**Podをデプロイして本当にマウントされているか確認します。**

```bash:test-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment1
  labels:
    service: test1
spec:
  replicas: 2
  selector:
    matchLabels:
      service: test1
  template:
    metadata:
      labels:
        service: test1
    spec:
      volumes:
        - name: test-volume
          persistentVolumeClaim:
            claimName: test-pvc　# PVCを要求する
      containers:
        - name: test
          image: alpine:latest
          command: [ "sleep" ]
          args: [ "infinity" ]
          resources:
            limits:
              cpu: 500m
              memory: 128Mi
          volumeMounts: # test-volume という名前のボリュームを /mnt/test にマウントする
          - mountPath: /mnt/test
            name: test-volume
```

```bash
# Deploymentをデプロイ
kubectl apply -f test-deployment.yaml

# 名称を変えて別の Deploymentを作ってみる。
vim test-deployment.yaml
> metadata:
  name: test-deployment2

kubectl apply -f test-deployment.yaml

# pod を確認
k get pods
> NAME                                READY   STATUS    RESTARTS   AGE
test-deployment1-5fd94f4bc6-k9nfc   1/1     Running   0          7m38s
test-deployment1-5fd94f4bc6-vpgq9   1/1     Running   0          9m16s
test-deployment2-5fd94f4bc6-768fj   1/1     Running   0          7m4s
test-deployment2-5fd94f4bc6-qtplg   1/1     Running   0          7m4s

# podに入る
kubectl exec -it test-deployment1-5fd94f4bc6-k9nfc -- sh # - 1
kubectl exec -it test-deployment1-5fd94f4bc6-vpgq9 -- sh # - 2 1と同じデプロイメントにひもづくPod
kubectl exec -it test-deployment2-5fd94f4bc6-768fj -- sh # - 3 別のデプロイメントにひもづくPod

> /
/ cd mnt/test # - 1, 2, 3 で実行
echo "Hello World!!" > test.txt # -1 で実行
cat test.txt # - 2, 3で実行
> Hello Workd!!
vi test.txt, rm test.txt # 2, 3 で変更や削除を確認
```

という感じで複数のPodからファイルの確認や変更ができるようになりました。

## **注意**

作成したリソースをそのままにすると結構な料金が発生してしまうので、
作成後使わない場合は忘れず削除してください！

```bash:削除用のコマンド
# GCE
gcloud compute instances delete temp-workload --zone=asia-northeast1-b
# Filestore
gcloud filestore instances delete test-filestore --zone=asia-northeast1-b
# GKE
gcloud container clusters delete test-autopilot-cluster-1 --region asia-northeast1
```

## むすびに

sweeepでは一緒に働くエンジニアを募集しています！
なぜを追求して、技術選定を行いたい!
自動化大好き!インフラ効率化したい！技術大好き！
なエンジニアの方一緒に働きましょう！

https://corp.sweeep.ai/recruit
