# 監査と設計レビュー

2種類の品質管理を**混同しない**。

| 種類 | 問い | 記録先 |
|---|---|---|
| メカニクス監査 | ルール解釈は正しいか？ | MECHANICS_AUDIT |
| 設計レビュー | コードとして良い設計か？ | DESIGN_REVIEW |

同じ実装に対して両方の指摘が独立に成り立つ。

## メカニクス監査 (MECHANICS_AUDIT)

### エントリ構成

```
### [機能名](フェーズX)
**日付** / **関連実装** / **ステータス**

#### ルール解釈
#### 実装
#### 監査結果 (✅/⚠️/❌)
#### 課題・申し送り
```

### 運用ルール

- 新規実装: 1エントリ追加
- 影響ある変更: ステータスを「再監査待ち」
- フェーズ完了: 全エントリ通読・最新化
- 1エントリ = Claude Code 1プロンプト単位
- **ルール本文引用・特技名網羅リスト化禁止** (著作権)

### エントリ追記先

エントリは本ファイル下部に追記する。

## 設計レビュー (DESIGN_REVIEW)

### エントリ構成

```
### [機能名](フェーズX)
**レビュー対象** / **ステータス**

#### レビュー観点 (保守性/拡張性/パフォーマンス/FVTT慣行/整合)
#### 良かった点
#### 課題
#### 推奨アクション
```

### 運用ルール

- 新規実装: 1エントリ追加
- 影響ある変更: 「リファクタ予定」等に更新
- フェーズ完了: 全エントリ通読・最新化

### 記録済みエントリ (主要)

| 範囲 | 内容 |
|---|---|
| B-0〜B-9 | DataModel 移行全サブフェーズ |
| フェーズ1-0/1/2 | v13 UI 対応 |
| 2-0〜2-2 | User 移行 |

詳細: [Design_Review_Entries.md](Design_Review_Entries.md)

## 実施タイミング

| タイミング | 作業 |
|---|---|
| 新規実装時 | MECHANICS_AUDIT + DESIGN_REVIEW 各1エントリ |
| 既存変更時 | 影響エントリのステータス更新 |
| フェーズ完了時 | 全エントリ最新化 |
| 書き手 | Claude Code |
| 最終承認 | ユーザー |

## B-0 設計方針 (DataModel 参照)

設計レビューの B-0 エントリに DataModel 全体方針が記録:

- 命名: プレフィックスなし
- 配置: `scripts/data/`
- 継承: Mixin パターン
- シート接続: 既存維持 (フェーズB スコープ外)

→ Wiki: [DataModel.md](../01_Wiki/Architecture/DataModel.md)

## 関連

- [Development_Rules.md](Development_Rules.md)
- [Phase_Operations.md](Phase_Operations.md)

---

## メカニクス監査エントリ

### [TnxRollTable（カード引きロールテーブル）](フェーズ5)

**日付**: 2026-06-11
**関連実装**: `scripts/module/tnx-draw-table.mjs`（フック注入方式で再実装）
**ステータス**: 監査済(問題なし)

#### ルール解釈

- TNX の判定・表引きはカードベースであり、ダイスを使う通常の RollTable は（ユーザー自作物を除き）使用場面がない → コアシートの完全上書きをユーザーが承認。
- 仮想 DoC は Joker / 2〜A の 14 区分で結果を出す（Joker 2/54、各数値 4/54 のフルデッキ 54 枚分布）。
- 設定デッキモードは、引かれたシーンカード（主にニューロデッキ）によって特殊効果を付与するシステムの基盤。発火点は HUD のカードドロー操作。

#### 実装

- 仮想 DoC: シート上の「ドロー」ボタン → `drawVirtualDoC()` が 54 枚分布で乱択 → 結果をチャット投稿。
- 設定デッキ: `drawNeuroCard()` 後に `lookupFromCard()` が開いているシートの deckId と照合し、カード名部分一致でエントリを引いてチャット投稿。

#### 監査結果

✅ ユーザー確認済みのルール解釈（14 区分・チャット出力・ニューロカード発火）に準拠。

#### 課題・申し送り

- カード名の部分一致（双方向 includes）は同名を含むカード名が複数ある場合に誤マッチしうる。実運用で問題が出たら完全一致オプションを検討。

#### 差し戻し記録（2026-06-11）

実装全体を revert。原因: `documentTypes.RollTable` は宣言不可（RollTable はサブタイプ非対応）であり、
サーバーが documentTypes 全体を無効化 → 全 Actor/Item/Card が検証エラーになる重大不具合が発生。
加えて、コアシートの完全置換は「元の UI・機能（結果行の編集ボタン → 別ウィンドウ編集）をほぼそのまま残す」
というユーザー要求に反していた。再設計方針: コア RollTable / TableResult / コアシートを保持し、
ドロー手段（ダイス → カード）だけを差し替える最小拡張とする。

#### 再実装記録（2026-06-11・ドロー表）

コア RollTable を保持したフック注入方式で再実装（コミット 8907d2e）。ルール対応:

- 仮想 DoC の値対応はユーザー指定どおり A〜K = 1〜13、ジョーカー = 99
  （`tnx-playing-cards.mjs` の card.value と一致することを確認済み）。
- 54 枚分布（各値 4 枚 + ジョーカー 2 枚）の仮想ドロー。
- 設定デッキモードの照合キーはカード名から抽出したラベル
  （切り札配布と同じ `：〜/` 抽出規則）。value はニューロカードで重複があるため不採用
  （コモン / カブキ がともに value=0）。
- 発火点は HUD のニューロカードドロー。シートを開いているテーブルのみ発火（有効化のセマンティクス）。

### [キャストシート詳細タブ（ライフパス・キャラクター説明）](フェーズ5-6)

**日付**: 2026-06-12
**関連実装**: `scripts/data/actor/cast.mjs`, `scripts/actor/tnx-cast-sheet.mjs`, `templates/actor/cast-sheet.hbs`
**ステータス**: 監査済(問題なし)

#### ルール解釈

- ライフパスは「出自」「経験」「邂逅」の 3 区分で構成される（ユーザー提示）。
- キャストはライフパスを各区分 1 つずつ持つ。シート上では区分ラベル・名前・概要を表示する。
- 所属組織を持たないキャストの表記は「フリーランス」（ユーザー指示。従来の「無所属」から変更）。

#### 実装

- `lifePath.origin / experience / encounter` を各 `SchemaField({itemUuid, name, summary})` に拡張。
  ドロップ時に参照 UUID と名前をキャッシュ保存（ライフパスアイテム本体の実装はフェーズ6）。
- 閲覧モード: 区分ラベル + 名前 + 概要 / 編集モード: 区分ラベル + インポートエリア（設定済みならアイテムボタン + 削除）。
- キャラクター説明エディタはシート自体の編集/閲覧モード切替に連動（トグルボタン方式の例外）。

#### 監査結果

✅ 3 区分構成・表示項目・フリーランス表記ともユーザー指示に準拠。

#### 課題・申し送り

- summary はドロップ時点では空文字。ライフパスアイテム側の概要フィールドとの連携はフェーズ6 で確定する。

#### 修正記録（2026-06-15）

- **ProseMirror 編集→閲覧切替時の自動保存バグ修正（初回）**: `_onToggleEditMode` を `async` 化し、
  `_isEditMode = false` を先にセットしてから `actor.update()` を呼ぶことで自動保存を実装。
- **切替トグルが反応しなくなるバグ修正（追補）**: `actor.update()` 後に `this.render()` を呼ばずに `return`
  していたため、説明文が未変更の場合（Foundry が差分なしと判定し更新フックを発火しない）に
  シートが再描画されず視覚的に編集モードのまま固まる不具合を修正。`await actor.update()` の直後に
  `this.render()` を追加して再描画を保証した（`tnx-cast-sheet.mjs:_onToggleEditMode`）。

### [キャストシート技能コンテキストメニュー（複製の扱い）](フェーズ5)

**日付**: 2026-06-12
**関連実装**: `scripts/actor/tnx-cast-sheet.mjs` `_activateContextMenus()`
**ステータス**: 監査済(問題なし)

#### ルール解釈

- 初期習得技能（初期技能カテゴリ・名乗り技能の初期分）の削除不可は、コンテキストメニュー経由でも維持される（ユーザー明示）。
- 複製の扱い（ユーザー承認済 2026-06-12）:
  - 初期技能カテゴリ（〈白兵〉等の基本技能）は複製不可（同一基本技能の重複所持は無意味なため）。
  - 名乗り技能の初期分を複製した場合、コピーは `isInitial = false` の通常名乗り技能になる
    （経験点を通常消費し、削除も可能。バリエーション作成がユースケース）。

