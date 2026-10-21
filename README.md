# sell-bridge-claude-plugins

SellBridge 公式の Claude プラグインのマーケットプレイスです。

ここには `sell-bridge` プラグインが 1 つ入っています（[plugins/sell-bridge](plugins/sell-bridge)）。
SellBridge コネクタと Shopee 出品スキル `shopee-listing` がまとめて入り、claude.ai のチャット（ブラウザ・デスクトップアプリ）、Cowork、Claude Code で使えます。
使い方と送るデータは [plugins/sell-bridge/README.md](plugins/sell-bridge/README.md) にあります。

出品は常に非公開（UNLIST）で作られます。公開は出品者が行います。

## インストール

### claude.ai・デスクトップアプリ（チャット・Cowork）

**Customize > Plugins > Add** から追加します。

- マーケットプレイスとして追加する: **Add marketplace** にこのリポジトリの URL を入れ、`sell-bridge` を追加します
- ファイルから追加する（試験用）: `plugins/sell-bridge` フォルダを zip にして **Upload plugin** で選びます

```
cd plugins && zip -r ../sell-bridge.zip sell-bridge -x '*.DS_Store'
```

追加したら、プラグインの **Connectors** タブで SellBridge に接続します。
使うときは、チャットの入力欄で `/` を打って `sell-bridge:shopee-listing` を選び、ASIN を添えて送ります。

### Claude Code

```
/plugin marketplace add darumad-inc/sell-bridge-claude-plugins
/plugin install sell-bridge@sell-bridge
```

claude.ai のアカウントで追加したプラグインは、Claude Code にも同期されます（次のセッションの開始時）。
使うときは `/sell-bridge:shopee-listing B0XXXXXXXX` と入力します。SellBridge への接続は `/mcp` から行えます。

## 更新

プラグインは `main` ブランチの内容がそのまま配信されます。変えたら `plugin.json` と `marketplace.json` の `version` を上げてください。

Claude Code で最新版にするには、次を実行してから Claude Code を再起動します。

```
/plugin marketplace update sell-bridge
/plugin update sell-bridge@sell-bridge
```

## 作りの前提

- スキルはチャットだけで最後まで動くように書いています。チャットはサブエージェント（`agents/`）・hooks・ローカルのファイルを使えないため、どれにも頼りません（[Plugin feature support across platforms](https://claude.com/docs/plugins/platform-support)）
- `.mcp.json` の URL は、claude.ai のコネクタ一覧に出ている SellBridge コネクタと同じにしています。両方を入れた人にも、ツールは1組だけ見えます

## ライセンス

All rights reserved. 詳しくは [LICENSE](LICENSE) を参照してください。
