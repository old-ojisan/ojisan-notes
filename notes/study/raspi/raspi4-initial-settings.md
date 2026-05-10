# AIが初学者に教える、ラズパイの初期設定で行った方がいいもの

> ⚠️ おことわり：
このページは、構築に不慣れなおじさんがAIに質問を繰り返し壁打ちした内容をベースにまとめたものです。
偉そうに解説・説明する文面が含まれますが、AI様が勝手に言ってることですので、あらかじめご了承ください。
なるべく裏付けをとるようにしましたが、正確でない情報が含まれる可能性もあります。

## 概要

この文書は、Raspberry Pi Imager(以下、Imager)でセットアップした後の設定と、対応方針をどうするかをまとめたものです。

- Imagerで設定できるか、できないか
- Imagerでセットアップ直後だとどういった状態か
- デフォルト状態で有効になっていない場合、それはなぜか

なお、この記事に登場するRaspberry Piは、MacBookのVS Codeから接続し、GitHub Codespacesの代替として利用することを想定した機器である。

---

## 進捗サマリー
<!-- TODO: sudo NOPASSWORDの追記は保留 -->
| 項目 | 状態 |
|------|------|
| 1. SSHのパスワード認証を公開鍵認証に変更する | VS Code設定時に実施 |
| 2. rootログインの無効化 | 完了 |
| 3. ファイアウォール（ufw）の設定 | 完了 |
| 4. タイムゾーンの設定（Asia/Tokyo） | 完了 |
| 5. ロケール・キーボードの設定 | 見送り(英語のまま) |
| 6. ホスト名の確認・変更 | 完了 |
| 7. 不要なサービスの無効化 | 完了 |
| 8. 固定IPアドレスの設定 | 見送り |
| 9. Wi-FiよりLANケーブル接続を優先する設定 | 見送り |
| 10. 自動アップデートの設定 | 完了 |
| 11. ログローテーションの確認 | 完了 |
| 12. 基本ツールのインストール | 完了 |

---

## セキュリティ系
---

### 1. SSHのパスワード認証を公開鍵認証に変更する

**背景**
パスワード認証はブルートフォース攻撃に弱い。公開鍵認証は数学的な鍵のペアで認証するため総当たり攻撃が事実上不可能。

**Imager対応状況**
最新のImagerの「サービス」タブで「公開鍵認証のみを許可する」を選択でき、その画面内で鍵ペアの生成も可能。Mac側にすでに鍵ペアが存在する場合は自動的に読み込まれる。

**デフォルトで設定されていない理由**
公開鍵の生成・管理はある程度の知識が必要なため。ラズパイは教育・ホビー用途が多く、最初から高いセキュリティを強制すると入門の障壁になるという設計思想。

**実施結果**
- Mac上のVS CodeからRaspberry PiにSSH接続するタイミングでRaspberry Pi専用のSSH鍵作成と鍵の送信を行う
- 手順は上記作業手順にて記載
- ラズパイ側でパスワード認証無効化（sshd_configの編集）は行わない方針とする

---

### 2. rootログインの無効化

**背景**
rootはすべての操作が許可された最強のユーザー。万が一突破された場合にシステム全体を乗っ取られる。

**Imager対応状況**
Raspberry Pi OSのデフォルトはrootにパスワードが設定されておらず、SSHでのrootログインもできない状態。実質的に対応済み。ただし設定ファイルで明示的に禁止しているわけではないため確認は必要。

**デフォルトで設定されていない理由**
rootパスワードなし＝事実上ログイン不可という設計で、明示的な禁止設定なしでも安全とみなされているため。

**実施結果**

`sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup`でバックアップした後に、`sudo vi /etc/ssh/sshd_config`で編集し、PermitRootLoginの設定を修正

変更前：
```
#PermitRootLogin prohibit-password
```
変更後：
```
PermitRootLogin no
```

保存後に、`sudo systemctl restart ssh`でSSHサービスを再起動、再起動後に`sudo grep PermitRootLogin /etc/ssh/sshd_config`で設定反映を確認



---

### 3. ファイアウォール（ufw）の設定

**背景**
外部からの不要な通信をブロックする仕組み。必要なポートだけ開放し、それ以外を遮断することで攻撃の入口を最小限にできる。

**Imager対応状況**
対応不可。ufwはインストールされていない。

**デフォルトで設定されていない理由**
「どのポートを開けるか」はユーザーの用途次第で決まるため、用途が決まっていない段階では設定できない。また家庭内LANであればルーターがある程度の防壁になるため必須とまではみなされていない。

**実施結果**

