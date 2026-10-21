---
name: shopee-listing
description: SellBridge につないだ Shopee のストアに、Amazon の ASIN（またはリサーチ候補）から出品を UNLIST（非公開）で作る。画像の目視確認、Shopee 上のストア状態の確認、コンプライアンス判定、カテゴリ・属性・配送の決定、ストアの言語での文言作成、prepare_listing の事前チェックを経て、ストアごとに出品を作り、Shopee 上の実体・在庫監視・割引の要否を確かめて報告する。「この ASIN を Shopee に出品して」「リサーチ候補を出品して」と頼まれたとき、ASIN と一緒に Shopee・出品・SellBridge の話が出たときに使う。1つの会話で処理する ASIN は1つ。公開（listed=true）は行わない。
---

# Shopee 出品（UNLIST で作成）

## 対象

出品する ASIN（英大文字・数字10桁）またはリサーチ候補ID（数字）は、出品者のメッセージに書かれている
（`/sell-bridge:shopee-listing B0XXXXXXXX` のように呼ばれたときは、その後ろ: $ARGUMENTS）。
複数書かれていても、処理するのは最初の1つだけ。残りは新しい会話で頼むよう伝える。
どちらも書かれていなければ、出品したい ASIN か候補IDを出品者に聞く。

出品は SellBridge の公開ツール（SellBridge コネクタ）だけで行える。ツールの説明には、ツールの順番や
「`submission_id` を全ストアで共有する」「認証番号を推測しない」などの前提がすでに書いてある。
このスキルが足すのは**判断と順番**である。どの画像を外すか、どのストアを見送るか、
どの順でストアに作るか、いつ止めて出品者に聞くか。

SellBridge のツールが見えない、または認証を求めるエラーが返るときは、出品者に
「SellBridge コネクタに接続（または再接続）してください。claude.ai の設定 → コネクタ、またはプラグインの Connectors タブから行えます」と伝えて止める。

## 絶対規則

- **`set_listing_status(listed=true)` を呼ばない。** 作るのは常に UNLIST。公開は出品者が行う。
- **`delete_item` を呼ばない。** `unlist_item` / `update_item` / `update_item_price` / `migrate_canonical_sku` も、出品者の確認なしには呼ばない。
- **同じ submit を同じ入力で送り直さない。** 例外はレート制限で拒否されたとき（「レート制限」の節）だけ。
- **重量・認証番号（TIS / FDA / BSMI など）・言語・カテゴリ・ブランドを推測しない。** 仕入原価は Step 2 の規則でだけ決める。ブランドの一致に自信が無ければ `brand_id: 0`。
- **作成の前に必ず `prepare_listing` を通す。** `status: ready` のストアだけに作り、`stores[].create_request` は画像以外を変えずに送る。価格を書き換えない。自分で価格を計算しない。
- **Shopee 上の状態が `NORMAL` のストアにだけ作る**（Step 0）。
- 会話や報告に認証情報（API キー、トークン、パスワード）を書かない。

## Step 0. 出品先のストアを決める（最初に1回）

### 0-1. 出品先ストアの候補

`list_stores` を呼び、次をすべて満たすストアを候補にする。

- Shopee のストアである
- `listing_language` が付いている（無いストアは言語を推測せず、出品先から外して出品者に伝える）
- `token_status` が `valid` か `expiring`（`expiring` は出品者に「近く連携が切れる」と伝える。`expired` / `invalid` は外し、SellBridge の画面で再連携するよう伝える）

出品者が出品先を指定していれば（「PH と TH だけ」など）、その中から選ぶ。指定が無ければ候補の全部。
**候補から外したストア（言語なし・`expired` / `invalid`）も理由付きで控え、完了報告に残す。** 外したことを黙って落とさない。
同じリージョンにストアが複数あるときは、ラベルを `<リージョン>-<provider_id>` にして区別する。

### 0-2. Shopee 上のストア状態を確かめる

出品先の**全ストア**で `get_shopee_shop_info(provider_id)` を呼ぶ。
`status` が `NORMAL` 以外のストアと、エラーが返ったストアは、**この会話の出品先から外す**。
完了報告に「Shopee 上のストア状態が NORMAL でない（<status またはエラー>）」と残し、残りのストアで続ける。
残りが0になったら止めて出品者に伝える。

