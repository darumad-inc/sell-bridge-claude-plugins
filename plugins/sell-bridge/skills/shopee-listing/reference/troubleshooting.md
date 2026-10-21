# 作成で失敗したとき・作成後に直すとき

`shopee-listing` スキルの Step 8 以降で、サブエージェントが失敗を返したとき、または作成済みの出品に
誤りが見つかったときに読む。どの場合も**同じ create を同じ入力で送り直さない**（例外はレート制限だけ）。

## まず「create が送られたか」を見る

サブエージェントの result ファイルの `step` と `create_sent` を読む。

| result | 意味 | 次の行動 |
| --- | --- | --- |
| `create_sent: false` | create の前（画像の準備・アップロード）で止まった。Shopee に出品は作られていない | 原因を直し、同じストアのサブエージェントを起動し直してよい |
| `create_sent: true` かつ `rate_limited: true` | create はレート制限で拒否され、実行されていない | 下の「レート制限」に従う |
| `create_sent: true` でそれ以外の失敗 | create が Shopee まで届いた可能性がある | 下の表で文言を見分ける。分からなければ作り直さない |

create が届いた後のやり直し（下の表で「やり直してよい」とされた場合だけ）は、前の result を残したまま、
指示書を書き直して `result_path` を `result_<submission_id>_<ストア>_r2.json` のような新しいパスにする。
サブエージェントは、既存の result が「create が届いた可能性あり」を示していると止まるようになっている。
Step 9 の照合では、そのストアの最新の result を使う。

## レート制限

- `Rate limit exceeded for the <window> window. Retry after Ns.`: そのツールは実行されていない。
  N 秒以上たってから、同じストアのサブエージェントに**同じ呼び出しを1回だけ**送り直させる
  （SendMessage で同じサブエージェントを再開する。使えなければ、同じ指示書で起動し直す。
  result ファイルに `create_sent: true` かつ `rate_limited: true` と書かれていることを先に確かめる）。
  待つあいだは、次の ASIN の画像確認など、書き込みの無い作業を進めてよい。
  2回目も拒否されたら、そのストアを止めて報告する。
- `Rate limit service is unavailable`: 送り直さない。ランを止め、時間を置いてやり直すよう出品者に伝える。
- 台湾（TW）のストアは書き込みの反映が数分遅れることがある。429 が続くときは、そのストアへの呼び出しの間隔を空ける。

## 422 が返ったとき

| エラー | 意味 | 次の行動 |
| --- | --- | --- |
| `ERR_VALIDATION_FAILED`（`issues[]` 付き） | prepare のルール（オプション画像・オプション名・ギャラリー・仕入原価・ASIN）に反している。何も作られていない | create を繰り返さない。`issues[].code` は prepare と同じなので、入力を直して `prepare_listing` からやり直す |
| `PURCHASE_PRICE_INVALID` | `purchase_price` が無いか0以下。何も作られていない | `create_request` を書き換えていないか確かめる。prepare の応答には必ず入っている |
| `ERR_VARIATION_TEXT_TOO_LONG` など文字数 | 何も作られていない | 文言を直して prepare からやり直す |

## タイ（TH）の分割払い

タイのストアでクレジットカード分割払いが有効だと、Shopee は 1000 THB 未満のモデルを持つ出品を拒否する。
SellBridge の作成処理がこれを吸収するので、このスキルは価格を嵩上げしたり分割払いを切り替えたりしない。
それでも create が `The price must be at least 1000.00 to enable CC installment` で失敗したら、
送り直さず、「作りかけの確認」をしてから報告する。

## 502・タイムアウトのとき

502 は**エラー文言で**2種類に分かれる。

| 文言 | 何が起きたか | 次の行動 |
| --- | --- | --- |
| `failed to add base item: ...` | 親アイテムを作る**前**に Shopee が拒否した。何も作られていない | 原因を1つ直す（多いのは配送チャネルの寸法・重量オーバー。`Channel detail` に出たチャネルを外す）。`list_shopee_items(item_status="UNLIST")` で新しい item が増えていないことを確かめてから、prepare からやり直す |
| `failed to init tier variation` など、上以外 | 親アイテムを作った**後**で失敗した可能性がある | 作り直さない。下の「作りかけの確認」をして報告する |
| タイムアウト・応答なし | create が届いたかどうか分からない | 作り直さない。下の「作りかけの確認」をして報告する |

### 作りかけの確認

1. `get_shopee_shop_info(provider_id)` で、そのストアが `NORMAL` のままか確かめる。
   `FROZEN` / `BANNED` になっていたら、そのストアをこのセッションの出品先から外す。
2. `list_shopee_items(provider_id, item_status="UNLIST", offset=0, page_size=20)` を呼び、Step 8 で控えた
   `total_count` と比べる。知らない新しい item_id があれば、`get_shopee_item` で中身（親 SKU、
   バリエーションの有無）を読む。
3. 見つけた item_id と状態を報告し、どうするかを出品者に聞く。**自分で削除しない・作り直さない。**

## 作成後に直すとき

作成済みの出品を直すときは、**何を直すかを出品者に確かめてから**、更新のツールを使う。削除して作り直さない。
削除すると item_id が変わり、レビュー・リンク・広告が切れ、SKU の割れも生む。

| 直したいもの | 手段 |
| --- | --- |
| 商品名・説明・`item_sku` | `update_item`（1ストアずつ。応答のフィールドごとの success を確かめる） |
| 価格 | `update_item_price`（バリエーションは `get_shopee_item_models` でそのストアの model_id を引く） |
| 在庫 | `update_stock` |
| 全ストアの SKU をそろえる | `migrate_canonical_sku`（応答の `completed` を必ず読む） |
| 画像・寸法・配送 | SellBridge または Shopee の画面で直すよう出品者に伝える |

- **削除した出品が持っていた SKU は再利用できる。** SKU の割れを直すときに、削除済みの出品の SKU を理由に候補から外さなくてよい。
  それでも `delete_item` はこのスキルでは使わない。
- どの更新も、応答だけで成功とみなさない。`get_shopee_item` / `get_shopee_item_models` で Shopee 上の実体を読み直して確かめる。
- 台湾（TW）は反映が数分遅れることがある。直後に古い値が見えても、数分待って読み直す。送り直しは1回まで。

## 公開について

このスキルは公開しない。出品者が公開しようとして「ショップの出品枠の上限に達した」と拒否された場合、
UNLIST の出品はそのまま残るので作り直す必要はない。出品枠を数えるときは `NORMAL` に加えて `REVIEWING` の件数も足す。
