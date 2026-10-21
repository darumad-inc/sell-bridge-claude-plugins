---
name: shopee-listing
description: SellBridge MCP を使い、Amazon の ASIN（またはリサーチ候補）から Shopee の出品を UNLIST（非公開）で作る。画像の目視確認、Shopee 上のストア状態の確認、コンプライアンス判定、カテゴリ・属性・配送の決定、ストアの言語での文言作成、prepare_listing の事前チェックを経て、ストアごとに出品を作り、Shopee 上の実体・在庫監視・割引の要否を確かめて報告する。1回で複数の ASIN を順に処理できる。「この ASIN を Shopee に出品して」「リサーチ候補を出品して」と頼まれたときに使う。公開（listed=true）は行わない。
argument-hint: "<ASIN またはリサーチ候補ID> [...]"
---

# Shopee 出品（UNLIST で作成）

## 対象: $ARGUMENTS

引数は ASIN（英大文字・数字10桁）またはリサーチ候補ID（数字）を、空白区切りで1つ以上。
引数が無ければ、出品したい ASIN か候補IDを出品者に聞く。

出品は SellBridge MCP の公開ツールだけで行える。ツールの説明には、ツールの順番や
「`submission_id` を全ストアで共有する」「認証番号を推測しない」などの前提がすでに書いてある。
このスキルが足すのは**判断と順番**である。どの画像を外すか、どのストアを見送るか、
サブエージェントをどう並べるか、いつ止めて出品者に聞くか。

## 絶対規則

- **`set_listing_status(listed=true)` を呼ばない。** 作るのは常に UNLIST。公開は出品者が SellBridge の画面から行う。
- **`delete_item` を呼ばない。** `unlist_item` / `update_item` / `update_item_price` / `migrate_canonical_sku` も、出品者の確認なしには呼ばない。
- **同じ create を同じ入力で送り直さない。** 例外はレート制限で拒否されたとき（「レート制限」の節）だけ。
- **価格・重量・認証番号（TIS / FDA / BSMI など）・言語・カテゴリ・ブランドを推測しない。** ブランドの一致に自信が無ければ `brand_id: 0`。
- **作成の前に必ず `prepare_listing` を通す。** `status: ready` のストアだけに作り、`stores[].create_request` は画像以外を変えずに送る。価格を書き換えない。自分で価格を計算しない。
- **Shopee 上の状態が `NORMAL` のストアにだけ作る**（Step 0）。
- 作業ファイルに認証情報（API キー、トークン、パスワード）を書かない。
- 作業ファイルはプラグインのフォルダに置かない。プラグインのフォルダは更新のたびに置き換わる。

## Step 0. セッションの準備（最初に1回）

### 0-1. 作業フォルダ

出品者の作業フォルダ（Claude Code を開いているフォルダ）に `shopee-listing/<YYYYMMDD>/` を作り、
このセッションのファイルをすべてそこに置く。以下 `<work>` と書く。ファイルは**絶対パス**で扱う。

### 0-2. 出品先ストアを決める

`list_stores` を呼び、次をすべて満たすストアを候補にする。

- Shopee のストアである
- `listing_language` が付いている（無いストアは言語を推測せず、出品先から外して出品者に伝える）
- `token_status` が `valid` か `expiring`（`expiring` は出品者に「近く連携が切れる」と伝える。`expired` / `invalid` は外し、SellBridge の画面で再連携するよう伝える）

候補を出品者に見せ、出品先にするストアを確かめる（指定が無ければ候補の全部）。
同じリージョンにストアが複数あるときは、ラベルを `<リージョン>-<provider_id>` にして区別する。

### 0-3. Shopee 上のストア状態を確かめる

出品先の**全ストア**で `get_shopee_shop_info(provider_id)` を呼ぶ。
`status` が `NORMAL` 以外のストアと、エラーが返ったストアは、**このセッションの出品先から外す**。
台帳と完了報告に「Shopee 上のストア状態が NORMAL でない（<status またはエラー>）」と残し、残りのストアで続ける。
残りが0になったら止めて出品者に伝える。