なぜ必要か: `list_stores` の `status` / `token_status` は SellBridge との接続状態で、
Shopee 側の凍結や BAN は映らない。凍結中のストアに作ると、Shopee が親アイテムを作ったあとで
バリエーションを拒否し、そのストアでの親 SKU が使えなくなる。作成側の API にもガードはあるが、
状態の取得に失敗したときは素通しし、状態を数分キャッシュするので、保証にはならない。

### 0-3. テンプレートとフレーム（任意）

- 説明文テンプレートを使うと出品者に言われたときだけ、`list_listing_templates` / `get_listing_template` で選び、`template_id` を Step 7 で渡す。テンプレートの言語別の中身は `list_stores` の `listing_language.template_key` で引く（`code` ではない）。
- フレームは、指定しなければ各ストアの既定フレームが使われる（無ければフレーム無し）。別のフレームを使うと言われたときだけ `list_listing_frames` で選び、`frame_id` を渡す。

## Step 1. Amazon の情報を取る

`prefill_from_asin(asin)` で商品名・説明・画像・バリエーション・価格を取る。

- prefill の重量は使わない（Step 2 で出品者が確かめた値が正）。寸法は下書きとして Step 2 で見せる。
- **`variations[].image` は商品写真とは限らない。** Amazon がシリーズ全体で共有する色見本のことがあり、
  別の形・別の商品が写っていることがある。オプション画像は、各オプションの ASIN を個別に
  `prefill_from_asin` して、その ASIN 自身の `images[0]` を候補にする。
- `prefill_from_asin` は毎分10回まで（キャッシュから返る呼び出しも数える）。オプションが多いときは間隔を空ける。

リサーチ候補IDが渡されたら、先に `get_research_candidate(id)` で ASIN と値を取り、その ASIN で prefill する。
`research_candidate_id` は Step 7 で渡す。在庫数は `planned_listing_quantity`、個数バッジは `set_quantity`。この2つを取り違えない。

## Step 2. 入力をそろえて、出品者に1回だけ確かめる

次の値が要る。

| 項目 | 内容 |
| --- | --- |
| `purchase_price` | 販売1単位あたりの仕入原価（円、整数）。下の「仕入原価の決め方」に従う |
| `weight_g` | 販売1単位あたりの**実測**の梱包重量（グラム、整数）。kg ではない。バリエーションはオプションごと。**出品者に聞く**（Amazon の値で埋めない） |
| `dimension` | 梱包の長さ・幅・高さ（cm、整数）。prefill の値は下書きとして見せ、出品者に確かめてもらう |
| `stock` | 初期在庫。**全モデル共通の整数1つ**（モデルごとには指定できない。違う在庫にしたいときは create_request を書き換えず、出品後に出品者が SellBridge の画面で直すよう伝える）。言われなければ 1 を提案する |
| `set_quantity` | 1販売単位に入っている個数（3個セットなら 3）。セットでなければ省略 |
| 出品先 | Step 0 で残ったストアのうち、この ASIN を出すストア |

**仕入原価の決め方:**

- 出品者が仕入原価を言っていれば、それを使う（常に出品者の値が優先）。
- 言っていなければ、Step 1 の `prefill_from_asin` の `price`（amazon.co.jp の Buy Box 価格。無ければ新品最安などの現在価格、円）を使い、
  **どの価格を使ったか（Buy Box か、新品最安か、いくらか）を出品者に伝える**。
- `price` は ASIN を1回買う価格である（2個入りパックなら、そのパック1つの価格）。販売1単位を作るのに同じ ASIN を複数回買うときだけ、
  その購入回数を掛ける。**セット個数 `set_quantity` を掛けない。**
- `price` が `null` のとき、または amazon.co.jp 以外の Amazon で引いたときは、推測せず出品者に聞く。
- バリエーションはオプションごとに、そのオプションの ASIN の `price` で同じ規則を当てる。

ほかの注意:

- **価格と重量に `set_quantity` を掛けない。** どちらも、もともと販売1単位あたりの値である。
  `set_quantity` はサムネイルの個数バッジと、セットの整合チェックにだけ使う。
- 値が「ASIN 1個ぶん」なのか「販売1単位ぶん」なのか分からないときは、推測せず出品者に聞く。

`submission_id` はこのスキルが作る。`<ASIN>-L1`（8〜64文字の英数字・`_`・`-`）のように**日付を入れず ASIN から決まる値**にし、
全ストアで共有する。日付を入れると、別の日にやり直したときに二重出品のガードが効かない。
一度 submit を送った ASIN では、やり直しでも**同じ値を使い続ける**。
`ERR_SUBMISSION_ALREADY_LISTED` は「作成済み」の意味である。`submission_id` を作り直して回避しない。

