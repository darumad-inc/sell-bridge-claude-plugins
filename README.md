# sell-bridge-claude-plugins

SellBridge 公式の Claude Code プラグインのマーケットプレイスです。

ここには `sell-bridge` プラグインが 1 つ入っています。インストールすると、次の 3 つがまとめて使えるようになります。

- **SellBridge MCP サーバーへの接続**（`https://mcp.sell-bridge.com/mcp`）。MCP サーバーを手で登録する必要はありません
- **Shopee 出品スキル** `shopee-listing`（ASIN から Shopee の出品を UNLIST で作る）
- **出品用サブエージェント** `shopee-store-lister`（1 ストア分の出品作成と、Shopee 上の実体の検証を行う）

出品は常に非公開（UNLIST）で作られます。公開は SellBridge の画面から行ってください。

## 必要なもの

- SellBridge のアカウント。一部のツールはご契約プランによって使えない場合があります（プランごとの対応は確認中です）
- Claude Code（Claude デスクトップアプリの「Code」タブ、または CLI）

## インストール

### 1. マーケットプレイスを追加する

Claude Code で次のコマンドを入力します。

```
/plugin marketplace add darumad-inc/sell-bridge-claude-plugins
```

### 2. プラグインをインストールする

次のコマンドを入力します。

```
/plugin install sell-bridge@sell-bridge
```

手順 1 と 2 は次の 1 行でまとめて行える場合があります（対応バージョンは確認中です。動かないときは手順 1 と 2 を順に実行してください）。

```
/plugin install sell-bridge --marketplace darumad-inc/sell-bridge-claude-plugins
```

デスクトップアプリでは、マーケットプレイスを追加したあと **+ → Plugins** から `sell-bridge` を選んでインストールすることもできます。

### 3. SellBridge にログインする

インストール後、SellBridge のツールを初めて使うときに、ブラウザで SellBridge のログインと連携の許可を求められます。画面の指示に従って許可してください。

ログイン画面が開かない場合は、`/mcp` を入力し、一覧から SellBridge のサーバーを選んで認証してください（一覧での表示名は確認中です）。

## 使い方

Claude Code で、出品したい ASIN（またはリサーチ候補ID）を添えて頼みます。

```
/sell-bridge:shopee-listing B0XXXXXXXX B0YYYYYYYY
```

「この ASIN を Shopee に出品して」と書くだけでもスキルが使われます。

始める前に、ASIN ごとに次の値を用意してください。Amazon の表示値ではなく、実際に確かめた値を使います。

- 販売1単位あたりの仕入原価（円）
- 販売1単位あたりの梱包重量（グラム、実測）。バリエーションはオプションごと
- 梱包の寸法（cm）、初期在庫、セット販売なら1セットの個数

スキルは次の順で進みます。

1. 出品先のストアと、Shopee 上のストアの状態を確かめる（凍結などのストアには出品しない）
2. Amazon の商品情報と画像を取り、画像を目で確かめて、載せてはいけない画像を外す
3. コンプライアンス、カテゴリ、必須属性、ブランド、配送方法をストアごとに決める
4. 各ストアの言語で商品名と説明を書き、SellBridge の事前チェック（価格の計算を含む）を通す
5. ストアごとに出品を UNLIST で作り、Shopee 上の実体を確かめる
6. 在庫監視に乗ったかと、手で割引に入れる必要がある商品を報告する

作業の記録は、Claude Code を開いているフォルダの `shopee-listing/<日付>/` に残ります。

- 価格はあなたの SellBridge の手数料設定からサーバーが計算します。手数料設定が未設定だと出品は止まります
- 1つの ASIN に数分かかります（SellBridge の書き込みは毎分20回までです）。1回の会話で出品する ASIN は1つにしてください
- 同じ ASIN を2つの会話で同時に出品しないでください

## 更新

プラグインは `main` ブランチの内容がそのまま配信されます。最新版にするには、マーケットプレイスの一覧を更新してからプラグインを更新します。

```
/plugin marketplace update sell-bridge
/plugin update sell-bridge@sell-bridge
```

更新は Claude Code を再起動すると反映されます。

## データの置き場所

出品の作業ファイル（商品の仕様、価格の結果、出品の記録など）は、プラグインのフォルダではなく、作業しているフォルダに置いてください。プラグインのフォルダは更新のたびに置き換わります。

## ライセンス

All rights reserved. 詳しくは [LICENSE](LICENSE) を参照してください。
