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

動作確認環境
`nodejs version 15.14.0`
`nuxt version 2.15.7`


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
[クイックスタート: Google Cloud CLI をインストールする  \|  Cloud SDK のドキュメント](https://cloud.google.com/sdk/docs/install-sdk)
Google Cloud CLIをインストールする必要があります。

* `automatic_scaling` の詳細については [app\.yaml リファレンス  \|  Python 2 の App Engine スタンダード環境  \|  Google Cloud](https://cloud.google.com/appengine/docs/standard/python/config/appref#automatic_scaling) に詳しい記載があるので、こちらを参考にしてください。
`target_cpu_utilization`, `target_throughput_utilization` によって インスタンスをスケールさせるCPUの閾値を設定しています。

* `handlers` については公式ドキュメントに以下のように説明が記載されています。

    > handlers 要素は、app.yaml 構成ファイルの必須要素です。この要素は、URL パターンと処理方法の説明リストを提供します。App Engine で URL を処理するには、アプリケーション コードを実行するか、画像、CSS、JavaScript など、コードと一緒にアップロードされた静的ファイルを提供します。

    詳細については
    [app\.yaml リファレンス  \|  Python 2 の App Engine スタンダード環境  \|  Google Cloud](https://cloud.google.com/appengine/docs/standard/python/config/appref#handlers_element)
    に詳しい記載があるので、こちらを参考にしてください。

* NuxtJSの handlersに関して
    [Nuxt \- Google App Engine](https://nuxtjs.org/deployments/google-appengine/) こちらを参考に作成しました。

## CloudBuild から GAEにアプリケーションをデプロイする

### ビルド構成ファイルの作成の作成

CloudBuildでデプロイを動かすために必要な
構成ファイルを作成します。
`json` または `yaml` 形式ををサポートしていますが
今回は `yaml` で作成します。

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

* cloudbuildの構成
基本的なビルド構成ファイルは下記のようなstepsフィールドからなります。

```yaml
- name: 'node:16' # タスクを実行するコンテナイメージ
args: - install # 引数のリスト
id: install dep # ステップのID
entrypoint: npm # 指定がない場合は args の最初の要素 or コンテナエントリポイント
```

```yaml
timeout: 3600s
```

のような追加のビルド構成フィールドを含めることができます。

[公式ドキュメント](https://cloud.google.com/build/docs/configuring-builds/create-basic-configuration) 詳細はこちら

* GAEのマニフェストファイルを環境毎に置換する

```yaml
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
```

    CloudBuildトリガーの環境変数(後述)を `sed` で app.yamlに埋め込んでいます。
    _SERVICE_NAME を変数として扱うことで複数環境を用意できます。
    テスト環境が複数必要という場合に役立っています。

    _INSTANCE_CLASS,　_MIN_INSTANCES, _MAX_INSTANCESは
    環境毎に要求されるパフォーマンスとコストの要件を満たすため変数として埋め込んでいます。

* node_modulesをキャッシュすることでデプロイ時間を短縮している

```yaml
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
# ~~~ デプロイ
---　
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
```

    デプロイ後 `node_modules` を zip化したものを
    gcpのバケットに保存しています。
    デプロイ実行時に 解凍し node_modulesに再配置しています。
    `package.json` に変更がある場合は追加でライブラリのインストールが
    行われます。
    導入前と比べデプロイにかかる時間が 3分半 ~ 4分程度短縮されました。

    作成にあたり下記の記事を参考にさせていただきました。
    [Cloud Buildでnode\_modulesをキャッシュしてビルド時間を高速化する](https://sunday-morning.app/posts/2021-04-07-cloud-build-cache-node-modules)

    [CloudBuildで特定のディレクトリをcacheして高速化する \- Qiita](https://qiita.com/moyashidaisuke/items/777b543c0d8a7a35a731)


* SecretManager からサーバサイドの秘匿情報をマウントしている

```yaml
- name: gcr.io/cloud-builders/gcloud
args:
    - '-c'
    - |
    gcloud secrets versions access latest --secret $_SECRET_NAME >> file_path/hoge
id: mount secret
```


* 環境変数を CloudBuildに寄せたいが期待通りに動かない問題

デプロイ時に環境変数がうまく渡らずちょっと詰まりました。
`.env`ファイルを環境毎に読ませて build コマンドを環境毎に用意する方法もありましたが、
サーバーサイドで使う環境変数が出てきたら別途保管しないといけないと思い
CloudBuildに寄せることにしました。
結果 `nuxt.config.js` の修正とマニフェストを以下の通り変更したら
期待通り動くようになりました。
CloudBuildトリガーの環境変数 と `npm run build` 実行時の環境変数は一致しないようです。

```javascript:nuxt.config.js
buildModules: {
    [
        '@nuxtjs/dotenv',
        { systemvars: process.env.NODE_ENV === 'production'}
    ],
}
```

```yaml
env:
    - NUXT_ENV_BASE_URL=$_NUXT_ENV_BASE_URL
    - NUXT_ENV_ENV_TYPE=$_NUXT_ENV_ENV_TYPE
```

### CloudBuildの設定

* トリガーの作成 + 環境変数を設定する
* サービスアカウントの権限を設定する

![](/images/facd673b424ecd/gae-cloudbuild.jpg =400x)

## むすびに

立ち上げたばかりなので荒削りで荒削りな部分も散見しますが、
同じように NuxtJSをGAEにデプロイしようとしている人の助けになればと
一緒に

