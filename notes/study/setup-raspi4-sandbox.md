# ラズパイとDockerでSandbox環境を構築する

> ⚠️ おことわり：
> このページは、構築に不慣れなおじさんがAIに質問を繰り返し壁打ちした内容をベースにまとめたものです。
> 偉そうに解説・説明する文面が含まれますが、AI様が勝手に言ってることですので、あらかじめご了承ください。
> なるべく裏付けをとるようにしましたが、正確でない情報が含まれる可能性もあります。


## この記事の目的と構成の説明

この記事のゴールは「MacbookのVS Codeから、ラズパイ(RasPi)上のDockerコンテナの中でPythonスクリプトを実行できる」状態を作ることです。ラズパイはラズパイ4を使用しています。

```
Macbook の VS Code
    ↓（Remote - SSH で接続）
ラズパイ
    └── Docker コンテナ
            └── Python スクリプトを実行
```

| 環境 | インストール | 備考 |
|---|---|---|
| Macbook| VS Code (別途導入済み) | GitとPythonは不要 |
| ラズパイ | Git (初期導入済み)、Docker | Pythonは不要 |
| ラズパイ上のDocker | Python、その他パッケージ | 動作時のDockerfile次第 |
| | | |


### GitHub Codespaces の代替として

- GitHub Codespacesとは
  - ブラウザやVS Codeから、GitHubが用意したクラウドでコードを実行できるサービス
- そのCodespacesクラウドを手元のラズパイに置き換え
  - MacbookのVS CodeからSSH経由でラズパイに接続
  - ラズパイ上でコードを編集・実行する構成
  - クラウドを使わず手元のハードで同様の開発体験を実現する
- ポイント
  - 極論、Macbook本体にはGitもPythonも一切インストール不要
  - Macbookの環境を汚さずに開発環境を持てることもメリットのひとつ

### なぜラズパイ本体ではなくDockerコンテナでPythonを動かすのか

ラズパイ本体に直接Pythonの実行環境を作ることも技術的には可能ですが、Dockerのメリットもあります。

- Dockerコンテナを使うメリット
  - ラズパイ本体の環境を汚さずに済む
  - プロジェクトごとに異なるPythonバージョンや依存ライブラリを使い分けられる
  - 環境そのものをDockerfileとしてGitで管理できる
  - 環境を壊しても、コンテナを作り直すだけでリセットできる

### この環境の用途と運用方針

この環境はSandbox・Playgroundとしての用途を想定しています。様々な言語やライブラリを気軽に試すための場所です。本番運用するサービスや重要なデータを扱うことは想定していません。

セキュリティについては、dockerグループへのユーザー追加（後述）はroot相当のリスクを伴う設定です。本記事ではその点を理解した上で、習熟するまでは未使用時にラズパイを停止させる運用によってリスクを受け入れる判断をしています。この記事に基づいて常時稼働・外部公開するサービスを構築することは推奨しません。

### 作業の流れ

| フェーズ | 内容 | 作業場所 |
|---|---|---|
| フェーズ1 | ラズパイのOSインストールと初期設定 | Mac → ラズパイ |
| フェーズ2 | Dockerのインストール | ラズパイ（MacからSSH） |
| フェーズ3 | VS CodeからのSSH接続設定 | Mac |
| フェーズ4 | GitHubとの連携 | ラズパイ（VS Code経由） |
| フェーズ5 | DockerコンテナでPythonを動かす | ラズパイ（VS Code経由） |

---

## 事前に用意するもの

### ハードウェア

- **Raspberry Pi 4**（4GB または 8GB RAM モデル推奨）
- **microSD カード**（64GB 以上、Class 10 以上推奨）
- **電源アダプタ**（USB-C 5V/3A）
- **Wi-Fi 環境**（Macbook と同じネットワークに接続できること）

### Macbook 側の前提条件

- VS Code がインストール済みであること
- GitHubアカウントを持っていること
- VS CodeとGitHubの連携、MacbookへのGitのインストールは不要です（コードの編集・実行・GitHubとの連携はすべてラズパイ上で行うため）

---

## フェーズ1：Raspberry Pi 4 の初期設定

