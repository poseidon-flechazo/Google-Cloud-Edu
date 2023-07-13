# Serverless
2023/07/05

## 1. Cloud Pub/Sub
### Cloud Pub/SubはMessaging Service
- データを発行する側：Publisher
- データを受け取る側：Subscriber

### Cloud Pub/Subの特徴
- グローバルに利用可能
- 低いレイテンシでメッセージを受信可能
- サーバーレスでインフラの考慮が不要

### TopicとSubscriptionの作成
- Topic: Pub/Subで配信されたメッセージを保存する場所
- Subscription: メッセージの配信先
---
#### topicの作成
```bash
gcloud pubsub topics create topic1
```
結果：
```
Created topic [projects/プロジェクト名/topics/topic1].
```
---
#### メッセージの配信
```bash
gcloud pubsub topics publish topic1 --message 'Hello!!!'
```
---
#### Subscriptionの作成
```bash
gcloud pubsub subscriptions create sub1 --topic projects/プロジェクト名/topics/topic1
```
結果：
```
Created subscription [projects/プロジェクト名/subscriptions/sub1].
```
---
#### messageのSubscribe
```bash
gcloud pubsub subscriptions pull --auto-ack projects/プロジェクト名/subscriptions/sub1
```
結果：
```
Listed 0 items.
```
それ、**Subscriberが受信できるメッセージはSubscriptionが作成されてから配信されたメッセージ**

---
#### もうっかいmessageを配信し、Subscriptionしてみる
```bash
gcloud pubsub topics publish topic1 --message 'Celery TeamB!'

gcloud pubsub subscriptions pull --auto-ack projects/プロジェクト名/subscriptions/sub1
```
結果：
```
┌───────────────┬──────────────────┬──────────────┬────────────┬──────────────────┬────────────┐
│      DATA     │    MESSAGE_ID    │ ORDERING_KEY │ ATTRIBUTES │ DELIVERY_ATTEMPT │ ACK_STATUS │
├───────────────┼──────────────────┼──────────────┼────────────┼──────────────────┼────────────┤
│ Celery TeamB! │ 8636672474356578 │              │            │                  │ SUCCESS    │
└───────────────┴──────────────────┴──────────────┴────────────┴──────────────────┴────────────┘
```
ナイス！Subscribeできた

---
## 2. GCSのPubSub Notification
Pub/Sub Notificationはパケットに対して、新しいオブジェクト（ファイル）と新規作成したり、削除したりしたら、通知してくれる機能
### notificationsを有効化
```bash
gcloud storage buckets notifications create gs://バケット名 \
    --topic=projects/プロジェクト名/topics/topic1
```
警告出ても大丈夫