1行の表（ASIN、商品名、仕入原価とその出どころ、重量、寸法、在庫、セット数、出品先）を出品者に見せて、
**1回だけ**確認を取る。重量のように出品者しか知らない値は、この確認でまとめて聞く。
確認が取れたら、以後は「止めて出品者に聞くとき」に当たらない限り止めずに最後まで進める。

## Step 3. 画像を目で確かめる（省略しない）

ギャラリーとオプションの**全画像**を `preview_listing_images(image_urls)` で見る（1回20枚まで。
応答サイズの上限を超えた分は失敗として返るので、その画像以降をもう一度呼ぶ）。
画像が1枚も返らなければ、出品者に画像の確認を頼む。
一部だけ見られなかった画像は、取得し直しても見られなければ**採用せず外す**（見ていない画像を出さない）。それが必須のオプション画像で外せないときは、出品者に聞く。判断できなければそのストアは見送り、報告に出す。

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

## Step 4. コンプライアンスを判定する

`get_compliance_rules` を、英語の商品名案・ブランド・カテゴリと、出品先の
`stores`（`shopee_<リージョン小文字>` の形。例 `shopee_sg`）で呼ぶ。

- 知らないストア名は黙って捨てられ、グローバルのルールしか返らない。応答の `storeRules` に
  出品先の全リージョンが載っていることを数えて確かめる。載っていないリージョンのストアは判断できないので**見送り**、理由を報告に残す。
- `keywordMatchHighlights` は文字の一致にすぎない。商品・ブランド・カテゴリ・説明文を読んで、
  ストアごとに「可 / 不可 / 要確認」を自分で決める。
- **「不可」と「要確認」のストアには出品しない。** 理由を報告に残す。
- 3辺合計が150cmを超える梱包は越境配送できない。

## Step 5. ストアごとにカテゴリ・制限・属性・ブランド・配送を決める

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

## Step 6. ストアの言語で文言を書く

ストアごとに `item_name`・`description`、バリエーションなら軸名 `tier_name` とオプション名 `option_names` を書く。

- 言語は `list_stores` の `listing_language.code`。日本語を残すと日本語のまま公開される。英語に統一しない。
- その言語の正書法で書く（ベトナム語は声調記号を省かない、など）。
- 文字数は Step 5 の上限に収める。軸名は多くのリージョンで14文字まで、オプション名は30文字まで。
  型番などでオプション名が長くなるときは、型番を説明文へ移す。
- Step 3 で外した画像の情報、同梱物、注意書きを説明文に入れる。
- コンプライアンスで引っかかる語（痩身の効能、SNS の名前など）を入れない。
- `badge_axis`（サムネイルのバッジ用の分類）は `Size` か `Type` の2択。買い手には出ない。
  長さ・容量の軸も `Size`。軸名の翻訳とは別に決める。

## Step 7. `prepare_listing` で事前チェックする

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

最後の応答の `expected_parent_sku` と、ストアごとの `kind`・`create_request`・`computed.discount_percent` を Step 8 以降で使う。
会話が長くなっても取り違えないよう、`ready` のストアごとに（ラベル、provider_id、kind、販売価格、discount_percent）を一度表にまとめて出品者に見せる。

## Step 8. ストアごとに作る

`ready` のストアに、**1ストアずつ順に**作る。並べて同時に作らない。

**1. 作成前の数を控える。** 作成するストアごとに `list_shopee_items(provider_id, item_status="UNLIST", offset=0, page_size=1)` の
`total_count` を控える（Step 9 の照合に使う）。

**2. 送り済みでないかを確かめる。** 各ストアで、作る直前に `get_listing_operation(provider_id, submission_id)` を呼ぶ。
見つからない（404）なら未送信なので作ってよい。操作が返ってきたら、そのストアには**この `submission_id` で送り済み**である。submit せず、返った状態から 8-7 へ進む。
（前の会話で途中まで進めた ASIN を頼み直されたときに、二重に作らないため。）

**3. 1店目を作り切ってから、残りに進む。** 1店目が `succeeded` で `product_id` が入るまで、2店目に進まない。
どちらかが欠けたら、残りのストアに作らない。「作りかけの確認」（`reference/troubleshooting.md`）を先に行い、出品者に聞く。
1店目の作成が商品マスタ（product_id）を作り、残りのストアが同じ商品にまとまる。順番を崩すと商品マスタが割れる。

