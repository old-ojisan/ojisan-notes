# フェーズ3：デプロイおよびStatic Web Appsの動作確認
> ⚠️ おことわり：
このページは、構築に不慣れなおじさんがAIに質問を繰り返し壁打ちした内容をベースにまとめたものです。
偉そうに解説・説明する文面が含まれますが、AI様が勝手に言ってることですので、あらかじめご了承ください。
なるべく裏付けをとるようにしましたが、正確でない情報が含まれる可能性もあります。

| 順番 | 作業内容 |
|------|---------|
| 3-1 | リポジトリ構成とローカルへのclone |
| 3-2 | 動作確認用のフロントエンド（HTML/CSS）のファイルを作成する |
| 3-3 | 動作確認用のAzure Functions（TypeScript）のファイルを作成する |
| 3-4 | GitHubのCI/CDワークフローの確認を行う |
| 3-5 | GitHubにpushしてCI/CDが正常に動作することを確認する |
| 3-6 | AzureのデフォルトURLでWebサイトの動作を確認する |
| 3-7 | GitやGitHubのブランチの基礎知識(参考) |
| 3-8 | AzureのGitHub連携で、ブランチを利用したCI/CDを活用する |



---

## フェーズ3の目的

フェーズ3では、アプリケーションの本格実装（Blob Storageへの読み書きなど）は行わない。  
目的は「CI/CDパイプラインが正しく機能すること」と「Static Web Apps(SWA)でWebサイトが配信されること」の2点を確認することである。

そのため、フロントエンドとAzure FunctionsはいずれもシンプルなHello World相当の内容にとどめる。

---

## 3-1. リポジトリ構成とローカルへのclone　

### リポジトリ構成
 
フェーズ1でリポジトリ`myazure`の作成、フェーズ2でSWAの作成を行った。  
今後、リポジトリ / ディレクトリは以下の構成で進めていく。
```
~/projects/
└─── myazure/
    ├── .gitignore     ← リポジトリ作成時に自動配置済み
    ├── .github/
    │   └── workflows/ ← SWA作成時に自動配置済み
    ├── frontend/      ← フロントエンド（HTML・CSS）※本フェーズで作成
    ├── api/           ← Azure Functions（TypeScript）※本フェーズで作成
    └── README.md      ← リポジトリ作成時に自動配置済み
```

### リポジトリのclone
作業を行うMacBookのローカルにcloneする。

VS Codeを起動し、ターミナルで作業ディレクトリを作成する。

```bash
mkdir -p ~/projects
```

作業ディレクトリ配下に、リポジトリ`myazure`をクローンする、

```bash
cd ~/projects
git clone git@github.com:<GitHubのユーザー名>/myazure.git
cd ~/projects/myazure
ls -la
```
`.git`、`.gitignore`、`README.md`が表示されればcloneは完了している。  

完了したら、VS Codeで`~/projects/myazure`フォルダを開く。  


### ブランチの進め方
ソースの作成とビルドおよびデプロイを行い、AzureでWebサイトが正常動作するかの確認を優先する。  
それまでは、開発(dev)ブランチは作成せず本番(main)で作業を進めることとする。一通りの動作が確認できたのち、開発(dev)ブランチに対応する環境を立ち上げる。


## 3-2：動作確認用のフロントエンド（HTML/CSS）のファイルを作成する

### 作成するファイルの配置

フェーズ1で決定したディレクトリ構成に従い、フロントエンドのファイルは `frontend/` ディレクトリ以下に配置する。

```
myazure/
├── .github/
│   └── workflows/           # GitHub Actionsワークフロー
└── frontend/
    ├── index.html
    └── style.css
```

### 作業手順

1. VS Codeのエクスプローラーで `frontend/` ディレクトリを作成する

2. `frontend/` 以下に `index.html` を新規作成し、以下の内容を記述する

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
  <button id="btn">APIを呼び出す</button>
  <p id="result"></p>
  <script>
    document.getElementById('btn').addEventListener('click', async () => {
      const res = await fetch('/api/hello');
      const data = await res.json();
      document.getElementById('result').textContent = data.message;
    });
  </script>
