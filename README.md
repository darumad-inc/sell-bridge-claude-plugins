# sell-bridge-claude-plugins

SellBridge 公式の Claude Code プラグインのマーケットプレイスです。

ここには `sell-bridge` プラグインが 1 つ入っています。インストールすると、次の 3 つがまとめて使えるようになります。

- **SellBridge MCP サーバーへの接続**（`https://mcp.sell-bridge.com/mcp`）。MCP サーバーを手で登録する必要はありません
- **Shopee 出品スキル** `shopee-listing`（準備中）
- **出品用サブエージェント** `shopee-store-lister`（1 ストア分の出品作成と、Shopee 上の実体の検証を行う）

出品は常に非公開（UNLIST）で作られます。公開は SellBridge の画面から行ってください。

## 必要なもの

- SellBridge のアカウント（有料プラン）。無料プランでは MCP のツールがすべて拒否されます
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

Claude Code 2.1.275 以降なら、手順 1 と 2 を次の 1 行でまとめて行えます。

```
/plugin install sell-bridge --marketplace darumad-inc/sell-bridge-claude-plugins
```

デスクトップアプリでは、マーケットプレイスを追加したあと **+ → Plugins** から `sell-bridge` を選んでインストールすることもできます。

### 3. SellBridge にログインする

インストール後、SellBridge のツールを初めて使うときに、ブラウザで SellBridge のログインと連携の許可を求められます。画面の指示に従って許可してください。

ログイン画面が開かない場合は、`/mcp` を入力し、一覧から `plugin:sell-bridge:sell-bridge` を選んで認証してください。

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