#### 実装

- コンテキストメニューを「閲覧 / 編集 / 削除」（全アイテム）+「複製」（一般技能・スタイル技能のみ）に統一。
- 削除・複製の可否は ContextMenu の `condition` コールバックでアイテム状態を判定。

#### 監査結果

✅ 削除制限の維持・複製時の isInitial 解除ともユーザー承認済みの解釈に準拠。

#### 課題・申し送り

- なし。

### [装備品共通パラメータ・ライフパス区分・トループ人数の DataModel 化](フェーズ6-0)

**日付**: 2026-06-12
**関連実装**: `scripts/data/item/common/outfit-base.mjs` / `scripts/data/item/outfit-categories.mjs` /
`scripts/data/item/life-path.mjs` / `scripts/data/actor/troop.mjs`
**ステータス**: 監査済(問題なし)

#### ルール解釈

ルールの正本は [Outfits.md](../01_Wiki/Game_Rules/Outfits.md)(2026-06-12 ユーザー言語化)。要点:

- 購入値(buy)・隠匿値(hide)は「なし / 数値 / 解説参照」の 3 状態を持つ。
  購は常備化経験点と対で「購：購入値／常備化経験点」表記(解説参照時は常備化経験点を表記しない)。
  隠は危険値と対で「隠：隠匿値／危険値」表記(解説参照でも危険値は別個に設定される)。
- 常備化経験点(preserveExp)・危険値(appearancePenalty)は当初「必ず数値が入る」として NumberField で実装。
  → フェーズ6-4 で「なし / 数値」の 2 状態に変更（下記エントリ参照）。
- カテゴリは大分類 8 種 + 小分類の二階層で、小分類ごとに対応する Item type が定まる。
- 装備部位(part)はスロット消費ルール(後置数字・籠手の排他等)があるが種類が多く、UI は自由入力とする。
- ライフパスの区分は出自・経験・邂逅の 3 種のみ(旧コメントの「運命」は誤りと確認)。
- トループは人数(heads)が HP のように機能する。

#### 実装

- buy / hide → `{mode: none/value/reference, value: Number}` の SchemaField。
- preserveExp / appearancePenalty → 当初 NumberField（6-4 で {mode,value} に変更）。
- majorCategory / minorCategory → choices 付き StringField(`OUTFIT_CATEGORIES` が正本、blank 許容)。
- lifePathType → choices 付き StringField(origin / experience / encounter。キャスト側スロットキーと対応)。
- troop.heads → `{value, max}` SchemaField(リソースバー用構造。バー連携は 6-8)。
- isCarrying(携帯中)を outfitBase に追加。

#### 監査結果

✅ 3 状態の持ち分け・必ず数値のフィールド・カテゴリ対応表・ライフパス 3 区分とも
ユーザー言語化(Outfits.md)に準拠。表示形式(購/隠の対表記)はシート実装(6-1)で監査する。

#### 課題・申し送り

- ~~hack / timing / exclusive の形式は実装サブフェーズ着手時にユーザー確認~~ → 6-1 で確定(下記エントリ)。
- 購入判定(★信用)・登場判定・〈知覚〉〈電脳〉判定との連動は判定システム(フェーズ7-8)の領域。本フェーズでは扱わない。

### [アウトフィット共通シート(表示形式・hack/timing/exclusive の確定)](フェーズ6-1)

**日付**: 2026-06-12
**関連実装**: `scripts/item/tnx-outfit-sheet.mjs` / `templates/item/outfit-sheet.hbs` /
`scripts/data/item/common/outfit-base.mjs`(微調整)
**ステータス**: 監査済(問題なし)

#### ルール解釈

2026-06-12 ユーザー確認:

- hack(電脳制御値)は「なし / 数値」の 2 状態のみ(解説参照はない)。
- timing の選択肢はスタイル技能のタイミングと共通(アクション/プロセスの二段選択を含む)。
- exclusive(専用)は自由記述。自動化はしない。

#### 実装

- hack → nullable NumberField(null = なし、閲覧時は「-」表示)。
- timing → スタイル技能と同一の配列構造 + 同一の選択肢マップ(TnxSkillUtils 共用)。
- 閲覧モードの表記: 「購：購入値／常備化経験点」(解説参照時は常備化経験点を非表示)、
  「隠：隠匿値／危険値」(解説参照でも危険値は表記)— Outfits.md の表記ルールに準拠。
- カテゴリは大分類→小分類の連動ドロップダウンで、Item type に有効な小分類のみ提示。

#### 監査結果

✅ 表記ルール・3 状態の持ち分け・タイミング共通化ともユーザー確認済みの解釈に準拠。実機確認済み。

#### 課題・申し送り

- ~~isPrepared 等の表示ラベルは Code の推測~~ → 全フラグの意味をユーザーが言語化し確定
  (Outfits.md 参照。準備済み/携帯中/サイバーウェア/プリプレイ購入/オプション/生体装備/消費アイテム)。

### [戦闘系装備(weapon / armor / cyborg)の表記と構造](フェーズ6-2)

**日付**: 2026-06-12
**関連実装**: `scripts/data/item/weapon.mjs` / `scripts/data/item/common/outfit-base.mjs` /
`scripts/data/item/common/extensible.mjs` / `scripts/data/item/helpers.mjs` /
`scripts/item/tnx-outfit-sheet.mjs` / `templates/item/outfit-sheet.hbs`
**ステータス**: 監査済(問題なし)

#### ルール解釈

ルールの正本は [Outfits.md](../01_Wiki/Game_Rules/Outfits.md)(2026-06-12 ユーザー言語化)。要点:

- ダメージ種別は S(斬撃)/P(貫通)/I(衝撃)/X(装甲無視)の 4 種。「攻：I+4」表記。
- 受け値(受)はパリィ失敗時にダメージから引く値。射程(射)は 5 種で最低～最大の範囲設定。
- 消費アイテム(旧 isthrow を一般化): 個数 0 でセッション中使用不可、終了で復活、
  常備化経験点は個数(max)基準で消費しても復活しない。
- 防は「5/4/6」(S/P/I)並記、制は制御値修正。義体の攻/受は義体を直接使う宣言が必要(自動化しない)。
- スロットは装備側が数のみを設定。意識スロットは表層意識/深層意識/無意識の 3 種。

#### 実装

- 概要表記順: weapon = 購/隠/攻/受/射/ス/電制/部位、armor = 購/隠/防/制/電制/部位、
  cyborg = 購/隠/防/攻/受/電制/部位(view.summary の配列組み立てで型差分を吸収)。
- quantity {value, max} の二層で「消費しても経験点が復活しない」を構造的に表現
  (EXP 連携・セッション終了時復活の自動化は後続フェーズ)。

#### 監査結果

✅ 表記・構造ともユーザー言語化に準拠。攻撃範囲の判定アイテム連携(上書き)はフェーズ7-8 の領域。

#### 課題・申し送り

- ~~複数ダメージ種別の表記区切り~~ → damageType は単一選択に確定(6-2 実機確認フィードバック)。区切り表記は不要。
- ~~attack.mod 概要表記~~ → 実機確認で問題なし。attack は種別+値の 1 行表示で落ち着いた。

### [生活系装備(ヴィークル / 住宅施設 / 住宅エリア / コンバイナー)](フェーズ6-4)

**日付**: 2026-06-13
**関連実装**: `scripts/data/item/vehicle.mjs` / `residence.mjs` / `housing-area.mjs` /
`outfit-categories.mjs` / `scripts/item/tnx-outfit-sheet.mjs` / `tnx-housing-area-sheet.mjs`
**ステータス**: 監査済(問題なし)

#### ルール解釈

ルールの正本は [Outfits.md](../01_Wiki/Game_Rules/Outfits.md)(2026-06-13 ユーザー言語化)。要点:

- ヴィークル: SF(移動力、段階移動)/ 乗員(操縦者含む)/ 攻・防を持つ。表記順 購/隠/攻/SF/防/制/乗員/ス/電制/部位。
- 住宅エリアは厳密にはアウトフィットでなく、住宅施設への修正値の集合。住宅施設は登場判定目標値・
  セキュリティ値(電/ア)を持ち、**危険値と電制を持たない**(隠は隠匿値のみ表記)。
