# ojisan-notes

## 自己紹介

ほぼ初学者のおじさん。  
長らくIT業界にいるものの、何もかもほとんど触ったことがない。  
今こそAIのウェーブに乗るため、AIを活用し、いろんな記事を参考にさせてもらいながら、色々と学習中。自分自身の手順書や技術情報のメモをまとめて保管。

## 初学者がAIで学んだのち、サービスを開発し公開
- [ブクマッター](https://bukumatter.com/) ：  AIに読ませる、共有オンラインブックマークのサービス
  - '26/04開発、'26/05/08公開
  - [ブクマッターを使ったAIケーススタディ](./notes/bukumatter/bukumatter-casestudy.md)
  - [ブクマッターの開発雑記](./notes/bukumatter/bukumatter-dev-notes.md)


## AIが初学者に教える シリーズ('26/03)  

### MacBook AirにVS Codeを入れ、GitHubと連携
- [Git / GitHub / VS Code 連携環境構築](./notes/study/setup-mac-git-github-vscode.md) 
    - (補足) [Gitの.gitディレクトリで気になったこと](./notes/study/git/git-dot-git-behavior.md)

### ラズパイをGitHub Codespacesの代わりに使う
- [ラズパイとDockerでSandbox環境を構築する](./notes/study/setup-raspi4-sandbox.md) 
    - (補足) [ラズパイの初期設定で行った方がいいもの](./notes/study/raspi/raspi4-initial-settings.md)
    - (補足) [Dockerインストールに際しての事前検討事項](./notes/study/docker/docker-install-method.md)
    - (補足) [AIと初学者が殴り合い：Dockerインストール検証の議論](./notes/study/docker/docker_trust_model.md)
    - (補足) [AIに初学者が粘着：GPGキーとsigned-byをめぐる議論](./notes/study/docker/gpg-signed-by.md)
    - (補足) [dockerグループ：よくわからないけど、何か違和感](./notes/study/docker/docker-group-problem.md)

### 初めてクラウドに触れる Azure編、ブランチやCI/CDも学ぶ
- [Azure初利用に向けた、AIを使ったプランニング](./notes/study/azure/azure-first-use-planning.md)
  - フェーズ1：[事前準備](./notes/study/azure/azure-setup-phase1.md)
    - 構成や配置の検討、ローカルのセッティング
  - フェーズ2：[Azureリソースの作成](./notes/study/azure/azure-setup-phase2.md)
    - Azureアカウントの作成やAzureリソースの作成
  - フェーズ3：[デプロイおよびStatic Web Appsの動作確認](./notes/study/azure/azure-setup-phase3.md)
    - 超シンプルなWebアプリ、ブランチ作成、GitHubのCI/CD
  - フェーズ4：[アプリケーションの実装](./notes/study/azure/azure-setup-phase4.md)
    - Blobのデータを読み書きする機能を作成
  - フェーズ5：[カスタムドメインを設定する](./notes/study/azure/azure-setup-phase5.md)
    - 手持ちのドメイン名でアクセスできるようにする