前提：
1. 自宅内は192.168.0.0/24であると仮定
2. ルータは192.168.0.1であると仮定

実施内容：

- `which ufw` でufwが未インストールであることを確認
- `sudo nft list ruleset` でメモリ上にルールが何も読み込まれていないことを確認
- `cat /etc/nftables.conf` で骨格のみでルールなしであることを確認
- `sudo systemctl is-enabled nftables`で`nftables.service` がdisabledであることを確認
- 現状はファイアウォールが実質的に機能していない状態であることを把握済み
- ファイアウォール戦略を以下の通り決定：
  - INPUT：
     - デフォルト全拒否。初期はSSH(22番)のみ開放。今後の用途に応じて追加する。
  - OUTPUT・自宅内LAN向け：
     - 自宅内の他の機器への横移動を防ぐため(ラテラルムーブメント防止)、明示的に拒否。ただしDNS・DHCP・avahi/mDNSは例外として許可。
     - DHCP・avahi/mDNSは許可不要でも良さそうだが、ヘッドレス運用のため念の為許可。
  - OUTPUT・インターネット向け：
     - 現時点では全許可。
- `sudo apt install ufw` でufwをインストール
- `sudo ufw default deny incoming` でINPUTのデフォルトを全拒否に設定
- `sudo ufw default allow outgoing` でOUTPUTのデフォルトを全許可に設定
- `sudo ufw allow in 22` でSSH（ポート22）を許可 ※ `in` は省略可能
- `sudo ufw allow out to 192.168.0.1 port 53` でDNS（TCP）を許可
- `sudo ufw allow out to 192.168.0.1 port 53 proto udp` でDNS（UDP）を許可
- `sudo ufw allow out to 192.168.0.1 port 67 proto udp` でDHCP（UDP）を許可
- `sudo ufw allow out to 192.168.0.255 port 5353 proto udp` でavahi/mDNS（UDP）を許可
- `sudo ufw deny out to 192.168.0.0/24` で自宅内LANへのOUTPUTを拒否
- ※ 許可ルールは必ずDENY OUTルールより前に追加すること
- `sudo ufw enable` でufwを有効化
- `sudo ufw status numbered` でルールの順番を確認済み
- `ping 8.8.8.8` でインターネットへの通信が正常であることを確認
- `ping 192.168.0.1` で自宅内LAN（ルーター）への通信が拒否されていることを確認
- `ping deb.debian.org` でDNS名前解決が正常であることを確認
- 再起動後にMacからSSH接続できることを確認済み

    >このように設定したところで、DockerがiptablesをバイパスしてufwのINPUT全拒否設定が意図通りに機能しない結果も想定される。リスクを承知の上で解決策を実施しないことを意図的に選択。自宅内LAN環境でのリスクレベルを踏まえた判断。


**補足事項**

ファイアウォールの階層構造
```
netfilter（Linuxカーネルの本丸）
    ↑
nftables・iptables（カーネルを操作するツール）
    ↑
ufw・firewalld（さらに簡単に操作するツール）
```
ufwはnftablesの「上位フロントエンド」であり、「原始的・高度」という関係ではなく「下位・上位」という階層関係。OSとしては下位のnftablesだけを用意しておき、ufwを使うかどうかはユーザーの判断に委ねるという設計思想。

DockerとufwのINPUT方向の問題

DockerはポートをPublishするとiptablesを直接操作するため、ufwのINPUT全拒否設定をバイパスしてしまう。つまりufwで意図通りにポートを制限したつもりでも、Dockerがマップしたポートは外部に開いてしまう。失敗談として、ufwでブロックしていたはずのポートがShodanなどの検索エンジンで外部に公開されていることが発覚したケースや、知らない間にサービスがWorldwideに公開されていたケースが報告されている。

参考記事：
- https://yoshinorin.net/articles/2022/02/18/docker-through-ufw/
- https://www.ryotosaito.com/blog/?p=492
- https://pc.atsuhiro-me.net/entry/2023/05/01/193433
- https://turgenev.hatenablog.com/entry/2024/02/27/015827
- http://hogetan.net/note/diary/20240531_docker_iptables.html

---

## システム系

---

### 4. タイムゾーンの設定（Asia/Tokyo）

**背景**
タイムゾーンがズレているとログの時刻が実際と異なり、トラブル発生時の原因特定が困難になる。

**Imager対応状況**
「ロケールを設定する」の項目でAsia/Tokyoを設定できる。

**デフォルトで設定されていない理由**
地域によって異なるためデフォルト値を一意に決められない。ユーザー個々の環境依存の設定。

**実施結果**
- `timedatectl` でAsia/Tokyo (JST, +0900)を確認済み

