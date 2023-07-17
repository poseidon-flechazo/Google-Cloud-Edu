# Cloud Logging
2023/07/17

- Cloud LoggingはGoogle Cloudのサービスのログを収集し、検索や分析を行うためのサービス
- Cloud Runを使用してリクエストログとアプリケーションログを体験

## 1. Cloud Runのリクエストログ
1. expressサービスをデプロイ

`Artifact Registry`に作成したリポジトリ（図のリポジトリは`cloudbuild`）に入って、`express`を選択し、最新のイメージを`Cloud Run`にデプロイする

![express deploy](/image/logging-1.png)

`Cloud Run`で確認できる
![cloud run](/image/logging-2.png)

サービスの詳細のセキュリティの認証種類によってアクセス権が違う
![express service](/image/logging-3.png)
- 未認証の呼び出しを許可を選択した場合
    - 矢印が指されるURLをアクセスできる
    - 安全性低い
- 認証が必要を選択した場合
    - 矢印が指されるURLをクリックしたら`Error:Forbidden`というエラーメッセージで拒否される
    - 自分の認証情報`id token`を加えてリクエストを送る

2. `ロギング`の`ログエクスプローラ`で確認

![logs explorer](/image/logging-4.png)

下記のクエリを入力し、クエリを実行ボタンを押す
```
resource.type = "cloud_run_revision"
resource.labels.service_name = "express"
resource.labels.location = "asia-northeast1"
severity>=DEFAULT
```
クエリ結果について
> ① GET 403
- expressサービスのセキュリティ：認証が必要
- リクエスト：GET
- 結果：403コードで拒否された
> ② HEAD 200
- expressサービスのセキュリティ：認証が必要
- リクエスト：HTTP
- 結果：ヘッダーに`id token`を加えて認証できて、200コードでアクセスできた
> ③ GET 200
- expressサービスのセキュリティ：未認証の呼び出しを許可
- リクエスト：GET
- 結果：200コードでアクセスできた

## 2. Cloud Runのアプリケーションログ

リクエストログは設定なしで収集されるが、アプリケーションログはコードで示す必要がある

1. `index.js`ファイルで`console.log`や`consloe.error`で出力するメッセージを定義
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
2. `console.log`のメッセージをLoggingで確認

![app logging](/image/logging-5.png)
`Example app listening on port 3000`と`Hello World!`が確認できた

### 3. 構造化ロギング

#### ログのメッセージをJSON形式で出力する
```
jsonPayload.foo="bar"
```
![jsonPayload](/image/logging-6.png)

- `severity`フィールド重要度を表す
    - 赤色のびっくりマーク
        - エラー
        - 重大
        - アラート
        - 緊急
    - 黄色のびっくりマーク
        - 警告

#### 親子化
リクエストログとアプリケーションログを紐づけてみる

1. traceIDの確認

Cloud Runはリクエストを受け取って、アプリケーションに転送する際に、`X-Cloud-Trace-Context`というHTTPヘッダをリクエストに自動的に付与する
![trace](/image/logging-7.png)
**trace**の`3b6dcf1231e67ef8f551b0852eae4d3f`はtraceIDと呼ぶ

2. `index.js`ファイルを編集し、再度Cloud Runにデプロイする
```js
const express = require('express')
const app = express()
const port = 3000

const PROJECT_ID = process.env.PROJECT_ID; // 環境変数からPROJECT_IDを取得

app.get('/', (req, res) => {
  const traceHeader = req.headers['x-cloud-trace-context'];
  const [traceId] = traceHeader.split("/"); // 不要な文字を取り除いてtraceIDを取得
  const object = {
    "severity": "ERROR",
    "message": "Hello World!",
    "foo": "bar",
    "key": "value",
    "logging.googleapis.com/trace": `projects/${PROJECT_ID}/traces/${traceId}`, // traceIDを指定
  };
  console.log(JSON.stringify(object));
  res.send('Hello World!')
})

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`)
})
```
3. Loggingでリクエストを確認
![trace](/image/logging-8.png)
`>|`は現在展開しているリクエストログに紐づくアプリケーションログであることを示している