なぜ必要か: `list_stores` の `status` / `token_status` は SellBridge との接続状態で、
Shopee 側の凍結や BAN は映らない。凍結中のストアに作ると、Shopee が親アイテムを作ったあとで
バリエーションを拒否し、そのストアでの親 SKU が使えなくなる。作成側の API にもガードはあるが、
状態の取得に失敗したときは素通しし、状態を数分キャッシュするので、保証にはならない。

### 0-4. テンプレートとフレーム（任意）

- 説明文テンプレートを使う出品者なら、`list_listing_templates` / `get_listing_template` で選び、`template_id` を Step 7 で渡す。テンプレートの言語別の中身は `list_stores` の `listing_language.template_key` で引く（`code` ではない）。
- フレームは、指定しなければ各ストアの既定フレームが使われる（無ければフレーム無し）。別のフレームを使うときだけ `list_listing_frames` で選び、`frame_id` を渡す。

## Step 1. 入力をそろえて、出品者に1回だけ確かめる

ASIN ごとに次の値が要る。**出品者が確かめた値だけを使い、Amazon（Keepa）の値で埋めない。**

| 項目 | 内容 |
| --- | --- |
| `purchase_price` | 販売1単位あたりの仕入原価（円、整数）。セット販売ならセット全体の原価 |
| `weight_g` | 販売1単位あたりの**実測**の梱包重量（グラム、整数）。kg ではない。バリエーションはオプションごと |
| `dimension` | 梱包の長さ・幅・高さ（cm、整数）。prefill の値は下書きとして見せ、出品者に確かめてもらう |
| `stock` | 各モデルの初期在庫 |
| `set_quantity` | 1販売単位に入っている個数（3個セットなら 3）。セットでなければ省略 |
| 出品先 | Step 0 のストアのうち、この ASIN を出すストア |

- **価格と重量に `set_quantity` を掛けない。** どちらも、もともと販売1単位あたりの値である。
  `set_quantity` はサムネイルの個数バッジと、セットの整合チェックにだけ使う。
- 値が「ASIN 1個ぶん」なのか「販売1単位ぶん」なのか分からないときは、推測せず出品者に聞く。
- リサーチ候補IDが渡されたら `get_research_candidate(id)` で値を取り、`research_candidate_id` として Step 7 で渡す。
  在庫数は `planned_listing_quantity`、個数バッジは `set_quantity`。この2つを取り違えない。

`submission_id` はこのスキルが作る。`<ASIN>-<YYYYMMDD>`（8〜64文字の英数字・`_`・`-`）とし、
全ストアで共有する。一度 create を送った ASIN では、やり直しでも**同じ値を使い続ける**。

ASIN ごとに1行の表（ASIN、仕入原価、重量、寸法、在庫、セット数、出品先）を出品者に見せて、
**1回だけ**確認を取る。確認が取れたら、以後は「止めて出品者に聞くとき」に当たらない限り止めずに進める。

## ASIN ごとの手順

ASIN は**1つずつ順に**処理する。1つの ASIN の Step 2〜12 が終わってから次へ進む。

### Step 2. Amazon の情報を取る

`prefill_from_asin(asin)` で商品名・説明・画像・バリエーションを取る。

- prefill の重量と価格は使わない（Step 1 の値が正）。
- **`variations[].image` は商品写真とは限らない。** Amazon がシリーズ全体で共有する色見本のことがあり、
  別の形・別の商品が写っていることがある。オプション画像は、各オプションの ASIN を個別に
  `prefill_from_asin` して、その ASIN 自身の `images[0]` を候補にする。
- `prefill_from_asin` は毎分10回まで（キャッシュから返る呼び出しも数える）。オプションが多いときは間隔を空ける。

### Step 3. 画像を目で確かめる（省略しない）

ギャラリーとオプションの**全画像**を `preview_listing_images(image_urls)` で見る（1回20枚まで。
応答サイズの上限を超えた分は失敗として返るので、その画像以降をもう一度呼ぶ）。
画像が1枚も返らなければ、出品者に画像の確認を頼む。

**必ず外す画像:**

- QR コード、外部サイトや SNS への誘導
- 同梱されない別の商品・オプション・交換部品が、同梱品のように写っているもの
- 越境販売で提供できない保証（「◯年保証付」など）
- 条件が日本語でしか書かれていない性能グラフや断定的な効能
- パッケージの裏面・台紙（バーコード、販売元の住所・電話、MADE IN 表記）
- 売る商品と色・形・サイズ・スタイルが違う商品