---

### 5. ロケール・キーボードの設定

**背景**
言語・文字コード・日付形式などの地域設定。ズレていると日本語が文字化けしたり予期しない動作の原因になる。

**Imager対応状況**
Imagerで設定を行っても反映されないバグが報告されている。

**デフォルトで設定されていない理由**
地域依存の設定のため。ただしImagerのバグにより設定が反映されないケースがあることに注意。

**実施結果**
- `locale` でen_GB.UTF-8であることを確認済み
- SSHでのコマンド操作が中心の用途では英語ロケールのままでも実用上の問題はほとんどない。むしろエラーメッセージが英語のままの方がWeb検索で解決策を探しやすいため、英語のまま運用することに決定

---

### 6. ホスト名の確認・変更

**背景**
同じLAN内に複数のラズパイがある場合や、ログを見たときにどのマシンか識別するために重要。

**Imager対応状況**
「一般」タブでホスト名を設定できる。

**デフォルトで設定されていない理由**
デフォルト値（raspberrypi）は存在するが、ユーザーが任意の名前をつけるべきものなのでImagerで設定を促す設計になっている。

**実施結果**
- Imagerで設定した通りのホスト名になっていることを確認→ 対応済み

---

### 7. 不要なサービスの無効化

**背景**
起動時に自動で立ち上がるサービスが多いほどメモリ・CPUを無駄消費し、攻撃の入口も増える。ラズパイ4はリソースが限られるため特に意識したい。

**Imager対応状況**
対応不可。どのサービスを無効化するかImagerで指定不可。

**デフォルトで設定されていない理由**
「何が不要か」はユーザーの用途次第で異なるため、OSとしてデフォルトで無効にすることはできない。用途が決まってから判断する性質の設定。

**実施結果**
- `systemctl list-unit-files --type=service | grep enabled` で自動起動サービスの一覧を確認済み

**保留事項**
- ❌のものを無効化した
- 停止は以下のコマンドで実行する

    >実行中のサービスを停止する  
    >`sudo systemctl stop <サービス名>`
    >
    >自動起動を無効化する  
    `sudo systemctl disable <サービス名>`
    >
    >状態を確認する  
    >`sudo systemctl is-enabled <サービス名>`  
    >`sudo systemctl is-active <サービス名>`  
    >
    > `disabled`と`inactive`が表示されれば完了です。

**補足事項**

必須・無効化してはいけないもの
- **ssh.service／sshd-keygen.service／sshswitch.service**：SSH接続に必要。これを無効にするとリモート接続できなくなる。
- **NetworkManager.service／NetworkManager-dispatcher.service／NetworkManager-wait-online.service**：ネットワーク接続の管理全般。無効にするとネットワークが使えなくなる。
- **wpa_supplicant.service**：Wi-Fi接続の認証管理。Wi-Fiを使う場合は必須。
- **cron.service**：定期的なタスクのスケジューラ。OS内部でも使われている。
- **systemd-timesyncd.service**：時刻の自動同期（NTP）。ログの正確な時刻管理に必要。
- **apparmor.service**：セキュリティの強制アクセス制御。無効にするとセキュリティが下がる。
- **regenerate_ssh_host_keys.service**：初回起動時にSSHのホスト鍵を生成する。
- **keyboard-setup.service／console-setup.service**：キーボードとコンソールの初期設定。
- **getty@.service**：コンソール（画面）へのログイン窓口。
- **rpi-eeprom-update.service**：ラズパイのファームウェア更新管理。ハードウェア固有の必須サービス。
- **systemd-pstore.service**：クラッシュ時のログ保存。
- **e2scrub_reap.service**：ファイルシステムの整合性チェック補助。
- **udisks2.service**：USBメモリなどのストレージデバイス管理。

GUIデスクトップ用途向けで、サーバー用途では不要なもの
- ❌ **lightdm.service**：グラフィカルなログイン画面（デスクトップ環境）の管理。画面を使わないなら不要。
- ❌ **wayvnc-control.service**：VNCリモートデスクトップの制御。GUIを使わないなら不要。
- ❌ **glamor-test.service**：GPU（グラフィック）のテスト用サービス。デスクトップ不使用なら不要。
- ❌ **cups.service／cups-browsed.service**：プリンター管理。プリンターを使わないなら不要。
- ❌ **accounts-daemon.service**：GUIのユーザーアカウント管理。デスクトップ環境向けで、CLIのみなら不要。