各ストアでは、次の 8-4〜8-8 を行う。**`create_request` で変えてよいのは画像の `image_id` / `image_url` だけ**
（先頭画像は 8-4 の戻り値で丸ごと置き換える）。価格（`original_price` / `price_jpy`）、`purchase_price`、`weight`、`dimension`、文言、
カテゴリ、属性、配送、`sku`（空文字のまま）、`submission_id`、`amazon_asin`、`product_name`、
`option_images_absent_approval`（あればそのまま。無ければ足さない）は1文字も変えない。`weight` はグラムで入っている。kg に直さない。

**8-4. 先頭画像（hero）。** `create_request.images[0]` に `frame_id` があるときだけ行う。

| 場合 | 呼び方 |
| --- | --- |
| バリエーションでオプション画像あり | `compose_variation_thumbnail`：`provider_id`、`image_urls` = `badge_axis` の軸のオプション画像 URL をオプションの出品順に1つずつ（同じ URL を共有するオプションがあっても除かない。バッジの数字はこの配列の長さ）、`variation_name` = `badge_axis`（買い手向けの軸名ではない）、`frame_id` = `images[0].frame_id`、`set_quantity` は2以上のときだけ |
| 単品、またはオプション画像なし | `apply_frame_to_image`：`provider_id`、`image_url` = `images[0].original_url`、`frame_id` = `images[0].frame_id`、`set_quantity` は2以上のときだけ |

戻り値の `badge` が `applied` / `applied_over_product` / `not_requested` / `not_configured` なら続ける。
`axis_unrecognised` やそれ以外（`style_unavailable`、`no_empty_corner` など）なら、**そのストアには作らずに止め**、報告に出す（`badge_axis` を自分で言い換えない）。
続けるときは、戻り値の `image_id` / `image_url` / `original_url` / `frame_id` の**4つとも**で `images[0]` を置き換える。

**8-5. 画像のアップロード。** `upload_image_url(provider_id, image_url)` で次をアップロードする。

- `images` のうち hero 以外の全部（hero が無ければ `images[0]` から全部）の `image_url`。
  hero の元画像と `images[1]` は同じ URL だが、`images[1]` は素の画像として別にアップロードする
- バリエーションなら、`tier_variation[0].options[].image.image_url` の全部。**同じ URL は1回だけ**アップロードし、同じ URL のオプションには同じ結果を使う

アップロードに成功した数がアップロードすべき URL の数と一致するまで、submit しない。
応答に `image_url` が欠けていたら、その1枚だけを1回アップロードし直す。URL を自分で組み立てない。

**8-6. 画像を埋めて submit（1回だけ）。**

- `images[k]`（hero 以外）の `image_id` / `image_url` を、その `image_url` のアップロード結果で置き換える。
- オプション画像は、各オプションの元の `image_url` のアップロード結果を**同じ順番で1対1**に入れる。hero の戻り値をオプション画像に使わない。
- submit の直前に読み返す: `image_id` が空の画像が無い／`images` の枚数が `create_request` と同じ／
  オプション画像の `image_id` は同じ元 URL どうしでだけ一致し、hero の `image_id` とは違う／
  トップレベルの `purchase_price` がある（バリエーションは各 `model[].purchase_price` も）／`sku` と全 `model[].sku` が空文字／画像以外は `create_request` と同じ。

`kind` が `variation` なら `submit_listing_with_variations`、`single` なら `submit_listing` に渡す。
同期版の `create_listing` / `create_listing_with_variations` は使わない（タイの分割払いの下限を吸収せず、タイムアウトすると作成の成否が分からなくなる）。

submit がエラーを返したとき:

- **422 `ERR_VALIDATION_FAILED`** → 何も作られていない。送り直さない。`issues[].code` は prepare と同じなので、入力を直して Step 7 からやり直す
- **409 `ERR_SUBMISSION_ALREADY_LISTED`** → そのストアには作成済み。送り直さず、8-7 で状態を読む
- **応答が返らない・タイムアウト・5xx** → **新しい submit をしない。** 8-7 で同じキーの状態を読む
- `Rate limit exceeded ... Retry after Ns.` → submit は実行されていない。N 秒待って、**同じ引数で1回だけ**送り直す
- それ以外 → エラー文言をそのまま報告に書き、`reference/troubleshooting.md` に従う

**8-7. 作成が終わるのを待つ。** `get_listing_operation(provider_id, submission_id)` を `retry_after_seconds` 後に読み直す。
終端は `succeeded` / `failed` / `needs_review` だけ。

