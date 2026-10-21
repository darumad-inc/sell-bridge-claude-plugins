# SellBridge

Amazon の ASIN を伝えるだけで、SellBridge につないだ Shopee の各国ストアに、出品を**非公開（UNLIST）**で作るプラグインです。
claude.ai のチャット（ブラウザ・デスクトップアプリ）、Cowork、Claude Code で使えます。

## 入っているもの

- **SellBridge コネクタ**（`https://mcp.sell-bridge.com/mcp`）。SellBridge のアカウントでログインして使います
- **Shopee 出品スキル** `shopee-listing`

## 使い方

1. プラグインを追加したら、プラグインの **Connectors** タブ（または claude.ai の設定 → コネクタ）で SellBridge に接続し、SellBridge のアカウントでログインします。
2. 新しいチャットで、入力欄に `/` を打って `sell-bridge:shopee-listing` を選び、ASIN を添えて送ります。

   ```
   /sell-bridge:shopee-listing B0XXXXXXXX
   ```

   「ASIN B0XXXXXXXX を Shopee に出品して」と書くだけでも、このスキルが使われます。
3. Claude が商品名・仕入れ値・重さ・梱包サイズ・在庫・出品先を1つの表にして確かめます。実際に量った梱包後の重さを伝えてください。仕入れ値を伝えなければ、amazon.co.jp の現在の価格（カート価格など）を使い、どの価格を使ったかを伝えます。
4. Claude が画像の確認、コンプライアンス、カテゴリと必須属性、各国語の商品名と説明、SellBridge の事前チェック（価格の計算を含む）を行い、ストアごとに出品を作ります。
5. 完了報告で、ストアごとの商品番号・価格・見送ったストアと理由・割引に入れる商品・在庫監視の状態を確かめ、内容を確認してから自分で公開します。

詳しい手順は [Claude と会話して Shopee に出品する](https://help.sell-bridge.com/guides/list-with-claude) をご覧ください。

## 注意

- SellBridge の有料プラン（ベーシック以上）と、SellBridge に接続済みの Shopee ストアが必要です
- 出品は常に非公開（UNLIST）で作られます。Claude は公開しません
- 価格は SellBridge の価格計算の設定（手数料・送料・割引率・目標利益）からサーバーが計算します
- 1つの会話で出品する ASIN は1つにしてください。1つの ASIN に数分かかります（SellBridge の書き込みは毎分20回までです）
- 同じ ASIN を2つの会話で同時に出品しないでください

## 送るデータ

このプラグインは、SellBridge コネクタを通じて、あなたの SellBridge アカウントに次のデータを送ります。ASIN、商品名と説明（各国語）、画像の URL、仕入れ値・重さ・寸法・在庫、出品先のストアとカテゴリ・属性。
SellBridge はこれを使って Amazon の商品情報を取得し、価格を計算し、Shopee に出品を作ります。プラグイン自体は何も保存しません。
SellBridge でのデータの扱いは [SellBridge のプライバシーポリシー](https://sell-bridge.com/privacy) に従います。

## ライセンス

All rights reserved. 詳しくは [LICENSE](LICENSE) を参照してください。