- 住宅オプション/アクセサリ・IANUSオプションは一般アウトフィット(general)。
- コンバイナー: 二つのアウトフィットを合成。電制は高い方、隠は X(Y)、部位/分類は両方を継承。
  本体パラメータは一般アウトフィットのもの。

#### 実装

- 住宅エリアは専用シート(説明 + 修正値 7 種)。住宅施設の重複していた *Mod は除去。
- residence は危険値・電制をシートで非表示にし、隠は隠匿値単体で表示。
- カテゴリの小分類→type 対応を配列化済みのため、general への移動は対応表の修正のみ。
- コンバイナー本体は一般アウトフィットシートで動作。合成ロジック(他アイテムの live 参照)は未実装。

#### 監査結果

✅ 各型の表記順・住宅施設/エリアの責務分離・危険値非在ともユーザー言語化に準拠。
コンバイナーの合成表示・住宅実効値算出・準備可能数は後続フェーズ(6-6 / 7-8)で扱う。

#### 課題・申し送り

- ~~コンバイナーの合成連携~~ → 6-4 で実装済み(住宅エリア連携エントリ・コンバイナーシート UI 刷新エントリ参照)。
- residence の `housingArea`(エリア ID リンク)フィールドは保持しているが、実効値算出は 6-6。

### [住宅エリア連携・コンバイン連携](フェーズ6-4 追補)

**日付**: 2026-06-13
**関連実装**: `scripts/data/item/{housing-area,combiner}.mjs` / `scripts/item/tnx-outfit-sheet.mjs` /
`scripts/item/tnx-housing-area-sheet.mjs` / `scripts/actor/tnx-cast-sheet.mjs` / `system.json`
**ステータス**: 監査済(問題なし。一体インポートは 6-6 へ)

#### ルール解釈

2026-06-13 ユーザー確定(Outfits.md):

- 住宅エリアは住宅施設への修正値の集合 + エリア(セキュリティランク)。住宅施設に compendium ドロップダウン /
  アイテムドロップの 2 方式で紐づけ、説明タブは合算実効値を表示。アクターには持たせない(ドロップ拒否)。
- コンバイナーは二つの元を合成。部位=両方占有、分類=両方、電制=高い方、隠=X(Y)。
  本体パラメータは一般アウトフィット。コンバイン条件は実装上は自由選択。

#### 実装

- housingArea に area(なし/レッド/グリーン/ホワイト/サンクチュアリ)。housing-areas pack を system.json に追加。
- 住宅施設: 住宅エリアを live 解決し、合算した実効値を説明タブに、供給値を設定タブに入力不可で表示。
- コンバイナー: combine{source1, source2, appearance}。プレビューで確定的な合成結果を算出。

#### 監査結果

✅ 住宅エリアの責務分離・合算・ドロップ拒否、コンバインの確定的合成ルールともユーザー言語化に準拠。
コンバイン条件の自由選択化もユーザー判断どおり。

#### 課題・申し送り

- 住宅エリア辞典は空で出荷(著作権配慮。ユーザーが内容を投入)。
- アクター一体インポート(コンバイナー)・住宅実効値の保存はフェーズ6-6。

---

### コンバイナーシート UI 刷新(フェーズ6-4 修正、2026-06-13)

**日付**: 2026-06-13
**関連実装**: `scripts/data/item/combiner.mjs` / `scripts/item/tnx-outfit-sheet.mjs` /
`templates/item/outfit-sheet.hbs` / `css/tnx2.css`
**ステータス**: 監査済(問題なし)

#### ルール解釈

ユーザー指摘(2026-06-13)による設計修正:

- 見た目選択は元アイテム行のインラインに配置する(standalone form-group は不適切)。
- パラメータ取捨選択 = 「両方の元に値があって食い違う項目すべてに個別ラジオ」。
  ラジオは横並び・元1/元2 ラベルを上付き(使用回数制限入力欄と同形式)。
- カテゴリはサービス/コンバイナーで確定。選ばせる必要なし。
- 性質セクションはコンバイナーでは全て false 確定のため非表示。
- 常備化経験点 = コンバイナー本体 + 元1 + 元2 の合計(「高い方を選択」ではない)。

#### 実装

- `combine.params` SchemaField を追加(8 フィールド: appearancePenalty / controlMod / attack /
  defence / guardValue / range / speedFactor / passenger、各 StringField initial "1")。
- `COMBINE_PARAM_DEFS` 定数でパラメータ定義(exists/get/fmt/eq)。
- `_prepareCombinePreview` に paramRows 生成ロジックと preserveExpTotal 計算を追加。
- `_prepareContext` でコンバイナーのカテゴリをサービス/コンバイナーに自動補正(不一致時のみ update)。
- テンプレート: 見た目ラジオをインライン化(目アイコン)、パラメータ選択セクション追加、
  カテゴリセレクタ非表示、性質 fieldset を `{{#unless isCombiner}}` で除外。
- CSS: `.combine-appearance-eye`(`:has(input:checked)` でアクセントカラー切替)、
  `.combine-param-section`。

#### 監査結果

✅ ユーザー指摘の5点すべてに対応。常備化経験点の「合計」解釈はユーザー明言どおり。
paramRows は「両方に存在かつ値が食い違う場合のみ」生成するため余分なラジオは表示されない。

#### 課題・申し送り

- 実機での UI 確認(目アイコン操作感、パラメータラジオの視認性)はユーザーが実施。
- アクター一体インポートはフェーズ6-6 で対応予定。

---

### [DataModel フィールドの {mode,value} 全面統一・isBiological 削除](フェーズ6-4 追補 2)

**日付**: 2026-06-14
**関連実装**: `scripts/data/item/helpers.mjs` / `scripts/data/item/common/outfit-base.mjs` /
`scripts/data/item/common/extensible.mjs` / `scripts/data/item/{armor,cyborg,tap,vehicle,ianus,weapon}.mjs` /
`scripts/data/item/housing-area.mjs`（hideMod 削除）/ `scripts/data/item/residence.mjs`（useHousingAreaDrop 追加）
**ステータス**: 監査済(問題なし)

#### ルール解釈

2026-06-14 ユーザー確定（Outfits.md 更新済み）:

- preserveExp（常備化経験点）は**「なし / 数値」の 2 状態**。常備化経験点がないアウトフィットが存在する
  （旧解釈「必ず数値が入る」を修正）。
- appearancePenalty（危険値）も**「なし / 数値」の 2 状態**（同上）。
- guardValue（受け値）/ defence（防御値）/ controlMod（制御値修正）/ speedFactor / passenger /
  cycle / combatSpeedMod / slots[].count も同様に**「なし / 数値」の 2 状態**。
  「0」と「そもそもない」を区別する意味がある（例: 防0は防がある武器に見える誤解を避けるため「-」表示が正しい）。
- range（射程）の最低射程に「なし」を追加（射程自体がないアウトフィット）。最長射程から「至近」を除く
  （最低射程が「至近」の単点表記で完結するため、最長が「至近」になるケースはない）。
- isBiological は削除。生体装備はカテゴリ「生体装備」で識別する方針に確定。
- 住宅エリアの修正値から hideMod を削除。住宅施設の隠匿値は住宅エリアを加算しない
  （hide は住宅施設固有の値であり、エリア側から修正されない。ユーザー確認済み）。

#### 実装

- `modeValueField(choices)` を `outfit-base.mjs` 局所定義から `helpers.mjs` へ昇格（全モデルで共有）。
- `defenceField()` に `mode: none/value` を追加（armor / cyborg / vehicle）。
- outfitBase: preserveExp / appearancePenalty を `{mode,value}` 化、isBiological を削除。
- extensible: slots[].count を `{mode,value}` 化。
- 個別フィールド: armor.controlMod / cyborg.guardValue / tap.cycle + combatSpeedMod /
  vehicle.speedFactor + passenger + controlMod / ianus.controlMod / weapon.guardValue を `{mode,value}` 化。
- weapon.range: `WEAPON_RANGE_MIN_OPTIONS`（なし+5択）と `WEAPON_RANGE_MAX_OPTIONS`（なし+4択、至近除外）に分離。
- 全モデルに `static migrateData(source)` を実装（旧 NumberField 形式のデータを `{mode,value}` に移行）。
- housing-area: `hideMod` フィールドを削除（MOD_FIELDS 定数も更新）。
- residence: `useHousingAreaDrop: BooleanField` を追加（ワールドアイテムドロップ時の状態保持フラグ）。

