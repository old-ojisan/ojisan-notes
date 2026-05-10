# フェーズ1：事前準備
> ⚠️ おことわり：
このページは、構築に不慣れなおじさんがAIに質問を繰り返し壁打ちした内容をベースにまとめたものです。
偉そうに解説・説明する文面が含まれますが、AI様が勝手に言ってることですので、あらかじめご了承ください。
なるべく裏付けをとるようにしましたが、正確でない情報が含まれる可能性もあります。
 
| 順番 | 作業内容 |
|------|---------|
| 1-1 | 構成・配置の詳細を検討し決定する |
| 1-2 | ローカルの開発環境を準備する |
| 1-3 | GitHubに新規リポジトリを作成する |

 
## 1-1：構成・配置の詳細を検討し決定する
 
本プロジェクトで使用するプロジェクト名・ディレクトリ構成・認証情報の置き場所を以下のとおり決定する。
 
### プロジェクト名
 
| 項目 | 値 |
|------|-----|
| プロジェクト名 | `myazure` |
| GitHubリポジトリ名 | `myazure` |
 
特に事情がない限り、Azureのリソース名にも `myazure` をプレフィックスとして踏襲する。
 
### リポジトリ / ディレクトリ構成
 
```
~/projects/
└─── myazure/          ← GitHubからcloneするリポジトリ
    ├── .gitignore
    ├── .github/
    │   └── workflows/ ← GitHub Actionsワークフロー
    ├── frontend/      ← フロントエンド（HTML・CSS）
    ├── api/           ← Azure Functions（TypeScript）
    └── README.md
```
---
 
## 1-2：ローカルの開発環境を準備する

ローカルでは、MacBookでコード作成編集のみを行う。

- MacBook Air M1
  - VS Code
  - Git (GitHubへのSSHアクセス設定済み)
  - ビルド・実行環境は持たない
- 構築手順は、[Git / GitHub / VS Code 連携環境構築](../setup-mac-git-github-vscode.md)に記載


**注意**  
初めてAzureを使ってみるという目的を優先し、今回はローカルでのビルドや動作確認は行わないという特殊な構成としている。  
ローカルでのビルド・実行環境構築は、今後の対応とする。

---
 
## 1-3：GitHubに新規リポジトリを作成する
 
### 手順
 
1. ブラウザで `https://github.com` にアクセスしてサインインする
2. 右上の「+」→「New repository」をクリックする
3. 以下の設定をして、「Create repository」でリポジトリを作成する
 
| 設定項目 | 設定値 |
|---------|--------|
| Repository name | `myazure` |
| Public / Private | **Private** |
| Add a README file | **On にする** |
| Add .gitignore | **Node** を選択 |
| Choose a license | なし |
 
`.gitignore` は Node を選択することで、Azureのテスト環境で使用する `node_modules/` や `local.settings.json` などが最初から除外対象となる。

ローカルの開発環境へのcloneは以降のフェーズで行う。

---
 
## まとめ
 
フェーズ1で完了したことは以下の通り。
 
- 構成・配置の決定
- ローカル開発環境の準備
- GitHubへのプライベートリポジトリ `myazure` の作成
 
 [「フェーズ2：Azureリソースの作成」](./azure-setup-phase2.md) へ進む

---
作成日：2026年3月 文書名：フェーズ1詳細手順