---
### オブジェクトをアップロードしてsubcriptionからメッセージを確認
```bash
touch notification.txt
gcloud storage cp notification.txt gs://バケット名

gcloud pubsub subscriptions pull --auto-ack projects/プロジェクト名/subscriptions/sub1
```
結果：
```
┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬──────────────────┬──────────────┬──────────────────────────────────────────────────────────────────────────────────┬──────────────────┬────────────┐
│                                                                         DATA                                                                         │    MESSAGE_ID    │ ORDERING_KEY │                                    ATTRIBUTES                                    │ DELIVERY_ATTEMPT │ ACK_STATUS │
├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────┼──────────────┼──────────────────────────────────────────────────────────────────────────────────┼──────────────────┼────────────┤
│ {                                                                                                                                                    │ 8626015785226059 │              │ bucketId=プロジェクト名-test                                                    │                  │ SUCCESS    │
│   "kind": "storage#object",                                                                                                                          │                  │              │ eventTime=2023-07-04T16:10:56.860421Z                                            │                  │            │
│   "id": "プロジェクト名-test/notification.txt/1688487056845920",                                                                                    │                  │              │ eventType=OBJECT_FINALIZE                                                        │                  │            │
│   "selfLink": "https://www.googleapis.com/storage/v1/b/プロジェクト名-test/o/notification.txt",                                                     │                  │              │ notificationConfig=projects/_/buckets/プロジェクト名-test/notificationConfigs/1 │                  │            │
│   "name": "notification.txt",                                                                                                                        │                  │              │ objectGeneration=1688487056845920                                                │                  │            │
│   "bucket": "プロジェクト名-test",                                                                                                                  │                  │              │ objectId=notification.txt                                                        │                  │            │
│   "generation": "1688487056845920",                                                                                                                  │                  │              │ payloadFormat=JSON_API_V1                                                        │                  │            │
│   "metageneration": "1",                                                                                                                             │                  │              │                                                                                  │                  │            │
│   "contentType": "text/plain",                                                                                                                       │                  │              │                                                                                  │                  │            │
│   "timeCreated": "2023-07-04T16:10:56.860Z",                                                                                                         │                  │              │                                                                                  │                  │            │
│   "updated": "2023-07-04T16:10:56.860Z",                                                                                                             │                  │              │                                                                                  │                  │            │
│   "storageClass": "STANDARD",                                                                                                                        │                  │              │                                                                                  │                  │            │
│   "timeStorageClassUpdated": "2023-07-04T16:10:56.860Z",                                                                                             │                  │              │                                                                                  │                  │            │
│   "size": "0",                                                                                                                                       │                  │              │                                                                                  │                  │            │
│   "md5Hash": "1B2M2Y8AsgTpgAmY7PhCfg==",                                                                                                             │                  │              │                                                                                  │                  │            │
│   "mediaLink": "https://storage.googleapis.com/download/storage/v1/b/プロジェクト名-test/o/notification.txt?generation=1688487056845920&alt=media", │                  │              │                                                                                  │                  │            │
│   "crc32c": "AAAAAA==",                                                                                                                              │                  │              │                                                                                  │                  │            │
│   "etag": "CODY7Lm49f8CEAE="                                                                                                                         │                  │              │                                                                                  │                  │            │
│ }                                                                                                                                                    │                  │              │                                                                                  │                  │            │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴──────────────────┴──────────────┴──────────────────────────────────────────────────────────────────────────────────┴──────────────────┴────────────┘
```
---
## 3. Cloud RUN
コンテナイメージをデプロイするだけで、HTTPリクエストを受け付けるWebサービスを作成できるサービス
### コンテナアプリケーションの作成
`nginx`は公開されているコンテナイメージで、注目して欲しいのがポート番号は$80$番
```bash
gcloud run deploy nginx 
 --image=nginx:stable-alpine 
 --platform=managed 
 --port=80 
 --region=asia-northeast1
 --allow-unauthenticated
```
結果：
```
Deploying container to Cloud Run service [nginx] in project [プロジェクト名] region [asia-northeast1]
OK Deploying... Done.                                                                                                                                        
  OK Creating Revision...                                                                                                                                    
  OK Routing traffic...                                                                                                                                        OK Setting IAM Policy...                                                                                                                                   Done.                                                                                                                                                        Service [nginx] revision [nginx-00002-gul] has been deployed and is serving 100 percent of traffic.
Service URL: https://nginx-fxmln72kfq-an.a.run.app
```
---
> オプションの補足
>- --cpu, --memory
>- --min-instances, --max-instances
>   - 一台もインスタンスを起動しない`ゼロスケール`が可能
>- --set-env-vars, --env-var-file
---
### Artifact Registryを利用したコンテナイメージの管理
- Artifact Registryとrepositryとは
  - Artifact registry：レジストリはコンテナイメージを保存・管理する**サービス**
    - Container Registryとの違い：Artifact RegistryはContainer Registryの拡張としてコンテナイメージとコンテナ以外のアーティファクトもサポートできる
    - 他のサービス
      - Docker Hub
      - Quay.ioなど
  - repositry：リポジトリは`nginx`などの**イメージが保管される場所**
    - Rigistryというサービスの中に、それぞれのイメージの集まりはrepositryにある
    - GitHubのrepositryはイメージしやすい

1. repositryを作成
```bash
gcloud artifacts repositories create リポジトリ名 --repository-format=docker --location=asia-northeast1
```
結果：
```
Create request issued for: [リポジトリ名]
Waiting for operation [projects/プロジェクト名/l
ocations/asia-northeast1/operations/07372454-365d
-4b01-91ac-919a90d098a8] to complete...done.
Created repository [リポジトリ名].
```
---
2. レジストリから`nginx:stable-alpine`イメージを取得（初めての時）

`nginx`は公開されているので、gitと同様に`pull`で取ってくる
```bash
docker pull nginx:stable-alpine
```
> ローカル環境にはDockerがインストールされないため、CloudShellで実行

結果：
```
stable-alpine: Pulling from library/nginx
4db1b89c0bd1: Pull complete 
6f8beccece3b: Pull complete 
efba416b8a87: Pull complete 
6e7bc8944b52: Pull complete 
72805f9582fb: Pull complete 
4c6615db462e: Pull complete 
d799d200ba56: Pull complete 
Digest: sha256:5e1ccef1e821253829e415ac1e3eafe46920aab0bf67e0fe8a104c57dbfffdf7
Status: Downloaded newer image for nginx:stable-alpine
docker.io/library/nginx:stable-alpine
```
---
3. `docker tag`で名前変更

