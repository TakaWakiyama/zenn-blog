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

動作確認環境
`nodejs version 15.14.0`
`nuxt version 2.15.7`

* 権限

## GAEにNuxtJSアプリケーションをデプロイする

### app.yaml の作成

GAEへNuxtJSアプリケーションを
デプロイするため app.yamlに設定を記述します。

```yaml:path/app.yaml
runtime: nodejs16 # Node.js ランタイム環境 バージョン
instance_class: _INSTANCE_CLASS # インスタンスのクラス デフォルト: F1
service: _SERVICE_NAME # GAE サービスの名前 デフォルト: default
automatic_scaling: # スケールアウトに関する設定
  target_cpu_utilization: 0.6
  target_throughput_utilization: 0.6
  min_instances: _MIN_INSTANCES # 最小インスタンス数
  max_instances: _MAX_INSTANCES # 最大インスタンス数
handlers: # URLsパターンと処理方法のセット 適用優先度準は上に記載されている順に高くなっている。
  - url: /_nuxt # リクエストのURLとのマッピングされる部分
    static_dir: .nuxt/dist/client # ルートディレクトリ起算のファイルパス
    secure: always # alwaysにすると http から httpsにリダイレクト
  - url: /(.*\.(gif|png|jpg|ico|txt))$
    static_files: static/\1
    upload: static/.*\.(gif|png|jpg|ico|txt)$
    secure: always
  - url: /.*
    script: auto
    secure: always
env_variables:
  HOST: "0.0.0.0"
  NODE_ENV: "production"
```

### 解説

* `instance_class`, `service`, `min_instances`, `max_instances` は
デプロイ時に各環境毎の設定を埋め込めるように実際の値ではなく `_環境変数名` としています。
```gcloud app deploy app.yaml --project [project-id]```
でターミナルからデプロイの動作確認を行う場合、上記項目を削除してデフォルト値でのデプロイを行なってください。

* `automatic_scaling` の詳細については [app\.yaml リファレンス  \|  Python 2 の App Engine スタンダード環境  \|  Google Cloud](https://cloud.google.com/appengine/docs/standard/python/config/appref#automatic_scaling) 公式ドキュメントに詳しく記載があるので、こちらを参考にしてください。
`target_cpu_utilization`, `target_throughput_utilization` によって インスタンスをスケールさせるCPUの閾値を設定しています。

* `handlers` については公式ドキュメントに以下のように説明が記載されています。

    > handlers 要素は、app.yaml 構成ファイルの必須要素です。この要素は、URL パターンと処理方法の説明リストを提供します。App Engine で URL を処理するには、アプリケーション コードを実行するか、画像、CSS、JavaScript など、コードと一緒にアップロードされた静的ファイルを提供します。

    詳細については
    [app\.yaml リファレンス  \|  Python 2 の App Engine スタンダード環境  \|  Google Cloud](https://cloud.google.com/appengine/docs/standard/python/config/appref#handlers_element)
    公式ドキュメントに詳しく記載があるので、こちらを参考にしてください。

* NuxtJSの handlersに関して
    [Nuxt \- Google App Engine](https://nuxtjs.org/deployments/google-appengine/) こちらを参考に作成しました。

## CloudBuild から GAEにアプリケーションをデプロイする

### cloudbuild.yaml の作成

```yaml: cloudbuild.yaml
steps:
  # app.yamlの環境変数の置換
  - name: gcr.io/cloud-builders/gcloud
    args:
      - '-c'
      - |
        sed -i -e "s/_SERVICE_NAME/$_SERVICE_NAME/g" app.yaml
        sed -i -e "s/_INSTANCE_CLASS/$_INSTANCE_CLASS/g" app.yaml
        sed -i -e "s/_MIN_INSTANCES/$_MIN_INSTANCES/g" app.yaml
        sed -i -e "s/_MAX_INSTANCES/$_MAX_INSTANCES/g" app.yaml
    id: replace appyaml env
    entrypoint: bash
  # cacheファイルを取得
  - name: gcr.io/cloud-builders/gsutil
    args:
      - 'cp'
      - 'gs://$_BACKET_NAME/cache.tar.gz'
      - 'cache.tar.gz'
    id: retrive moduel cache
  # cacheファイルを解凍
  - name: bash
    args:
      - 'tar'
      - 'xzf'
      - 'cache.tar.gz'
    id: unzip moduel cache
  # ライブラリインストール
  - name: 'node:16'
    args:
      - install
    id: install dep
    entrypoint: npm
  # secret マウント
  - name: gcr.io/cloud-builders/gcloud
    args:
      - '-c'
      - |
        gcloud secrets versions access latest --secret $_SECRET_NAME >> file_path/hoge
    id: mount secret
    entrypoint: bash
  # ビルド
  - name: 'node:16'
    env:
      - NUXT_ENV_BASE_URL=$_NUXT_ENV_BASE_URL
      - NUXT_ENV_ENV_TYPE=$_NUXT_ENV_ENV_TYPE
    args:
      - run
      - build
    id: build static file
    entrypoint: npm
  # デプロイ
  - name: gcr.io/google.com/cloudsdktool/cloud-sdk
    args:
      - '-c'
      - gcloud config set app/cloud_build_timeout 3600 && gcloud app deploy
    id: deploy to gae
    entrypoint: bash
  # zip化
  - name: bash
    args:
      - 'tar'
      - 'czf'
      - 'cache.tar.gz'
      - 'node_modules'
    id: 'archive node_modules'
    waitFor:
     - 'deploy to gae'
  # upload
  - name: gcr.io/cloud-builders/gsutil
    args:
      - 'cp'
      - 'cache.tar.gz'
      - 'gs://$_BACKET_NAME/cache.tar.gz'
    waitFor:
      - 'archive node_modules'
timeout: 3600s
```

### 解説
![](/images/facd673b424ecd/gae-cloudbuild.jpg =400x)

### GAEのマニフェストファイルを環境毎に置換している

### node_modulesをキャッシュすることでデプロイ時間を短縮している

### SecretManager からサーバサイドの秘匿情報をマウントしている

## 詰まったポイント

### 環境変数を CloudBuildに寄せたいが期待通りに動かない

## むすびに

## 参考記事