### 1-1. Raspberry Pi OS のインストール

**必ず 64bit 版**をインストールします。32bit 版では Docker が正常に動作しません。

[Raspberry Pi Imager](https://www.raspberrypi.com/software/) を Macbook にダウンロードし、microSD カードに書き込みます。OS 選択時は「Raspberry Pi OS（64-bit）」を選んでください。

Imager の設定画面（歯車アイコンまたは「OS のカスタマイズ」）で以下を事前に設定しておきます。モニターもキーボードもラズパイに繋がずに初期設定を完了させるための設定です（ヘッドレスセットアップと呼びます）。

- ホスト名（例：`raspi`）
- ユーザー名とパスワード
- Wi-Fi の設定（SSID とパスワード）
- SSH の有効化（「サービス」タブで設定）


### 1-2. 初回起動と OS のアップデート

microSD カードをラズパイに挿入して起動します。起動完了まで1〜2分程度かかります。

起動後、Macbook のターミナルから SSH 接続します。  
(**補足**：この時点ではパスワード認証でSSH接続します。公開鍵認証への切り替えはフェーズ3で行います。)

```bash
ssh <ユーザー名>@<ホスト名>.local
```

**注意**：`<ホスト名>.local`で接続できない場合は、ルーターの管理画面などでラズパイのIPアドレスを確認し、`ssh <ユーザー名>@<IPアドレス>`で接続してください。

接続できたら OS を最新の状態にします。

```bash
sudo apt-get update && sudo apt-get upgrade -y
```


### 1-3. 64bit OS の確認

必ず 64bit 版が動作していることを確認します。

```bash
uname -m
```

`aarch64` と表示されれば 64bit で動作しています。`armv7l` と表示された場合は 32bit 版のため OS を入れ直してください。

### 1-4. ラズパイ の OS の諸設定

OSの初期設定については別途まとめた記事を参照してください。

[ラズパイで初期設定したほうがいいもの](./raspi/raspi4-initial-settings.md)

---

## フェーズ2：Docker のインストール

> ⚠️ おじさんメモ：
> Dockerのインストールは複数の方法が提供されています。この手順では、`Docker公式リポジトリからインストールする`方法を採用しています。以下のページにいろんな方法と選んだ基準を記載しています。
>
> - 疑問点 : [ Dockerインストールに際しての事前検討事項](./docker/docker-install-method.md)  

### 2-1. Docker のインストールの前準備

古いバージョンが残っていれば削除します。

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt-get remove $pkg
done
```

**補足**：上記は2026年3月時点のDocker公式ドキュメントに記載されている手順に従っています。過去のインストール方法や時期によって異なるパッケージ名が使われていた可能性もあり、このリストで過去のあらゆるケースを網羅できているかどうかは筆者には判断できていません。

システムを最新の状態にしてから ca-certificates と curl をインストールします。  
なお、curlはあらかじめインストールされているようです。

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install ca-certificates curl
```

`/etc/apt/keyrings` ディレクトリを作成します。  
すでに作成済みで755の場合は飛ばして下さい。

```bash
sudo mkdir -p /etc/apt/keyrings
sudo chmod 0755 /etc/apt/keyrings
```

Docker の GPG キーを取得して保存します。  
保存後、docker.ascの権限が644であれば2行目は不要です。

```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Docker の apt リポジトリを OS のパッケージソース一覧に追加します。  

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

上記のコマンドが何を行っているか理解するために、構成要素を分解して説明します。  
- `$(dpkg --print-architecture)`  
  - CPUアーキテクチャを自動取得します。ラズパイ4の場合は`arm64`が入ります。  
- `$(. /etc/os-release && echo "$VERSION_CODENAME")` 
  - OSのバージョン名を自動取得します。Raspberry Pi OSの現行版では`trixie`が入ります。  
- `sudo tee`を使っているのはroot権限が必要なファイルへの書き込みをパイプ経由で行うための迂回手段です。
  - `> /dev/null`はターミナルへの表示を不要として捨てています。  
- 結果として`/etc/apt/sources.list.d/docker.list`に書き込まれる内容は実質的にこうなります。 
  - `deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian trixie stable`


> ⚠️ おじさんメモ：この操作は、DockerのaptリポジトリのURLと署名検証するGPGキーを紐づけているようです。2つ疑問点があったので、以下の記事にまとめています。
> - 疑問点1 : [GPGは何を保証しているのか？](./docker/docker_trust_model.md)  
> - 疑問点2 : [GPGとリポジトリURLをなぜ紐づける？(signed-by)](./docker/gpg-signed-by.md)


### 2-2. Docker のインストール
apt のパッケージリストを更新します。

```bash
sudo apt-get update
```

Docker をインストールします。

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

インストールされるパッケージの内訳は以下の通りです。
| パッケージ | 役割 |
|---|---|
| docker-ce | Docker本体（Community Edition）です。コンテナの管理・実行を担います。 |
| docker-ce-cli | ターミナルからDockerを操作するためのコマンドラインツールです。`docker run`などのコマンドはこれが提供しています。 |
| containerd.io | コンテナの実行を担う低レベルのランタイムです。Docker本体の下で動作します。 |
| docker-buildx-plugin | Dockerイメージをビルドするための拡張ツールです。複数のCPUアーキテクチャ向けのイメージを一度にビルドする機能などを提供します。 |
| docker-compose-plugin | 複数のコンテナをまとめて管理するための`docker compose`コマンドを提供します。 |
| | |


動作確認します。

```bash
sudo docker run hello-world
```
`Hello from Docker!` と表示されれば完了です。  
このコマンドは「hello-worldというDockerイメージをもとにコンテナを起動する」コマンドです。hello-worldはDocker社が動作確認用に公式で提供している最小限のイメージです。


### 2-3. Docker の操作方針

Docker の操作はdockerグループにユーザーを追加することで、結果的に`sudo`なしでDockerを実行できるようなります。  

**注意 :** dockerグループへの追加はroot権限の付与と実質同等のリスクを伴います。これはDockerの公式ドキュメント自体も認めていることです。本環境はSandbox用途であり、当面は未使用時にラズパイを停止させる運用によってリスクを受け入れた上でこの方針を採用しています。常時稼働・外部公開するサービスへの転用時は改めてセキュリティ設計を見直してください。 

> ⚠️ おじさんメモ：Dockerグループについて確認した内容を記載しています。 
> - 疑問点 : [dockerグループ：よくわからないけど、何か違和感](./docker/docker-group-problem.md)  
>  

usermodコマンドで現在のユーザをdockerグループに所属させます。

```bash
sudo usermod -aG docker $USER
```



設定を反映させるためにいったんSSH接続を切断して再接続します。

```bash
exit
```

再接続後、グループへの追加が反映されていることを確認します。

```bash
groups
```

出力に `docker` が含まれていれば正常です。

### 2-4. 動作確認

```bash
docker --version
docker compose version
```

以下のように表示されれば問題ありません（バージョン番号は異なる場合があります）。

```
Docker version 29.x.x, build xxxxxxx
Docker Compose version v5.x.x
```

ARM64 で正しく動作しているかも確認します。

```bash
docker info | grep Architecture
```

`aarch64` と表示されれば正常です。

`sudo`なしで動作することも確認します。

```bash
docker run hello-world
```

`Hello from Docker!` と表示されれば、dockerグループへの追加が正しく機能しています。

### 2-5. Docker の自動起動設定

ラズパイでは systemd を使って Docker の自動起動を設定します。  
aptリポジトリからインストールした場合はすでにenabledになっている可能性があるため、必ず現在の状態を確認するのが良いでしょう。 

まず現在の状態を確認し、`disabled` の場合のみ `enable` します。
おそらくenableになっていると思います。

```bash
# 現在の状態を確認
sudo systemctl is-enabled containerd
sudo systemctl is-enabled docker

# disabled の場合のみ実行
sudo systemctl enable containerd
sudo systemctl enable docker

# 設定後の状態を確認
sudo systemctl is-enabled containerd
sudo systemctl is-enabled docker
```

どちらも `enabled` と表示されれば完了です。

**補足**：`docker.service` は systemd 上で containerd への依存関係（`After`・`Requires`）が定義されており、containerd が起動していない場合は Docker も起動しない仕組みになっています。

---

## フェーズ3：Macbook からのリモート開発環境の設定

ここからはMacbook側での作業です。VS CodeのRemote - SSH拡張機能を使って、MacbookのVS Codeからラズパイの中を直接操作できるようにします。

**補足**：Remote - SSH拡張機能を使うと、VS Codeのウィンドウがそのままラズパイ上で動作しているかのような状態になります。ファイルの編集もターミナルでのコマンド実行も、すべてラズパイ上で行われます。

### 3-1. VS Code に Remote - SSH 拡張機能をインストールする

VS Codeを開き、左サイドバーの拡張機能アイコン（四角が4つ並んだアイコン）をクリックします。検索欄に `Remote - SSH` と入力し、Microsoftが提供する「Remote - SSH」をインストールします。

インストールが完了すると、VS Codeの左サイドバーにリモート接続用のアイコン（モニターに小さな矢印が付いたアイコン）が追加されます。

### 3-2. MacbookでSSH鍵を作成する

MacbookのターミナルでSSH鍵ペアを新規作成します。

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "<任意のコメント>"
```


>**⚠️ 既に、Macから別のサーバーにSSH鍵でアクセスしている方 / MacのGitでGitHubとSSH連携している方へ**  
>
>`~/.ssh/`に既存の鍵が存在する場合、上記コマンドでは別のファイル名（例：`-f ~/.ssh/id_ed25519_raspi`）を指定して新たに鍵を作成します。その後、`~/.ssh/config`にラズパイ用のブロックを以下の追記することで、どのホストにどの鍵を使うかを明示的に管理できます。
>
> ```
> Host raspi
>   HostName <ホスト名>.local
>   User <ユーザー名>
>   IdentityFile ~/.ssh/id_ed25519_raspi
> ```
>
> このように設定しておけば、ターミナルで`ssh raspi`と打つだけで自動的に該当する鍵が使われます。  
> 以下の記事では`id_ed25519.pub`は`id_ed25519_raspi.pub`と読み替えてください。

**補足**：ssh-keygenの各オプションの意味は以下の通りです。  
- `-t ed25519`：鍵の種類を指定します。ed25519は現在推奨される方式です。  
- `-f ~/.ssh/id_ed25519`：鍵の保存先とファイル名を指定します。  
- `-C`：鍵を識別するためのコメントです。メールアドレスや用途を書くことが多いです。省略可能です。  
- パスフレーズの入力を求められます。設定しておくとセキュリティが高まりますが、空のままEnterでも構いません。



### 3-3. ラズパイに公開鍵を転送する

作成したSSH鍵の公開鍵をラズパイに転送します。

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub <ユーザー名>@<ホスト名>.local
```

**補足**：`ssh-copy-id`はMacbook上の公開鍵をラズパイの`~/.ssh/authorized_keys`に追記するコマンドです。秘密鍵（id_ed25519）はMacbook上にあるままで、ラズパイにはコピーされません。このコマンドの実行時にはパスワード認証でラズパイにログインします。

転送後、公開鍵認証でSSH接続できることを確認します。

```bash
ssh <ユーザー名>@<ホスト名>.local
```

パスワードなしで接続できれば正常です。確認後は`exit`で切断します。

### 3-4. VS Code からラズパイに SSH 接続する

VS Codeを開き、左サイドバーのリモート接続アイコンをクリックします。「SSH」の項目の右にある`+`アイコンをクリックし、接続先として以下を入力します。

```
ssh <ユーザー名>@<ホスト名>.local
```

設定ファイルの保存先を聞かれたら`~/.ssh/config`を選択します。

>**⚠️ 既に、Macから別のサーバーにSSH鍵でアクセスしている方 / MacのGitでGitHubとSSH連携している方へ**  
>3-2の注釈に該当する方（既存の鍵が複数あり、すでに`~/.ssh/config`にラズパイ用のブロックを追記済みの方）は、ここで`ssh raspi`と入力してください。その場合、VS Codeは`~/.ssh/config`への書き込みを行わず、既存の設定をそのまま使って接続します。

接続先が追加されたら、`<ホスト名>.local`の右側にある矢印アイコン（「新しいウィンドウで接続」）をクリックします。新しいVS Codeウィンドウが開き、接続処理が始まります。初回接続時はラズパイ側にVS Code Serverのインストールが行われるため、数分かかる場合があります。

接続が完了すると、VS Codeの左下に `SSH: <ホスト名>.local` と表示されます。

### 3-5. 動作確認

VS Codeのメニューから「ターミナル」→「新しいターミナル」を選択します。ターミナルが開いたら、以下のコマンドを実行します。

```bash
hostname
```

ラズパイのホスト名（1-1で設定したもの）が表示されれば、VS CodeがラズパイにSSH接続できていることが確認できます。

---

## フェーズ4：GitHub との連携

ここからはフェーズ3で接続したVS Code上のターミナルでの作業です。つまり、コマンドはすべてラズパイ上で実行されます。

### 4-1. ラズパイ上での Git の初期設定

GitにGitHubアカウントの情報を設定します。  
GitHubで登録メールアドレスを**非公開に設定**している場合は、ダミーメールアドレス`例)xxx@users.noreply.github.com`を指定します。

```bash
git config --global user.name "<GitHubのユーザー名>"
git config --global user.email "<GitHubに登録しているメールアドレスかダミーメールアドレス>"
```

設定を確認します。

```bash
git config --global --list
```

`user.name`と`user.email`が正しく表示されれば完了です。


### 4-2. (補足) Gitの動作について

おじさんメモ：
Gitがどのディレクトリをどのように管掌するか、気になったので色々と調べました。以下の記事にまとめています。
- ⚠️疑問点 : [Gitの.gitディレクトリについて気になったこと](./git-dot-git-behavior.md)


### 4-3. (補足) git config --global 設定項目一覧


| 設定項目 | 内容 |
|---|---|
| `user.name` | コミット履歴に記録される作者名。未設定の場合はコミット時にエラーになる |
| `user.email` | コミット履歴に記録されるメールアドレス。未設定の場合はコミット時にエラーになる |
| `init.defaultBranch` | git initしたときに最初に作られるブランチ名。デフォルトは`master`、GitHubに合わせて`main`にすることが多い |
| `core.editor` | git commitでメッセージを書く際に起動するエディタ。デフォルトは`vim` |
| `core.autocrlf` | 改行コードの自動変換。デフォルトは`false`。Macでは`input`にするとWindowsの改行コードをLFに変換して保存する |
| `color.ui` | ターミナルのgit出力を色付きにする。デフォルトは`auto` |
| `push.default` | git pushしたときにどのブランチにプッシュするかの挙動。デフォルトは`simple` |
| `pull.rebase` | git pullしたときにマージではなくrebaseを使うかどうか。デフォルトは`false`（マージ） |
| `core.ignorecase` | ファイル名の大文字・小文字を区別するかどうか。Macのデフォルトは`true`（区別しない） |
| `diff.tool` | 差分を表示する際に使うツール。デフォルトはターミナル表示 |
| `merge.tool` | マージの競合解決に使うツール。デフォルトはターミナル表示 |
| `credential.helper` | GitHubへの認証情報をどこに保存するか。Macのデフォルトは`osxkeychain` |
| `alias.*` | よく使うgitコマンドに短縮名をつける。例：`alias.st status` |




### 4-4. ラズパイ上でGitHub連携用のSSH鍵を作成する

ラズパイからGitHubへのSSH接続に使う鍵をラズパイ上で新たに作成します。MacbookのSSH鍵とは別の鍵を使う理由は、ラズパイは常時起動に近い運用をするデバイスであり、Macbookの鍵をラズパイ上に置くことを避けるためです。

```bash
ssh-keygen -t ed25519 -C "<任意のコメント>"
```

保存先とファイル名を聞かれます。デフォルト（`~/.ssh/id_ed25519`）のままEnterを押します。

**補足**：ラズパイからSSHでアクセスする先はGitHubだけです。鍵も1つだけ存在します。この場合、SSHのデフォルト動作（`~/.ssh/`にある鍵を自動的に探して使う）に任せれば十分なため、ラズパイの`~/.ssh/config`には何も設定しません。接続先が増えたり鍵が複数になった段階で改めてconfigを書けばよいという考え方です。

作成した公開鍵の内容を表示します。

```bash
cat ~/.ssh/id_ed25519.pub
```

表示された内容（`ssh-ed25519 AAAA...`で始まる1行）をすべてコピーします。

### 4-5. GitHubに公開鍵を登録する

ブラウザでGitHubを開き、右上のアイコンから「Settings」→「SSH and GPG keys」→「New SSH key」をクリックします。

- Title：任意（例：`raspi`）
- Key type：Authentication Key
- Key：4-2でコピーした公開鍵の内容を貼り付け

「Add SSH key」をクリックして登録完了です。

ラズパイのターミナルからGitHubへのSSH接続を確認します。

```bash
ssh -T git@github.com
```

`Hi <ユーザー名>! You've successfully authenticated...` と表示されれば接続成功です。