コンテナのイメージ名は`LOCATION-docker.pkg.dev/プロジェクト名/リポジトリ名/イメージ名:タグ`というフォーマットで指定する必要があるので、`tag`でイメージ名を変更する

```bash
docker tag nginx:stable-alpine asia-northeast1-docker.pkg.dev/プロジェクト名/リポジトリ/nginx:stable-alpine
```
> ローカル環境にはDockerがインストールされないため、CloudShellで実行

---
4. 認証

dockerコマンドがGoogle Cloudに認証するため
```bash
gcloud auth configure-docker asia-northeast1-docker.pkg.dev
```
結果：
```
Adding credentials for: asia-northeast1-docker.pkg.dev
```
---
5. Docker Run
```bash
docker run -d -p 127.0.0.1:8080:80 nginx:stable-alpine
```
- `-d`はコンテナーをバックグラウンド実行し、コンテナーIDを出力するって意味
- `-p`はコンテナのポート`8080`を`127.0.0.1`上の`80`ボートにバインド（割り当て）するって意味

> ローカル環境にはDockerがインストールされないため、CloudShellで実行

結果：
```
df0b7c7003a45d75cd32a782eee8d4eccdd49f59100df784e38d3c6a9dc29791
```
長い謎文字列はコンテナID

---
6. ローカルホストでアクセスできるか確認
```bash
curl localhost:8080
```
---
7. push

Artifact Registryへイメージをpushする準備を完了、pushする
```bash
docker push asia-northeast1-docker.pkg.dev/プロジェクト名/リポジトリ名/nginx:stable-alpine
```
> ローカル環境にはDockerがインストールされないため、CloudShellで実行
---

### Serviceの認証と公開範囲
デフォルトで認証なしで公開されるのはよくないが、ユーザ認証と交換範囲を指定すれば良い

1. サービスの公開アクセス設定を取り除き
```bash
gcloud run services remove-iam-policy-binding nginx \
    --member=allUsers \
    --role=roles/run.invoker \
    --region=asia-northeast1
```
結果：
```
Updated IAM policy for service [nginx].
etag: BwYALzvMKW4=
version: 1
```
2. 自分のアカウントを許可
```bash
gcloud run services add-iam-policy-binding nginx \
    --member='user:xxx@xxx.com' \
    --role='roles/run.invoker' \
    --region=asia-northeast1
```
結果：
```
Updated IAM policy for service [nginx].
bindings:
- members:
  - user:xxx@xxx.com
  role: roles/run.invoker
etag: BwYAL0wlqSw=
version: 1
```

3. 確認

普通に`curl`してリクエストしてもアクセスできないはず
```bash
curl -I https://nginx-自分のサービスURL.a.run.app/
```
結果：
```
HTTP/2 403 
date: Tue, 11 Jul 2023 05:38:09 GMT
content-type: text/html; charset=UTF-8
server: Google Frontend
content-length: 295
```
4. id tokenを加えてリクエスト

自分の認証情報を加えるとアクセスできる
```bash
curl -I -H "Authorization: Bearer $(gcloud auth print-identity-token)" https://nginx-fxmln72kfq-an.a.run.app
```
- `gcloud auth print-identity-token`で取ってくる文字列はid token
- 最も広く使われているJSON形式のフォーマットは**jwt**と書いて「ジョット」と読む

結果：
```
HTTP/2 200 
content-type: text/html
last-modified: Tue, 11 Apr 2023 17:21:57 GMT
etag: "64359735-267"
accept-ranges: bytes
x-cloud-trace-context: 1d3cabb28975c5a8f254992e7a53daf4;o=1
content-length: 615
date: Tue, 11 Jul 2023 05:39:30 GMT
server: Google Frontend
```

### Severless Network Endpoint Group
Cloud RunのサービスをGCLBのBackend Serviceとして利用可能

1. NEGの作成
```bash
gcloud compute network-endpoint-groups create nginx-serverless-neg \
    --region=asia-northeast1 \
    --network-endpoint-type=serverless  \
    --cloud-run-service=nginx
```
結果：
```
Created [https://www.googleapis.com/compute/v1/projects/プロジェクト名/regions/asia-northeast1/networkEndpointGroups/nginx-serverless-neg].
Created network endpoint group [nginx-serverless-neg].
```
---
2. backend-serviceの作成
```
gcloud compute backend-services create nginx-backend-service \
    --global \
    --load-balancing-scheme=EXTERNAL_MANAGED
```
結果：
```
Created [https://www.googleapis.com/compute/v1/projects/プロジェクト名/global/backendServices/nginx-backend-service].
NAME                   BACKENDS  PROTOCOL
nginx-backend-service            HTTP
```