</body>
</html>
```

3. `frontend/` 以下に `style.css` を新規作成し、下記の簡単な内容を記述する

```css
button {
  padding: 8px 16px;
  cursor: pointer;
}
```

### ポイント：`/api/hello` について

`index.html` 内のボタンをクリックすると `/api/hello` というURLにリクエストが送られる。  
SWAでは、`/api/` 以下のパスは自動的に組み込みのAzure Functionsにルーティングされる。  
そのため、フロントエンドからAPIを呼び出す際にドメインやポート番号を意識する必要がない。

---

## 3-3：動作確認用のAzure Functions（TypeScript）のファイルを作成する

### Azure Functionsプロジェクトとは

Azure Functionsのプロジェクトは、複数の「関数」をまとめて管理するための単位である。  
SWAに組み込むAzure Functionsは、`api/` ディレクトリ以下にプロジェクトを作成する。

### 作成するファイルの配置

```
myazure/
└── api/
    └─ src
    │ └─ functions/
    │     └─ hello.ts          ← 関数本体（TypeScript）
    ├── host.json              ← Functions全体の設定
    ├── package.json           ← Node.jsプロジェクト設定
    └── tsconfig.json          ← TypeScriptコンパイル設定
```


### 作業手順

1. VS Codeのエクスプローラーで `api/` ディレクトリを作成する

2. `api/src/functions`を作成、そこに`hello.ts` を新規作成し、以下の内容を記述する

```typescript
import { app, HttpRequest, HttpResponseInit, InvocationContext } from "@azure/functions";

export async function hello(request: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> {
    context.log('hello function processed a request.');

    return {
        status: 200,
        jsonBody: {
            message: "Hello from Azure Functions!"
        }
    };
}

app.http('hello', {
    methods: ['GET', 'POST'],
    authLevel: 'anonymous',
    handler: hello
});
```


3. `api/` 以下に `host.json` を新規作成し、以下の内容を記述する

- ポイント
  - 最小構成ならこれだけで十分

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  }
}
```

4. `api/` 以下に `package.json` を新規作成し、以下の内容を記述する
- ポイント
  - TypeScript をビルドして JS に変換するための設定
  - npm run build で TypeScript → JavaScript に変換
  - GitHub Actions でもこのコマンドを実行する

```json
{
  "name": "myazure-api",
  "version": "1.0.0",
  "description": "Azure Functions for myazure",
  "main": "dist/src/functions/*.js",
  "scripts": {
    "build": "tsc",
    "watch": "tsc -w",
    "prestart": "npm run build",
    "start": "func start"
  },
  "dependencies": {
    "@azure/functions": "^4.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0"
  }
}

```

5. `api/` 以下に `tsconfig.json` を新規作成し、以下の内容を記述する
- ポイント
  - `outDir: "dist"` → ビルド後の JS が `api/dist` に出る
  - SWA の GitHub Actions は `api` を指定すれば自動で拾う

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "ES2020",
    "outDir": "dist",
    "rootDir": ".",
    "strict": false,
    "esModuleInterop": true
  }
}
```

6. 本来ならこの段階でローカルでビルドし、動作確認を行うタイミングであるが、本文書ではそのまま次のCI/CDのワークフローの作成に進む。

### 作業後のファイル構成
```
myazure/
├── .github/
│   └── workflows/           # GitHub Actionsワークフロー
├── api/                     # Azure Functions（TypeScript）
│   └─ src
│   │ └─ functions/
│   │     └─ hello.ts
│   ├── host.json
│   ├── package.json
│   └── tsconfig.json
│
└── frontend/                # フロントエンド（HTML, CSS）
　   ├── index.html
 　  └── style.css
 ```

### 補足：`authLevel: 'anonymous'` について

Azure Functionsのエンドポイントには認証レベルを設定できる。

| 認証レベル | 内容 |
|-----------|------|
| `anonymous` | 認証不要。誰でもAPIを呼び出せる |
| `function` | 関数キーが必要 |
| `admin` | マスターキーが必要 |

本フェーズでは動作確認用のため `anonymous` を使用する。本番実装時は適切な認証レベルを検討する。

---

## 3-4：GitHubのCI/CDワークフローの確認を行う

フェーズ2でSWAを作成した際に以下の2つがリポジトリに追加されていることを確認した。

| 確認項目 | 状況 |
|---|---|
| yml追加 | .github/workflows/azure-static-web-apps-<固有URL>.yml |
| Secrets追加 | AZURE_STATIC_WEB_APPS_API_TOKEN_<固有名> |


### ワークフローファイル(yml)の中身を確認する

VS Codeで `.github/workflows/azure-static-web-apps-<固有名>.yml` を開いて内容を確認しておく。

```yaml
name: Azure Static Web Apps CI/CD

on:
  push:
    branches:
      - main
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches:
      - main