> **補足**：初回接続時は「このホストを信頼しますか？」という確認メッセージが表示されます。`yes`と入力して進んでください。

### 4-6. リポジトリ  の作成（GitHub 上での作業）

ブラウザでGitHubを開き、新しいリポジトリを作成します。

- Repository name：任意（例：`sandbox`）
- Description：任意（例：`Python Sandbox`）
- Public / Private：Privateを推奨
- 「Initialize this repository with a README」にチェックを入れる
- 「Add .gitignore」で `Python` を選択する

「Create repository」をクリックして作成完了です。

**補足**：`.gitignore`にPythonを選択しておくと、Pythonが自動生成するキャッシュファイル（`__pycache__/`や`.pyc`など）がGit管理対象から除外されます。これらはGitHubにプッシュする必要がないファイルです。

### 4-7. リモート接続のVS Codeでクローンを行う際の注意点 ： Gitコマンドで行う

**注意：** VS Codeのエクスプローラーに「Clone Repository」ボタンが表示されますが、ここでは使用しません。ラズパイ上でのGitHub認証はSSH鍵で行うため、ターミナルのコマンドでクローンします。　　

| | Macローカルを開くVS Code | リモートを開くVS Code |
|---|---|---|
| VS CodeとGitHubの連携 | OAuthで連携済み | 連携なし |
| GitとGitHubの連携 | Mac上のSSH鍵で連携済み | ラズパイ上のSSH鍵で連携済み |
| VS CodeのGitHub機能 | 使える（リポジトリ一覧・PR・Issueなど） | 使えない |
| クローンの方法 | VS CodeのGUIまたはターミナル | ターミナルのみ推奨 |
| VS Codeの役割 | ローカルで処理も表示も担う | 表示・入力の送受信のみ（処理はラズパイ側） |

