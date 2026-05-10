# フェーズ4：アプリケーションの実装
 
> ⚠️ おことわり：
> このページは、構築に不慣れなおじさんがAIに質問を繰り返し壁打ちした内容をベースにまとめたものです。
> 偉そうに解説・説明する文面が含まれますが、AI様が勝手に言ってることですので、あらかじめご了承ください。
> なるべく裏付けをとるようにしましたが、正確でない情報が含まれる可能性もあります。
 
| 順番 | 作業内容 |
|------|---------|
| 4-1 | Blob StorageへのアクセスのためにAzureポータルで環境変数を設定する |
| 4-2 | `package.json` にBlob Storage SDKを追加する |
| 4-3 | 「登録」APIを実装する（`register.ts`） |
| 4-4 | 「表示」APIを実装する（`list.ts`） |
| 4-5 | フロントエンド（`index.html`）に登録・表示機能を追加する |
| 4-6 | GitHubにpushしてデプロイ・動作確認する |
| 4-7 | 掲載のソースコードに関する注意事項 |
 
---
 
## フェーズ4の目的
 
フェーズ3で確認した「CI/CDパイプライン」と「Static Web AppsでのWebサイト配信」を土台として、アプリケーションの本機能を実装する。
 
具体的には以下の2つのAPIとフロントエンドの実装を行う。
 
| 機能 | 内容 |
|------|------|
| 登録 | テキストボックスに入力した文字列を、登録日時とセットでBlob Storageに保存する |
| 表示 | Blob Storageに保存されたデータを新着順に5件取得して画面に表示する |
 
---
 
> ⚠️ブランチ運用について：  
> フェーズ3では、devブランチを切ってPR・マージするという一連のブランチ運用を確認した。  
> 本フェーズでは、アプリケーションの機能実装そのものに集中することを優先し、mainブランチに直接コミット・pushする形で進める。  
> 本来であればdevブランチで開発してmainにマージするのが望ましい運用であり、その点はご留意いただきたい。
 
## 4-1：Blob StorageへのアクセスのためにAzureポータルで環境変数を設定する
 
### 接続文字列とは
 
Azure FunctionsからBlob Storageにアクセスするには「接続文字列」が必要である。  
接続文字列にはストレージアカウントのエンドポイントとアクセスキーが含まれており、これをコードに渡すことで接続が可能になる。
 
### 接続文字列の取得
 
1. Azureポータルでストレージアカウント（`myazurestorage`）を開く
2. 左メニューの「セキュリティとネットワーク」→「アクセスキー」を開く
3. `キー1` の「接続文字列」をコピーする
 
> ⚠️ キー1・キー2はどちらも同じ権限・同じ用途。2つあるのはキーのローテーション（定期交換）のためである。どちらを使っても動作は同じ。
 
### 接続文字列の扱いについて
 
接続文字列はストレージアカウントへのフルアクセスが可能な機密情報である。  
コードにハードコードすると、GitHubに公開されてしまうリスクがある。  
そのため、Azureポータルの「環境変数」として設定し、コードからは変数名で参照する方式をとる。
 
| 方法 | 採用 | 理由 |
|------|------|------|
| コードにハードコード | ❌ | GitHubに漏れるリスクがある |
| **Azureポータルの環境変数** | ✅ | コードには書かない。GitHubにも上がらない |
 
### 環境変数の設定手順
 
1. Azureポータルで `myazure-swa` を開く
2. 左メニューの「設定」→「環境変数」を開く
3. 「＋追加」をクリックして以下を入力する
 
| 項目 | 値 |
|------|-----|
| 名前 | `AZURE_STORAGE_CONNECTION_STRING` |
| 値 | 前の手順でコピーした接続文字列 |
 
4. 「適用」をクリックすると環境変数一覧に追加され、さらに「適用」をクリックすると反映される。
 
---
 
## 4-2：`package.json` にBlob Storage SDKを追加する
 
### SDKとは
 
SDKとは、特定のサービスを簡単に操作するためのライブラリ（部品集）のことである。  
`@azure/storage-blob` はAzureが公式に提供するBlob Storage用のTypeScript/JavaScript SDKである。  
このSDKを使うことで、Blob Storageへの読み書きを数行のコードで実現できる。
 
### `dependencies` と `devDependencies` の違い
 
`package.json` には2種類の依存関係の記載箇所がある。
 
| 項目 | 用途 | 本番デプロイ時 |
|------|------|--------------|
| `dependencies` | 本番環境でも実行時に必要なパッケージ | インストールされる |
| `devDependencies` | ビルド・開発時だけ必要なパッケージ | インストールされない |
 
`@azure/storage-blob` はAzure上で実行時に使用するため `dependencies` に追加する。
 
### 変更内容
 
`api/package.json` の `dependencies` に `@azure/storage-blob` を追加する。
 
変更前：
 
```json
"dependencies": {
  "@azure/functions": "^4.0.0"
},
"devDependencies": {
  "typescript": "^5.0.0",
  "@types/node": "^20.0.0"
}
```
 