**採用してよい画像:** 商品の実写、使用シーン、言語に依存しない図（寸法図・構成図など）。
日本語の見出しが乗っていても、情報の中身を写真や図が担っていれば採用してよい。
外した画像の情報のうち買い手に必要なものは、説明文の言葉で補う（「◯◯は別売り・同梱されません」など）。

枚数・比率・SKU・重量を照らす機械的な検証は、画像の中身を1ピクセルも見ていない。
ここが画像の誤りを止められる唯一の場所である。見た枚数と外した理由の分類を完了報告に書く。

**オプション画像の決め方:**

- 各オプション自身の ASIN の商品写真を使う。色見本は使わない。
- 見た目が同じオプション（日本語版と英語版など）は、同じ URL を2つまで共有してよい。
- 1枚の画像を3つ以上のオプションで使う、または全オプションで同じ1枚になるなら、色見本を掴んでいる。取り直す。
- オプション画像なし（`option_images: absent`）にできるのは、サイズ・長さ・容量のように見た目で区別しない軸だけ。
  理由を `absent_reason` に書く。色・柄・デザインの軸、色名を含むオプション名では使えない。
  固有の写真がどうしても取れないなら、その ASIN を見送るか出品者に聞く。

### Step 4. コンプライアンスを判定する

`get_compliance_rules` を、英語の商品名案・ブランド・カテゴリと、出品先の
`stores`（`shopee_<リージョン小文字>` の形。例 `shopee_sg`）で呼ぶ。

- 知らないストア名は黙って捨てられ、グローバルのルールしか返らない。応答の `storeRules` に
  出品先の全リージョンが載っていることを数えて確かめる。
- `keywordMatchHighlights` は文字の一致にすぎない。商品・ブランド・カテゴリ・説明文を読んで、
  ストアごとに「可 / 不可 / 要確認」を自分で決める。
- **「不可」と「要確認」のストアには出品しない。** 理由を台帳と報告に残す。
- 3辺合計が150cmを超える梱包は越境配送できない。

### Step 5. ストアごとにカテゴリ・制限・属性・ブランド・配送を決める

出品先のストアごとに次を行う。

1. `recommend_category(provider_id, <英語の商品名>)` → `search_category(provider_id, category_ids=[...])` で名前とリーフを確かめる。
   **`has_children=false` のリーフしか使えない。** 第1候補が親カテゴリのことがある。
2. `get_shopee_category_limits(provider_id, category_id)` で商品名・説明の文字数の上限を見る。
   **リージョンで大きく違う**（台湾は商品名60文字など）。`source=fallback` は推測値なので、厳しい方の値を使う。
3. `get_attributes(provider_id, category_id, mandatory_only=true)` で必須属性を見る。**同じカテゴリでも必須属性はリージョンごとに違う。**
   値を選んだら、その値の `requires_attributes` を見る（例: 対象年齢の選び方で認証番号が必須になる）。
   認証番号は推測しない。埋められない組み合わせしか無ければ、そのストアは見送る。
   選択式の属性で選択肢が応答に無ければ、`mandatory_only` を外して取り直し、`value_id` を推測しない。
4. `get_brands(provider_id, category_id, query=<ブランド名>)`。名前が部分一致しただけで使わない
   （同名の別会社が別のブランドIDで登録されていることがある）。確信が持てなければ `brand_id: 0`。
   取得がタイムアウトし続けるストアも `brand_id: 0` にする。
5. `get_logistics_channels(provider_id)`。**有効にしたチャネルが1つでも寸法・重量の上限を超えると、create 全体が失敗する。**
   有効なチャネルを1つずつ見て、最重量のモデルと梱包寸法が収まらないチャネルを外す。
   - 1辺ごとの上限（`item_max_dimension` の長さ・幅・高さ）を見る。3辺合計だけでは足りない。
   - `dimension_sum` が0なら3辺合計の上限は未設定。
   - 上限が全部0や `unit: UNKNOWN` のチャネルは上限が読めない。確実に収まるチャネルが他にあれば、そちらだけを残す。

### Step 6. ストアの言語で文言を書く

ストアごとに `item_name`・`description`、バリエーションなら軸名 `tier_name` とオプション名 `option_names` を書く。