リモートを開くVS Codeはあくまでもラズパイ上のVS Code Serverに繋いでいるだけなので、仮にOAuth認証をする場合でもVS Code Server側で行う必要があるようで、複雑そうです。  
個人の利用の範囲であれば、そこまで必要な機能追加もないため、OAuth認証は行いません。

---

### 4-8. 実際にGitコマンドでクローンを行う

VS Codeのリモートウィンドウのターミナルでラズパイ上にクローンします。
まず作業ディレクトリを作成します。

```bash
mkdir -p ~/projects
cd ~/projects
```

GitHubのリポジトリページから「Code」→「SSH」タブのURLをコピーし、クローンします。

```bash
git clone git@github.com:<GitHubのユーザー名>/<リポジトリ名>.git
cd <リポジトリ名>
```

クローンできたことを確認します。

```bash
ls -la
```

`README.md`と`.gitignore`が表示されれば完了です。

---

## フェーズ5：Docker コンテナで Python を動かす

### 5-1. リポジトリ のフォルダ構成を作る

フォルダ構成は以下のようにする前提で進めていきます。

```
<リポジトリ名>/
├── .gitignore
├── README.md
└── hello-python/
    ├── Dockerfile
    ├── docker-compose.yml
    ├── requirements.txt
    └── src/
        └── main.py
```

