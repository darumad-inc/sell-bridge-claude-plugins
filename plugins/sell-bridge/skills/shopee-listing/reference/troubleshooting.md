# 作成で失敗したとき・作成後に直すとき

`shopee-listing` スキルの Step 8 以降で、作成が失敗したとき、または作成済みの出品に
誤りが見つかったときに読む。どの場合も**同じ submit を同じ入力で送り直さない**（例外はレート制限だけ）。

## まず「submit が届いたか」を見る

| 状況 | 意味 | 次の行動 |
| --- | --- | --- |
| hero の加工・画像のアップロードで止まった | submit の前で止まった。Shopee に出品は作られていない | 原因を直し、同じストアを Step 8 の 8-4 からやり直してよい |
| submit がレート制限で拒否された | submit は実行されていない | 下の「レート制限」に従う |
| submit がそれ以外のエラー・タイムアウトになった | submit が届いた可能性がある | `get_listing_operation(provider_id, submission_id)` で状態を読む。見つからない（404）ときだけ未送信。分からなければ作り直さない |

`get_listing_operation` が `failed` / `needs_review` を返したときは、`error_code` と `error_message` で下の表を見分ける。
`item_id` が入っていれば Shopee 上に残骸があるので、「作りかけの確認」をしてから出品者に聞く。

## レート制限

- `Rate limit exceeded (<window> limit). Retry after Ns.`: そのツールは実行されていない。
  N 秒以上たってから、**同じ呼び出しを1回だけ**送り直す。
  待つあいだは、次のストアの準備など、書き込みの無い作業を進めてよい。
  2回目も拒否されたら、そのストアを止めて報告する。
- `Rate limit service is unavailable`: 送り直さない。止めて、時間を置いてやり直すよう出品者に伝える。
- 台湾（TW）のストアは書き込みの反映が数分遅れることがある。429 が続くときは、そのストアへの呼び出しの間隔を空ける。

## 422 が返ったとき

| エラー | 意味 | 次の行動 |
| --- | --- | --- |
| `ERR_VALIDATION_FAILED`（`issues[]` 付き） | prepare のルール（オプション画像・オプション名・ギャラリー・仕入原価・ASIN）に反している。何も作られていない | submit を繰り返さない。`issues[].code` は prepare と同じなので、入力を直して `prepare_listing` からやり直す |
| `PURCHASE_PRICE_INVALID` | `purchase_price` が無いか0以下。何も作られていない | `create_request` を書き換えていないか確かめる。prepare の応答には必ず入っている |
| `ERR_VARIATION_TEXT_TOO_LONG` など文字数 | 何も作られていない | 文言を直して prepare からやり直す |

## タイ（TH）の分割払い

タイのストアでクレジットカード分割払いが有効だと、Shopee は 1000 THB 未満のモデルを持つ出品を拒否する。
SellBridge の作成処理がこれを吸収するので、このスキルは価格を嵩上げしたり分割払いを切り替えたりしない。
それでも `The price must be at least 1000.00 to enable CC installment` で失敗したら、
送り直さず、「作りかけの確認」をしてから報告する。

## 502・タイムアウトのとき

502 は**エラー文言で**2種類に分かれる。

| 文言 | 何が起きたか | 次の行動 |
| --- | --- | --- |
| `failed to add base item: ...` | 親アイテムを作る**前**に Shopee が拒否した。何も作られていない | 原因を1つ直す（多いのは配送チャネルの寸法・重量オーバー。`Channel detail` に出たチャネルを外す）。`list_shopee_items(item_status="UNLIST")` で新しい item が増えていないことを確かめてから、prepare からやり直す |
| `failed to init tier variation` など、上以外 | 親アイテムを作った**後**で失敗した可能性がある | 作り直さない。下の「作りかけの確認」をして報告する |
| タイムアウト・応答なし | submit が届いたかどうか分からない | **再送しない。** `get_listing_operation` で状態を読む。それでも分からなければ「要確認」として、下の「作りかけの確認」で `list_shopee_items`（ページ送り）と `get_listing_data` を使って確かめ、結果を報告する |

### 作りかけの確認

1. `get_shopee_shop_info(provider_id)` で、そのストアが `NORMAL` のままか確かめる。
   `FROZEN` / `BANNED` になっていたら、そのストアをこのセッションの出品先から外す。
2. `list_shopee_items(provider_id, item_status="UNLIST", offset=0, page_size=100)` を呼び、Step 8 の 8-1 で控えた
   `total_count` と比べる。1ページ目に新しい item があるとは限らない。`has_next_page` が真の間は応答の `next_offset` で続きを読む。知らない新しい item_id があれば、`get_shopee_item` で中身（親 SKU、
   バリエーションの有無）を読む。
3. `product_id` が分かっているとき（先に作ったストアの結果など）は、`get_listing_data(product_id, provider_id)` で SellBridge に紐付いた出品があるかも見る。
   紐付けが無いと 404 になる。それは「SellBridge に記録が無い」という情報であり、Shopee 上に item が無い証拠ではない。
4. 見つけた item_id と状態を報告し、どうするかを出品者に聞く。**自分で削除しない・作り直さない。**

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