- 言語は `list_stores` の `listing_language.code`。日本語を残すと日本語のまま公開される。英語に統一しない。
- その言語の正書法で書く（ベトナム語は声調記号を省かない、など）。
- 文字数は Step 5 の上限に収める。軸名は多くのリージョンで14文字まで、オプション名は30文字まで。
  型番などでオプション名が長くなるときは、型番を説明文へ移す。
- Step 3 で外した画像の情報、同梱物、注意書きを説明文に入れる。
- コンプライアンスで引っかかる語（痩身の効能、SNS の名前など）を入れない。
- `badge_axis`（サムネイルのバッジ用の分類）は `Size` か `Type` の2択。買い手には出ない。
  長さ・容量の軸も `Size`。軸名の翻訳とは別に決める。

### Step 7. `prepare_listing` で事前チェックする

`prepare_listing` は何も書き込まないドライランで、何度呼んでもよい。次を組み立てて呼ぶ。

- 共通: `submission_id`、`amazon_asin`（バリエーションなら親）、`product_name`（日本語の商品名）、`purchase_price`、
  `weight_g`、`dimension`、`stock`、`gallery_image_urls`（Step 3 で採用した URL を先頭から。フレームが付くなら8枚まで）、
  必要なら `set_quantity`・`research_candidate_id`・`template_id`・`frame_id`・`internal_category_id`
  （SellBridge の商品カテゴリ。分かる場合だけ。省くと商品一覧でカテゴリが空になる）
- `stores[]`: ストアごとに `provider_id`、`category_id`、`brand_id`、`attributes`、`logistics_channel_ids`、`content`
- `variation`（バリエーションのとき）: `badge_axis`、`option_images`、`absent_reason`、`options[]`
  （オプションごとに `amazon_asin`・`purchase_price`・`weight_g`・`image_url`、必要なら `asin_pack_size`・`purchase_quantity`）

応答の読み方:

- `request_errors` が空でなければ、どのストアにも作らない。入力を直して呼び直す。
- ストアごとに `status` を見る。**判断は常に `status` で行う。** `ready` のストアだけを作る。
- `errors` は止める issue、`warnings` は止めない issue。**知らない `code` が `errors` / `request_errors` にあれば止める issue として扱い**、
  直し方が分からなければそのストアを見送って報告する。

`code` ごとの対応:

| code | 対応 |
| --- | --- |
| `SHOP_NOT_OPERATIONAL` | そのストアを見送る |
| `CONTENT_NOT_IN_STORE_LANGUAGE` / `TEXT_TOO_LONG` / `TEXT_TOO_SHORT` | 文言を直す（`message` に上限が書いてある） |
| `TEMPLATE_UNUSABLE` | `template_id` を外すか、その言語の中身があるテンプレートにする |
| `FEE_SETTINGS_DEFAULT` | **ランを止める。** 出品者に SellBridge で手数料設定をしてもらう（価格がその出品者のものではない） |
| `PRICING_FAILED` / `PRICE_BOUNDS_UNVERIFIED` | そのストアを見送る。別の価格で埋めない |
| `PRICE_ABOVE_MAXIMUM` / `PRICE_BELOW_MINIMUM` / `PROFIT_BELOW_MINIMUM` | そのストアを見送る。`price_override_local` は出品者が価格を明示したときだけ使う |
| `HIGH_SHIPPING_RATIO`（warning） | 送料が販売価格に対して大きい。売れにくいので、`message` を見せて、そのストアに出すか出品者に聞く（他のストアの作業は止めない） |
| `GALLERY_DUPLICATE` / `TOO_MANY_IMAGES` | ギャラリーを直す |
| `OPTION_IMAGES_INCONSISTENT` / `OPTION_IMAGE_SHARED_TOO_WIDELY` / `OPTION_IMAGES_REQUIRED_FOR_VISUAL_AXIS` | Step 3 に戻ってオプション画像を取り直す |
| `OPTION_IMAGE_SHARED`（warning） | 2つのオプションが同じ写真。見た目が同じ組なら問題ない |
| `OPTION_NAME_DUPLICATE` / `BADGE_AXIS_INVALID` | オプション名・`badge_axis` を直す |
| `SET_QUANTITY_MISMATCH` | `purchase_quantity × asin_pack_size` を確かめ直す。分からなければ出品者に聞く |
| `STORE_DUPLICATE` / `PURCHASE_PRICE_INVALID` / `ASIN_INVALID` | 入力を直す |
| `ATTRIBUTE_INVALID` | `get_attributes` を `mandatory_only` なしで取り直して属性を直す |
| `CREATE_REQUEST_INVALID` | `message` を読んで直す。直せなければそのストアを見送る |
| `SHOP_STATUS_UNKNOWN`（warning） | Step 0 で `NORMAL` を確かめたストアなら続ける |