#### 監査結果

✅ preserveExp/appearancePenalty の「なし/数値」化・guardValue 等の同統一・isBiological 削除・hideMod 削除・
range 選択肢分離ともユーザー確認済みの解釈に準拠。

#### 課題・申し送り

- シート側の modeValueField 表示対応（outfit-sheet / cast-sheet）は同日コミット済み（別設計レビューエントリ）。
- activeEffect による {mode,value} フィールドへの修正（強制上書き vs 加算）は ActiveEffect フェーズで設計する。

---

### lifePath アイテム + キャスト詳細タブ連携（フェーズ6-5）

**日付**: 2026-06-14
**レビュー対象**: `scripts/data/item/life-path.mjs`（lifePathType / skillName）、`scripts/data/actor/cast.mjs`（lifePath スロット構造）
**ステータス**: 監査済(問題なし)

#### ルール解釈

2026-06-14 ユーザー確定:

- lifePath アイテムの **lifePathType**（種別）は「—（指定なし）/ 出自 / 経験 / 邂逅」の 4 値。
  キャストシートの 3 スロット（origin / experience / encounter）と値を統一することで、
  ドロップ時のスロット・型整合チェックが単純な文字列比較で済む。
  **空文字（—）はどのスロットにもドロップ可能**（汎用ライフパスの想定）。
  非空値がスロットキーと一致しない場合のみ拒否してユーザーに通知。
- **skillName**（習得技能）は**自由記述文字列**として保存する。
  自動化する場合、その技能名を持つ **固有名詞技能**（`generalSkillCategory: "onomasticSkill"`）アイテムを
  キャストアクターに直接生成する設計を予定（フェーズ未定）。
  技能 UUID の参照としないのは、固有名詞技能はキャラクターと共に生成されるものであり、
  既存技能アイテムとは別のライフサイクルを持つため。
- キャストシートの lifePath スロットに表示する **description** は、
  ライフパスアイテムの `system.description` を毎回 fromUuid で live 解決して表示する。
  `summary` フィールドをキャスト DataModel に持たせてドロップ時に同期するキャッシュ方式は採用しない
  （フェーズ6-0 でユーザーが確定した「名前キャッシュ以外の値キャッシュなし」方針に準拠）。

#### 実装

- `scripts/data/actor/cast.mjs`: lifePath 各スロットから `summary` フィールドを削除。
  残存フィールドは `itemUuid`（参照 UUID）と `name`（削除時フォールバック用キャッシュ名）の 2 つのみ。
- `scripts/data/item/life-path.mjs`: `lifePathType` の choices に origin / experience / encounter を追加済み（6-0）。`skillName: StringField` は 6-0 より存在。
- `scripts/item/tnx-life-path-sheet.mjs`（新設）: 説明タブ（ProseMirror）+ 設定タブ（種別・習得技能・識別キー）の 2 タブ構成。
- `scripts/actor/tnx-cast-sheet.mjs`: `_onDropItem` にライフパスのみ受け付ける型チェックと lifePathType × スロットキー整合チェックを追加。`_prepareContext` で async ループにより fromUuid live 解決 + TextEditor.enrichHTML を実施。

#### 監査結果

✅ lifePathType のキーとスロットキーの統一・空文字→どこでも受付の型検証ロジック・skillName の文字列保持方針・
description の live 解決方針、いずれもユーザー確認済みの解釈に準拠。

---

### アウトフィット集計・携帯/準備フラグ（フェーズ6-6）

**日付**: 2026-06-14
**レビュー対象**: `scripts/tnx.mjs`（集計関数群）、`scripts/actor/tnx-cast-sheet.mjs`（フラグトグル UI）、
`scripts/data/actor/cast.mjs`（outfitMod / appearanceModifier）、`scripts/data/item/common/outfit-base.mjs`（hide "control" モード）

#### ルール解釈

2026-06-14 ユーザー確定:

- **携帯中（isCarrying）**: アクトに「持って来ている」状態。危険値（appearancePenalty）は携帯中のアウトフィット全体から算出し、
  キャストの `appearanceModifier` に集計する（外見値判定で参照するため）。
- **準備済み（isPrepared）**: 即使用可能な状態（「装備中」相当）。制御値修正（controlMod）は
  **準備済み**かつ携帯中のアウトフィットのみ加算する。
- **CS 修正（combatSpeedMod）**: 携帯中であれば準備済み不問で加算する（TAP は常時稼働のため）。
  ただし**ゴースト登場中は CS 修正を無効化**する（ゴーストは物理現場不在のため）。
- **住居（residence）は「携帯」の概念が存在しない**ため isCarrying トグルを表示しない。
  住居が提供する修正値は住宅エリア集計で別途処理する（フェーズ未定）。
- **`hide` モードの "control"**: 制御値依存の隠匿値を表す 4 番目のモード。
  数値固定の "value" / ルルブ参照の "reference" に加えて追加。

#### 実装

- `scripts/tnx.mjs`:
  - `calcAppearanceModifier(actor)`: 携帯中かつ `appearancePenalty.mode === "value"` のアウトフィットの
    `appearancePenalty.value` を合算し `actor.system.appearanceModifier` に書き戻す。
  - `calcOutfitMods(actor)`: 準備済み携帯中の `controlMod.value` を合算して `outfitMod.control` に格納。
    CS 修正は `isGhost` 非時のみ加算（ghost ガードを関数内に持つ）。
  - `createItem` / `deleteItem` / `updateItem` フックで `recalcOutfitAggregates` を呼び出し。
    `ready` 時に全キャストを対象に初期集計を実施。
  - `updateActor` フック: `isGhost` 変更検出時に `updateCastOutfitMods` を再実行。
- `scripts/actor/tnx-cast-sheet.mjs`:
  - `_onToggleOutfitFlag`: `isCarrying` / `isPrepared` を行内ボタンでトグル。
    `isPrepared` は `isCarrying === false` の場合 disabled（携帯しない装備品を準備済みにできない）。
  - `_onRecalculateBounty`: 現在の `mundaneTotalValue`（外界値合計）を `bountyBase` として DB に書き込む。

#### 監査結果

✅ isCarrying / isPrepared の分離・控除対象の差分（control は準備済み必須、CS は携帯のみ）・
isGhost 時の CS 無効化・住居の isCarrying 非表示・hide "control" モード追加、
いずれもユーザー確認済みのルール解釈に準拠。

---

### [判定エンジン・カード値計算・スート対応](フェーズ8-2)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-judgment-engine.mjs`
**ステータス**: 監査済(問題なし)

#### ルール解釈

- カード値: 2〜10 は数字通り、J/Q/K = 10（手札ではファンブルにならない）、A = 11 または達成値21固定（プレイヤーが選択）、Joker = 宣言値。
- 山札判定で J/Q/K が出た場合のみファンブル（`isFromDeck: true` でトリガー）。
- スート → 能力値キー: ♠理性 / ♣感情 / ♥生命 / ♦外界。出したカードのスートで能力値が自動決定する。
- 技能判定: 達成値 = カード値 + 対応能力値実効値 + 報酬点。技能レベルは加算しない。
- 制御判定: カード値 ≤ 制御値で成功（下方判定）。A は 11（21固定オプションは制御判定では使用不可）。
- スート不一致: 起動を拒否しない。カードをプレイして達成値 0 の通常失敗扱いで投稿する。
- 組み合わせ技能の使用可能スート: 各技能の suits の積集合（AND）。

#### 実装

- `SUIT_TO_ABILITY` 定数: `{ spade: "reason", club: "passion", heart: "life", diamond: "mundane" }` を `Object.freeze` で不変化。
- `getCardJudgmentValue({ numericValue, isJoker, fixedAt21, declaredValue, isFromDeck })`: Symbol 値（FUMBLE_VALUE / FIXED_21）を使い、型安全に「ファンブル」「21固定」を表現。
- `calcSkillCheck`: `bountyUsed` を加算。A の 21固定は `fixedAt21 === true` のとき `achievement = 21` で完全固定（能力値・報酬点を加算しない）。
- `calcControlCheck`: 制御値 = `abilitiesCtx[key].totalControl`。`success = cardValue <= controlVal`。
- `getComboSuits(skillSystems)`: `ALL_SUITS.filter` で積集合を計算。

#### 監査結果

