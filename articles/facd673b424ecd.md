---
title: "Cloud Buildを使ってGAE に NuxtJSアプリケーションを自動デプロイしてみた"
emoji: "👓"
type: "tech"
topics: ["GCP", "CloudBuild", "GAE", "NuxtJS", "CD"]
published: false
---

## はじめに

sweeep Box のフロントエンドの開発で
CD構築をした際に詰まった部分や工夫した部分を中心に
どのように Google App Engine (以下GAE)
に NuxtJSアプリケーションをデプロイしているか解説していきます。
GAEやCloud Buildの基本的なマニフェスト構成・使い方については解説は行いません。

## 概略

* 権限

![](/images/facd673b424ecd/gae-cloudbuild.jpg =400x)

## 工夫したポイント

### GAEのマニフェストファイルを環境毎に置換している

### node_modulesをキャッシュすることでデプロイ時間を短縮している

### SecretManager からサーバサイドの秘匿情報をマウントしている

## 詰まったポイント

### 環境変数を CloudBuildに寄せたいが期待通りに動かない

## むすびに

## 参考記事