最初のプロジェクト用のフォルダ構成を作ります。

```bash
cd ~/projects/<リポジトリ名>
mkdir -p hello-python/src
```



### 5-2. Dockerfile を書く

VS Codeのエクスプローラーから`hello-python/Dockerfile`を新規作成し、以下の内容を記述します。

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ .

CMD ["python", "main.py"]
```

**補足**：各行の意味は以下の通りです。

- `FROM python:3.12-slim`：Pythonの公式イメージをベースにします。`slim`は最小限の構成で容量が小さいバリアントです。
- `WORKDIR /app`：コンテナ内の作業ディレクトリを`/app`に設定します。
- `COPY requirements.txt .`と`RUN pip install`：依存ライブラリをインストールします。
- `COPY src/ .`：`src/`ディレクトリの中身をコンテナ内の`/app`にコピーします。
- `CMD ["python", "main.py"]`：コンテナ起動時に実行するコマンドを指定します。

### 5-3. docker-compose.yml を書く

`hello-python/docker-compose.yml`を新規作成し、以下の内容を記述します。

```yaml
services:
  app:
    build: .
    volumes:
      - ./src:/app
```

**補足**：`volumes`の設定はホスト(ラズパイ)側の`./src`ディレクトリとDockerコンテナ内の`/app`を同期します。この設定があることで、`src/main.py`を編集するたびにコンテナを作り直さなくても変更が即座にコンテナ内に反映されます。

### 5-4. requirements.txt を書く

`hello-python/requirements.txt`を新規作成します。今回はHello World的な確認が目的のため、中身は空のままで構いません。

```
# 依存ライブラリをここに記述する
# 例：requests==2.31.0
```

### 5-5. コンテナをビルドして Python スクリプトを実行する

`src/main.py`を新規作成し、以下の内容を記述します。

```python
print("Hello from Docker container!")
```

`hello-python/`ディレクトリに移動してコンテナをビルドします。

```bash
cd ~/projects/<リポジトリ名>/hello-python
docker compose build
```

**補足**：初回ビルド時はPythonのベースイメージをDocker Hubからダウンロードするため、数分かかる場合があります。

ビルドが完了したらコンテナを起動してスクリプトを実行します。

```bash
docker compose run --rm app
```

**補足**：`--rm`はコンテナ実行後に自動削除するオプションです。使い捨てで実行したい場合に便利です。

### 5-6. 動作確認

以下のように表示されれば、DockerコンテナでPythonスクリプトの実行が確認できました。

```
Hello from Docker container!
```

最後にGitHubにプッシュして、一連の作業をリポジトリに保存します。

```bash
cd ~/projects/<リポジトリ名>
git add .
git commit -m "Add hello-python project"
git push origin main
```

`git clone`を実行したとき、Gitはクローン元のURL(今回であれば`git@github.com:<ユーザー名>/<リポジトリ名>.git`)に対して自動的に`origin`という名前をつけます。毎回長いURLを打たなくて済むようにするための仕組みです。  
`main`はプッシュ先のブランチ名です。GitHubでリポジトリを作成したとき、デフォルトで`main`というブランチが作られています。  
GitHubのリポジトリページを開き、`hello-python/`フォルダが追加されていれば完了です。  
もちろん、コマンドではなく、VS Codeでもコミットやプッシュを行うことも可能です。

---

## 次のプロジェクトを追加するときは

新しいプロジェクトを試したいときは、リポジトリのルートに新しいフォルダを追加するだけです。

```
例） 
<リポジトリ名>/
├── hello-python/   ← 今回作ったもの
├── project-02/     ← 次に追加するもの
└── project-03/     ← さらに追加するもの
```

各プロジェクトは独立したDockerfileとdocker-compose.ymlを持つため、Pythonのバージョンや依存ライブラリが異なっていても互いに干渉しません。

---

*作成日：2026年3月*  
*作業環境：MacBook Air M1 / macOS / zsh / Raspberry Pi 4 / Raspberry Pi OS trixie*
