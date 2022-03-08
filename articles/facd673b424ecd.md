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

## GAEにNuxtJSアプリケーションをデプロイする

1. app.yaml の作成

GAEへビルド後のファイルをデプロイするため app.yamlに設定を記述します。

```yaml:path/app.yaml
runtime: nodejs16
instance_class: _INSTANCE_CLASS
service: _SERVICE_NAME
automatic_scaling:
  target_cpu_utilization: 0.6
  target_throughput_utilization: 0.6
  min_instances: _MIN_INSTANCES
  max_instances: _MAX_INSTANCES
handlers:
  - url: /_nuxt
    static_dir: .nuxt/dist/client
    secure: always
  - url: /(.*\.(gif|png|jpg|ico|txt))$
    static_files: src/static/\1
    upload: src/static/.*\.(gif|png|jpg|ico|txt)$
    secure: always
  - url: /.*
    script: auto
    secure: always
env_variables:
  HOST: "0.0.0.0"
  NODE_ENV: "production"
```

## 工夫したポイント

### GAEのマニフェストファイルを環境毎に置換している

### node_modulesをキャッシュすることでデプロイ時間を短縮している

### SecretManager からサーバサイドの秘匿情報をマウントしている

## 詰まったポイント

### 環境変数を CloudBuildに寄せたいが期待通りに動かない

## むすびに

## 参考記事