✅ カード値ルール（手札絵札は 10、山札絵札はファンブル）・スート対応表・能力値計算式・制御判定の下方判定・スート不一致 = 達成値 0 失敗、いずれもユーザー確認済みのルール解釈に準拠。

#### 課題・申し送り

- AE ボーナス（ActiveEffect）は現在未集計。`abilitiesCtx[key].totalValue` の計算式に AE 分が含まれていないため、AE フェーズ完了後に `_buildAbilitiesCtx` を拡張する。

---

### [判定フロー・HUD 連携・報酬点フロー](フェーズ8-3)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-judgment-flow.mjs`, `scripts/module/tnx-judgment-dialog.mjs`, `scripts/module/tnx-hud.mjs`
**ステータス**: 監査済(問題なし)

#### ルール解釈

- 判定フロー（技能判定）: ① カードを出す → ② 報酬点の消費量を決定 → ③ 達成値計算 → ④ チャット投稿。
- 報酬点選択: カード選択後に別プロンプトで行う（カード選択前ではない）。ルール上「カードを出した時点で判定成立 → その後に報酬点を使うかどうか決定」の順序に準拠。
- スート不一致: 起動（カードのプレイ操作）は拒否しない。達成値 0 の通常失敗として投稿する。ファンブルとは区別しない。
- 切り札: 切り札を消費することを宣言 → 手札から 1 枚を Joker として使用 → 切り札はシーンカードへ。
- ×ボタンでダイアログを閉じると判定キャンセル（ユーザー要求、2026-06-15 確定）。

#### 実装

- `TnxJudgmentFlow` (singleton 静的クラス): `_context` / `_trumpMode` で状態管理。ダイアログクラスは `tnx.mjs` で注入（循環 import 回避）。
- `_execute`: ① ダイアログ close・state clear → ② カードプレイ（`TnxActionHandler.playCard`）→ ③ HUD 再描画 → ④ `_promptBountyUsage`（スート不一致・制御判定はスキップ）→ ⑤ 達成値計算 → ⑥ チャット投稿。
- `TnxJudgmentDialog._onClose`: `TnxJudgmentFlow.cancel()` を呼び、×ボタンでも判定キャンセルを実現。
- HUD カード達成値プレビュー: `getCardJudgmentValue + getAbilityBySuit` でホバー時に数値バッジを表示。スート不一致は赤バッジで「0」表示。

#### 監査結果

✅ 報酬点フローの順序（カード後）・スート不一致の扱い・切り札消費フロー・×ボタンキャンセル、いずれもユーザー指示に準拠。

#### 課題・申し送り

- 制御判定の `achievement` フィールドは `null`（達成値概念がない）。チャットテンプレートで制御判定専用の表示分岐を使用。将来的に型を明確化する場合はオブジェクト構造の見直しが必要。

---

### [報酬点フロー修正: usesBounty チェック・spinner 修正](フェーズ8-3 バグ修正)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-judgment-flow.mjs`, `scripts/actor/tnx-cast-sheet.mjs`
**ステータス**: 監査済(修正完了)

#### ルール解釈

- 報酬点を判定に使えるかどうかは、技能ごとに決まる（一般技能の `usesBounty` フラグで制御）。
- デフォルトは「使用不可」。ルールブックで明記された技能のみ `true` に設定。
- スタイル技能・制御判定・能力値判定には報酬点を適用しない（`bountyAvailable = 0`）。

#### 実装

- `_onStartSkillCheck`: `item.system.usesBounty === true` のときのみ `bountyAvailable = bountyBase + bounty`、そうでなければ `0`。これにより `_promptBountyUsage` 自体は変更不要（`bountyAvailable = 0` なら即 return 0）。
- `_promptBountyUsage`: `DialogV2.prompt` → `DialogV2.wait` に切り替え。`amount-input-dialog.hbs` テンプレートと `actions: { decrement, increment }` で number-input-spinner を実現。
- `_promptJokerDeclaration`: 値入力フィールドを `<input type="number">` → number-input-spinner に変換（`DialogV2.wait` + `actions: { decrementJokerValue, incrementJokerValue }`）。

#### 監査結果

✅ `usesBounty` フラグチェックによりルール通りの制御になった。number-input-spinner への変換は UI 規約（全数値入力はスピナー）に準拠。

#### 課題・申し送り

- HUD カードホバーの達成値プレビューは報酬点を含まない（カード選択前なので含められない）。これは設計上の仕様であり、プレビューは暫定値として扱う。

---

### [HUD プレビュー表示改善](フェーズ8-3/8-4)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-hud.mjs`, `templates/hud/hud.hbs`, `css/tnx2.css`
**ステータス**: 監査済

#### ルール解釈

- 判定待機中は「使用できないスートのカードを光らせない」のみで十分。使用可能スートを金色グローで強調し、それ以外は通常表示（モノクロ化しない）。
- プレビュー数値はカードの上に重ならず、カードの上部に独立して表示する（ユーザー確認済み）。

#### 実装

- `getAbilityBySuit()` が `{abilityKey, totalValue, totalControl}` オブジェクトを返すため、`.totalValue` で参照するよう修正（文字列連結バグ `n[object Object]` を解消）。
- CSS: モノクロ化ルール（`opacity: 0.35; filter: grayscale(80%)`）を削除。有効スートへのゴールドグローのみ残す。
- HUD テンプレート: `.card-in-hand` を `.card-hand-slot` ラッパーで包み、`.card-preview-label` を flex-column の先頭に配置（カードより上に表示）。
- ホバーアニメーション: スロット単位で `transform: translateY` を適用し、ラベルとカードが一体で浮き上がる。

#### 監査結果

✅ プレビュー表示位置・モノクロ廃止・達成値計算ともにユーザー指示に準拠。

#### 課題・申し送り

なし。

---

### [RL 判定要求フロー](フェーズ8-5)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-rl-request-app.mjs`, `scripts/tnx.mjs`, `templates/app/rl-request-app.hbs`, `templates/chat/judgment-request.hbs`
**ステータス**: 監査済

#### ルール解釈

- RL（GM）が判定種別・技能・目標値・対象 PL を指定して判定を要求する。
- 技能判定: identificationKey で技能を同定。PL は自分のキャラクター上で同一 identificationKey の技能を検索して判定。未所持の場合は判定不可。
- 能力値判定: 指定能力値のスートで判定。報酬点使用可。
- 制御判定: 指定能力値のスートで制御判定。報酬点使用不可。
- 目標値の非公開: PL には「（非公開）」と表示、GM には実値を表示。
- RL モードは全公開のみ（TNX はカードベースのため非公開判定が原理的に成立しない）。

#### 実装

- `TnxRlRequestApp` (ApplicationV2 + HandlebarsApplicationMixin): フォーム送信で `ChatMessage.create` を呼び、`flags.judgmentRequest` にデータを保持。
- `renderChatMessage` フック: 対象 PL（または GM）にのみ「判定する」ボタンを表示。結果 (`results[actorId]`) が存在する場合はボタンを結果表示に切り替え。
- `TnxRlRequestApp.onDoJudgment()`: identificationKey でアクター上の技能を解決し、`getComboSuits()` で validSuits を取得。技能未所持の場合は warn して早期 return。
- `getSceneControlButtons` フック: V13 では controls がオブジェクト（`tokens` キー）。配列・Map・オブジェクトの全パターンをハンドリング。

#### 監査結果

✅ identificationKey による技能同定・技能未所持の早期 return・rollMode 廃止（全公開のみ）・目標値非公開フラグ、いずれもユーザー指示に準拠。

#### 課題・申し送り

- `TnxRlRequestApp` ダイアログ内のチェックボックスは Foundry V13 の `appearance:none` + FA アイコン描画との相性で視覚的に不整合が残る（機能は正常）。将来的に CSS 調整の余地あり。
- コンペンディウム技能のキャッシュは `TnxRlRequestApp._compendiumSkillCache` に保持。セッション中にコンペンディウムが更新された場合は反映されない（許容範囲）。
- `renderChatMessage` フック名が V13 では `renderChatMessageHTML` に変更済み（2026-06-15 修正）。

---

### [Joker 宣言ダイアログ修正](フェーズ8)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-judgment-flow.mjs`
**ステータス**: 監査済

#### ルール解釈

- Joker はスートと「どの札として使うか（A〜K のランク）」をプレイヤーが宣言する。
- 宣言後はそのランクの通常カードとして扱う: A=11（または21固定）、J/Q/K=10（手札からは FUMBLE しない）、2〜10=数値通り。
- Joker であってもスート不一致の宣言は可能（プレイヤーに失敗する権利があるため）。
- 山札から Joker が出た場合、J/Q/K と宣言すると FUMBLE（山札判定の絵札ルールが適用される）。

