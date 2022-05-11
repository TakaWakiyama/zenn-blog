---
title: "GKE ファイル共有 Filestore"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GKE", "Kubenetes"]
published: false
---

## はじめに

GKEの永続ボリュームとして filestoreを使用しています。
導入時の背景とセットアップ方法について、説明していきます。

### 導入の経緯

下記の要件・制約の元でファイルの保存先を決める必要がありました。

1. ファイルへのアクセスを高速に行う
1. 障害への対策として複数のゾーンにまたがるGKEクラスタを利用する必要性

Objectストレージ(GCS)でファイル読み書きを行うことも検討しましたが、
速度的な面で一時保存はFilestore、最終保存先としてGCSを使用することにしました。

また可用性向上のため、GKEのリージョナルクラスタにデプロイする必要がありました。
複数のNodeから読み書きを行う必要があったので、
[アクセス方法](https://kubernetes.io/ja/docs/concepts/storage/persistent-volumes/#%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%83%A2%E3%83%BC%E3%83%89
) を `ReadWriteMany` にする必要がありました。
ゾーンクラスタの場合 [ComputeEngineの永続ディスク](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/gce-pd-csi-driver
)　を利用することが可能で、費用を安く抑えることができます。

[Compute Engine クライアントでのファイル共有のマウント](https://cloud.google.com/filestore/docs/mounting-fileshares?hl=ja)
を行うか Filestore を使ってマウントを行うかどちらかという選択肢の中で
費用は高くても高速でマネージドなFilestoreを使うことになりました。

### 実際にやってみる

* クラスタの準備: 30
* filestoreのデプロイ, フォーマッティング: 30
* PV, PVCのデプロイ: 30

1.5h

## 動作確認

* 30分

## むすびに

sweeepでは一緒に働くエンジニアを募集しています！
なぜを追求して、技術選定を行いたい!
自動化大好き!インフラ効率化したい！技術大好き！
なエンジニアの方一緒に働きましょう！

https://corp.sweeep.ai/recruit