jobs:
  build_and_deploy_job:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.action != 'closed')
    runs-on: ubuntu-latest
    name: Build and Deploy Job
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: true
          lfs: false
      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_<固有名> }}

          # Used for Github integrations (i.e. PR comments)
          repo_token: ${{ secrets.GITHUB_TOKEN }} 
          action: "upload"
          ###### Repository/Build Configurations
          app_location: "/frontend" # App source code path
          api_location: "/api" # Api source code path - optional
          output_location: "" # Built app content directory - optional


  close_pull_request_job:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    name: Close Pull Request Job
    steps:
      - name: Close Pull Request
        id: closepullrequest
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_<固有名> }}
          action: "close"
```
Azureが追加したSecretsは、ymlファイルの以下で呼び出されている。

```yaml
azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_<固有名> }}
```

### ワークフローファイルの処理の概要について

- mainにコミットされると、ビルドとAzureへのデプロイが行われる
- mainに対してPull Requestされると、ビルドとAzureへのデプロイが行われる
  - opened/synchronize/reopened の場合のみ、closeは対象外
  - すでに稼働中のmainとは別に、確認用のURLが新たに生成される
  - Stagingの確認に便利
- mainに対してPull Requestがcloseさせると確認用のURLの後片付けが行われる


---

## 3-5：GitHubにpushしてCI/CDが正常に動作することを確認する

### 除外ファイルの設定 `.gitignore` の確認

各フェーズを通してローカルでビルドは行わないため、現時点では特に気にする必要はない。  


### 作成した全てのファイルをGitHubにpushする

> ⚠️注意：  
> ここでGitHubに本フェーズで作成した全ファイルがpushされる。  
> pushが完了すると、GitHub Actionsでビルドされ、Azureへのデプロイが行われる。


VS Codeで全てのファイル作成が完了したら、コミットした上で、全ファイルをGitHubにpushする。
- **ポイント** (特に、この文書、この構成におけるポイントとして)
  - ブランチを切っていないため、mainブランチに対してpushされる。   
  - pushの後、GitHub Actionsのワークフロー(yml)の処理が始まる
    - ビルド
    - Azureへのデプロイ


### GitHub Actionsの実行を確認する

1. GitHubで `myazure` リポジトリを開く
2. 「Actions」タブを開く
3. 「Azure Static Web Apps CI/CD」ワークフローが実行中または完了していることを確認する
4. ワークフローをクリックして詳細を確認する
5. ワークフローが緑のチェックマークで完了していれば成功(完了まで1-2分かかる場合がある)

**失敗した場合**

ワークフローが赤いバツマークで失敗した場合は、ジョブをクリックして各ステップのログを確認する。  
よくある原因は以下の通り。

| 原因 | 対処 |
|------|------|
| シークレット名が一致しない | GitHubシークレットの名前とワークフローファイルの記述を照合する |
| `app_location` や `api_location` が誤っている | ワークフローファイルのパス設定とディレクトリ構成を照合する |
| TypeScriptのビルドエラー | ログのエラーメッセージを確認してコードを修正する |

---

## 3-6：AzureのデフォルトURLでWebサイトの動作を確認する

### デフォルトURLの確認

Static Web AppsのデフォルトURLはAzureポータルで確認できる。

1. Azureポータルで `myazure-swa` リソースを開く
2. 「概要」ページの「URL」欄に表示されている

### 動作確認

1. ブラウザでデフォルトURLにアクセスする
2. 「テスト」という文字と「APIを呼び出す」ボタンが表示されることを確認する
3. 「APIを呼び出す」ボタンをクリックする
4. 「Hello from Azure Functions!」というテキストがボタンの下に表示されれば成功

---


## 3-7. GitやGitHubのブランチの基礎知識(参考) 

> おことわり：  
> 参考のため、ご存じの方は読み飛ばして次のセクションへ。  



### ブランチの概念

| 項目 | 内容 |
|------|------|
| ブランチとは | コードの分岐した作業ライン |
| 「ブランチを切る」とは | 現在いるブランチから新しいブランチを作ること |
| どこから切られるか | **コマンド実行時に自分がいるブランチ**から切られる |


### ブランチ構成（3環境の場合）

次のように、3つの階層で行われることもある。

```
main（Production）
  └── staging（Staging）
        └── dev（Development）