#### 実装

- `_promptJokerDeclaration()` から `validSuits` 引数を廃止し、常に全4スート（`ALL_SUITS`）を表示。
- 値入力を「1〜13の数値スピナー」から「A, 2-10, J, Q, K のドロップダウン」に変更。
- 戻り値を `{ suit, numericValue }` に変更（旧: `{ suit, value }`）。
- `executeFromHand` / `executeFromDeck` / `_executeAsTrumpJoker` で `getCardJudgmentValue({ numericValue })` を呼び出す（`isJoker: true` の特例ルートを廃止）。
- `_handleAceChoice` に `extraParams = {}` を追加し、`fromDeck` / `trumpUsed` を引き継ぎ可能に。

#### 監査結果

✅ 全4スート選択可・宣言ランクが通常カードと同じルールで計算される点、ルールに準拠。

#### 課題・申し送り

なし。

---

### [山札判定スート読み取りバグ修正](フェーズ8)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-judgment-flow.mjs`
**ステータス**: 監査済

#### ルール解釈

- 山札から引いたカードのスートで能力値を決定する。

#### 実装（バグ修正）

- `executeFromDeck` で `discard.draw()` 直後のカードは裏向き（face: -1）であり `card.suit` が null になる問題を修正。
- `updateEmbeddedDocuments("Card", [{ face: 0 }])` の await 後に `discard.cards.get(card.id)` で再取得し、表向き状態のスートを読む。
- スート不一致（有効スートに含まれないスート）の場合は達成値 0 の失敗として処理（手札判定と同様に `suitMismatch: true` を設定）。

#### 監査結果

✅ 修正後は引いたカードの実スートで能力値を決定する。スート不一致処理はルール「両判定に適用」に準拠。

#### 課題・申し送り

なし。

---

### [報酬点ダイアログ改善・FUMBLE 時スキップ](フェーズ8)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-judgment-flow.mjs`
**ステータス**: 監査済

#### ルール解釈

- ファンブル確定時は報酬点を消費できない（報酬点ダイアログを表示しない）。
- 報酬点ダイアログには報酬点加算前の現在の達成値を表示し、プレイヤーが加算効果を判断できるようにする。

#### 実装

- `_execute` の条件に `cardJudgmentValue !== "FUMBLE"` を追加。FUMBLE 確定時は `_promptBountyUsage` を呼ばない。
- `abilitiesCtx` を `_execute` の先頭で一度だけ構築（従来: 計算直前）。
- 報酬点 0 時のベース達成値を `calcSkillCheck(bountyUsed: 0)` で先計算し、`_promptBountyUsage` に `baseAchievement` として渡す。
- ダイアログに「現在の達成値: N（報酬点1点 = +1 加算）」を表示。

#### 監査結果

✅ ファンブル時のダイアログ非表示・ベース達成値の事前表示、いずれもルールおよびユーザー要件に準拠。

#### 課題・申し送り

なし。

---

### [手札補充自動化](フェーズ8)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-action-handler.mjs`（`autoReplenishHand`）、`scripts/module/tnx-judgment-flow.mjs`（`_execute`）
**ステータス**: 監査済

#### ルール解釈

- 手札からカードを出した直後、**手札上限まで自動補充する**。
- 山札判定では手札を消費しないため自動補充は行われない。
- 山札が不足している場合は引ける枚数だけ引く（エラーにしない）。

#### 実装

- `TnxActionHandler.autoReplenishHand()` を新設: 上限 - 現在手札枚数を計算し、その枚数を山札からドロー。
- `TnxJudgmentFlow._execute()` で `playCard()` の直後に `autoReplenishHand()` を呼び出す（`!fromDeck` のブランチ内）。

#### 監査結果

✅ 手札消費直後の自動補充タイミング、山札不足時の安全処理、山札判定では発動しない点、いずれもルールに準拠。

#### 課題・申し送り

なし。

---

### [達成値プレビューをホバー時のみ表示](フェーズ8)

**日付**: 2026-06-15
**関連実装**: `templates/hud/hud.hbs`, `css/tnx2.css`
**ステータス**: 設計レビュー済

#### 設計

- `.card-preview-label` を `.card-in-hand` 内部（カード画像の後）に移動。
- `position: absolute; top: 0; left: 0; right: 0` でカード上端にオーバーレイ表示。
- デフォルト `opacity: 0; visibility: hidden`。`.card-hand-slot:hover .card-preview-label:not(:empty)` でフェードイン。
- `.card-hand-slot` の `gap: 2px` を削除（ラベルがフロー外になったため不要）。
- `.card-in-hand` はすでに `position: relative` であり変更不要。
- `overflow: hidden` を持つ `.hand-cards-row` のクリッピング問題を回避（ラベルがカード内に収まるため）。

#### 監査結果

✅ 常時表示による手札エリア上部の余白が解消。ホバー時のみ表示でレイアウト体系を破壊しない。

---

### [HUD空エリアクリックによるカード補充](フェーズ8)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-hud.mjs`（`_onReplenishHand` / `_onResetRlTrump`）、`templates/hud/hud.hbs`（空手札プレースホルダー）、`scripts/module/tnx-action-handler.mjs`（`autoReplenishHand` / `resetRlTrump`）
**ステータス**: 設計レビュー済

#### 設計

- **手札エリア（PL・GM共通）**: 手札が0枚のとき表示されるプレースホルダー（`.card-empty-placeholder`）に `data-action="replenishHand"` と `.card-empty-clickable` クラスを付与。クリックで `TnxActionHandler.autoReplenishHand()` を呼び出す。
- **切り札エリア（GM専用）**: GMの切り札エリアが空のとき `data-action="resetRlTrump"` + `.card-empty-clickable` クラスを付与。クリックで `TnxActionHandler.resetRlTrump()` を呼び出す。非GMは非クリッカブルなプレースホルダーのみ。
- いずれも `TnxHud.DEFAULT_OPTIONS.actions` に登録し、`_onReplenishHand` / `_onResetRlTrump` ハンドラが呼び出される。
- `.card-empty-clickable` CSS クラスで `cursor: pointer` + ホバー時アクセントカラー変化を付与（既存スタイル再利用）。

#### 監査結果

✅ 手札補充は `autoReplenishHand()` を再利用しており、上限計算ロジックの重複がない。切り札リセットはGM専用制御が `hbs` 側の条件分岐で保証されている。

---

### [カード操作チャット通知の全面抑制](フェーズ8)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-action-handler.mjs`、`scripts/module/tnx-hud.mjs`、`scripts/journal/tnx-scenario-sheet.mjs`
**ステータス**: 設計レビュー済

#### 設計

- Foundry V13 の Cards API（`draw` / `pass` / `recall` / `shuffle`）はデフォルトでチャットへ操作通知を投稿する。
- TNX では独自の `ui.notifications` によるフィードバックを使用しており、Cards API の自動通知はノイズになるため、全操作に `{ chatNotification: false }` を付与して抑制する。
- 対象ファイルはシステム内の Cards API 呼び出し箇所すべて。`tnx-draw-table.mjs` の `table.draw()` は `RollTable` APIであり別系統のため除外。

#### 監査結果

✅ 全 Cards API 呼び出し箇所（`tnx-action-handler.mjs` / `tnx-hud.mjs` / `tnx-scenario-sheet.mjs` / `tnx-card-setup-app.mjs` / `tnx-judgment-flow.mjs`）に `chatNotification: false` が付与されていることを確認。

---

### [usage edit 接続・skillRefs 編集 UI（KI-002）](フェーズ8-6)

**日付**: 2026-06-15
**関連実装**: `scripts/item/tnx-item-sheet.mjs`（`_onUsageEdit`）、`scripts/module/tnx-dialog.mjs`（`UsageCreationDialog.editAction`）、`templates/dialog/usage-edit-dialog.hbs`
**ステータス**: ~~設計レビュー済~~ → **フェーズ8-6改訂(2026-06-17)で TnxUsageSheet に全面移行、このエントリは無効**

（後継エントリ: 下記「TnxUsageSheet 全面実装・コンボ UI 完成（フェーズ8-6改訂）」を参照）

---

### [UsageTemplate DataModel 全面改訂・用途タイプ確定](フェーズ8)