変更後：
 
```json
"dependencies": {
  "@azure/functions": "^4.0.0",
  "@azure/storage-blob": "^12.0.0"
},
"devDependencies": {
  "typescript": "^5.0.0",
  "@types/node": "^20.0.0"
}
```
 
---
 
## 4-3：「登録」APIを実装する（`register.ts`）
 
### ファイルの配置
 
`api/src/functions/register.ts` を新規作成する。
 
### 処理の流れ
 
```
① フロントエンドからPOSTリクエストを受け取る（リクエストボディにtextが含まれる）
      ↓
② textが空の場合は400エラーを返す
      ↓
③ 環境変数から接続文字列を取得する
      ↓
④ 登録日時（ISO形式）をファイル名としたJSONファイルをBlob Storageに保存する
      ↓
⑤ 登録成功のレスポンスを返す
```
 
### ソースコード
 
```typescript
import { app, HttpRequest, HttpResponseInit, InvocationContext } from '@azure/functions';
import { BlobServiceClient } from '@azure/storage-blob';
 
app.http('register', {
  methods: ['POST'],
  authLevel: 'anonymous',
  handler: async (req: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> => {
 
    const body = await req.json() as { text?: string };
    const text = body?.text;
 
    if (!text || text.trim() === '') {
      return {
        status: 400,
        jsonBody: { message: 'textが空です' }
      };
    }
 
    const connectionString = process.env.AZURE_STORAGE_CONNECTION_STRING;
    if (!connectionString) {
      return {
        status: 500,
        jsonBody: { message: '接続文字列が設定されていません' }
      };
    }
 
    const blobServiceClient = BlobServiceClient.fromConnectionString(connectionString);
    const containerClient = blobServiceClient.getContainerClient('entries');
 
    const now = new Date();
    const blobName = `${now.toISOString()}.json`;
    const content = JSON.stringify({
      text: text,
      registeredAt: now.toISOString()
    });
 
    const blockBlobClient = containerClient.getBlockBlobClient(blobName);
    await blockBlobClient.upload(content, Buffer.byteLength(content), {
      blobHTTPHeaders: { blobContentType: 'application/json' }
    });
 
    return {
      status: 200,
      jsonBody: { message: '登録しました', blobName: blobName }
    };
  }
});
```
 
### ポイント
 
| 項目 | 内容 |
|------|------|
| Blobのファイル名 | 登録日時のISO形式（例：`2026-03-31T12:00:00.000Z.json`） |
| ファイル名を日時にする理由 | 後でファイル名を降順ソートするだけで新着順の並び替えが実現できる |
| 保存形式 | `{ text: "...", registeredAt: "..." }` のJSON |
 
---
 
## 4-4：「表示」APIを実装する（`list.ts`）
 
### ファイルの配置
 
`api/src/functions/list.ts` を新規作成する。
 
### 処理の流れ
 
```
① フロントエンドからGETリクエストを受け取る
      ↓
② 環境変数から接続文字列を取得する
      ↓
③ Blob Storageの `entries` コンテナ内のBlobを一覧取得する
      ↓
④ ファイル名（日時）で降順ソートして新着順に並べる
      ↓
⑤ 上位5件のBlobの中身を読み込む
      ↓
⑥ 5件のデータをレスポンスとして返す
```
 
### ソースコード
 
```typescript
import { app, HttpRequest, HttpResponseInit, InvocationContext } from '@azure/functions';
import { BlobServiceClient } from '@azure/storage-blob';
 
app.http('list', {
  methods: ['GET'],
  authLevel: 'anonymous',
  handler: async (req: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> => {
 
    const connectionString = process.env.AZURE_STORAGE_CONNECTION_STRING;
    if (!connectionString) {
      return {
        status: 500,
        jsonBody: { message: '接続文字列が設定されていません' }
      };
    }
 
    const blobServiceClient = BlobServiceClient.fromConnectionString(connectionString);
    const containerClient = blobServiceClient.getContainerClient('entries');
 
    const blobs: { name: string }[] = [];
    for await (const blob of containerClient.listBlobsFlat()) {
      blobs.push({ name: blob.name });
    }
 
    blobs.sort((a, b) => b.name.localeCompare(a.name));
    const top5 = blobs.slice(0, 5);
 
    const entries = [];
    for (const blob of top5) {
      const blockBlobClient = containerClient.getBlockBlobClient(blob.name);
      const download = await blockBlobClient.downloadToBuffer();
      const content = JSON.parse(download.toString());
      entries.push(content);
    }
 
    return {
      status: 200,
      jsonBody: { entries: entries }
    };
  }
});
```
 
### ポイント
 
| 項目 | 内容 |
|------|------|
| `listBlobsFlat()` | コンテナ内のBlobを一覧取得するSDKのメソッド |
| 降順ソートの方法 | ファイル名がISO形式の日時なので、文字列の降順ソート＝新着順になる |
| `downloadToBuffer()` | BlobをBufferとして読み込むSDKのメソッド |
 