```

| ブランチ | 環境 | 役割 |
|---------|------|------|
| `main` | Production | 本番環境 |
| `staging` | Staging | 本番前の最終確認環境 |
| `dev` | Development | 日常の開発作業 |


### 変更の流れ
```
dev → (PR) → staging → (PR) → main
```

- `dev` での作業を `staging` にPRしてマージ
- `staging` で確認後、`main` にPRしてマージ


### ブランチを作るためのgitコマンドの例
```bash
# 上流から順番に切る
git switch main          #mainに切り替え
git switch -c staging    #mainからstagingを作って切り替える
git push -u origin staging  #stagingをクローン元(GitHub)にpushする

git switch -c dev        #stagingにいるので、そこからdevを作って切り替える
git push -u origin dev      #devをクローン元(GitHub)にpushする
```

| コマンド | 意味 |
|---------|------|
| `git switch ブランチ名` | ブランチを切り替える |
| `git switch -c ブランチ名` | ブランチを作って切り替える |
| `git push -u origin ブランチ名` | リモートに(-u：upstreamとして)pushする |

Git 2.23以降で`git switch` が使用可能。(以前は`git checkout`だった)



### ブランチを削除するためのgitコマンドの例
削除するためには、削除対象でないブランチに移動してから行う必要がある。  
なお、リモート(GitHub)のブランチも削除する場合はリモートから先に削除するほうが好ましい。

```bash
# 削除対象でないブランチ(mainなど)に移動する
git switch main

# リモートのブランチを削除
git push origin --delete ブランチ名

# ローカルのブランチを削除
git branch -d ブランチ名
```

| オプション | 内容 |
|-----------|------|
| `-d` | マージ済みのブランチのみ削除（安全） |
| `-D` | 強制削除 |



### PRの基礎知識

| 項目 | 内容 |
|------|------|
| PRとは | あるブランチの変更を別のブランチに取り込むことを依頼する仕組み |
| 正式名称 | Pull Request |
| どこで行うか | GitHub上の操作 |



### 3ブランチ構成でのPRの流れ

```
dev → (PR) → staging → (PR) → main
```

| PR | マージ元 | マージ先 | タイミング |
|----|---------|---------|-----------|
| 1回目 | `dev` | `staging` | 開発作業が一区切りついたとき |
| 2回目 | `staging` | `main` | 本番リリースの準備ができたとき |

---

### PRの流れ（例：stagingからmainへのPRを行う場合）
```
1. stagingブランチに切り替える
      ↓
2. 変更をcommit
      ↓
3. リモート（GitHub）にpush
      ↓
4. GitHub上でPRを作成（stagingからmainへ）
      ↓
5. レビュー・確認
      ↓
6. マージ → mainに変更が取り込まれ、PRがcloseされる
      ↓
7. ローカルのmainに反映する
```
コマンド例
```bash
# 1. stagingブランチに切り替える
git switch staging

# 2. VS Codeでもstagingに切り替わるので、各種修正ののち、変更をcommit

# 3. リモート(GitHub)にpush
git push -u origin staging

# 4〜6. GitHub上でPRを作成・レビュー・マージ（ブラウザ操作）