直して呼び直しても2回続けて同じ issue が残るなら、そのストアを見送るか出品者に聞く。

最後の応答を `<work>/prepare_<submission_id>.json` に保存する。そこから `expected_parent_sku` と、
ストアごとの `computed.discount_percent` を Step 11 で使う。

### Step 8. ストアごとに作る（`shopee-store-lister` に任せる）

画像のアップロード、create、Shopee 上の実体の検証は、サブエージェント `shopee-store-lister`
（プラグインでは `sell-bridge:shopee-store-lister`）に**1ストアにつき1体**で任せる。
親が自分でアップロードや create を呼ばない（親のコンテキストがすぐに尽きる）。

**1. 指示書を書く。** `ready` のストアごとに `<work>/job_<submission_id>_<ストア>.json` を書く。

```jsonc
{
  "store": "SG",                       // ストアのラベル
  "provider_id": 101,
  "kind": "variation",                 // prepare の stores[].kind（single / variation）
  "submission_id": "B0XXXXXXXX-20261021",
  "expected_parent_sku": "...",        // prepare の expected_parent_sku（照合用。作成後の SKU と違うこともある）
  "hero": {                            // create_request.images[0] に frame_id があるときだけ。無ければ null
    "tool": "compose_variation_thumbnail",   // 単品・オプション画像なしは "apply_frame_to_image"
    "image_urls": ["https://..."],     // compose のときだけ。オプション画像を出品順に、同じ URL は1回
    "badge_axis": "Size",              // compose のときだけ
    "set_quantity": null               // 2以上のときだけ数値
  },
  "create_request": { },               // prepare の stores[].create_request を1文字も変えずに入れる
  "result_path": "/absolute/path/to/result_B0XXXXXXXX-20261021_SG.json"
}
```

- サイズ×色のようにオプションに色が繰り返し出るときは、`hero.image_urls` に**色ごとに1枚**だけ入れる
  （全オプションを入れるとバッジの数が実際の色数と合わなくなる）。
- 価格・文言・カテゴリは create_request の中にある。指示書に別の値を書き足さない。

**2. 作成前の数を控える。** 作成するストアごとに `list_shopee_items(provider_id, item_status="UNLIST", offset=0, page_size=1)` の
`total_count` を控える（Step 9 の照合に使う）。

**3. 1店目だけを先に、同期で動かす。** `shopee-store-lister` を1体起動し、終わるのを待つ。
result ファイルを読み、`ok: true` と `product_id` を確かめる。全ストアを同時に作ると、
商品マスタ（product_id）が割れる。1店目の create が商品を作ってから、残りが同じ商品にまとまる。

**4. 残りのストアは同時に最大3体。** 1つのメッセージでまとめて起動する。4ストア以上残っていれば、
3体が終わってから次を起動する。

起動時のプロンプトに書くのは、**指示書の絶対パス**と、次の1文だけ:

> 指示書の create_request は画像以外を変えずに送る。オプション画像は各オプションの image_url をアップロードした結果を同じ順で1対1に割り当て、先頭画像（hero）の戻り値をオプション画像に使わない。

規則:

- **起動の直前に、その (submission_id, ストア) の result ファイルが無いことを確かめる。** あれば起動しない（作成済みの可能性がある）。
  例外は、result が `create_sent: false`（create の前で止まった）か `rate_limited: true`（レート制限で拒否された）で、
  `reference/troubleshooting.md` に従ってやり直すと決めたときだけ。
- **result ファイルが正。** サブエージェントの完了通知は、遅れて届く・矛盾する内容が2通届く・
  サブエージェントが終わる前に届くことがある。通知に書かれた item_id や product_id を使わず、result ファイルを読む。
