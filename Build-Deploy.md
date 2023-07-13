# Cloud Build & Cloud Deploy
2023/07/12

## 1. 前提知識
### CI/CDパイプライン
一文でいうと、プロジェクトで作業をトリガーする際に実行する一連のプロセスの集まり
- CI: 継続的インテグレーション
    - plan→code→build→test
- CD: 継続的デプロイメント
    - release→deploy→(hold)→operate→monitor

**Cloud BiuldとCloud DeployはCI/CDの二つの部分と理解して良い**

### yamlファイル
- yamlとは構造的なデータ集合を文字列として表記する
- ソフトウェアの設定ファイル、異なる間のデータ交換などで使われる

## 2. Cloud Build
### Node.jsアプリケーションのビルド
1. 必要な準備（四つのファイル）
    - Node.jsプロジェクト
        - index.js
        - package.json：パッケージの情報やバーションなど
    - Dockerfile
    - cloudbuild.yaml：ビルド構成ファイル

`indxe.js`
```js
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  const object = {
    "severity": "ERROR",
    "message": "Hello World!",
    "foo": "bar",
    "key": "value",
  };
  console.log(JSON.stringify(object));
  res.send('Hello World!');
});

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`);
});
```
`package.json`
```json
{
  "name": "cloudbuild",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "dependencies": {
    "express": "^4.18.2"
  }
}
```
`Dockerfile`
```Dockerfile
FROM node:16-alpine

COPY index.js package.json /app/

WORKDIR /app

RUN yarn install

ENTRYPOINT [ "node" ]

CMD [ "/app/index.js" ]
```

Cloud Buildの構成ファイル`cloudbuild.yaml`
```yaml
steps:
# 依存関係をインストール
- name: 'gcr.io/cloud-builders/yarn'
  args: ['install']

# イメージをビルドする
- name: 'gcr.io/cloud-builders/docker'
  id: 'build'
  args: ["build", "-t", "asia-northeast1-docker.pkg.dev/${PROJECT_ID}/リポジトリ名/イメージ名express", "-f", "Dockerfile", "."]

# registryへイメージをpush
- name: 'gcr.io/cloud-builders/docker' 
  id: 'push'
  args: ["push", "asia-northeast1-docker.pkg.dev/${PROJECT_ID}/リポジトリ名/イメージ名express"]
```
- `name`フィールドはタスクを実行するコンテナイメージを指定する
    - `gcr.io/cloud-builders/docker`はContainer Registryに格納される事前ビルド済みDockerイメージ（Google Cloudで管理されてる）
- `args`フィールドはステップの引数を追加する
    - `Dockerfile`とソースコードが異なるディレクトリにある場合、`-f`でDockerfileのpathを追加
    ```yaml
    - name: 'gcr.io/cloud-builders/docker'
    args: [ 'build', '-t', 'LOCATION-docker.pkg.dev/PROJECT_ID/REPOSITORY/IMAGE_NAME', '-f', 'DOCKERFILE_PATH', '.' ]
    ```
2. ビルド開始
```bash
gcloud builds submit --config cloudbuild.yaml
```
---

### Cloud Buildのトリガー
1. GitHubリポジトリに接続(トリガーのWeb UIで)
- asia-northeast1(東京)リージョン、Github(Cloud Build GitHubアプリ)
- 認証
- リポジトリを選択

> 注意点：
>- GitHubリポジトリの下に`Dockerfile`、`cloudbuild.yaml`とソースコードを用意する
>- GitHubリポジトリにフォルダ構造を作成する場合、`Dockerfile`、`cloudbuild.yaml`の位置を指定する必要がある

---
2. トリガーの作成
```bash
gcloud builds triggers create github \
    --repo-name=GitHubのリポジトリ名 \
    --repo-owner=オナー \
    --branch-pattern="^main$" \
    --build-config=cloudbuild.yaml \
    --region=asia-northeast1
```
結果：
```
Created [https://cloudbuild.googleapis.com/v1/projects/プロジェクト名/locations/asia-northeast1/triggers/5d544a6c-87c4-4294-8d79-c97a5d747988].
NAME     CREATE_TIME                STATUS
trigger  2023-07-12T08:05:01+00:00
```
---

## 3. Cloud Deploy 
1. `clouddeploy.yaml`を作成
```yaml
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: deploy-pipeline
description: main application pipeline
serialPipeline:
  stages:
  - targetId: run-express-dev
    profiles: [dev] # devというプロファイルを適用するDeliveryPipelineを定義
---

apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: run-express-dev
description: Cloud Run development service
run:
  location: projects/PROJECT_ID/locations/asia-northeast1
```
---
2. このマニフェストファイルを適用
```bash
gcloud deploy apply --file=clouddeploy.yaml --region=asia-northeast1
```
結果：
```
Created Cloud Deploy resource: projects/プロジェクト名/locations/asia-northeast1/targets/run-express-dev.
```
---
3. `skaffold.yaml`を作成
```yaml
apiVersion: skaffold/v3alpha1
kind: Config
metadata:
  name: deploy-run-quickstart
profiles:
- name: dev # devプロファイルがservice.yamlを利用することを定義
  manifests:
    rawYaml:
    - service.yaml
deploy:
  cloudrun: {}
```
---
4. `service.yaml`を作成
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: express
  labels:
    cloud.googleapis.com/location: asia-northeast1 # リージョンを指定
  annotations:
    run.googleapis.com/ingress: all
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: '0'  # インスタンスの最小数
        autoscaling.knative.dev/maxScale: '10' # インスタンスの最大数
        run.googleapis.com/execution-environment: gen2 # 第2世代のインスタンスを利用
    spec:
      containers:
      - image: asia-northeast1-docker.pkg.dev/PROJECT_ID/REPOSITORY_NAME/express:latest
        ports:
        - containerPort: 3000 # コンテナのポート番号
```
5. release
```bash
gcloud deploy releases create RELEASE_NAME --delivery-pipeline=deploy-pipeline --region=asia-northeast1 --skaffold-file=skaffold.yaml
```
