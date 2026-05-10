# dockerグループ：よくわからないけど、何か違和感を感じた

## はじめに

これは答えのある記事ではない。Dockerを使い始めて気になったことを書き留めたものだ。詳しい人から見れば的外れな部分もあるかもしれないし、そもそも問題でも何でもないのかもしれない。ただ、腑に落ちないまま先に進むのが嫌なので書いておく。

---

## 引っかかったこと

テック系の記事などでDockerのインストール手順を調べると、このコマンドが出てくることが多い気がする。

```bash
sudo usermod -aG docker $USER
```

「これを実行しないとsudoが必要になって不便」という説明とともに紹介されていることも。実際に実行する前に少し調べてみると、Dockerの公式ドキュメントにこんな記述があった。

> dockerグループへの追加はroot権限の付与と同等のリスクがある

同じ公式ドキュメントが、インストール手順でdockerグループへの追加を紹介しつつ、そのリスクも認めている。これが何を意味するのかは、正直よくわからない。

---

## sudoersとdockerグループの関係

「dockerグループに追加しなければ安全」という考え方があるようだが、自分にはこれがよくわからない。

ユーザーとグループの組み合わせを整理すると、以下の4つのケースが考えられる。

- userA：sudoersに入っており、dockerグループにも入っている
- userB：sudoersに入っているが、dockerグループには入っていない
- userC：sudoersには入っていないが、dockerグループには入っている
- userD：sudoersにもdockerグループにも入っていない

userAとuserBの違いについては、sudoersに入っているユーザーは`sudo usermod -aG docker $USER`を自分で実行できるし、`sudo docker`でdockerを操作することもできる。つまりuserBがuserAより安全かどうか、自分には判断できない。

userCはsudoを持たないため一見権限が低いように見えるが、dockerグループに入っている時点で`docker run -v /:/mnt ...`のような操作でホストのファイルシステム全体にアクセスできる。「低権限」と呼べるかどうか、自分には判断できない。

userDのケースがRootlessモードに相当するのかもしれないが、これも自分には正確なところがわからない。

---

## Rootlessモードというものがある

調べると、Dockerには2019年からRootlessモードというものが導入されているらしい。Dockerデーモン自体を一般ユーザーとして動かす仕組みで、dockerグループへの追加も`sudo`も不要になるという。

これが設計上の正解なのかもしれないが、多くのインストール手順やプロダクトがRootlessモードを前提としておらず、依然としてdockerグループへの追加が一般的な方法として紹介されている。なぜそうなっているのかは、自分にはよくわからない。

---

## OpenClawというプロダクトを使おうとして気づいた違和感

TelegramとClaudeを連携させるOpenClawというプロダクトを試した。セットアップスクリプトを実行したところ、`sudo`なしでの実行を前提としているようで、dockerグループへの追加が実質的に必要になった。

OpenClawはインターネット経由で外部と通信するサービスであり、プロンプトインジェクション攻撃への脆弱性も指摘されている。「自宅だから安全」と言い切れる状況ではないのかもしれないと感じた。

ただしこれが本当に問題なのか、どの程度のリスクなのか、自分には正確に判断できない。詳しい人の意見を聞きたいところだ。

---

## 世の中の実態はよくわからない

多くのエンジニアがdockerグループへの追加をそのまま採用しているように見えるが、それがリスクを理解した上での判断なのか、単に深く考えていないだけなのかは、外からではわからない。

オープンソースの世界では個人の思想や慣習が積み重なって現在の形になっているものが多く、設計上の不整合がそのまま普及してしまうことがあるのかもしれない。ただこれも自分の印象に過ぎず、実際のところはわからない。

---

## おわりに

結局のところ、何が正解なのかは自分にはわからない。Rootlessモードが正解なのか、専用ユーザーを作るべきなのか、そもそもこれが問題なのかどうかすら、自信を持って言えない。

ただ「広く普及している手順だから安全」とも思えなかった。何かおかしい気がするが、何がおかしいのかを正確に言語化できないままでいる。考えすぎか？


## 参考：Docker公式ドキュメント
[Linux post-installation steps for Docker Engine](./https://docs.docker.com/engine/install/linux-postinstall/)より抜粋

以下、「dockerコマンドの前にsudoを付けたくない場合は、」との希望条件の記載があるのを見るに、`sudo docker`が正規の実行のようにも見えるのだが...

```
The Docker daemon binds to a Unix socket, not a TCP port.
By default it's the root user that owns the Unix socket,
and other users can only access it using sudo.
The Docker daemon always runs as the root user.

If you don't want to preface the docker command with sudo,
create a Unix group called docker and add users to it.

WARNING:
The docker group grants root-level privileges to the user.

NOTE:
To run Docker without root privileges, see
Run the Docker daemon as a non-root user (Rootless mode).
```

```
(日本語訳)  
DockerデーモンはTCPポートではなくUnixソケットにバインドします。
デフォルトではrootユーザーがそのUnixソケットを所有しており、
他のユーザーはsudoを使った場合にのみアクセスできます。
Dockerデーモンは常にrootユーザーとして動作します。

dockerコマンドの前にsudoを付けたくない場合は、
dockerという名前のUnixグループを作成し、ユーザーを追加してください。

警告：
dockerグループはユーザーにroot相当の権限を与えます。

注記：
root権限なしでDockerを動かしたい場合は、
Rootlessモードを参照してください。
```

*作成日：2026年3月*  