**日付**: 2026-06-17
**関連実装**: `scripts/data/item/common/usage.mjs`
**ステータス**: 監査済(問題なし)

#### ルール解釈

2026-06-17 ユーザー確定:

- 用途タイプは `check` / `attack` / `declaration` / `damageBoost` / `damageReduce` / `modification` の 6 種（controlCheck / miracle は廃止）
- `timing.value = "blank"` / `target = "blank"` は「自由（指定なし）」= どのタイミング・対象とも組み合わせ可能（制約なし）
- コンボルール修正: アクション技能は**必ずベース技能になる**が、「組み合わせにはアクション技能が必要」は誤り。非アクション技能もベースになりうる
- ベース技能（`baseSkillRef`）は用途ドキュメントが明示的に保持する。generalSkill 新規作成時に親技能 ID で自動設定

#### 実装

- `_id` フィールド追加（randomID）: インデックス参照を廃止し ID で管理
- `timing`（value/actionName/processName/timingOther）/ `target` / `effects` / `baseSkillRef` / `weaponRef` / `damageType` / `formula` / `damageCategory` / `modifiableParams` を追加
- `migrateData`: 既存エントリに `_id` と `baseSkillRef` を自動付与
- `preCreateItem` フック: `generalSkill` 新規作成時に `check` 用途を 1 件自動挿入

#### 監査結果

✅ 用途タイプ 6 種・blank=自由の意味・baseSkillRef 導入方針、いずれもユーザー確認済みの解釈に準拠。

#### 課題・申し送り

- `usage.test.mjs` に `type.initial === ""` を期待する既存テストがある（現在は `"check"`）。フェーズ8-7 で修正予定。

---

### [TnxUsageSheet 全面実装・コンボ UI 完成（フェーズ8-6改訂）](フェーズ8)

**日付**: 2026-06-17
**関連実装**: `scripts/module/tnx-usage-sheet.mjs`（新設）、`templates/app/usage-sheet.hbs`（新設）、`scripts/item/tnx-item-sheet.mjs`、`templates/parts/usage-list.hbs`、`scripts/actor/tnx-cast-sheet.mjs`（`_onStartSkillCheck` 改訂）、`css/tnx2.css`
**ステータス**: 設計レビュー済

#### 設計

- **TnxUsageSheet（ApplicationV2）**: UsageCreationDialog を全面置き換え。タブ構成（基本 / 起動 / 種別固有 / エフェクト）、`submitOnChange` 自動保存。
- **`_patchUsage` のバグ修正**: 旧実装の `mergeObject` は配列フィールド（skillRefs 等）でマージ挙動になり配列が壊れ、またドット記法キー（`baseSkillRef.itemId`）も正しく書き込まれない問題があった。`setProperty` ループに変更して解消。
- **コンボ UI**:
  - ベース技能候補（`availableBaseSkills`）: アクション技能を含む全技能（自身以外）
  - コンボ候補（`availableSkills`）: アクション技能を除外（アクション技能は必ずベースになるため）
  - アクション技能の用途はベース技能をテキスト固定表示（セレクター非表示）
  - ドロップダウン選択で skillRefs に即時追加（「追加」ボタン廃止）
  - スタンドアロンアイテム（`actor === null`）でも親アイテム自身の名前を表示
- **`_onStartSkillCheck` 改訂**: 用途（check タイプ）を起点に baseSkillRef からベース技能を解決、skillRefs + 親技能自動追加でコンボを構成

#### 監査結果

✅ アクション技能のベース固定・非アクション技能のベース可能・コンボ候補からアクション技能を除外、いずれもユーザー確認済みのルール解釈（2026-06-17 修正後）に準拠。

#### 課題・申し送り

- 「コンボ設定の他技能への自動ミラーリング」（組み合わせ設定を1つの技能に設定すると参加技能全員に反映）は実装を先送り。必要が生じた際に追加する。
- タイミング条件のバリデーション（全技能タイミングの一致確認）は未実装。現状はスートの積集合のみ確認。

---

### [用途シート エフェクトタブをセレクト方式へ再設計](フェーズ8-6改訂)

**日付**: 2026-06-20
**関連実装**: `scripts/module/tnx-usage-sheet.mjs`、`templates/app/usage-sheet.hbs`、`css/tnx2.css`
**ステータス**: 設計レビュー済

#### 設計

- 旧実装（アイテムの全 ActiveEffect をチェックボックスで羅列し、トグルで `usage.effects` を増減）を、D&D 5e の「適用される効果」方式へ置換。
- **セレクト追加**: 未追加の ActiveEffect をセレクトボックスで選ぶと `usage.effects` に `{effectId}` を即時追加。`select.effect-select` の change ハンドラは skillRefs と同じ `_onRender` の editable ブロックに集約。
- **追加済みリスト＋解除**: 追加済みは行表示し、`effectRemove` アクションでデタッチ（効果ドキュメント自体は削除しない＝アイテムシートのエフェクトタブの責務と分離）。
- **空状態**: アイテムに ActiveEffect が無くても空のセレクトを表示（ユーザー要求）。`addedEffects` / `availableEffects` / `hasAnyEffect` をコンテキストに供給。
- **UI 統一・CSS**: 行は既存の `.usage-skill-ref-row`、追加セレクトは `.usage-skill-ref-add` を流用。不要になった `.usage-effect-row` / `.usage-effect-label` 系を削除（汎用セレクタの拡張なし）。

#### 監査結果

✅ ルール解釈に関わる変更なし（純 UI）。`usage.effects`（`{effectId}` 配列）の DataModel は不変で、フィールド追加・改名なし（§4.3 抵触なし）。

#### 課題・申し送り

- 用途使用時に効果を実際に対象へ適用するランタイム処理は ActiveEffect 基盤（フェーズ9）の領域。本変更は参照の保持・編集 UI まで。
- 他タブ（起動・種別固有）への追加項目は今回見送り。`targetValue`（スタイル技能内包の目標値）・`range`・`confrontation`（対決）はルール上存在するが、消費する判定フローが攻撃判定・対決（フェーズ12）／AE ボーナス集計（フェーズ9）に属するため、§4.3「先回りで作らない」に従い現フェーズでは usage への追加を行わない。

---

### [用途シート発動タブ拡張・使用回数消費（メカニクス監査）](フェーズ8後追い)

**日付**: 2026-06-20
**関連ルール**: 射程の優先度・使用回数の消費（[Judgment_Rules.md](../01_Wiki/Game_Rules/Judgment_Rules.md)）

#### 監査対象のルール解釈（ユーザー確定）

- **射程の優先度**: 至近※ ＞ 武器 ＞ 超遠 ＞ 遠 ＞ 中 ＞ 近 ＞ 至近。変更不可（※）が複数なら最短を優先。ルルブに優先度記述がなくユーザーが裁定・追記要望したもの。
- **使用回数の消費**: 消費は判定（用途）起動に紐づく機能であり、回復/リセット（フェーズ15）から分離。参加技能（ベース＋コンボ）の使用回数制限を各 −1、組み合わせ技能は遠隔消費。Roadmap フェーズ15「行動回数制限は自動化しない」（＝ターン行動回数）とは別物。
- **対決**: 相手側リアクションの自動解決はしない（フェーズ13）。対決不可フラグ（`isUnopposable`）のみ今フェーズで保持、対決可能技能は参加技能から読み取り専用表示。

#### 監査結果

✅ いずれもユーザーが明示的に確定・裁定したルール解釈に準拠。射程優先度は一次ルールに記述がないためユーザー裁定を正本として記録。

---

### [用途シート発動タブ再構成・自動入力・使用回数消費（設計レビュー）](フェーズ8後追い)

**日付**: 2026-06-20
**関連実装**: `scripts/data/item/common/usage.mjs`、`scripts/module/tnx-usage-sheet.mjs`、`templates/app/usage-sheet.hbs`、`templates/app/usage-sheet-combo.hbs`（新設）、`scripts/tnx.mjs`（partial 登録）、`scripts/module/tnx-judgment-flow.mjs`、`scripts/actor/tnx-cast-sheet.mjs`、`css/tnx2.css`
**ステータス**: 設計レビュー済

#### 設計