用途次第で判断が分かれるもの
- **avahi-daemon.service**：`<ホスト名>.local` でSSH接続するために使っている場合は必要。使っていなければ不要。
- ❌ **bluetooth.service**：Bluetoothデバイスを使わないなら不要。セキュリティ上の観点からも無効化が望ましい。
- ❌ **cloud-init系**（cloud-config／cloud-final／cloud-init-local／cloud-init-main／cloud-init-network）：クラウド環境（AWSなど）向けの初期設定サービス。家庭内ラズパイには不要。
- ❌ **ModemManager.service**：モバイル回線（SIMカード）の管理。使わないなら不要。
- ❌ **rpcbind.service／nfs-blkmap.service**：ネットワーク越しにファイル共有（NFS）をする場合に必要。使わないなら不要。
- ❌ **rp1-test.service**：ラズパイ5向けのI/Oコントローラ（RP1チップ）用サービス。ラズパイ4では不要。

---

## ネットワーク系

---

### 8. 固定IPアドレスの設定

**背景**
通常ルーターはDHCPでIPを毎回変える可能性がある。IPが変わるとSSH接続先やControl UIのアクセス先も変わり不便。

**Imager対応状況**
対応不可。

**デフォルトで設定されていない理由**
IPアドレスはネットワーク環境によって異なるためOSが決めることができない。またルーター側でMACアドレスによるDHCP固定割り当てという別の手段もあり、OS側設定だけが唯一の方法ではない。

**実施結果**
- `<ホスト名>.local` でSSH接続できることを確認済み
- avahi-daemon.serviceが機能しているためIPアドレスが変わっても接続先が変わらないとして対応不要と判断

    > ヘッドレス運用のため、念の為DHCPの許可ルールをufwに追加したが、固定IPにすることでこのルールが不要になりufwの設定がシンプルになるメリットがある。それでも今回はDHCP運用を行うこととし、固定IP化は見送る。

---

### 9. Wi-FiよりLANケーブル接続を優先する設定

**背景**
Wi-Fiと有線LANを両方使える場合、OSがどちらを優先するか明示しておかないと意図しない方が使われることがある。常時稼働サーバー用途では有線が安定。

**Imager対応状況**
対応不可。

**デフォルトで設定されていない理由**
有線・無線の優先度はユーザーの環境や用途次第であり、OSがデフォルトで決める性質のものではない。

**実施結果**
- `nmcli connection show` でethernetとwifiの存在を確認
- `nmcli -f NAME,TYPE,AUTOCONNECT-PRIORITY connection show` で優先度がどちらも0であることを確認
- LANケーブル接続での運用予定はないため設定は見送り

**補足事項**

設定方法

方法1：LANケーブルが刺さっている時は有線優先、刺さっていない時はWi-Fiで接続する
```
sudo nmcli connection modify "<有線のプロファイル名>" connection.autoconnect-priority 100
```
LANケーブル接続時に実行する。プロファイル名はnmcli connection showのNAME列の値を使う。

方法2：LANケーブルのみで運用し、Wi-Fiを無効にする
```
sudo nmcli connection modify "<Wi-Fiのプロファイル名>" connection.autoconnect no
```
Wi-Fiを完全に削除するわけではないため、必要時に手動で有効に戻せる。

---

## 運用系

---

### 10. 自動アップデートの設定

**背景**
脆弱性は日々発見される。手動アップデートを忘れると古い脆弱性が放置される。ただしアップデートには「セキュリティパッチ」「パッケージのアップデート」「OSバージョンアップ」の3種類があり、すべてを自動化するのは適切ではない。常時稼働サーバー用途では「セキュリティパッチのみを自動適用し、それ以外は手動で管理する」が最も適切な設計。

**Imager対応状況**
対応不可。

**デフォルトで設定されていない理由**
自動アップデートはタイミングによって予期しない動作変更やサービス停止を引き起こす可能性がある。サーバー用途では慎重に扱うべき設定のため全員に強制するのが適切でないとみなされている。

**実施結果**
- `dpkg -l unattended-upgrades` でunattended-upgradesがインストール済みか確認→ 未インストールを確認
- `sudo apt install unattended-upgrades` でインストール
- `sudo cat /etc/apt/apt.conf.d/50unattended-upgrades` で自動更新対象の設定ファイルを確認
- `origin=Debian,codename=${distro_codename},label=Debian` をコメントアウトしてセキュリティパッチのみが対象になるよう編集
- `sudo cat /etc/apt/apt.conf.d/20auto-upgrades` で自動実行の設定ファイルを確認
- `APT::Periodic::Update-Package-Lists "1"` と `APT::Periodic::Unattended-Upgrade "1"` がすでに有効になっていることを確認→ 追加設定不要
- 毎日パッケージリストの更新とセキュリティパッチの自動適用が行われる状態になっていることを確認→ 対応済み