- `succeeded` → `item_id` / `product_id` / `model_ids` を控えて 8-8 へ
- `failed` / `needs_review` → `error_code` と `error_message`（Shopee の原文）を報告に書く。`item_id` が入っていれば**残骸がある**ので、それも書く。
  `reference/troubleshooting.md` に従う
- **10回読んでも終端にならない** → そのストアは「処理中」として報告に書き、次のストアへ進む（1店目なら残りに進まない）。
  タイで 1000 THB 未満のモデルがあると、作成後に分割払いを外して価格を戻すまで、数分〜30分ほど `configuring` のまま待つことがある。
  最後のストアまで終えたら、処理中のストアをもう一度読み直す

**8-8. Shopee 上の実体を確かめる。** `get_shopee_item(provider_id, item_id)` を呼び、次を照合する。

- `weight`: `create_request.weight`（グラム）÷ 1000 の KG 文字列と一致
- `item_status == "UNLIST"`
- `item_sku` が空でない
- `item_name` が `create_request.item_name` と一致
- 画像の枚数が `images` の枚数と一致し、順番どおりに並んでいる（hero があれば先頭が hero、その次がギャラリー先頭の素の画像）
- バリエーションなら、オプション画像が `tier_variation` のオプションでアップロード結果と順番どおり1対1に一致し、hero の画像と違う

バリエーションなら、さらに `get_shopee_item_models(provider_id, item_id)` で、各モデルの `tier_index` / `original_price` / `stock` が
`create_request.model` と一致し、各 `model_sku` が空でないことを確かめる。
`dimension` は Shopee の読み取りに無いので検証しない。

台湾（TW）では書き込みの反映が遅れることがある。TW で不一致が出たら、他の照合を終えてから1回だけ読み直す。
それでも違えば「要確認」として報告する。**検証で不一致が出ても、自分で直さない。**

**ストアを1つ終えるごとに、親 SKU（`item_sku`）と `product_id` を、それまでのストアと比べる。** 割れていたら、
その時点で残りのストアに作らず、すぐ出品者に報告する（SKU の割れは取り返しのつかない事故の前段）。
`expected_parent_sku` と違うだけなら、既存の商品の SKU を使った可能性があるので、報告に書いて続ける。

## Step 9. 全ストアの結果を照らし合わせる

1. **作成漏れ:** 作ると決めたストアを並べ、作成・見送り・失敗のどれかに必ず入っていることを確かめる。
2. **product_id と親 SKU:** 作成した全ストアで同じで、空でない。
3. **Shopee 上の実在:** ストアごとに `list_shopee_items(provider_id, item_status="UNLIST", offset=0, page_size=100)` を呼ぶ。
   作った item_id が一覧にあること、`total_count` が 8-1 から作った件数だけ増えていることを確かめる。
   **ページ送りを前提にする**: 1ページ目に作った item があるとは限らない。`has_next_page` が真の間は応答の `next_offset`（自分で計算しない）で続きを読む。
   読み切っても見つからなければ、「確認できず」として報告する（実在と書かない）。
   知らない新しい item_id がある・増え方が合わないときは、作りかけの出品が残っている可能性があるので報告する
   （他の会話や出品者の操作でも増えるので、ここでは何も消さない・作り直さない）。

## Step 10. 在庫監視に乗ったか確かめる（報告だけ）

作成後の在庫監視の自動登録は、モデルに ASIN が無い・在庫監視の設定がオフ・プランの上限に達した、
のどれかで**黙ってスキップ**される（作成は成功のまま返る）。

1. `get_stock_monitor_status({product_ids: [<product_id>]})` で概況を見る。エラーになったら「0件」と読まず、「確認できず」として報告する。
2. `list_stock_monitor_subscriptions({search: <親SKU>, page_size: 100})` を呼び、返った行を
   **`sku` の完全一致**と今回の `product_id` で絞ってから数える。
   出品したストアごとに、単品なら1行、バリエーションならモデルの数だけ行があること。
   `enabled: false` の行と `abnormal_status` が入った行は数えない。作成直後の `last_checked_at: null` は問題ない。
3. 足りなければ、**ここで有効にしない。** 足りないストア・モデルを報告し、
   SellBridge の商品一覧から在庫監視を有効にするよう出品者に伝える（よくある原因: ASIN の無いモデル、設定オフ、プランの上限）。

## Step 11. 割引の To Do を作る（報告だけ）

