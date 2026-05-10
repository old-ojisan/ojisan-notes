# ブクマッターを使ったAIケーススタディ


ブクマッターとAIを使ったケーススタディをご紹介します。


## ケース1. マーケットニュースをAIで定期チェック(Claude Code)

最もシンプルな利用方法をサンプルとしてまとめます。   

AIを活用される方は以下のような悩みがあるかもしれません。
- AIがリストアップしたURLがそもそもおかしい
- AIがリストアップしたURLにアクセスしてもうまくいかない

ブクマッターはJSONでブックマークの一覧を公開できます。

- AIに読ませるURLの収集はブラウザで行う。
  - 日々新しいニュースソースを発見したら追加する。
- URLはいちいちAI用のmdファイルに記録しない。
  - ブクマッターのJSONだけをAIには参照させて読み込ませる。
- ブクマッターで他の人が公開しているURLも時には参考に。
  - 自分のブックマークに取り込むことも可能。

### 準備するもの

- ブクマッターにRSSのリストを登録
  - [ブックマークのサンプルページ](https://bukumatter.com/folder/ojisan01/rg75f)
  - [このサンプルのjson](https://bukumatter.com/api/json/ojisan01/rg75f)
- Claude Code
  - 外部のデータ取得はClaude Coworkだと限界がある


### Claude Codeの構成イメージ
- ~/claude_data/checknews
- ~/claude_data/checknews/CLAUDE.md (指示書)
- ~/claude_data/checknews/data/ (データの読み書き場所)

### Claude Codeの指示書イメージ

指示書自体もChatGPTやClaudeに作成させています。用途に応じてAIに修正させるといいでしょう。

```
# newscheckの Claude Code セッション

## 役割
RSSをチェックして投資関連のニュースを収集する

## RSS取得時に工夫することの一例

### アクセス面

- 一部サイトは通常のRSS URLに見えてもHTMLページや404を返すためXMLとして取得できるかを確認。
- `curl -L`を使い、リダイレクトがあるRSSでも最終レスポンスまで追跡した。
- `--max-time`を指定し、応答が遅い・詰まるフィードで処理が止まらないようにした。
- `-A 'Mozilla/5.0'`を付け、User-Agent未指定で弾かれる可能性を下げた。

### パース面

- RSS本文は文字列検索だけで済ませず、Rubyの`REXML`でXMLとしてパースし
- 先頭ニュースは`//item`の最初の要素を取り、そこから`title`と`link`を抽出。
- RSS 2.0形式では`//item/title`と`//item/link`を基本形として扱う。
- 名前空間つきRSSやRDF形式も想定し、必要に応じて`local-name()`で要素名を見られる形を使う。
- XMLパースに失敗した場合は、RSSではなくHTMLや不正なレスポンスが返っている可能性があるものとして扱う。
- `head`で本文の冒頭を確認し、XML宣言・`rss`・`channel`・`item`の有無を目視でも検証。

### 実行例

curl -L --silent --max-time 15 -A 'Mozilla/5.0' \
  'https://xxxxxx.jp/rss/news.xml' |
ruby -r rexml/document -e '
  doc = REXML::Document.new(STDIN.read)
  item = REXML::XPath.first(doc, "//item")
  abort("no item") unless item

  title = REXML::XPath.first(item, "title").text
  link  = REXML::XPath.first(item, "link").text

  puts [title, link].join("\n")
'

## ニュース取得タスク

ニュース取得タスクは次の複数のタスクで構成される。

### ソース確認タスク
- ソースデータ: ブクマッターのhttps://bukumatter.com/api/json/ojisan01/rg75fに記載されるURL一覧
- 日本語のサイトと英語のサイトがあるが(JP)(EN)というサイト名で識別される

### RSS取得タスク
- それぞれのソースにアクセスし、RSSの内容を取得しそのまま保存する
- 概ねニュースのタイトルとニュースのサマリー、参照先のURL、配信日時が含まれていると思われる。
- 保存先: data/newslist_YYYYMMDDHHMM.json
- 形式: JSON

### 重複チェック兼記録タスク
- 参照・保存先: data/newslist.json
- 形式: JSON
- RSS取得タスクで取得したdata/newslist_YYYYMMDDHHMM.jsonと既存のdata/newslist.jsonを重複チェックし、data/newslist.jsonに存在しなければ追記する。

### レポートタスク
- data/newslist.jsonから重要なニュースを5件表示する。
- 英語の場合は英語と日本語訳を併記すること。
- ニュースの日付も表示すること。
- 重要と評価した根拠も表示すること。
```

### Claude Codeでタスク実行

Claude Codeで、ニュース取得タスクを実行して、と依頼すると処理が始まります。  
何度か権限の承認を求められますが、これ外部URLへのアクセスやパースのための承認です。
データの取得が終わると、data/以下にjsonが保存され、チャット上では5件のニュースが表示されます。

新しいニュースソースを見つけた場合は、ブクマッターに追加登録すると良いでしょう。