- **DataModel（usage.mjs）**: スタイル技能と同形の flat フィールドを 9 個追加（`targetOther` / `isFixedTarget` / `range` / `rangeOther` / `isFixedRange` / `targetValue` / `targetValueNumber` / `targetValueOther` / `isUnopposable`）。全て安全初期値で ArrayField(SchemaField) の cleanData が既存エントリへ自動補完するため migrateData 改変不要。
- **タブ再構成**: 基本 / 発動 / 効果 の3タブ。種別固有タブを効果タブへ統合。ベース＋コンボ UI を `usage-sheet-combo.hbs` パートに切り出し共用。
- **自動入力**（`_onAutoFill`）: 参加技能の固有値を優先度関数（`targetRank` / `rangeRank` / `resolveTarget` / `resolveRange` / `resolveTargetValue`）で合成。ボタン式で常時上書きを回避。
- **使用回数消費**: 起動時 `planUsesConsumption`（消費ダイアログ＋原則ブロック）、実行時 `_consumeUses`（`updateEmbeddedDocuments` で遠隔消費）。消費確定と減算を分離しキャンセル時の誤消費を防止。

#### 課題・申し送り

- 自動入力のタイミング合成はベース技能の先頭 timing を採用する best-effort（スタイル技能 timing は配列のため）。目標値は数値最大の近似。いずれも手編集前提。
- 対決の統合ルール（コンボ時に複数技能の対決をどう束ねるか）・相手リアクション解決はフェーズ13。
- 対象/射程の AE による変更と `isFixed*` による抑止の実機能はフェーズ9/13。現状はフラグ保持のみ。
- 使用回数の回復・リセット（カット/シーン/アクト境界）はフェーズ15。

---

### [対決の解釈・使用回数の spent 化（メカニクス監査）](フェーズ8後追い)

**日付**: 2026-06-20
**関連ルール**: 対決の解釈・使用回数の消費（[Judgment_Rules.md](../01_Wiki/Game_Rules/Judgment_Rules.md)）

#### 監査対象のルール解釈（ユーザー確定）

- **対決「技能名」（無印）は限定でなく付与・例示**。書かれていない技能でのリアクションを排除しない。根拠は〈電脳〉ドッジ付与技能の存在（限定説だと機能しなくなる）という実プレイ整合。公式は「対決:〈回避〉」を「リアクションできる」程度の緩い意味で運用しているとの解釈。
- **「技能名※」のみ真の制約**（その技能を必ずリアクションに含める）。単体判定も「組み合わせ」に含む。
- 機械的に意味があるのは「対決可能／不可」と「※必須」のみ。無印技能名は例示扱い、強制しない。※の必須化強制はフェーズ13。
- 使用回数は D&D 方式の「消費済み（spent）」表現に変更（残り = max − spent）。消費の意味自体は不変。

#### 監査結果

✅ ユーザーが実プレイ整合の根拠とともに確定したルール解釈に準拠。当初 Code が傾いた「限定説」は前提（射撃をパリー不可）が誤りだったため撤回し、付与説で確定。

---

### [使用回数 spent 化・mixin 静的 migrateData 合成（設計レビュー）](フェーズ8後追い)

**日付**: 2026-06-20
**関連実装**: `scripts/data/abstract.mjs`、`scripts/data/item/helpers.mjs`、`scripts/data/item/style-skill.mjs`、`scripts/data/item/common/outfit-base.mjs`、`scripts/module/tnx-judgment-flow.mjs`、`scripts/item/tnx-style-skill-sheet.mjs`、`scripts/item/tnx-outfit-sheet.mjs`、`templates/item/style-skill-sheet.hbs`、`templates/item/outfit-sheet.hbs`、`templates/app/usage-sheet.hbs`
**ステータス**: 設計レビュー済

#### 設計

- **mixin の静的 migrateData 合成**: `SystemDataModel.mixin()` がプロトタイプメソッドのみコピーしていたため template の静的 `migrateData` が concrete モデルで死んでいた（潜在バグ）。mixin 内で各 template の静的 `migrateData` を順に適用してから親へ委譲するよう修正。`OutfitBaseTemplate` 等の既存マイグレーションも有効化される。
- **uses.value → uses.spent**: 残り回数管理から D&D 方式の「消費済み」管理へ。`migrateUsesValueToSpent()`（`helpers.mjs` 共用）で `spent = max − value` に移行。style-skill は自身の migrateData、outfit-base は template の migrateData に追加（上記 mixin 修正で全アウトフィットに適用）。
- 判定フロー: `planUsesConsumption`（残り = max − spent でブロック判定）・`_consumeUses`（spent +1）。シートのラベル「現在値」→「消費済み」、isLimit 解除リセットは spent=0。
- 対決表示: 無印技能名を「リアクション例」として表示し、網羅と誤読させない注記を追加。

#### 監査結果

✅ DataModel フィールド改名・mixin 挙動変更はユーザー承認済み（§4.3）。757 テスト green（mixin 合成・移行の単体テスト追加）。

#### 課題・申し送り

- mixin 修正で既存の dead だった template マイグレーション（outfit の appearancePenalty/preserveExp、usage の _id 等）が有効化された。いずれもガード付きで冪等のため安全だが、今後 template に migrateData を足す際は concrete で走る前提で書くこと。
- 使用回数の回復・リセット（境界）はフェーズ15。

---

### [能力値・制御値の実効値一本化・0clamp（メカニクス監査）](フェーズ9-1)

**日付**: 2026-06-20
**関連実装**: `scripts/data/helpers.mjs`（`computeAttributeFinal`）、`scripts/data/actor/cast.mjs`（`prepareDerivedData`）
**ステータス**: メカニクス監査済

#### ルール解釈

- 能力値・制御値の最終実効値 = `growth + Σ(スタイル基本値 × スタイルレベル) + mod + effectMod + outfitMod`。スタイル基本値は**3スロット全てを加算**する（"装備中のみ" のような equipped 概念はルールに存在しない）。
- 最終値は **0clamp**（全修正を合算した最終値に対して。順序非依存。能力値・制御値の両方）。負方向修正で0を下回る場合の「効果を受けられない」を 0clamp として実装（ユーザー確定、[Character_Creation.md](../01_Wiki/Game_Rules/Character_Creation.md)）。

#### 実装

- `computeAttributeFinal()` でスタイル寄与（×レベル）の合算＋各修正合算＋0clamp を一括算出。`prepareDerivedData` が `system.<key>.total` / `.totalControl` を書く。

#### 監査結果

✅ ルール正本（Character_Creation）に準拠。判定が実在しない `system.equipped` でスタイルを除外し達成値からスタイル基本値を欠落させていた（KI-021）点を一本化で是正し、シート表示と同値になった。

#### 課題・申し送り

- 増減ペアを持つ技能の矛盾（clamp でデメリットだけ消える等）は自動化しない（手動 AE 調整）。
- AE の effectMod 着地点への流入はフェーズ9-3。outfitMod の派生化（B-2）はフェーズ9-2。

---

### 能力値実効値の派生パイプライン確立（設計レビュー）（フェーズ9-1）

**日付**: 2026-06-20
**関連実装**: `scripts/data/actor/cast.mjs`、`scripts/data/helpers.mjs`、`scripts/actor/tnx-cast-sheet.mjs`、`scripts/module/tnx-judgment-flow.mjs`、`tests/data/helpers.test.mjs`
**ステータス**: 設計レビュー済

#### 設計

- **システム初の `prepareDerivedData` オーバーライド**を `CastDataModel` に新設（CLAUDE.md §3 が「現状なし」としていた箇所。本フェーズが整備の正規タイミング）。実効値を `system.<key>.total` / `.totalControl` に集約し**単一の真実**とした。
- 式は純粋関数 `computeAttributeFinal`（Item 非依存・素オブジェクト配列を受ける）に切り出しテスト可能化。
- 重複3箇所（シート `_getAbilitiesData`・`_computeMundaneTotalValue`・判定 `_buildAbilitiesCtx`）を集約値参照に置換。EXP コスト（`updateCastExp`）は別式・§4.2 のため対象外。
- `outfitMod` は当面永続フィールドを参照（B-2 化は9-2）。`effectMod` は AE 着地点として合算済み（流入は9-3）。

#### 監査結果

✅ `defineSchema` 不変（`total`/`totalControl` は派生プロパティで §4.3 非該当）。766 テスト green（`computeAttributeFinal` 単体テスト追加）、lint 新規警告ゼロ。

#### 課題・申し送り

- guest / troop も同式の実効値を要するが本フェーズ範囲外。フェーズ11 で同 helper を各 prepareDerivedData から呼ぶ。
- 実機回帰（シート表示不変・判定達成値にスタイル基本値が載る）はユーザー確認待ち。