- **result ファイルが無いことを失敗と読み替えない。** まだ動いていることが多い。それだけを理由に同じストアを再起動しない。
- サブエージェントが失敗を返したら、同じ指示書のまま再起動しない。エラー文言を読み、
  `${CLAUDE_SKILL_DIR}/reference/troubleshooting.md`（このスキルのフォルダの `reference/troubleshooting.md`）に従う。
- 422 `ERR_VALIDATION_FAILED` が返ったら、create を繰り返さない。`issues[]` の `code` は prepare と同じなので、
  入力を直して Step 7 からやり直す。

### Step 9. 全ストアの結果を照らし合わせる

1. **起動漏れ:** 作ると決めた (submission_id, ストア) の組を先に並べ、result ファイルの組と比べる。足りない組は起動漏れ、余分な組は二重起動。
2. **result の中身:** 全ストアで `ok: true` で、`verified` の値が `true`（単品の `option_image` / `models` だけ `null` でよい）。
   バリエーションで `option_image` が `null` なら、prepare で `option_images: absent` にしたときだけ認める。
3. **product_id:** 全ストアで同じ。
4. **親 SKU（`item_sku`）:** 全ストアで同じで、空でない。**割れていたら、この ASIN の残りの作成を止めてすぐ報告する**（SKU の割れは取り返しのつかない事故の前段）。
   `expected_parent_sku` と違うだけなら、既存の商品の SKU を使った可能性があるので、報告に書いて続ける。
5. **Shopee 上の実在:** ストアごとに `list_shopee_items(provider_id, item_status="UNLIST", offset=0, page_size=20)` を呼ぶ。
   作った item_id が一覧にあること、`total_count` が Step 8-2 から作った件数だけ増えていることを確かめる。
   知らない新しい item_id がある・増え方が合わないときは、作りかけの出品が残っている可能性があるので報告する
   （他のセッションや出品者の操作でも増えるので、ここでは何も消さない・作り直さない）。

### Step 10. 在庫監視に乗ったか確かめる（報告だけ）

作成後の在庫監視の自動登録は、モデルに ASIN が無い・テナントの在庫監視設定がオフ・プランの上限に達した、
のどれかで**黙ってスキップ**される（create は成功のまま返る）。

1. `get_stock_monitor_status({product_ids: [<product_id>]})` で概況を見る。
2. `list_stock_monitor_subscriptions({search: <親SKU>, page_size: 100})` を呼び、返った行を
   **`sku` の完全一致**と今回の `product_id` で絞ってから数える。
   出品したストアごとに、単品なら1行、バリエーションならモデルの数だけ行があること。
   `enabled: false` の行と `abnormal_status` が入った行は数えない。作成直後の `last_checked_at: null` は問題ない。
3. 足りなければ、**ここで有効にしない。** 足りないストア・モデルを報告し、
   SellBridge の商品一覧から在庫監視を有効にするよう出品者に伝える（よくある原因: ASIN の無いモデル、設定オフ、プランの上限）。

### Step 11. 割引の To Do を作る（報告だけ）

`prepare_listing` の `stores[].computed.discount_percent` が0より大きいストアでは、推奨価格が
「その率の割引に入る」前提で計算されている。割引に入れないまま公開すると、想定より高い値段で並び続ける。

- 該当する (ストア, item_id, モデル数, 割引率) を完了報告の To Do に並べ、出品者が手で割引に入れるよう伝える。
- Shopee は **UNLIST のバリエーション商品**を割引に入れられない（`discount.item_status_invalid`）。それらは「公開後に入れる」と書く。
- このスキルは割引を操作しない。

### Step 12. 台帳に記録する

`<work>/listing-log.jsonl` に **1行 = 1つの ASIN × 1ストア** で追記する。作成したストアだけでなく、
見送った・失敗したストアも理由付きで1行残す（後から「なぜこのストアに出ていないか」を辿れるように）。

```json
{"timestamp": "2026-10-21T10:00:00+09:00", "submission_id": "B0XXXXXXXX-20261021", "asin": "B0XXXXXXXX",
 "store": "SG", "provider_id": 101, "status": "created_unlist", "reason": null,
 "item_id": 123456789, "product_id": 98765, "item_sku": "...", "price_local": 25.9, "currency": "SGD",
 "discount_percent": 10, "stock_monitor": "ok", "listed": false}
```