---
 
## 4-5：フロントエンド（`index.html`）に登録・表示機能を追加する
 
フェーズ3で作成した `frontend/index.html` に、登録・表示機能を追加する。  
フェーズ3で動作確認済みの「APIを呼び出す」ボタンは残したまま、機能を追加する。
 
### ソースコード
 
```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>myazure</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <p>テスト</p>
 
  <button id="btn-hello">APIを呼び出す</button>
  <p id="result-hello"></p>
 
  <input type="text" id="input-text" placeholder="テキストを入力">
  <button id="btn-register">登録</button>
  <p id="result-register"></p>
 
  <button id="btn-list">表示</button>
  <ul id="result-list"></ul>
 
  <script>
    document.getElementById('btn-hello').addEventListener('click', async () => {
      const res = await fetch('/api/hello');
      const data = await res.json();
      document.getElementById('result-hello').textContent = data.message;
    });
 
    document.getElementById('btn-register').addEventListener('click', async () => {
      const text = document.getElementById('input-text').value;
      const res = await fetch('/api/register', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text: text })
      });
      const data = await res.json();
      document.getElementById('result-register').textContent = data.message;
    });
 
    document.getElementById('btn-list').addEventListener('click', async () => {
      const res = await fetch('/api/list');
      const data = await res.json();
      const ul = document.getElementById('result-list');
      ul.innerHTML = '';
      for (const entry of data.entries) {
        const li = document.createElement('li');
        li.textContent = `${entry.registeredAt} : ${entry.text}`;
        ul.appendChild(li);
      }
    });
  </script>
</body>
</html>
```
 
---
 
## 4-6：GitHubにpushしてデプロイ・動作確認する
 
### 変更したファイルの確認
 
| ファイル | 変更内容 |
|---------|---------|
| `api/package.json` | `@azure/storage-blob` を追加 |
| `api/src/functions/register.ts` | 新規作成 |
| `api/src/functions/list.ts` | 新規作成 |
| `frontend/index.html` | 登録・表示機能を追加 |
 
### pushとデプロイ
 
VS Codeで変更したファイルをコミットしてmainにpushする。  
pushが完了すると、GitHub Actionsのワークフローが起動してAzureへのデプロイが行われる。  
GitHubの「Actions」タブでビルド・デプロイの成功を確認する。
 
### 動作確認
 
1. ブラウザでSWAのデフォルトURLにアクセスする
2. テキストボックスに文字を入力して「登録」ボタンをクリックする
3. 「登録しました」と表示されることを確認する
4. 数件登録したのち「表示」ボタンをクリックする
5. 新着順に最大5件が表示されることを確認する
6. Azureポータルでストレージアカウント `myazurestorage` を開き、コンテナ `entries` にJSONファイルが保存されていることを確認する
 
---
 
## 4-7：掲載のソースコードに関する注意事項
 
> ⚠️ 注意事項：学習用コードについて  
> 本フェーズで掲載しているソースコードは、AzureのサービスとAPIの動作を理解することを目的とした学習用のものである。  
> 実際の運用に用いる場合は、以下のリスクを認識した上で、適切な対策を講じること。
 
**`register.ts` のリスク**
 
| # | リスク | 内容 |
|---|--------|------|
| 1 | 認証なし | `authLevel: 'anonymous'` のため、誰でもAPIを叩ける |
| 2 | 入力値の検証が不十分 | 文字数上限がないため、極端に長いテキストを送り込める |
| 3 | リクエストの連打 | 短時間に大量リクエストを送ることで、Blob Storageに大量のファイルが作成される |
| 4 | Blob名の衝突 | 同一ミリ秒に複数リクエストが来た場合、同じファイル名で上書きされる可能性がある |
| 5 | 接続文字列の権限が過剰 | ストレージアカウントキーの接続文字列は全操作（読み・書き・削除）が可能 |
 
**`list.ts` のリスク**
 
| # | リスク | 内容 |
|---|--------|------|
| 1 | 認証なし | 同上。誰でもデータを取得できる |
| 2 | 件数の上限がない | Blobが大量にある場合、`listBlobsFlat()` で全件取得してからスライスするため、件数が増えると処理が重くなる |
| 3 | エラーハンドリングなし | Blob読み込み失敗時に例外がそのまま返る |
 
---
 
## まとめ
 
フェーズ4で完了したことは以下の通り。
 
- Azureポータルで環境変数（接続文字列）を設定した
- `package.json` にBlob Storage SDKを追加した
- 登録API（`register.ts`）を実装した
- 表示API（`list.ts`）を実装した
- フロントエンド（`index.html`）に登録・表示機能を追加した
- デプロイ後、登録・表示・Blobへの書き込みをすべて確認した
 
[「フェーズ5：カスタムドメインを設定する」](./azure-setup-phase5.md) へ進む
 
---
作成日：2026年3月 文書名：フェーズ4詳細手順