**補足事項**

アップデートの3つの分類
1. **セキュリティパッチ**：OSやインストール済みパッケージの脆弱性を修正するもの。影響範囲は小さく動作への影響も最小限。自動化の対象として最も適している。
2. **パッケージのアップデート**：カーネルやファームウェアを含むインストール済みソフトウェアの最新化。動作に影響が出る可能性があるため、自動化するかどうか慎重に判断が必要。
3. **OSバージョンアップ**（例：bullseyeからbookwormへ）：手動で行うものであり、自動化の対象には含めない。

---

### 11. ログローテーションの確認

**背景**
常時稼働でログが蓄積しストレージを圧迫する可能性がある。古いログを自動削除・圧縮する仕組みが正しく動いているかの確認。

**Imager対応状況**
対応不可。ただし両系統ともデフォルトで動作しているため、新規設定は基本不要で「確認」が目的。

**デフォルトで設定されていない理由**
両系統ともデフォルトで動作している。journaldはデフォルトでRAM上に揮発性保存する設定のため、サーバー用途では保存先をストレージに変更するかどうかを用途に応じて判断する必要がある。

**実施結果**
- journald・logrotate両系統の設定を確認済み
- journaldは揮発性のまま運用することに決定（microSDの消耗を抑えるための判断）
- logrotateは週次ローテーション・4世代保持で正常動作中を確認→ 対応済み

**補足事項**

重要な前提：現在のRaspberry Pi OS（bookworm）では従来のsyslogが廃止されjournaldが標準のログ管理に変わっている。そのためログ管理は「journaldが管理するログ」と「logrotateが管理するテキスト形式のログ（/var/log/auth.logなど）」の2系統が存在する。

確認方法

journald系
```
#ログ保存先を確認 → Storage=volatile(RAMに保存)となっている
sudo systemd-analyze cat-config systemd/journald.conf

#ストレージ保存の場合の保存先に何かログがあったりしないか？→ ない
ls /var/log/journal/
```

logrotate系
```
#logrotateの全体設定を確認
cat /etc/logrotate.conf

#logrotateの個別設定ファイルの一覧
ls -la /etc/logrotate.d/

#logrotateが実際に各ログファイルを最後にローテーションした日時の記録
sudo cat /var/lib/logrotate/status
```

---

### 12. 基本ツールのインストール（git、curl、vimなど）

**背景**
フェーズ2以降の作業でgitやcurlは必ず使う。テキストエディタも設定ファイル編集に必要。事前に揃えておくことでつまずきを防ぐ。

**Imager対応状況**
対応不可。何を入れるかはImagerで指定しない。

**デフォルトで設定されていない理由**
どのツールが必要かは用途次第で異なる。OSに不要なソフトをあらかじめ入れることはセキュリティ・容量の観点から避けるのがLinuxの一般的な設計思想（必要なものだけ入れる）。

**実施結果**
- git・curlは導入済みを確認
- 設定ファイルの修正程度なのでvimは導入しない判断。vi(`/usr/bin/vi`)で十分。

---

## 出典・参考サイト

- https://zenn.dev/tryeverything/articles/a0023_raspberry-pi-ssh
- https://raspida.com/rpi-imager181/
- https://elirlab.com/publickey-ssh/
- https://raspida.com/firewall4raspbian-ufw/
- https://rofumi.net/raspberrypi-ufw-setting/
- https://hitoshiarakawa.com/blogs/2025/2025-01-29_installing-ufw-on-raspberry-pi-os-bookworm/
- https://raspida.com/rpi-imager-advanced-options-issue/
- https://zenn.dev/ishikawa096/articles/f4270ce9439cef
- https://raspida.com/nmcli-static-ipaddress/
- https://www.mikan-tech.net/entry/raspi-unattended-upgrades
- https://raspida.com/rpios-update-upgrade/
- https://blog.treedown.net/entry/2025/02/04/010000
- https://qiita.com/Typhon/items/3508726632c46c295d6b
- https://linux-jp.org/?p=5480
- https://zenn.dev/19931/articles/14473c10e2974f
- https://yoshinorin.net/articles/2022/02/18/docker-through-ufw/
- https://www.ryotosaito.com/blog/?p=492
- https://pc.atsuhiro-me.net/entry/2023/05/01/193433
- https://turgenev.hatenablog.com/entry/2024/02/27/015827
- http://hogetan.net/note/diary/20240531_docker_iptables.html

*作成日：2026年3月*  
*作業環境：MacBook Air M1 / macOS / zsh / Raspberry Pi 4 /  bookworm*