`prepare_listing` の `stores[].computed.discount_percent` が0より大きいストアでは、推奨価格が
「その率の割引に入る」前提で計算されている。割引に入れないまま公開すると、想定より高い値段で並び続ける。

- 該当する (ストア, item_id, モデル数, 割引率) を完了報告の To Do に並べ、出品者が手で割引に入れるよう伝える。
- Shopee は **UNLIST のバリエーション商品**を割引に入れられない。それらは「公開後に入れる」と書く。
- このスキルは割引を操作しない。

## レート制限

上限は SellBridge のアカウント全体で共有される（同じアカウントの全接続・全会話の合計）。

| 種類 | 上限 |
| --- | --- |
| 全ツール | 毎分60回 |
| 書き込み（`upload_image_url`、`apply_frame_to_image`、`compose_variation_thumbnail`、submit など） | 毎分20回 |
| `prefill_from_asin` | 毎分10回 |

- 6ストアで ASIN 1つあたり約40回の書き込みになる。**ASIN 1つに少なくとも数分かかる**と、始める前に出品者に伝える。
- `Rate limit exceeded (<window> limit). Retry after Ns.` は、**そのツールが実行されなかった**という意味。
  N 秒待ってから、**同じ呼び出しを1回だけ**送り直す。submit も同じ。同じ入力での送り直しが許されるのはこの場合だけ。
- `Rate limit service is unavailable` は送り直さない。止めて、時間を置いてやり直すよう出品者に伝える。

## 会話の長さと同時出品

- **1つの会話で出品する ASIN は1つだけ。** 複数の ASIN を頼まれたら、最初の1つだけを処理し、残りは新しい会話で頼むよう伝える。
  会話が長くなると前半の判断（どの画像を外したか、どのストアを見送ったか）が失われ、作り直すより高くつく。
- 大きなツール応答（prefill、カテゴリ、属性、Shopee の商品）は、必要な値だけを読む。全文を会話に書き写さない。
- **同じ ASIN を2つの会話で同時に出品しない。** 同じ submit が二重に走り、商品マスタや SKU が割れる。

## 止めて出品者に聞くとき

- Step 2 の値（重量・寸法・在庫・セット数、`price` が無いときの仕入原価）が欠けている、または何ひとつぶんの値か分からない
- `listing_language` の無いストアに出したいと言われた
- コンプライアンスが「要確認」
- 認証番号・属性・ブランド・配送の上限を確定できない
- `FEE_SETTINGS_DEFAULT`、または `HIGH_SHIPPING_RATIO` のストアに出すかどうか
- 色・柄の軸なのに、オプションごとの写真が取れない
- 作成後の検証で不一致が出た、親 SKU や product_id がストア間で割れた
- 作成済みの出品を直す必要がある（直す前に必ず確認を取る）
- 同じエラーを2回続けて切り分けられない

## ここでは止めない

- 1つのストアの見送り・失敗。他のストアは続ける（ただし親 SKU・product_id が割れたら残りに作らない。1店目が `succeeded` かつ `product_id` ありにならないときも、残りに作らない）
- 「止めて出品者に聞くとき」に無い理由で見送ると決まったストア。見送って報告する
- レート制限。待って1回送り直すだけ
- 在庫監視の不足と割引の To Do。報告に書いて続ける

## 完了報告

最後に次をまとめて報告する。出品者が後から見返せるよう、ストアごとの結果は表にする。

- ストアごとの結果: 作成なら item_id・product_id・親 SKU・価格（現地通貨）、見送り・失敗なら理由。
  **Step 0 で出品先から外したストア（理由付き）も、ここに必ず並べる**。検証で不一致が出たもの・処理中のものは「要確認」として別に挙げる
- 仕入原価: 使った金額と出どころ（出品者の指定 / Amazon の Buy Box / 新品最安など）
- 画像の確認結果: 見た枚数、外した枚数と理由の分類（「確認した」の一言で済ませない）
- 照合の結果: product_id と親 SKU の一致、Shopee 上の実在（Step 9）
- 在庫監視の結果: 「全ストア×全モデル登録済み（N行）」か、足りないストア・モデル
- 割引の To Do: (ストア, item_id, モデル数, 割引率)。UNLIST のバリエーション商品は「公開後に入れる」
- `submission_id`（同じ ASIN を頼み直すときは、この値で送り済みかを確かめる）
- **公開（`listed=true`）は一度も行っていない**こと。内容を確かめてから、出品者が Shopee Seller Center か SellBridge で公開するよう伝える