`status` は `created_unlist` / `skipped` / `failed` のどれか。`listed` は常に `false`。
バリエーションで価格がモデルごとに違うときは最安値を `price_local` に入れ、幅を `reason` ではなく報告に書く。
同じ (submission_id, ストア) の行を二重に書かない。

## レート制限

上限はテナント全体で共有される（同じテナントの全 API キー・全接続・全セッションの合計）。

| 種類 | 上限 |
| --- | --- |
| 全ツール | 毎分60回 |
| 書き込み（`upload_image_url`、`apply_frame_to_image`、`compose_variation_thumbnail`、create など） | 毎分20回 |
| `prefill_from_asin`（Keepa） | 毎分10回 |

- 6ストアで ASIN 1つあたり約40回の書き込みになる。**ASIN 1つに少なくとも2〜3分かかる**と見込む。
- サブエージェントを同時に4体以上動かさない。待ちが連鎖して全体が遅くなる。3体は**テナント単位の予算**で、
  別のセッションも出品しているなら合計で3体以内にする。
- `Rate limit exceeded for the <window> window. Retry after Ns.` は、**そのツールが実行されなかった**という意味。
  N 秒待ってから、**同じ呼び出しを1回だけ**送り直す。create も同じ。同じ入力での送り直しが許されるのはこの場合だけ。
- `Rate limit service is unavailable` は送り直さない。ランを止め、時間を置いてやり直すよう出品者に伝える。

## コンテキストと並行セッション

- ASIN 1つで親のコンテキストを約40k トークン使う。**1セッションの目安は10 ASIN まで。** それより多いときはセッションを分ける。
  途中で会話の圧縮が入ると、どの画像を外したか・どのストアを見送ったかという判断が失われ、作り直すより高くつく。
- 重い作業（アップロード・create・検証）は必ずサブエージェントに任せ、親で近道しない。大きなツール応答は要点だけを読む。
- **同じ ASIN を2つのセッションで同時に出品しない。** 何もロックしていないので、同じ create が二重に走り、
  商品マスタや SKU が割れる。セッションを分けるときは ASIN を重ならないように分ける。

## 止めて出品者に聞くとき

- Step 1 の値（仕入原価・重量・寸法・在庫・セット数）が欠けている、または何ひとつぶんの値か分からない
- `listing_language` の無いストアに出したいと言われた
- コンプライアンスが「要確認」
- 認証番号・属性・ブランド・配送の上限を確定できない
- `FEE_SETTINGS_DEFAULT`、または `HIGH_SHIPPING_RATIO` のストアに出すかどうか
- 色・柄の軸なのに、オプションごとの写真が取れない
- 作成後の検証で `false` が出た、親 SKU や product_id がストア間で割れた
- 作成済みの出品を直す必要がある（直す前に必ず確認を取る）
- 同じエラーを2回続けて切り分けられない

## ここでは止めない

- 1つのストアの見送り・失敗。他のストア・他の ASIN は続ける（ただし同じ ASIN の SKU・product_id が割れたら、その ASIN の作成は止める）
- 「止めて出品者に聞くとき」に無い理由で見送ると決まったストア。見送って報告する
- レート制限。待って1回送り直すだけ
- 在庫監視の不足と割引の To Do。報告に書いて続ける

## 完了報告

最後に次をまとめて報告する。

- ASIN × ストアごとの結果: 作成なら item_id・product_id・親 SKU・価格、見送り・失敗なら理由
- 画像の確認結果: ASIN ごとに見た枚数、外した枚数と理由の分類（「確認した」の一言で済ませない）
- 照合の結果: 起動漏れ・二重起動の有無、product_id と親 SKU の一致、Shopee 上の実在（Step 9）
- 在庫監視の結果: ASIN ごとに「全ストア×全モデル登録済み（N行）」か、足りないストア・モデル
- 割引の To Do: (ストア, item_id, モデル数, 割引率)。UNLIST のバリエーション商品は「公開後に入れる」
- 台帳のパス（`<work>/listing-log.jsonl`）と書いた行数
- **公開（`listed=true`）は一度も行っていない**こと。公開は SellBridge の画面から行うよう伝える