# 7. ローカルのmainに反映する
git switch main
git pull origin main
```


---

## 3-8. AzureのGitHub連携で、ブランチを利用したCI/CDを活用する

先ほど実際に行ったCI/CD動作は、全てmainに対するpushのため、ymlに記載されるブランチからのPR(Pull Request)をトリガーとするワークフローには実行されていない。  
このセクションではブランチの作成とそれに伴うCI/CDを実施していく。併せてAzureの動作も検証する。


### このセクションのブランチ方針

このセクションでは、簡素化のため、Product(main) / Staging / Developmentという3つのブランチ構成ではなく、Product(main) / Development(dev) の２階層で進めていく。  
devブランチを切ってファイルを微修正し、mainにPR・マージするという流れで行う。

### SWAとGitHub連携のポイント

SWAはブランチのPRをトリガーとするプレビューURLという機能を提供している。

例：devブランチからmainへのPR・マージ、クローズを行う場合

| 操作 | SWAの動作 |
|------|----------|
| PRを作成 | devの内容を確認できるプレビューURLがAzure上に自動生成される |
| PRをマージ | Azureのmainに対応する環境にデプロイされ、プレビューURLが自動削除される |
| PRをマージせずクローズ | AzureからdevのプレビューURLが自動削除される |

### PRを作成し、devの修正をプレビューURLで確認する

1. VS Codeのターミナルでmainからdevブランチを作成する(`~/project/myazure/`で行う)

```bash
git switch -c dev     #devブランチ作成(devブランチに切り替わる)
git push -u origin dev   #devブランチをGitHubへpush
```

2. VS Codeの左下がmainからdevに切り替わったことを確認
3. ソース`api/src/functions/hello.ts`を適当に修正する

```
message: "Hello! Branch Test!"
```

4. VS Codeで修正した`hello.ts`をコミットしpushする。この段階ではdevブランチへのpushなので、GitHubのワークフローは起動しない。

5. ブラウザでGitHubのリポジトリを開き、devブランチからmainへのPRの内容を設定する

- devブランチを開き、`Compare & pull request`のボタンを押す
  - ドロップダウンで「`base:main`」、「`compare:dev`」となっていることを確認
  - 内容は今は適当でいい
  - tilte：PR from dev to main
  - description：messageの文字列を修正
  - 変更点がmesssageだけかどうかも念のため確認


> ⚠️注意：  
> この後、devからmainへのPRを作成(open)する作業に移る。  
> このフェーズで作成したワークフローのymlではPRがopenされるとワークフローが始まる。

6. `Create pull request`ボタンを押すと、devからmainのPRが作成(open)される。

7. ワークフロー`PR from dev to main`が開始するので、リポジトリのActionsタブでの動作を確認する。ワークフローが緑のチェックマークで完了していれば成功。

8. Azureでリソース`myazure-swa`を開き、左メニューの「設定」-「環境」を開く

9. 「環境のプレビュー」に分岐`dev`が追加されているので、右の「参照」を選ぶとプレビューURLで開くことができる。「APIを呼び出す」ボタンを押すと、`Hello! Branch Test!`と表示される。

10. mainのURLで「APIを呼び出す」ボタンを押しても`Hello from Azure Functions!`のままとなっている。

以上の流れで、devブランチの作成とそれに応じたAzureでのプレビュー確認を行うことができた。


### PRをマージし、本番(main)への変更反映を確認する

プレビューURLでdevブランチの動作確認が行えたので、今度はdevからmainにマージを行う。マージを行った後もGitHubのワークフローが実行されビルドとデプロイが処理される。  
この処理によって、mainのURLで変更内容が反映されるか、プレビューURLがどうなるかを確認していく。


1. ブラウザでGitHubのリポジトリを開き、`Pull Requestsタブ`を開き、`PR from dev to main`を選択。

2. 行える操作は、`Merge pull request`か`Close pull request`のいずれか。

    - `Close`はマージせずにPRをcloseすることを意味する(リクエスト却下)
    - ここではマージするため`Merge pull requests`を選び、`Confirm merge`を実行する

3. マージすると、2つのワークフローが実行される。

    - devの修正がmainにコミットされるので、mainのビルドとAzureの本番URLへのデプロイ
    - devからmainのPRがマージによりcloseさえるので、プレビューURLの後片付け

4. Azureでリソース`myazure-swa`を開き、左メニューの「設定」-「環境」を開くと、devの分岐がなくなっていることが確認できる(プレビューURLの削除)。

5. mainのURLを開き、「APIを呼び出す」ボタンを押すと、`Hello! Branch Test!`となり、修正反映が確認できる。

6. GitHub上でマージしたので、ローカルのmainにも忘れずに反映しておく。

```bash
git pull origin main
```

### devブランチの後片付け(オプション)

作成したdevブランチを削除してしまいたい場合は、VS Codeのターミナルで以下の操作を行う。


```bash
git switch main              # 削除対象でないブランチ(mainなど)に移動する
git push origin --delete dev # リモート(GitHub)のdevブランチを削除
git branch -d dev            # ローカルのdevブランチを削除
```
GitHubのリポジトリでdevブランチが削除されていることを確認する。

### SWAとGitHub連携のまとめ

- ブランチを作成しPRを作成すると、プレビューURLが用意されそこにデプロイされる
- プレビューURLでの確認を終え、ブランチの修正をmainにマージすると
   - 本番URLに修正がデプロイされる
   - プレビューURLは廃棄される


---

## まとめ

フェーズ3で完了したことは以下の通り。

- フロントエンド（`frontend/index.html`・`frontend/style.css`）の作成
- Azure Functions（`hello.ts`）と、関連ファイルの作成
- GitHubシークレットとワークフローファイルの確認
- GitHubへのpushによるCI/CDパイプラインの挙動確認
- AzureにデプロイされたWebアプリを実際に確認
- Git/GitHubのブランチの概念について整理
- ブランチの作成とそれに伴うCI/CDパイプラインの挙動の変化も確認


[「フェーズ4：アプリケーションの実装」](./azure-setup-phase4.md) へ進む

---

作成日：2026年3月  文書名：フェーズ3詳細手順
