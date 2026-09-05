# Handlebars テンプレート

## ディレクトリ構成

```
templates/
  actor/
    cast-sheet.hbs          # cast シートのシェル（タブ nav＋partial 差し込み。フェーズ11-2 で部品化）
    guest-sheet.hbs         # guest シートのシェル（履歴タブなし。フェーズ11-3）
    troop-sheet.hbs         # troop シートのシェル（神業/報酬点/部位なし・heads あり。フェーズ11-4）
    extra-sheet.hbs         # extra シート（名前＋説明のみ・タブなし。フェーズ11-5・2026-07-04 再訂正）
    parts/                  # アクターシート共通部品（フェーズ11-2。cast/guest 等で共有・features でゲート）
                            #   sidebar / header / tab-abilities / tab-combat / tab-outfits /
                            #   tab-status / tab-history(cast 専用) / tab-profile
  item/
    style-sheet.hbs / miracle-sheet.hbs / general-skill-sheet.hbs /
    style-skill-sheet.hbs / organization-sheet.hbs /
    outfit-sheet.hbs / housing-area-sheet.hbs / life-path-sheet.hbs   # フェーズ6
  journal/
    scenario-sheet.hbs      # アクトシート
  hud/
    hud.hbs                 # 常駐 HUD
  chat/                     # チャットカード（2026-09-05 に**統一規格**へ。下記「チャットカードの統一規格」）
    parts/card.hbs          #   骨格 tnxCard（器＋見出し＋本文。長文があれば見出しが開閉の起点）
    parts/card-field.hbs    #   段: ラベル＝値の1行 tnxCardField
    parts/card-row.hbs      #   段: 同上（値が組み立てを要するとき）tnxCardRow
    parts/card-fold.hbs     #   段: 畳める内訳 tnxCardFold（見出しはそれ自体が情報の行）
    parts/card-text.hbs     #   段: カード本体の長文 tnxCardText
    parts/card-result.hbs   #   段: 結果の行 tnxCardResult
    parts/check-card-row.hbs / check-calc-rows.hbs              # 判定カードの段（カード行・標準計算行）
    scene-card.hbs / check-request.hbs / check-result.hbs       # フェーズ8
  user/
    record-sheet.hbs        # レコードシート (フェーズ2)
  app/                      # usage-sheet.hbs / usage-sheet-combo.hbs / rl-request-app.hbs
  apps/                     # damage-chart-text.hbs
  dialog/                   # 各種ダイアログ
  parts/                    # 共通パーシャル
```

## 共通パーシャル (`templates/parts/`)

| ファイル | 用途 | 使用シート |
|---|---|---|
| active-effects-list.hbs | ActiveEffect CRUD | miracle, generalSkill, styleSkill, organization, cast |
| bad-status-list.hbs | バッドステータス | cast |
| history-list.hbs | EXP 履歴 | cast (レコードシートは独自実装) |
| usage-list.hbs | 用途リスト | miracle, generalSkill, styleSkill |
| prosemirror-editor.hbs | リッチテキスト | 各所 |
| scenario-setting-wizard.hbs | セットアップ | ウィザード |

## data-action パターン

シートは `[data-action]` 属性 + リスナーディスパッチが主流。

### 修正済み不一致

`active-effects-list.hbs` の action 名を EffectsSheetMixin の switch 文と一致 (KI-001 解消):

| テンプレート | ハンドラ |
|---|---|
| create | create |
| toggle | toggle |
| edit | edit |
| delete | delete |

### 解消済み

`usage-list.hbs` の用途編集ハンドラは実装済み（`_onUsageEdit`・KI-002 解消・フェーズ8-6）

## フォーム name 属性

DataModel 移行後、フィールドパスは `system.xxx` 形式 (type 名プレフィックスなし)。

例: styleSkill の comboSkill → `system.comboSkill.N.value` (NOT `system.styleSkill.comboSkill...`)

## テンプレート読み込み

`loadTemplates()` でプリロード (v13 deprecation 警告のみ、動作は v15 まで保証)。

## ProseMirror / HTMLField

description フィールドは HTMLField。編集 UI は prosemirror-editor パーシャル等を使用。

`system.json` の `htmlFields: ["description"]` でサニタイズ宣言。

## チャットカードの統一規格（2026-09-05）

**全てのチャットカードは骨格 `tnxCard` の中に段を積む。** 器・見出し・段のマークアップを各所で組まない
（ユーザー指示「そもそもチャットカードの形式を完全に統一規格にしてください」）。

```hbs
{{#> tnxCard cardClass="tnx-miracle-card" typeLabel="神業" title=name furigana=furigana}}
    {{#*inline "cardProse"}}…効果文（見出しをクリックすると開く）…{{/inline}}
    {{> tnxCardField label="残り使用回数" value=usesLabel}}
{{/tnxCard}}
```

- **見出し**＝種別タグ（何のカードか）＋名前（技能名・アイテム名・配布物名）。並びは全カードで同じ。
  **タグは卓に意味のある種別があるときだけ**——FVTT の文書型（「一般アウトフィット」等）のような内部分類は出さない
  （2026-09-05 ユーザー指摘）。アイテムの解説カードはタグ無し（名前だけ）、スタイルの解説は在ゲームの区分として「スタイル」。
- **長文を持つカードは見出しそのものが開閉の起点**（`details`/`summary`）。長文は `{{#*inline "cardProse"}}` で渡す。
  畳むための見出しを別に足さない（解説カードが以前から採っていた作法）。
- **段は機能で選ぶ**: `tnxCardField`（ラベル＝値）／`tnxCardRow`（値が組み立てを要する）／`tnxCardFold`（畳める内訳・
  見出しはそれ自体が情報の行）／`tnxCardText`（カード本体の長文）／`tnxCardResult`（結果の行）。操作は全幅縦積みの
  `.tnx-chat-btn` を本文の最後に置く。
- **JS からライブ描画する段**（ダメージ台帳・対決の状態領域・状態の解決）は `scripts/module/chat-card.mjs` の
  `cardField` / `cardResult` / `cardFold` / `cardText` を使う。折らない塊の nowrap はそこで一括して掛かる。
- **クラス名**は `tnx-card__*`（器 `.tnx-card` ／見出し `__head`・`__tag`・`__title` ／本文 `__body` ／段
  `__field`・`__fold`・`__text`・`__result`・`__outcome`・`__targets`）。同じ役割に 2 つ目の名前を作らない。
- **段ごとに指定名が違う**（`cardClass` / `fieldClass` / `rowClass` / `foldClass` / `textClass` / `resultClass`）。
  Handlebars のパーシャルは囲みの文脈を引き継ぐため、同じ名前にすると囲みの指定が中の段へ流れ込む。
- **過去ログの互換**: カードは投稿時の HTML が保存される。マークアップやクラス名を変えるときは、**保存済みの
  カードがどう見えるか**を必ず実機で確認する（`css/tnx2.css` の「過去ログのカードの互換」の節に旧名を受ける）。
- **確認**: `T:/pw/check-cards.mjs` が全カードを実機で描画し、見出し・はみ出し・語の行またぎ・畳みの既定状態を自動判定する。
