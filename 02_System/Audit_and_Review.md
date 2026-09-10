# 監査と設計レビュー

2種類の品質管理を**混同しない**。

| 種類 | 問い | 記録先 |
|---|---|---|
| メカニクス監査 | ルール解釈は正しいか？ | 本ファイル（`Audit_and_Review.md`） |
| 設計レビュー | コードとして良い設計か？ | `Design_Review_Entries.md`（本ファイルにも一部） |

同じ実装に対して両方の指摘が独立に成り立つ。

## メカニクス監査

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

## 設計レビュー

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
| フェーズ3〜10 | 各フェーズの実装エントリ（本ファイル後半とDesign_Review_Entries に対で追記。最新はフェーズ10-1〜10-6） |

詳細: [Design_Review_Entries.md](Design_Review_Entries.md)

## 実施タイミング

| タイミング | 作業 |
|---|---|
| 新規実装時 | メカニクス監査 + 設計レビュー 各1エントリ |
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
**関連実装**: `scripts/module/tnx-check-engine.mjs`
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
- `getCardCheckValue({ numericValue, isJoker, fixedAt21, declaredValue, isFromDeck })`: Symbol 値（FUMBLE_VALUE / FIXED_21）を使い、型安全に「ファンブル」「21固定」を表現。
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
**関連実装**: `scripts/module/tnx-check-flow.mjs`, `scripts/module/tnx-check-dialog.mjs`, `scripts/module/tnx-hud.mjs`
**ステータス**: 監査済(問題なし)

#### ルール解釈

- 判定フロー（技能判定）: ① カードを出す → ② 報酬点の消費量を決定 → ③ 達成値計算 → ④ チャット投稿。
- 報酬点選択: カード選択後に別プロンプトで行う（カード選択前ではない）。ルール上「カードを出した時点で判定成立 → その後に報酬点を使うかどうか決定」の順序に準拠。
- スート不一致: 起動（カードのプレイ操作）は拒否しない。達成値 0 の通常失敗として投稿する。ファンブルとは区別しない。
- 切り札: 切り札を消費することを宣言 → 手札から 1 枚を Joker として使用 → 切り札はシーンカードへ。
- ×ボタンでダイアログを閉じると判定キャンセル（ユーザー要求、2026-06-15 確定）。

#### 実装

- `TnxCheckFlow` (singleton 静的クラス): `_context` / `_trumpMode` で状態管理。ダイアログクラスは `tnx.mjs` で注入（循環 import 回避）。
- `_execute`: ① ダイアログ close・state clear → ② カードプレイ（`TnxActionHandler.playCard`）→ ③ HUD 再描画 → ④ `_promptBountyUsage`（スート不一致・制御判定はスキップ）→ ⑤ 達成値計算 → ⑥ チャット投稿。
- `TnxCheckDialog._onClose`: `TnxCheckFlow.cancel()` を呼び、×ボタンでも判定キャンセルを実現。
- HUD カード達成値プレビュー: `getCardCheckValue + getAbilityBySuit` でホバー時に数値バッジを表示。スート不一致は赤バッジで「0」表示。

#### 監査結果

✅ 報酬点フローの順序（カード後）・スート不一致の扱い・切り札消費フロー・×ボタンキャンセル、いずれもユーザー指示に準拠。

#### 課題・申し送り

- 制御判定の `achievement` フィールドは `null`（達成値概念がない）。チャットテンプレートで制御判定専用の表示分岐を使用。将来的に型を明確化する場合はオブジェクト構造の見直しが必要。

---

### [報酬点フロー修正: usesBounty チェック・spinner 修正](フェーズ8-3 バグ修正)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-check-flow.mjs`, `scripts/actor/tnx-cast-sheet.mjs`
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
**関連実装**: `scripts/module/tnx-rl-request-app.mjs`, `scripts/tnx.mjs`, `templates/app/rl-request-app.hbs`, `templates/chat/check-request.hbs`
**ステータス**: 監査済

#### ルール解釈

- RL（GM）が判定種別・技能・目標値・対象 PL を指定して判定を要求する。
- 技能判定: identificationKey で技能を同定。PL は自分のキャラクター上で同一 identificationKey の技能を検索して判定。未所持の場合は判定不可。
- 能力値判定: 指定能力値のスートで判定。報酬点使用可。
- 制御判定: 指定能力値のスートで制御判定。報酬点使用不可。
- 目標値の非公開: PL には「（非公開）」と表示、GM には実値を表示。
- RL モードは全公開のみ（TNX はカードベースのため非公開判定が原理的に成立しない）。

#### 実装

- `TnxRlRequestApp` (ApplicationV2 + HandlebarsApplicationMixin): フォーム送信で `ChatMessage.create` を呼び、`flags.checkRequest` にデータを保持。
- `renderChatMessage` フック: 対象 PL（または GM）にのみ「判定する」ボタンを表示。結果 (`results[actorId]`) が存在する場合はボタンを結果表示に切り替え。
- `TnxRlRequestApp.onDoCheck()`: identificationKey でアクター上の技能を解決し、`getComboSuits()` で validSuits を取得。技能未所持の場合は warn して早期 return。
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
**関連実装**: `scripts/module/tnx-check-flow.mjs`
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
- `executeFromHand` / `executeFromDeck` / `_executeAsTrumpJoker` で `getCardCheckValue({ numericValue })` を呼び出す（`isJoker: true` の特例ルートを廃止）。
- `_handleAceChoice` に `extraParams = {}` を追加し、`fromDeck` / `trumpUsed` を引き継ぎ可能に。

#### 監査結果

✅ 全4スート選択可・宣言ランクが通常カードと同じルールで計算される点、ルールに準拠。

#### 課題・申し送り

なし。

---

### [山札判定スート読み取りバグ修正](フェーズ8)

**日付**: 2026-06-15
**関連実装**: `scripts/module/tnx-check-flow.mjs`
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
**関連実装**: `scripts/module/tnx-check-flow.mjs`
**ステータス**: 監査済

#### ルール解釈

- ファンブル確定時は報酬点を消費できない（報酬点ダイアログを表示しない）。
- 報酬点ダイアログには報酬点加算前の現在の達成値を表示し、プレイヤーが加算効果を判断できるようにする。

#### 実装

- `_execute` の条件に `cardCheckValue !== "FUMBLE"` を追加。FUMBLE 確定時は `_promptBountyUsage` を呼ばない。
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
**関連実装**: `scripts/module/tnx-action-handler.mjs`（`autoReplenishHand`）、`scripts/module/tnx-check-flow.mjs`（`_execute`）
**ステータス**: 監査済

#### ルール解釈

- 手札からカードを出した直後、**手札上限まで自動補充する**。
- 山札判定では手札を消費しないため自動補充は行われない。
- 山札が不足している場合は引ける枚数だけ引く（エラーにしない）。

#### 実装

- `TnxActionHandler.autoReplenishHand()` を新設: 上限 - 現在手札枚数を計算し、その枚数を山札からドロー。
- `TnxCheckFlow._execute()` で `playCard()` の直後に `autoReplenishHand()` を呼び出す（`!fromDeck` のブランチ内）。

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

✅ 全 Cards API 呼び出し箇所（`tnx-action-handler.mjs` / `tnx-hud.mjs` / `tnx-scenario-sheet.mjs` / `tnx-card-setup-app.mjs` / `tnx-check-flow.mjs`）に `chatNotification: false` が付与されていることを確認。

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
**関連ルール**: 射程の優先度・使用回数の消費（[Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)）

#### 監査対象のルール解釈（ユーザー確定）

- **射程の優先度**: 至近※ ＞ 武器 ＞ 超遠 ＞ 遠 ＞ 中 ＞ 近 ＞ 至近。変更不可（※）が複数なら最短を優先。ルルブに優先度記述がなくユーザーが裁定・追記要望したもの。
- **使用回数の消費**: 消費は判定（用途）起動に紐づく機能であり、回復/リセット（フェーズ15）から分離。参加技能（ベース＋コンボ）の使用回数制限を各 −1、組み合わせ技能は遠隔消費。Roadmap フェーズ15「行動回数制限は自動化しない」（＝ターン行動回数）とは別物。
- **対決**: 相手側リアクションの自動解決はしない（フェーズ13）。対決不可フラグ（`isUnopposable`）のみ今フェーズで保持、対決可能技能は参加技能から読み取り専用表示。

#### 監査結果

✅ いずれもユーザーが明示的に確定・裁定したルール解釈に準拠。射程優先度は一次ルールに記述がないためユーザー裁定を正本として記録。

---

### [用途シート発動タブ再構成・自動入力・使用回数消費（設計レビュー）](フェーズ8後追い)

**日付**: 2026-06-20
**関連実装**: `scripts/data/item/common/usage.mjs`、`scripts/module/tnx-usage-sheet.mjs`、`templates/app/usage-sheet.hbs`、`templates/app/usage-sheet-combo.hbs`（新設）、`scripts/tnx.mjs`（partial 登録）、`scripts/module/tnx-check-flow.mjs`、`scripts/actor/tnx-cast-sheet.mjs`、`css/tnx2.css`
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
**関連ルール**: 対決の解釈・使用回数の消費（[Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)）

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
**関連実装**: `scripts/data/abstract.mjs`、`scripts/data/item/helpers.mjs`、`scripts/data/item/style-skill.mjs`、`scripts/data/item/common/outfit-base.mjs`、`scripts/module/tnx-check-flow.mjs`、`scripts/item/tnx-style-skill-sheet.mjs`、`scripts/item/tnx-outfit-sheet.mjs`、`templates/item/style-skill-sheet.hbs`、`templates/item/outfit-sheet.hbs`、`templates/app/usage-sheet.hbs`
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
**関連実装**: `scripts/data/actor/cast.mjs`、`scripts/data/helpers.mjs`、`scripts/actor/tnx-cast-sheet.mjs`、`scripts/module/tnx-check-flow.mjs`、`tests/data/helpers.test.mjs`
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

---

### outfitMod / appearanceModifier の B-2 派生化（設計レビュー）（フェーズ9-2）

**日付**: 2026-06-20
**関連実装**: `scripts/data/actor/cast.mjs`（`_prepareOutfitAggregates`）、`scripts/data/helpers.mjs`（`computeOutfitAggregates` / `OUTFIT_ITEM_TYPES`）、`scripts/tnx.mjs`（フック撤去）、`tests/data/helpers.test.mjs`
**ステータス**: 設計レビュー済

#### 設計

- アウトフィット集計（outfitMod の制御値修正・CS修正、appearanceModifier の危険値合計）を、フック駆動の DB 書き戻し（B-1）から `prepareDerivedData` での都度算出（B-2）へ移行。
- 撤去: `calcOutfitMods` / `updateCastOutfitMods` / `calcAppearanceModifier` / `updateCastAppearanceModifier` / `recalcOutfitAggregates`（create/delete/updateItem の3フック）/ 起動時スキャンの集計呼び出し / isGhost 変更ハンドラ。`{syncing:true}` 書き込みも該当分が消滅。
- 集計ロジックは純粋関数 `computeOutfitAggregates(items, isGhost)` に切り出し（isPrepared/isCarrying/isGhost 条件を網羅テスト、8件）。`OUTFIT_ITEM_TYPES` も helpers に集約（tnx.mjs から移動、重複解消）。
- 能力値 total が outfitMod を読むため、`prepareDerivedData` 内で集計を**先に**実行。

#### 監査結果

✅ `defineSchema` 不変（outfitMod / appearanceModifier は schema フィールドのまま派生で上書き＝§4.3 非該当）。774 テスト green、lint 新規警告ゼロ。派生値を DB に焼かない本来形になり、`{syncing}` の小細工と「GM クライアントのみ集計」制約が解消。各クライアントがローカル算出する。

#### 課題・申し送り

- 旧 `system.outfitMod` / `system.appearanceModifier` の DB 保存値は以後 prepareDerivedData が常に上書きするため死蔵となる（読まれない）。実害なし。マイグレーションで除去するかは将来判断。
- 9-3 で危険値集計はアイテムの AE 込み実効値（`appearancePenalty.total`）を読むよう切替予定（現状は raw value）。
- guest / troop の集計は本フェーズ範囲外（11）。EXP 同期機構（updateCastExp / User flag）は B-2 対象外で不変。

---

### アイテム AE 着地点・実効値派生（設計レビュー）（フェーズ9-3a）

**日付**: 2026-06-20
**関連実装**: `scripts/data/item/helpers.mjs`（effectMod・`computeItemEffectiveValues`）、`scripts/data/item/common/outfit-base.mjs` / `skill-base.mjs`（prepareDerivedData）、`weapon.mjs` / `residence.mjs`、各テスト
**ステータス**: 設計レビュー済（表示切替・モードB・KI-020 は継続）

#### 設計

- **着地点 substrate（漏れ防止＝ヘルパー単位）**: `modeValueField` に effectMod（隠匿値/電制/危険値/購入値/常備化EXP/受け値/制御修正/SF/乗員/サイクル/CS修正/スロット数を一括網羅）、`defenceField` に S/P/I_effectMod、attack.effectMod（既済）、weapon.FAValueEffectMod、skill levelEffectMod、residence 3stat。フィールド手列挙は漏れるため構造単位で付与（ユーザー指摘 2026-06-20）。
- **除外**: 住宅エリア供給値・スタイル能力値（それ自体がエフェクト的でターゲットでない、ユーザー確定）。
- **実効値派生（representation はフェーズ9）**: `computeItemEffectiveValues` が **base を書き換えず** `.total` 系を生成。mode=value 時のみ effectMod 加算。`OutfitBaseTemplate.prepareDerivedData` 1ヶ所で全10アウトフィット型に一括適用（concrete 未定義のため継承）、技能は `SkillBaseTemplate`。
- **編集 UI は base / 表示・消費は total**（入力欄に実効値が漏れない＝AE 二重加算を防ぐ、ユーザー指摘）。

#### 監査結果

✅ §4.3 のスキーマ変更（effectMod 群）はユーザー承認済・ヘルパー単位で網羅。792 テスト green（substrate/computeItemEffectiveValues 単体テスト）、lint 新規警告ゼロ。

#### 課題・申し送り

- clamp はしない（controlMod / combatSpeedMod は負の修正がありうる）。0clamp は消費側（ダメージ算出フェーズ12）の責務。
- **mixin の prepareDerivedData は順序依存**（後勝ち・チェーンしない）。現状 prepareDerivedData 定義は cast / outfit-base / skill-base の3つのみで競合なし。今後 UsageTemplate 等に追加する際は super 委譲か concrete 集約を検討。
- 表示の total 参照切替（アウトフィット概要・キャストシート装備表示）は次の増分。モードB（アクター→アイテム横断）・KI-020 は継続。

---

### AE 適用モデル v2「total 直接適用」（メカニクス監査＋設計レビュー）（フェーズ9-3 v2）

**日付**: 2026-06-21
**関連実装**: `scripts/data/item/helpers.mjs`（`parseEffectTargetKey` / `computeCheckBonus` / `checkChangeMatches` / `computeItemEffectiveValues`）、`scripts/data/actor/cast.mjs`（`_applyEffectBuffs` / `_resolveBuffApplications`）、`scripts/data/helpers.mjs`（`computeAttributeFinal`）、`scripts/module/tnx-check-engine.mjs`・`tnx-check-flow.mjs`（checkBonus）、`scripts/tnx.mjs`（重複可 UI フック）、各 DataModel・テスト
**ステータス**: 実装完了・実機確認待ち（9-3a の effectMod/モードB 設計を**全面置換**）

#### メカニクス監査（ルール解釈）

- **同一効果の重複適用不可**（ユーザー確定 2026-06-21）: 明記なき限り同一効果は重複しない＝一般原則。値バフ・判定バフ双方に適用。ごく稀な「重複可」は明示フラグで表現。同一性は**安定 ID**（`flags.effectId` ?? document `_id`）で判定（名前変更耐性。辞典 effectId でコピー同士も同一視可能）。同名・異値は最大採用。
- **判定（check）への加算がトーキョーN◎VA 最重要バフ**（単一技能指定）。コンボで複数技能に該当しても同一エフェクトは1回（プレフィックスが2技能に当たっても +1）。別エフェクトはスタック。
- 制御判定バフは制御値を実効的に押し上げ（成功条件 `cardValue <= totalControl + checkBonus`）。
- アウトフィットは識別キーで個体指定しない（親/カテゴリ/条件で相対指定）。技能は識別キーで一意指定。区分バフは小分類キー、大分類も `category.<大キー>` で可（大小キー非衝突）。

#### 設計レビュー（技術的妥当性）

- **effectMod 中間枠を全廃し total へ直接 `effect.apply`**: ネイティブのモード（ADD/OVERRIDE/MULTIPLY/UPGRADE/DOWNGRADE）と priority をそのまま再利用。effectMod では OVERRIDE/MULTIPLY を表現できなかった限界を解消。
- **標準 UI のみ（キー＋モード＋値）**: 非存在フィールド・マクロに依存しない。条件はキー文字列内 `[...]` に畳み込む。重複可のみ詳細タブにチェックボックスを自前注入（`renderActiveEffectConfig`＋`setFlag`、フォーム送信非依存）。
- **アクター単一適用パス**（`_applyEffectBuffs`）: Foundry の Item が自己効果を適用しない・change.key が他アイテムへ解決しない制約を、`legacyTransferral=false`＋アクター集約で回避。transfer OFF でも横断バフが反映。
- **判定バフは判定時評価**（恒常 total に焼かない）: 文脈依存（コンボ）かつ非スタックのため。純粋関数 `computeCheckBonus` に切り出し単体テスト（重複不可・最大採用・スタック・stackable・非アクティブ）。

#### 監査結果

✅ §4.3 のスキーマ変更（effectMod 群の削除）はユーザー承認済。814 テスト green、lint 0 error（既存 15 warning）。9-3a の substrate（effectMod 着地・モードB aeTarget・死蔵ヘルパー）は本 v2 で撤去。

#### 課題・申し送り

- 値バフ重複排除は `(対象, total パス, モード, identity)` 単位の最大採用。OVERRIDE/MULTIPLY を**同名併用**する特殊ケースは「同名で最も有利な結果を残す」方針で、複雑な併用は明示 stackable で回避する整理。
- 値バフ重複排除・適用パスは Foundry ドキュメント依存のため単体テスト外（実機確認）。判定バフ側は純粋関数で網羅済み。
- 手札上限バフは現状 native `system.handMaxSizeMod`（KI-020 特例）。横断パスへ寄せて transfer 不要に統一するかは申し送り。
- スタイル/ワークス述語（`byStyle:` 等）は構造化（辞典ドロップダウン・専用欄）が前提で後続。当面は識別キー/プレフィックス/カテゴリ。

---

### コンディション(BS・戦闘不能)の AE 表現（メカニクス監査＋設計レビュー）（フェーズ9-4）

**日付**: 2026-06-22
**関連実装**: `scripts/module/conditions.mjs`（CONDITION_KINDS / readCondition / gatherConditionCheckSources / gatherConditionControlPenalty / computeJammingPenalty / getCheckBlock）、`scripts/module/tnx-check-flow.mjs`（_execute 重圧遮断・_computeCheckBonus 合流・_computeJammingSource・_gatherConditions）、`scripts/data/actor/cast.mjs`（_applyConditionControlPenalty）、`scripts/tnx.mjs`（statusEffects に conditionKind・詳細タブ effect-value UI）、`tests/module/conditions.test.mjs`
**ステータス**: 9-4 実装（in-phase 分）完了・実機確認待ち。発火依存(萎縮憎悪/捕縛=12、邪毒/恐慌/メインプロセス不可=13、回復=15)は申し送り

#### メカニクス監査（ルール解釈）

- **condition は自己完結する実体**: 同一性(status id=`conditionKind`/回復/除去可否) ＋ 可変効果値(condition 専用フラグ)。効果値は汎用 changes でなく専用フィールドで設定、適用はコードが kind で行う。D&D と違い TNX の BS は効果値が可変なため(ユーザー確定 2026-06-22)。
- **5論点裁定**: ①「あらゆる判定」は上方判定のみ(制御は controlCheck/control 指定時のみ)②重圧=該当能力値を使う上方判定を不可(制御は可)③萎縮/憎悪=対象UUID＋`game.user.targets`(発火は12)④重複明記なしは非重複・強度は最大n⑤ウェットは「ウェット」アウトフィットを識別キーで検出(専用フィールド不可)。
- **重複規則**: condition の非 stackable は**同 kind で重複排除**(同名 BS は複数発生しない)。stackable(衰弱/酩酊小大別/萎縮)は重ねる。憎悪は対象ごとに存在するがペナルティは非重複(同 kind で1回)。
- **電子妨害**: 準備中アウトフィットのみ。category(武器/アーマーギア/サイバーウェア/トロン/サイコアプリ)＋電制≤n の個数(上限10)。全身義体(fullCyborg)・ヴィークル準備/該当タップでゴースト(isGhost)→10、ウェットで該当1個以上→1。

#### 設計レビュー（技術的妥当性）

- **2注入点に condition 用コードを追加**(Active_Effects v2 と整合): 恒常＝派生パス(`_applyConditionControlPenalty` で全制御値減)、判定時＝`_computeCheckBonus` に酩酊/萎縮憎悪/電子妨害を合流(チャート内訳・HUDプレビューにも反映)。
- **重圧の遮断は `_execute` 冒頭・playCard 前**。判定に転送された(切り札リマップ後の)suit から能力値を拾う。**カード実体・クリックは読まない**(切り札はあらゆるカードを別スート/数字で使えるため、ユーザー指摘)。
- **純粋関数に分離**(conditions.mjs は Foundry 非依存): 重複排除・電子妨害計算・遮断判定を単体テスト(20件)。actor/item/targets の解決は呼び出し側。
- **付与 UI は既存 statusEffects**、効果値は詳細タブの kind 別フィールド(`renderActiveEffectConfig`、重複可と同じ手法で setFlag)。

#### 監査結果

✅ 836 テスト green、lint 0 error。effectMod 廃止後の v2 と一貫(condition は別系統＝kind 駆動・専用フィールド)。萎縮/憎悪・捕縛は器とロジック・UIを用意し、発火フック(isAttack/targets/武器)はフェーズ12 が供給する dormant 状態。

#### 課題・申し送り

- ウェット識別キー実値は辞典(17)で確定。準備制限・技能制限・対象外の enforcement は別スコープ(準備/技能系・12/13)。
- 持続単位フラグ(durationUnit)・小さなシート積み残し(residence表示・isGhost CS・手札上限統一)は 9-5。発火(メインプロセス不可・継続ダメージ・回復)は 13/15。詳細は Phase_9 積み残し管理表。

---

### 部位（part）管理システム（フェーズ10-1）

**日付**: 2026-06-26（実装完了 2026-06-27）
**関連実装**: `scripts/data/item/common/outfit-base.mjs`（part 行スキーマ・PART_KINDS/PART_RELATIONS・isOption 派生）、`scripts/data/item/part-helpers.mjs`（表示・占有の純粋関数）、`scripts/data/item/outfit-categories.mjs`（SLOT_KIND_MINOR_HOSTS）、`scripts/data/actor/cast.mjs`（partSlots・エイリアス）、`scripts/module/part-slot-preset-app.mjs`（プリセット設定・編集App）、`scripts/item/tnx-outfit-sheet.mjs`（部位エディタ・装備先UI）、`scripts/actor/tnx-cast-sheet.mjs`（占有パネル）、`scripts/tnx.mjs`（設定登録・preCreateActor 流し込み・既定プリセット自動設定）。正本 [Outfits.md](../01_Wiki/Game_Rules/Outfits.md)「部位管理(フェーズ10)」
**ステータス**: ✅ 実装完了（DataModel/プリセット/エイリアス/部位エディタ UI/装備先UI/占有計算①身体部位②③ホストスロット/キャスト占有パネル/部位表示ルール）。実機確認済（2026-06-27）

#### ルール解釈（ユーザー裁定 2026-06-25〜26）

- **part 5択 kind**: none/bodyPart/option/reference/other。種別は文字列内容から推測せずユーザーが明示設定（§4.2）。
- **任意（isOptional）**: 占有数に非カウント（重複可・無制限）。括弧内（指 等）は部位名として保持。素の任意は値空。
- **複数行の結合 relation(and/or)**: and=全部占有「A+B」、or=択一（装備先はトグルで選択・説明表示に不影響）。表示コネクタは種別で分岐＝身体部位「A、もしくはB」／任意「任意(A、B)」／スロット持ちホスト「A／B」、その他含む→末尾「など」（ルルブ表記に一致）。
- **非消費3型**: slots=0（武器0 等）／小分類=住宅アクセサリ／任意。占有母数から外す。
- **プリセット**: 体部位のみ。片手持ち/片腕/片脚/指=各2、他1。完全一致照合（耳≠両耳・腕部≠腕）。下垂体は脳下垂体を正、任意(下垂体)は非カウントで衝突せず。
- **エイリアス（occupiesOther）**: 両手持ち=片手持ち×2 等を「指定部位を複数占有」で表現。公式に無い排他部位・ルール追加にユーザー設定で対応。
- **オプションのホスト照合**: 大分類サイバーウェア指定は isCyber=true も拾う（※付き＝サイバーウェア扱いの裁定）。小分類は単一選択＋除外トグル（搭載兵器以外）。その他特徴は isLaser／**isCyber（武器(サイバーウェア)）**（2026-06-27 追加）。変異器官は 10-2 で新造。
- **解説参照**: 表示は常に「解説参照」（説明タブはルルブ踏襲）。実部位を入れ子で持ち占有計算に含む。入れ子の解説参照は不可。
- **部位表示は正本リスト（ユーザー提供）に逐条準拠**（2026-06-27 確定）。**例外は特殊弾のみ**＝「特殊弾」「特殊弾(スリング)＝種別(ホスト名)」（武器に畳まない）。タップの soft/hard は「タップ」。意識3種は固有名のまま（IANUS に畳まない）。意識系は「IANUS で解放される身体部位」とも見なせる（精神という部位の存在）。
- **isOption は部位 kind=option から派生**（旧チェック撤去を吸収。`prepareDerivedData`）。「オプション選択時」に装備先 UI が出る挙動と一致。
- **ホストスロット占有（②③）**: オプションは装備先ホスト（`parentItemId`）の `parentSlotKind` スロットを消費。占有チップのスロット名は意識3種・soft/hard は種別名のみ、normal はカテゴリ名。**ホスト名は同名スロット複数時のみ併記**（IANUS は単一準備で `IANUS(IANUS)` を出さない）。**スロット0／無スロットは装備不可ゆえ拾わない**（容量>0 のみ）。
- **準備＝装備（一覧表示。2026-06-27）**: 準備済み（`isPrepared`）オプションのみホスト配下に入れ子表示＝装着済みを表す。**未準備（携帯のみ）は装備対象に装着されていないので、配下から外して独立行で通常表示**する。

#### 実装

- DataModel は加算的（value/slots 維持、kind 既定 other で旧データ移行）。
- 表示・占有は **Foundry 非依存の純粋関数**（`part-helpers.mjs`・40 テスト）: `formatPartDesignation`／`formatOptionLabel`／`joinPartDesignations`（部位名表記）、`computePartOccupancy`（①身体部位 vs partSlots）、`computeHostOccupancy`（②③ホストスロット vs オプション消費）。kind→ラベル解決は呼び出し側（`SLOT_KINDS`）。
- 装備先 UI は撤去されていた `parentItemId`/`parentSlotKind`（孤児だった選択肢構築・変更ハンドラ）を部位カード最下段に再建。占有パネルは身体部位チップ＋ホストチップを 1 列統合（超過は赤）。
- アウトフィット一覧の入れ子は**記号を廃し回路トレース調**（縦トレース＋分岐ひし形＋横ブランチ＋名前横の丸ポチ・発光・`outfit-row--option`/`--last-option` で枝端制御）。スロット保有ホストに**スロット使用ゲージ**を常時表示（種別ごと・`--c` で色分け＝normal シアン/software 緑/hardware マゼンタ/表層意識 藤色/深層意識 青紫/無意識 藍・種別名ツールチップ・未準備でも 0/容量）。オプションに占有スロット種別タグ。表セル値は `squeeze` で縮小。

#### 監査結果

✅ 実装完了・実機確認済（2026-06-27）。設計はユーザー裁定でロック、表示は正本リストに逐条準拠。全 896 テスト green・lint 0。残課題は下記「課題・申し送り」。

#### 課題・申し送り

- 派生データ（武器＋タップ等のマルチ種別）・変異器官フラグは 10-2（取得時自動取得）へ合流。→ **10-2 で実装済**
- 任意の部位名/消費数の記録の細部・両手持ち等の体部位表記マッピングは部位表示 UI 完成時に確定。
- 戦闘タブ表示内容は 10-6（ユーザー言語化待ち）。→ **10-6 で実装済**

---

### 技能の組み合わせ解決・技能/対決欄の辞典カスケード・スタイル技能の自動取得（フェーズ10-2＋判定）

**日付**: 2026-06-29
**関連実装**: `scripts/module/skill-dictionary.mjs`（辞典ローダ・カスケード・カテゴリ全体トークン・comboSkill 名逆引き）、`scripts/module/skill-chain-resolution.mjs`（技能チェーン解決・必須/候補/ロック判定の純関数）、`scripts/module/style-skill-acquisition.mjs`（インポート時自動取得）、`scripts/data/item/style-skill.mjs`（comboSkill/confrontation のカスケード化・usesBounty/noCombo・自動取得・代用フィールド）、`scripts/item/tnx-style-skill-sheet.mjs`＋`templates/item/style-skill-sheet.hbs`（技能/対決の別セクション・カスケードUI）、`scripts/module/tnx-usage-sheet.mjs`＋`templates/app/usage-sheet-combo.hbs`（用途設定時の解決・個数制限・トリムダイアログ）、`scripts/actor/tnx-cast-sheet.mjs`＋`scripts/item/item.mjs`（ロール挙動・用途不備検知）。正本 [Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md)・[Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)
**ステータス**: ✅ 実装完了・実機レビュー済（2026-07-02・問題なし。具体データ依存の項目は未検証のまま持ち越し）

#### ルール解釈（ユーザー裁定 2026-06-28〜29）

- **「技能」欄(comboSkill)＝組み合わせ相手の指定**。技能名は**識別キーで格納**（辞典カスケード 辞典→種別/スタイル/組織→小分類→技能名）、表示は辞典から**都度逆引き**（保持名キャッシュなし）。
- **カテゴリ全体指定**: 固有名詞〈社会〉等は小分類リスト先頭、スタイル例外〈元力〉(`element_`)/〈血脈〉(`bloodline_`)はそのスタイルのリスト先頭（リスト全体先頭でない）。単一技能名でない＝自動固定しないが候補絞り込み対象。
- **対決(confrontation)も技能名/技能名※で同じカスケード**。技能名※は表示で「〈名〉※」。
- **判定は用途で解決済みを使う**（判定起動時に自動解決しない）。解決は**用途設定時**に baseSkillRef/skillRefs へ確定する。
- **技能チェーン解決**: 「技能」が単一技能名 X を指す間 X を辿る。「&」結合（両方必須）も単一名と同様に自動固定対象。「、」結合（いずれか1つ）は候補制限で扱う。
- **アクション技能は必ずベース**（チェーン＝起点＋足した組み合わせ技能の連鎖、その全体にアクションがあればそれがベース・他は全てコンボ）。**アクション技能同士は組み合わせ不可**＝参加にアクションが2つ以上になる追加はブロック。**アクション技能はベース欄以外に絶対入らない**（コンボから一律除外）。
- **代用**: 各段で本体優先・本体が無ければ代用で解決（代替先のプロパティは引かない＝自前のスート・報酬点フラグ）。代用がアクション技能の代替ならベースになる。ベース固定はハードロックでなく「指定技能＋その代用」候補からの選択（候補が増えれば選べる・本体優先で既定）。
- **組み合わせ個数上限＝ベース技能のレベル+1（ベース込み）**＝コンボ欄の数 ≤ level。追加で連れ込む分・アクション連れ込みでのベース入れ替わりを反映して事前ブロック。ベース変更で超過したらトリムで調整（必須は外せない・取り消しで元のベースへ）。
- **ロール時のアイテム挙動**: 用途(check)が無ければ D&D 方式で説明をチャット表示（アイテムの基本機能）。用途はあるが不備（ベース技能解決不可・共通スート無し）なら notification で不備を提示（未設定と区別）。
- **10-2 自動取得**: スタイル技能の習得時に対象（武器＝アイテム生成／トループ＝ID 保持）を自動取得。武器使用義務フラグ。由来武器は識別キープレフィックス照合で同種重複取得を防止（専用フラグは持たない）。**取得武器が複数設定されているときはインポート時にプルダウンで1つだけ選択**（複数取得は不可・候補1つなら自動取得）。**トループ取得対象は1つのみ設定可能**（武器は複数設定可・ドロップで置換）。（2026-06-29 裁定）

#### 監査結果

✅ ルール解釈はユーザー裁定に逐条準拠。解決・ロック判定の純関数はテストで固定（`tests/module/skill-chain-resolution.test.mjs`・`skill-dictionary.test.mjs`）。

#### 課題・申し送り

- 用途エディタの enforcement はレンダリング後に冪等で走り収束する設計。実機での収束（無限レンダリング無し）は要確認。→ **実機レビューで問題なし（2026-07-02）**
- トリムダイアログのインタラクティブなロック再計算（DialogV2 actions）は engine 未検証。→ 実機レビュー対象に含む（具体データ依存の細部は持ち越し）
- 10-3（経験点消費なし取得・取得数表示・レベル自動参照）は未着手。→ **10-3 で実装済**

### 派生データ・変異器官・式神装備フラグ／専用の辞典参照化／部位プリセット並び見直し（フェーズ10-2 後追い）

**日付**: 2026-06-29
**関連実装**: `scripts/data/item/common/outfit-base.mjs`（`isMutantOrgan`/`isShiki`/`shikiType`/`isDerivedData`/`hasDerivedData`/`derivedDataRefs`＋`SHIKI_TYPES`）、`scripts/data/item/part-helpers.mjs`（`PART_HOST_FEATURE_LABELS` に `isMutantOrgan`）、`scripts/item/tnx-outfit-sheet.mjs`＋`templates/item/outfit-sheet.hbs`（特性セクション・式神タイプ・派生データ参照UI・説明タブのタイプ表示）、`scripts/module/style-skill-acquisition.mjs`（`autoImportDerivedData`）、`scripts/tnx.mjs`（`createItem` フック）、`scripts/actor/tnx-cast-sheet.mjs`（経験点集計の `isDerivedData` 除外）、`scripts/module/part-slot-preset-app.mjs`（`DEFAULT_PART_SLOT_PRESET` 並び）。正本 [Outfits.md](../01_Wiki/Game_Rules/Outfits.md)
**ステータス**: ✅ 実装完了・実機レビュー済（2026-07-02・問題なし。派生データ等の具体データ依存項目は未検証のまま持ち越し）

#### ルール解釈（ユーザー裁定 2026-06-29）

- **派生データ（`isDerivedData`）**＝別アイテムから派生した従属データ。**常備化経験点を消費しない**（派生元が経験点を負担＝二重計上回避）。
- **派生元（`hasDerivedData`＋`derivedDataRefs`）**＝派生データを持つアウトフィット。取得（インポート）時に参照先を**全て自動生成**し `isDerivedData=true` を付す。**選択ダイアログは出さない**（派生データは親に内在＝ユーザーの選択対象でない）。例: Tウェポン（派生元）→ タップ＋武器（派生データ）。
- **変異器官（`isMutantOrgan`）**＝ヒルコの自製生体装備等。部位オプションの「その他特徴」ホスト条件で対象を絞る。
- **式神装備（`isShiki`＋`shikiType`）**＝しきそうび。タイプは攻撃/防御/操縦/独立の4種。説明タブではタイプを「部位」の1つ前に表示する。
- **専用（`exclusive`）**＝スタイル辞典・オーガニゼーション辞典からのプルダウン（2オプショングループ・複数指定可・`{type,key}[]`・名前は都度解決）。**自動化なし**（指定のみ）。模造技能・別組織のアウトフィット取得効果との兼ね合いで enforcement を持たせない（スタイル技能・ワークス専用技能の指定も同様）。旧自由記述文字列は移行で空配列にクリア。
- **部位プリセットの並び**＝頭→足→全身/組織→内的（心・電脳・霊）→スコープ。装着物は装着先の身体部位群末尾／装着先が不定の装着物（衣類・携帯・装飾・外付けの護符/結界/操縦）は全身近く／各部・独立は最も非局所として末尾。魂を追加。

#### 監査結果

✅ ルール解釈はユーザー裁定に逐条準拠。経験点二重計上回避（派生データ除外）・自動生成の無選択方針・式神タイプの表示位置を仕様どおり実装。

#### 課題・申し送り

- 派生データの UI（特性セクションのチェック・派生データのドロップ/参照ボタン/右クリック・説明タブのタイプ・自動派生インポート・「その他特徴」変異器官）は engine 未検証。
- 部位プリセット並びは**ワールド初回ロードのみ自動設定**。既存ワールドはプリセット編集 UI で再適用が必要。

### スタイル技能の取得・経験点（経験点消費なし／取得数表示／レベル自動参照）（フェーズ10-3）

**日付**: 2026-06-30
**関連実装**: `scripts/data/item/style-skill.mjs`（`expFree`/`excludeFromCount`/`levelRef{enabled,key}`＋`prepareDerivedData` でレベル参照を派生上書き）、`scripts/module/style-skill-acquisition.mjs`（純関数 `resolveLevelRef`/`groupStyleSkillsByStyle`）、`scripts/item/tnx-style-skill-sheet.mjs`＋`templates/item/style-skill-sheet.hbs`（取得・経験点 UI・アクター上は参照先ドロップダウン・レベル入力欄の無効化）、`scripts/actor/tnx-cast-sheet.mjs`（EXP 集計・レベル変更コストの 0 化・suit↔level 連動ガード・スタイル別グループ化）＋`templates/actor/cast-sheet.hbs`（スタイル技能をスタイル別グループ化し各群見出しに取得数・インラインレベル入力の無効化）＋`css/tnx2.css`（`.style-skill-group-header`）。正本 [Character_Creation.md](../01_Wiki/Game_Rules/Character_Creation.md)
**ステータス**: ✅ 実装完了（純関数層テスト済み・全テスト green 967）・実機レビュー済（2026-07-02・問題なし）

#### ルール解釈（ユーザー裁定 2026-06-25／補正 2026-06-30）

- **経験点消費なしで取得可（`expFree`）**＝自動習得（クロガネのフォルムどれか1つ等）・2技能セットの相方自動習得用。EXP 集計・レベル変更コストを 0 にする。
- **取得数カウント（秘技/奥義・個数・表示のみ・非強制）**＝上限は強制しない。**取得数はキャラ全体でなくスタイルごとに上限が決まる**（秘技＝スタイルレベル×2／奥義＝スタイルレベル）。**ユーザー裁定（2026-06-30）**: 表示は (a) 見出し1か所への全体合算＝誤り、(b) スタイル表示エリア（シート上部）＝スタイル技能セクションから離れていて関連が分かりにくく不可 → **スタイル技能リストをスタイル単位でグループ化し、各グループ小見出しにそのスタイルの取得数＋上限を表示**（該当技能の直上・スタイル単位・一か所集中を回避）。スタイル未一致（ワークス技能等）は末尾「その他」群（上限なし）。「取得数に含まない（`excludeFromCount`）」で表示カウントから除外（特殊な取得条件の技能用・リストには残す）。**チェックは単一**で、技能の種別（secret/mystery）に応じてラベルを「秘技/奥義の取得数に含まない」と出し分ける（wiki 記載の2ラベルを冗長なフィールドにせず忠実化）。
- **他スタイル技能レベルの自動参照（`levelRef`）**＝模造技能・アヤカシ〈血脈〉/〈血脈（魔性）〉など。**ユーザー補正（2026-06-30）**: 自動参照は便宜上の別ブロックで、**実体は1つの技能にパラメータ・効果が2組**あるにすぎない。したがって——
  - レベルは**派生計算（`prepareDerivedData`）で1回だけ解決**し `level`/`levelTotal` を参照先の値で上書きする（source 不変）。**判定フローは関与せず、確定済みレベルを通常通り読むだけ**（毎判定の再参照はしない）。
  - **二重計上しない**: 経験点コスト・秘技/奥義の取得数カウントから除外する（同一技能だから）。
  - 指定方法: 辞典/ワールド直下では識別キーの自由入力、アクター上は同型スタイル技能のドロップダウン（既存キーは自動選択）。スートは事後に手動選択。

#### 監査結果

✅ ルール解釈はユーザー裁定（および 2026-06-30 の「実体は1技能」「取得数はスタイル単位」補正）に準拠。レベル参照は判定時でなく派生パイプラインで解決し、判定は無改変（懸念の「毎判定参照」を回避）。取得数はスタイル単位で集計しグループ見出しに表示、経験点・取得数の二重計上回避を `expFree`/`levelRef.enabled` 両系統で実装。`resolveLevelRef`/`groupStyleSkillsByStyle` をテストで固定（`tests/module/style-skill-acquisition.test.mjs`）。

#### 課題・申し送り

- レベル参照の**連鎖（A→B→A 等）**は想定外。`resolveLevelRef` は参照先の base レベルを読むため循環でも無限ループはしないが、確定値は準備順依存になりうる。実運用では連鎖しない規約。
- 新規 UI（取得・経験点の各チェック・参照先ドロップダウン/識別キー入力・レベル入力欄の無効化・取得数バッジ）は engine 未検証＝実機確認待ち。→ **実機レビュー済（2026-07-02・問題なし）**

### 住宅アウトフィットのエリア表示（フェーズ10-4）

**日付**: 2026-07-01
**関連実装**: `scripts/actor/tnx-cast-sheet.mjs`（`_resolveHousingAreaMods` の戻り値に `area` を追加・`_prepareOutfitRow` で `housingAreaRank{key,label}` を付与）＋`templates/actor/cast-sheet.hbs`（アウトフィットタブの住宅施設名の横にランクバッジ）＋`css/tnx2.css`（`.outfit-area-badge`）。正本 [Outfits.md](../01_Wiki/Game_Rules/Outfits.md)「住宅施設（residence）・住宅エリア（housingArea）」
**ステータス**: ✅ 実装完了（全テスト green 968・lint clean）・実機レビュー済（2026-07-02・問題なし。追補修正5点を含む）

#### ルール解釈（既確定 2026-06-13）

- **住宅エリア**＝住宅施設に組み合わせる修正値の集合（厳密にはアウトフィットでない）。**エリア（セキュリティ・ランク）**は住宅エリアが供給し、なし/レッド/イエロー/グリーン/ホワイト/サンクチュアリの6段（`HOUSING_AREA_RANKS`・2026-07-01 イエローをレッドとグリーンの間に追加）。
- 10-4 は**アウトフィットタブでこのランクが一目で分かるよう表示**するもの（名前置換でなく**名前横バッジ**＝アイテム名を保つ）。表示のみ（実効値の合算は既存の住宅施設シート供給値表示が担う）。

#### 監査結果

✅ ランクは住宅施設に紐づく住宅エリア item を **live 解決**（`_resolveHousingAreaMods`＝`fromUuid` で無条件解決・type ガード付き）して読む。名前キャッシュを持たない（[[feedback-no-stale-caches]]）。リンク無しの住宅施設はバッジ非表示。

⚠ **実装中に既存不具合 KI-022 を発見・解消**（2026-07-01）: 旧 `_resolveHousingAreaMods` は `useHousingAreaDrop` で分岐し非ドロップ時に `pack.getDocument(UUID)` を呼んでいたが、`housingArea` は辞典選択・ドロップとも UUID 格納のため `getDocument` では解決できず null。**辞典で紐づけた住宅エリアは供給値（登/セ）もバッジも出ていなかった**（住宅施設シートは常に `fromUuid` で正常＝乖離）。`fromUuid` 無条件解決へ統一し、死んだ `HOUSING_AREA_PACK` 静的・分岐を撤去。バッジが出ない不具合の真因はこれ（CSS でなく解決ロジック）。正本 [Resolved_Issues.md](../01_Wiki/Issues/Resolved_Issues.md) KI-022。

#### 課題・申し送り

- バッジ表示・色分け（レッド/イエロー/グリーン/ホワイト/サンクチュアリ/なし）は engine 未検証。→ **実機レビュー済（2026-07-02・問題なし）**
- 実効値（住宅施設＋エリア供給値）の算出表示は既存（フェーズ6-6）。

#### 追補修正（2026-07-01・実機フィードバック5点）

1. **イエローとサンクチュアリのバッジ色が同色**→ イエロー `#ffe24d`（明レモン）／サンクチュアリ `#d4af37`（メタリックゴールド）に差別化。
2. **住宅エリア辞典ドロップダウンの順がバラバラ**→ `_housingAreaChoices` を**エリア（セキュリティ・ランク）順**に並べる（`getIndex({fields:["system.area"]})` でランク取得）。順番＝**レッド→イエロー→グリーン→ホワイト→サンクチュアリ→なし**。**「なし」は治安が悪いのでなくセキュリティの概念の外（たどり着けない/認識されない）エリアのため別枠で最下部**（ユーザー確定 2026-07-01）。**同ランク内は辞典内の元順を保持**（安定ソート＝名前順にはしない）。当初の名前ソートは撤回。
3. **住宅を装着する部位が無い**→ `DEFAULT_PART_SLOT_PRESET` の外付け系（護符/結界/操縦）に **`住宅`** を追加。※既存ワールドはプリセット編集 UI で再適用が必要。
4. **住宅施設アイテムシートのヘッダーに携帯トグルが残っていた**→ ヘッダー注入トグルを `type === "residence"` で `isCarrying` のみ除外（キャストシートと一致）。
5. **住宅エリアの常備化経験点修正が実 EXP 消費に未反映**（メカニクス監査）→ `updateCastExp` の item コスト集計で、住宅施設の計上時に**住宅エリアの `preserveExpMod` を実効常備化経験点へ加味**（実効値=基本+エリア修正・0clamp。購入判定/派生/preserveExp≠value は非加味）。これに伴い `_resolveHousingAreaMods` を **static 化**し、`_prepareOutfitRow`（実効値・バッジ）と `updateCastExp`（EXP）で共有。住宅エリアが供給する修正値（購入値/常備化経験点/…）のうち常備化経験点が EXP 計算へ流れるのは正本 [Outfits.md](../01_Wiki/Game_Rules/Outfits.md) の「住宅エリアの持つ修正値」どおり。全テスト green(968)・lint clean。

> ※上記 2〜5 と 10-5 の課題・申し送りは記録挿入ミスでファイル末尾（10-6 の後ろ）に紛れていたのを 10-8 通読時に本来の位置へ復元（2026-07-02）。

### コンバットスピード 3層パイプラインと入力 UI（フェーズ10-5・KI-010 解消）

**日付**: 2026-07-02
**関連実装**: `scripts/data/helpers.mjs`（`combatSpeedField` 3層化＝mod 削除・`computeOutfitAggregates` フラグ駆動）、`scripts/data/item/tap.mjs`（`combatSpeedModGhostIgnore`）、`scripts/data/item/helpers.mjs`（`parseEffectTargetKey` に cs 名前空間）、`scripts/data/actor/cast.mjs`（`_prepareCombatSpeedTotals`・cs 着地・cs.base 準備ゲート）、`scripts/data/actor/common/attributes.mjs`（素の実効値フォールバック）、`scripts/actor/tnx-cast-sheet.mjs`＋`templates/actor/cast-sheet.hbs`（戦闘タブ CS ブロック・プレアクト初期化）、`scripts/item/tnx-outfit-sheet.mjs`＋`templates/item/outfit-sheet.hbs`（タップのフラグ UI・（）表記）、`system.json`（initiative）。正本 [Combat_Flow.md](../01_Wiki/Game_Rules/Combat_Flow.md)・[Outfits.md](../01_Wiki/Game_Rules/Outfits.md)「タップの CS 修正」・[Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md) 大原則5/6
**ステータス**: ✅ 実装完了（全テスト green 973・lint clean）・実機レビュー済（2026-07-02・問題なし。**AE 配線は具体データ未整備で未検証のまま持ち越し**）

#### ルール解釈（2026-07-02 の言語化セッションで確定）

- **3層＝「決定値（再計算不可）＋修正値（ライブ）」**（実装表現の原則）: CSベースの能力値項は**プレアクト時点の実効値**を焼き込み（以後、能力値変化に追従しない＝「アクト中に能力値が変化しても CS は再計算されない」の実装）。CS はプレアクト決定。カレントはセットアップ末決定（自動管理13）。修正値（freeMod・タップ修正・AE 常時修正）はライブに乗り降りする。
- **「読み飛ばし」＝修正項の条件付き除外**: フラグ ON（携帯起点）のタップ修正はゴースト登場中に外れ、cs.base 着地の AE は保持アイテム未準備で外れる。決定値には触れないため再計算なしと矛盾しない（「無視」「考慮する必要は無い」の解釈）。
- **タップ CS 修正はフラグで一元化**（符号ルールを吸収した最終裁定）: OFF＝準備起点・ゴースト無関係（フラット・オリジナルデータ基準）／ON＝携帯起点・ゴースト時読み飛ばし・表示（）囲み（原作の括弧書き「CS：（-20）」＝フラグの表示形式）。
- **CS に 0clamp はしない**（負になりうる）。
- **シート表示は常に単一の「CS」・表示層は自動制御**（2026-07-02 ユーザー裁定）: ベース/CS/カレントの3層は内部データ。カット（開始済み戦闘）進行中＝カレント／それ以外＝CS を自動判定（`isActorInStartedCombat`）。3つを並べて常時表示しない・手動切替もしない。
- **カット開始時にカレントへ CS 実効値を自動セット**（「セットアップ末に決定・初期値は CS」の近似）。以後のセットアップ修正・メジャー後 0・待機 1 の自動管理はフェーズ13。
- **initiative**: 表示中の CS（`displayTotal`）を読む。カット開始直後はカレント＝CS のためセットアップ順も正しく、以後のカレント修正がメイン順に反映される。

#### 監査結果

✅ 3層の決定タイミング・非再計算・読み飛ばしの各裁定に逐条準拠。プレアクト初期化は base（能力値項）と value（CS）を同時セットし、ライブ修正項は保って写す。`computeOutfitAggregates`（純関数・テスト済）がフラグ別に通常分/読み飛ばし可能分を分離。パラメータは直接読み取り（大原則5）・AE は cs.\* の3層着地（大原則6）で表現系統を分離。

#### 課題・申し送り

- CSカレントのカット中自動管理（セットアップ末決定・メジャー後0・待機1）＝フェーズ13。initiative のカレント切替も13。
- extra は combatSpeed を持たず initiative 不可（従前どおり）。guest/troop はフォールバックで式解決可（本格対応は11）。
- 『CS：0（+1）』の例外タップは機械化せず手動/AE（既裁定）。
- cs.base の準備ゲートは **AE 着地の中で唯一の特例**。他の着地先への一般化は解説効果全体の扱いに波及するため見送り（必要になったら別途設計）。

### 戦闘タブ本体（武器・防具・状態・タイミング別一覧）（フェーズ10-6）

**日付**: 2026-07-02
**関連実装**: `scripts/data/actor/cast.mjs`（`weaponRefs`）、`scripts/actor/tnx-cast-sheet.mjs`（`_prepareCombatData`・`declareWeaponRef`）、`templates/actor/cast-sheet.hbs`（戦闘タブ）、`css/tnx2.css`。正本 [Outfits.md](../01_Wiki/Game_Rules/Outfits.md)「生身・パラメータの適用宣言・携帯/準備の適用範囲」・[Combat_Flow.md](../01_Wiki/Game_Rules/Combat_Flow.md)・[Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)
**ステータス**: ✅ 実装完了（全テスト green 977・lint clean）・実機レビュー済（2026-07-02・問題なし。生身変更装備・義体の実データ検証はフェーズ12 へ持ち越し）

#### ルール解釈（2026-07-02 の言語化・同日の数回の是正を経た確定仕様）

- **一覧は置かない**: 武器/防具の一覧はアウトフィットタブ・効果系（負傷/BS）は状態タブの領分（ユーザー確定）。戦闘タブが答える質問は「**いま自分の攻撃力・受け値・防御力はいくつか／このタイミングで使えるものは何か**」。
- **生身ライン（2026-07-02 後半の裁定で再設計）**: 戦闘タブの武器欄は**生身のデータのみ**を扱う（案2確定＝通常武器の使用は攻撃判定の用途に移管・〈二刀流〉等の合算は12の判定フロー側）。攻撃用/パリー用それぞれの「現在の生身」を表示し、書き換え元（準備済みの全身義体・生身変更装備 `isFleshChange`）を**攻撃とパリーで別々に選択可**（別の身体部位の使用は設定上も自然・便宜的事前設定で本義は都度宣言＝非強制）。**生身はアイテムとして用意せず**（部位「-」＝準備不可のため）、攻撃力 I+0・受け値 0 を**アクターのデータ**（`baseAttack`/`baseGuard`＝フェーズ9の休眠フィールドを正式化）として保持。空欄＝未変更の生身が既定のため「未宣言」状態は存在しない。書き換え意味論（生身は単一・相互合算不可）は [Outfits.md](../01_Wiki/Game_Rules/Outfits.md)「生身の変更」。
- **防御力**: **全ての義体を含む種別ごとの合計（S/P/I）のみ**を表示（防具は合算適用・義体の防御力は防具と加算）。防御力は物理ダメージ算出の減算項（精神攻撃には適用なし・X は軽減されない＝Damage_Rules）であり、単一の「防御」バイタルには昇格させない。
- **タイミングごとの使用技能・アウトフィット**（検索補助）: 標準のプロセス/アクション8種は**空でも常に表示**。差し込みタイミング（その他の自由記述）は**記述テキストごとにグループ化して非空のみ**標準群の後ろに。自由記述なしの「その他」と「解説参照」は**「その他」群**（非空のみ・末尾）。準備ゲート（`isPrepared || noPrepareRequired`）はアウトフィット側にのみ適用。
- **CS の表示（2026-07-02 後半の裁定）**: 公式プロファイルシートの CS ボックス様式（2段ラベル＋値・式キャプションは載せない）。**カット進行中はボックスのグローで示す**（チップ廃止・ユーザー案）。編集モードは「決定の連鎖」の縦積み（ベース算出＋修正 → CSを決定 → CS → 自動セット → カレント）。**CS への修正は原則 AE**・手入力は編集モードの例外的直接介入・閲覧モードに操作なし（Combat_Flow「手動修正の位置づけ」）。

#### 監査結果

✅ 参照解決（複数時のみ宣言・単一自動）・種別別防御合算・タイミング表示規則は、正本（Outfits.md 2026-07-02 節・Damage_Rules.md・Combat_Flow.md）とユーザー裁定に逐条準拠。攻/受/防はアイテムの実効値（total）を表示（フェーズ9 の表現原則どおり）。

#### 課題・申し送り

- **未実装（裁定待ち/後続）**: 生身変更装備の識別と「同時準備でも適用は一種類のみ」の表現／全身義体の置き換え（生身の完全置換）によるキャラ実効攻/受/防の派生——12 のダメージ算出と併せて設計。防御合計は複数義体を単純合算しており「一種のみ」の厳密化も12で。
- 宣言の妥当性チェック（未準備武器の宣言残り等）は非強制のまま（無効な宣言は解決時に読み飛ばし、候補1つなら自動参照にフォールバック）。
- レイアウトの最終確定はユーザーの実機レビューで行う。→ **実機レビュー済・確定（2026-07-02）**
- **監査上の反省**: 初版は「武器/防具の一覧」「カットの流れの可視化」を実装した——前者は誤記録（KI 相当の承認偽装記録）由来の安易な読み、後者は要望（分類再表示）の取り違え。表示要素はルールとユーザーの言葉から導出し、ジャンル慣習から持ち込まない。

### アクションランク（AR）の付与型モデルと戦闘タブ表示（フェーズ11 追補）

**日付**: 2026-07-03
**関連実装**: `scripts/data/helpers.mjs`（`ACTION_RANK_GRANT`・`actionRankField`・`resolveActionRankDisplayTotal`）、`scripts/data/actor/common/attributes.mjs`（スキーマ＋素の派生）、`scripts/data/actor/cast.mjs`（AE 適用後の 0clamp・表示確定・ar 着地・準備ゲート）、`scripts/data/item/helpers.mjs`（`parseEffectTargetKey` に ar 名前空間）、`scripts/tnx.mjs`（カット開始時の付与＝CS カレントと同フック）、`templates/actor/cast-sheet.hbs`＋`css/tnx2.css`（戦闘タブ AR ボックス）。正本 [Combat_Flow.md](../01_Wiki/Game_Rules/Combat_Flow.md)「アクションランク」・[Troops.md](../01_Wiki/Game_Rules/Troops.md)（分身 AR=1 固定）
**ステータス**: ✅ 実装完了（本体 `fb9c08d` ＋ 表示修正 `b0132e8`・全テスト green 992・eslint/stylelint 新規警告なし）・**実機確認済み（2026-07-03。非戦闘時表示の指摘 → 修正済み）**

#### ルール解釈（2026-07-03 言語化）

- **AR は付与型**: キャラクター作成で決まる常設ステータスではなく「カット進行のシーン開始時に 1 付与」。**ルール明記でなくユーザー解釈**（ユーザー自身が明示的に留保。原文との矛盾が判明したら見直す）。実装は基準値の入力欄を持たず、ルール定数 1（`ACTION_RANK_GRANT`）＋手動修正（freeMod）＋AE で実効付与値 `maxTotal` を派生する（「初期値」概念の不在をデータ構造で表現）。
- **変動源は基本的に高額アウトフィットの増強のみ**（まれ）→ AE 着地 `system.ar.max` で受ける。
- **クリンナップの回復は必ず付与値まで全回復**・**シーン進行中は AR を消費できない**（後者もユーザー解釈）→ ~~カット外は常に満額＝表示は付与値~~ **訂正（2026-07-03 実機確認）: カット進行外は AR の値自体を持たない＝表示は「なし」**（「満額表示」は Code の誤導出。ユーザー裁定で是正・`b0132e8`）。カット進行中のみ現在AR を数値表示する。
- 消費（メジャー後 −1・行動不能 −1）・クリンナップ全回復の**自動管理はフェーズ13**。今回は表示＋カット開始時の付与自動セット＋手動増減まで。
- 分身の AR=1 固定は修正を付けないデータ運用で表現（機構なし・[Troops.md](../01_Wiki/Game_Rules/Troops.md)）。

#### 監査結果

✅ 付与型の言語化に逐条準拠。負値は 0clamp（AE 適用後）。guest / troop も attributes フォールバックで同式解決（シート整備は 11-3 / 11-4）。extra は AR 自体を持たない（attributes 非保持＝「カット進行外では消費されない」性質とも整合）。

#### 課題・申し送り

- ~~**ar.max の AE 着地に cs.base と同じ準備状態ゲートを適用した（Code 判断・要ユーザー確認）**~~ → **裁定確定（2026-07-03）**: 「アウトフィットのデータの効果は必ず準備しないと適用されない（危険値と CS 修正が例外）」とユーザーが明言＝[Outfits.md](../01_Wiki/Game_Rules/Outfits.md)「携帯・準備の適用範囲（一般原則）」どおりで、ゲート適用は正しい。常時系の AE 着地（cs.base / ar.max）への準備ゲートは以後この一般原則の適用として扱う（10-5 の「cs.base は唯一の特例」の記述は本裁定で更新）。
- AE 配線（`system.ar.max`）は具体データ未整備で**実機未検証**（cs.\* と同様の持ち越し）。
- 消費・回復・イニシアチブプロセスの −1 の自動化＝フェーズ13。ダメージチャートの AR 系効果（驚愕/士気喪失）の発火＝12/13/15。

### ゲストシートと DataModel 共通基底（フェーズ11-3）

**日付**: 2026-07-03
**関連実装**: `scripts/data/actor/common/character-base.mjs`（新設）、`scripts/data/actor/cast.mjs`・`guest.mjs`（継承化）、`scripts/actor/tnx-guest-sheet.mjs`＋`templates/actor/guest-sheet.hbs`（新設）、`scripts/module/user-flag-schema.mjs`（GM ゲスト合算）、`scripts/tnx.mjs`（シート登録・シード処理）。正本 [Actor_Types.md](../01_Wiki/Documents/Actor_Types.md)・[Card_Operations.md](../01_Wiki/Game_Mechanics/Card_Operations.md)「手札上限」
**ステータス**: ✅ 実装完了（本体 `a76e01e`・全テスト green 1012・eslint 0 error）・実機確認待ち

#### ルール解釈（2026-07-03 確定分の実装）

- **ゲスト＝キャストからセッション履歴（EXP・User 同期）~~のみ~~と lifePath を除いたもの**（**同日再訂正・`9dc0928`**: 「セッション履歴以外差異なし」はライフパスの失念＝ユーザー再訂正。ライフパス除外はフェーズ6 記録どおり）: DataModel はセッション履歴クラスタ（ownerUserId / syncWithOwner / history / exp）＋ lifePath の**5フィールド**を cast 固有に残し、他は全て共通基底 `CharacterBaseDataModel` へ（報酬点・部位・生身・isGhost・handMaxSizeMod 含む）。**差分が正確にこの5フィールドであることをテストで不変条件化**。シートは features.lifePath でライフパス節をゲート（guest=false）。
- **RL の手札上限はゲストの技能で修正されうる**: GM ユーザーの実効上限にワールド内全ゲストの `handMaxSizeMod`（有効 AE 由来）を合算（上限修正は基本「メインプロセス中」効果＝重複懸念なし・ユーザー確認済み。登場概念導入後に「登場中のみ」へ差し替え可能）。
- **判定・EXP**: ゲストの判定はキャストと同経路（派生パイプライン共通化により実効値も同式）。技能レベル変更の EXP 連動は基底フックの既定（素通し）が効き、ゲストでは消費が発生しない。
- **新規ゲストのシード**: 部位プリセット＋基本13技能＋リンクトークンをキャスト同様に流し込み（データ同一の帰結）。所有権の自動設定（プレイヤー作成時のオーナー付与）と disposition=FRIENDLY は**キャストのみ**（ゲストは RL の持ち物・敵味方は場合による＝Code 判断）。

#### 監査結果

✅ 「セッション履歴以外データ的に何も違いがない」（2026-07-03 ユーザー明言）をスキーマ差分テストで逐条担保。GM 合算は Card_Operations.md の確定記述どおり。guest の EXP 非消費は基底フック設計（11-2）の帰結で追加分岐なし。

#### 課題・申し送り

- ゲストのゴースト登場・部位管理・戦闘タブ・判定起動は cast と同一コードパスのため、実機確認は guest シートの一巡で足りる（AE 配線の実データ検証は従来どおり持ち越し）。
- シード時の disposition・所有権の扱い（Code 判断）は実機確認時に要すれば調整。

### トループシートとエニグマモード（フェーズ11-4）

**日付**: 2026-07-03
**関連実装**: `scripts/data/actor/troop.mjs`（CharacterBaseDataModel 継承化・isEnigmaMode・memo 廃止/移行）、`scripts/actor/tnx-troop-sheet.mjs`＋`templates/actor/troop-sheet.hbs`（新設）、`scripts/actor/tnx-character-sheet-base.mjs`（SHEET_FEATURES 拡張・神業ガード）、`templates/actor/parts/sidebar.hbs`・`header.hbs`（features ゲート・heads パネル）、`scripts/tnx.mjs`（登録・シード・trackableAttributes）。正本 [Troops.md](../01_Wiki/Game_Rules/Troops.md)
**ステータス**: ✅ 実装完了（本体 `5c8151f`・全テスト green 1018・eslint/stylelint 新規警告なし）・実機確認待ち

#### ルール解釈（Troops.md・2026-07-03 言語化の実装）

- **トループの構成**（スタイル1・能力値・技能・アウトフィット・状態・神業なし）と**判定はキャストと同じ**: CharacterBaseDataModel 継承で判定・実効値・AE・アウトフィット集計・CS/AR をキャストと同経路にした。基本13技能もキャスト同様に流し込む（判定同一の帰結）。
- **heads＝純粋な人数**: 入力は手動（敵対トループ=RL 任意・取得技能由来=判定達成値の転記。「トループ側で自動化すべき点はない」＝ユーザー明言どおり自動化なし）。閲覧モードに±ボタン（ダメージ分の減少を RL が手で行う＝ダメージ例外の器。攻撃フローからの自動適用はフェーズ12）。
- **エニグマモード**: `isEnigmaMode` ON で heads の意味・ラベルが「人数」→「エニグマポイント」に切替（フィールドは heads を共用＝トークンバー連携がモードに依らず機能。挙動は同じ＝達成値付与・ダメージ分減少）。
- **神業不可**: 神業のドロップとスタイルインポート時の自動取得の両方をガード（features.miracles=false）。シートにも神業 UI を出さない。
- **分身**: troop として作成し、AR=1・CS/CSカレント=0 はデータ入力で表現（専用機構なし・Troops.md どおり）。スタイル複数は運用（データ制約なし）。
- **報酬点・部位管理・ゴースト等は троープのルール上未使用**: 基底由来のフィールドはスキーマ上残るが死蔵（シート非表示・GM 手札上限合算は guest のみ）。DataModel コメントと本記録に明記。

#### 監査結果

✅ Troops.md の言語化に逐条準拠（構成・判定同一・heads 手動・エニグマモード・神業不可・分身=troop）。memo→description の migrateData で既存データを保全。guest とのスキーマ差分が heads/isEnigmaMode の2フィールドに限られることをテストで不変条件化。

#### 課題・申し送り

- **基底継承の設計変更**: 11-3 レビューで「troop は attributes 直 mixin＋選択共有の見込み」としたが、派生値パイプライン（AE・集計・CS/AR）への依存が本体であるため CharacterBaseDataModel 継承に変更（死蔵フィールドの明記とトレードオフ）。
- トループの biography（パーソナルデータ欄）は集団に不要な項目も含むが、ユーザー意向「biography はあった方が良い」に従い全表示。実機確認で過剰なら絞る。
- 攻撃フローからの heads 自動減算＝フェーズ12、NPC取得用途（達成値→heads/ポイント）＝11-6。

#### 是正（2026-07-03・同日・ユーザー指摘5件 → 本体 `46a8584`）

初版はユーザーの実機レビューで大きく是正された。**「エニグマモード」フラグは Code の誤設計**（ユーザーの想定は種別ドロップダウン）:

1. **種別 `troopMode` ドロップダウン（トループ/エニグマ/分身）に刷新**（isEnigmaMode 廃止・migrateData 移行）。分身はリソース管理なし（1点で消滅）＝人数欄を出さない。
2. **人数編集は報酬点同様の平入力**（スピナーはサイドバー幅に対して広すぎ見切れ＝ユーザー指摘）。
3. **市民ランク・パーソナルデータを除去**し、市民ランク枠に**トループレベル**を追加。
4. **成長欄を除去**。**能力値＝スタイルの基本値＋トループレベル**（新ルール・2026-07-03 言語化。`_prepareAbilityTotals` を基底フック化し troop がオーバーライド。修正・アウトフィット・AE・0clamp は共通。**制御値も＋トループレベルの対称実装＝Code 判断・要ユーザー確認**）。
5. **ハンドル除去・名前固定**: トループ=「(スタイル名)・トループ」／分身=「(分身元キャラ)の分身」（`syncTroopName` フックで同期・分身元は当面手入力＝11-6 で自動化予定）。自由入力はエニグマのみ。

正本 [Troops.md](../01_Wiki/Game_Rules/Troops.md)「種別と名前の規則」「能力値の決定」に新ルールを記録。テスト 1023 green。

#### 追補（2026-07-03・同日・ユーザー指摘3件 → 本体 `8b28d3b`）

- **トループ名にレベル付加**: 「(スタイル名)・トループ（(トループレベル)レベル）」（例: カブトワリ・トループ（2レベル））。
- **「ワークスを設定」（hasWorks）フラグ新設**: トループは基本ワークスなしだが、持つトループは作成可能。ON＋組織ありで名前「(組織名)（(スタイル名)(トループレベル)レベル）」。**組織未設定の間は OFF と同じ挙動**（「フリーランス」表示の不整合回避＝ユーザー指定）。所属ブロックは ON のときだけ表示（showAffiliation）。
- **トループレベル枠のミニラベル ID→LV**（市民ランク枠 CSS の::before 継承を上書き）。
- 名前導出は純粋関数 `computeTroopFixedName`（`data/helpers.mjs`）へ切り出しテスト化（6件）。同期トリガーに troopLevel / hasWorks / 組織アイテムを追加。テスト 1030 green。
- ⚠️ **要確認**: ワークス形式のレベルはユーザー文言では「スタイルレベル」だったが、通常トループ名と同じ**トループレベル**の意と解釈して統一（[Troops.md](../01_Wiki/Game_Rules/Troops.md) に留保を記録）。スタイルアイテムのレベルを指す意図なら要修正。

#### 追補2（2026-07-03・同日・確定2件＋部署技能 → 本体 `7e7a4cc`）

- **裁定確定2件**: ①制御値へのトループレベル加算（対称実装を承認） ②ワークス名のレベル＝**トループレベル**（当初文言「スタイルレベル」は取り違えとユーザー確認）。留保を各正本から解除。
- **部署技能（新ルール・2026-07-03 言語化）**: スタイル技能に `special.works.isDepartment` を新設（ワークス技能 ON のときのみ設定可のチェックボックス＝実装方式はユーザーから一任され簡明なフラグ案を採用）。取得していると**所属名（ワークス名）の表示がこの技能名で上書き**——キャスト/ゲスト＝ヘッダー所属表示、トループ＝トループ名の組織名部分（例: 千早グループ（クグツ2レベル）→ 後方処理課第二班（クグツ2レベル））。判別は純粋関数 `findDepartmentSkillName`（テスト付き）・トループ名同期トリガーに styleSkill を追加。
- **スタイル技能シートのセクション再編（ユーザー指摘）**: 旧「特技専用」は名称不明瞭で、**代用技能が特技限定であるかのような配置は誤り**（そのような裁定は存在しない）。→「**ワークス技能**」（特技のみ＝Character_Creation.md の正本どおり category ゲート維持）／「**代用技能**」（全カテゴリで設定可・独立）／「**神業の書き換え**」（unique=miracleChange ゲート）に分割。
- テスト 1034 green。

### エキストラシートと固定値判定（フェーズ11-5）

**日付**: 2026-07-03
**関連実装**: `scripts/actor/tnx-extra-sheet.mjs`＋`templates/actor/extra-sheet.hbs`（新設）、`scripts/actor/tnx-character-sheet-base.mjs`（features.abilities/combat ゲート・固定値判定フロー）、`scripts/data/item/common/usage.mjs`（fixedResult）、`templates/app/usage-sheet.hbs`（発動タブ入力）、`templates/actor/parts/tab-abilities.hbs`（能力値グリッドのゲート）、`scripts/tnx.mjs`（登録・リンクトークン）。正本 [Actor_Types.md](../01_Wiki/Documents/Actor_Types.md)・[Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「固定値判定」・[Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)（エキストラはダメージ概念なし）
**ステータス**: ✅ 実装完了（本体 `b37abeb`＋`7e68c96`・全テスト green 1035・eslint/stylelint 新規警告なし）・実機確認待ち

#### ルール解釈（2026-07-03 言語化の実装）

- **エキストラ＝基本は名前のみ**（biography のみ・attributes 非保持）: シートは 技能／アウトフィット／詳細 の3タブ（能力値グリッド・戦闘・状態・履歴タブなし＝能力値もダメージ・状態の概念もない）。スタイル・神業・技能・アウトフィットの所持は可。**神業は使用できる可能性がある**ため神業表示・使用ボタンは残す。
- **判定は固定値でのみ可能**: usage の check 用途に**固定達成値（fixedResult）**を新設。設定された用途は**カードも出さず能力値も参照せず、記載の数値がそのまま達成値**（〈知覚〉10 等）。エキストラは固定達成値の用途のみ実行可（通常判定はガードして通知）。**他アクターでも固定達成値の用途は同様に動く**（ルール上は主にエキストラ由来のアウトフィット）。
- 固定値判定の結果チャットは通常判定と同じ `checkResult` フラグ形（achievement 等）で投稿し、対決の読み取り等から同様に扱えるようにした。
- 新規エキストラはリンクトークンのみ設定（**基本13技能は流し込まない**＝「基本は名前のみ」の帰結・Code 判断）。

#### 監査結果

✅ 「固定値で達成値が算出（カードも能力値も参照しない）」（Check_Rules 2026-07-03 言語化）を fixedResult の実装がそのまま符号化。エキストラの通常判定不可・神業使用可能性・ダメージ概念なし（状態タブ非搭載）に逐条準拠。

#### 課題・申し送り

- 固定達成値の入力はスピナーでなく**平入力**（空欄=通常判定の nullable 意味論のため。スピナー原則の例外として記録）。
- エキストラの initiative は従前どおり不可（combatSpeed 非保持・フェーズ13 で扱う）。
- 固定値判定と対決の突き合わせ UI・目標値付き固定判定はフェーズ12 以降の判定拡張で必要になれば追加。

#### 再訂正（2026-07-04・エキストラは完全にデータなし → 本体 `e93a082`）

- **エキストラは完全にデータを持たない**（名前と説明のみ）とユーザーが再訂正。フェーズ6 記録以来の「スタイル・神業・技能・アウトフィットを持てる」は**神業以外を持たないゲストとの取り違え**だった。神業が要る場合は口頭宣言か、データを制限したゲストとして作る。
- シートをポートレート＋名前＋説明エディタのみに縮小（タブなし・小型ウィンドウ）。**アイテムのドロップは全て拒否**・判定不可。
- **固定値判定の担い手は「アウトフィットとして取得できるエキストラ」**（小分類エキストラ・キャスト/ゲスト所持）で、**設定方法は check 用途の「固定達成値」とする**（2026-07-04 ユーザー確定・一般技能アイテムは無改造の方針）。スート・レベル・カード・能力値は一切読まない（fixed 分岐は技能解決より前で短絡＝実装済みのまま）。
- 経緯: Code が①用途→②アウトフィットのデータ欄(提案)→③一般技能フィールド(着手)と置き場を振ったが、ユーザー裁定で**用途に確定**。②③は撤回（③のスキーマ変更は revert 済み・コミットに含まれず）。


## フェーズ11-6: NPC取得用途・消費先設定・分身の使用回数共有・所有トループ経験点（2026-07-05）

**ルール正本**: [Troops.md](../01_Wiki/Game_Rules/Troops.md)「NPC取得」「事前作成と所有者記録」「所有トループの経験点」・[Phase_11_Tasks_Detail.md](../01_Wiki/Phases/Phase_11_Tasks_Detail.md) 11-6 設計計画（実装前にユーザー承認済み・2026-07-05 実装開始指示）。

- **NPC取得用途（npcAcquire）**: モードは明示選択（エキストラ/トループ/エニグマ/分身。参照先からの導出は却下済み）。エキストラ=判定なしで派生取得＋extraActorRef の共有アクターのトークン配置を**同時**に実施。トループ/エニグマ=通常判定→**heads（現在/最大）＝達成値**を事前作成済みアクターへ転記（複製しない）。分身=目標値10・**達成値10以上で成功**（目標値の一般規約）・成功時のみ配置・sourceName を所有者名で自動設定。
- **対象解決は所有者逆引き**（troop 側 ownerActorRef=使用者・種別一致。複数該当は選択ダイアログ）。フェーズ10 の autoAcquireActors は本フローでは不使用（残置）。
- **レベル転記**: 取得技能のレベルがそのままトループ/エニグマのレベル（起動時に転記・分身対象外・敵対トループは手入力のまま）。エニグマの呼称は「エニグマレベル」。
- **消費先設定**: 全消費は用途の consumeTargets からのみ（親自動消費・コンボ遠隔消費の自動スキャン全廃＝ユーザー確定。組み合わせ技能の消費は手動設定が必要）。消費量可変（既定1）。既存 check 用途は migrateData で「親×1」に互換化。check/npcAcquire 判定系=判定実行時に消費・他タイプ=使用時。神業 usageCount への消費アクセスも本設定経由（§4.2: 回数管理への機械的アクセスのみ・効果自動化ではない）。
- **分身の使用回数共有**: 本体側カウンターが唯一の正本（識別キー一致→名前一致で照合・照合不能はローカル＋警告）。分身直下のアイテムシートは使用回数欄を共有表示・入力無効化。
- **経験点**: 所有トループ級（troop/enigma・**分身除外**）のアイテムコストを**取得元キャストの消費経験点に算入**（所有者参照が計上の前提・User 直結ではない）。対象=スタイル技能の取得と成長/アウトフィット常備化/一般技能の成長（Lv1 無償は既存規約と同一）。トループレベルは取得技能側で計上済みのため独立消費なし。ゲスト所有は算入なし（ユーザー確定）。
- **権限**: トークン配置は TOKEN_CREATE が無ければ不可（警告のみ・委譲機構なし＝D&D 同様）。配置 UX は dnd5e 実挙動踏襲（トークン半透明プレビュー追従・左クリック確定・右クリックスキップ・グリッドスナップ+Shift 解除）。

**ステータス**: 実装完了→反復実機レビューを経て**確認済み（2026-07-08 ユーザー明言・フェーズ11 完了 v0.11.0）**。
**レビュー中の設計変更（すべてユーザー裁定・Troops.md に正本）**: ①取得対象は用途側の「取得するアクター」参照（所有者逆引き廃止・2026-07-07）→さらに**分身は対象設定なしの直接召喚**（永続1体自動確保・召喚数指定・2026-07-08） ②分身=差分反映方式（本体から全アイテム複製・成長の修正値焼き込みで数値完全一致・神業除外・レベル概念なし） ③sourceName（分身元テキスト入力）廃止=所有者参照から導出 ④技能側 autoAcquireActors 廃止（用途側一本化） ⑤消費行の自動入力導出（作成時+ボタン・親×1+isLimit参加技能×1） ⑥スタイル等のドロップを種別ルーティングに（閲覧モードでも神業同時インポート）。


### 判定バフのスタイル/ワークス・グループ参照（フェーズ12 補・式/AE参照拡張）

**日付**: 2026-07-10
**関連実装**: `scripts/data/item/helpers.mjs`（`parseEffectTargetKey` に group 解析・`checkChangeMatches` の照合）、`scripts/module/tnx-check-flow.mjs`（`_computeCheckBonus` の criteria）。正本 [Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md)「判定バフ」・[Formula_Reference.md](../01_Wiki/Game_Mechanics/Formula_Reference.md)
**ステータス**: ✅ 実装完了（本体 `68b0d3f`・全テスト green 1147・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-10 言語化）

- **特定スタイルのスタイル技能・特定ワークスのワークス技能の判定に恒常ボーナスを与える能力が複数存在する**。固有名詞技能の大分類（操縦/社会）は識別キー前方一致（`check.operate*`）で既に束ねられるが、スタイル/ワークスは識別キーでなく技能の `system.style` / `system.special.works.organization` で束ねる必要がある。
- 判定バフの「値」は**リテラルのみ**（式で判定に足す用途は「判定ボーナス」欄が担う＝役割分担・2026-07-10 ユーザー確定）。

#### 監査結果

✅ `check.style.<スタイル識別キー>` / `check.works.<組織識別キー>` を追加。判定に使った技能のうち該当 style/organization を持つものがあれば適用（重複排除は既存の identity 規約に乗る）。既存の完全一致/前方一致・能力値/制御判定は不変。

#### 課題・申し送り

- **「特定スタイルを持つキャラクターへのダメージ増加」**能力 → 下記「ダメージ対象バフ」で実装（AE `damage.vsStyle.*`）。当初 `@target` 式案を出し「AE は対象文脈を持てず不適」と述べたが、**これは値バフ（`system.*`）に限った話で誤り**——判定バフ `check.*` と同じ「実行時評価の別系統」にすれば AE で表せる（ユーザー指摘で修正）。
- 値バフ（`system.*`）側の同グループ化は未着手（当面 category/識別キーで賄う＝[Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md) §6）。styleKey/orgKey の入力補助（辞典ドロップダウン）も未提供。


### ダメージ対象バフ（対象のスタイル/ワークス依存ダメージ・フェーズ12 補）

**日付**: 2026-07-10
**関連実装**: `scripts/data/item/helpers.mjs`（`parseEffectTargetKey` の damageVs・`damageVsChangeMatches`/`gatherDamageVsSources`/`collectActorEffectBuffs`）、`scripts/data/actor/common/character-base.mjs`（`_applyEffectBuffs` の除外）、`scripts/module/damage-flow.mjs`（`collectDamageVsBonuses` と台帳合算）。正本 [Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md) §1-2・[Formula_Reference.md](../01_Wiki/Game_Mechanics/Formula_Reference.md)
**ステータス**: ✅ 実装完了（本体 `a76c7fe`・全テスト green 1152・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-10 言語化）

- **攻撃対象が特定のスタイルを持つ場合にダメージが増加する能力が複数存在する**（例: 〈アヤカシ〉持ちに +N）。これを**データ（AE 一行）で再現したい**というのがユーザーの主目的。
- 対象依存の値は**値バフ（prepareDerivedData 焼き込み）では表せない**（対象未定）。判定バフ `check.*` が既に「実行時にアクション文脈を見る」パターンを持つので、その**ダメージ・対象参照版**として実装するのが正しい設計（ユーザー確定＝AE で表現）。

#### 監査結果

✅ `damage.vsStyle.<スタイル識別キー>` / `damage.vsWorks.<組織識別キー>` を追加。恒常適用パスから除外し、ダメージフローが攻撃対象を解決→対象の style（`type:"style"` アイテム）/works（ワークス技能の組織）で照合→合致分を用途のダメージ修正と同じ台帳行としてダメージへ合算。値はリテラル・重複規約は判定バフと同一（`_gatherBonusSources` を共用）。

#### 課題・申し送り

- AE `damage.vs*` はフラット +N（キャラ恒常）。**式ベース・用途固有の対象依存は用途のダメージ修正欄で `@target.*` を参照する形で実装済み**（下記「用途ダメージ修正の @target 参照」・`f66db00`）。「組み合わせた物理攻撃が〈スタイル名〉を対象とする場合～」等の用途固有効果はそちらで表現する。
- 対象の「ワークス所属」は当面ワークス技能の組織で近似（キャラクター自身のワークス所属フラグは未整備＝[Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md) §6）。

#### 追記（2026-07-11・本体 `7c669fd`）: 与えるダメージキー `damage.dealt[.<系統>]`

- **最重要キーの欠落をユーザー指摘**——無条件の「与えるダメージ +値」が無かった。`damage.dealt`（全系統）/`damage.dealt.physical|mental|social`（系統別）を同じ実行時評価系統に追加（`damageDealtChangeMatches`/`gatherDamageDealtSources`・攻撃の系統で照合・台帳は「ダメージ修正（効果名）」行・重複規約は判定バフと同一）。


### 用途ダメージ修正の @target 参照（式で攻撃対象を参照・フェーズ12 補）

**日付**: 2026-07-10
**関連実装**: `scripts/module/tnx-formula.mjs`（`buildFormulaData` の target 引数・`buildTargetData`・`evaluateBonusRows`）、`scripts/data/item/helpers.mjs`（`targetStyleWorksKeys`）、`scripts/module/damage-flow.mjs`（対象解決と配線）。正本 [Formula_Reference.md](../01_Wiki/Game_Mechanics/Formula_Reference.md)「@target.*」
**ステータス**: ✅ 実装完了（本体 `f66db00`・全テスト green 1155・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-10 言語化）

- **「組み合わせた物理攻撃が〈スタイル名〉を対象とする場合、ダメージ +N」**のような効果は、AE のフラット対象バフでなく**用途（その攻撃）に紐づく式**で表すのが自然（ユーザー明言＝「むしろこちらでないと再現できない」）。
- ダメージ算出時に攻撃対象を確定するので、ダメージ修正/`damageBoost` の式に攻撃対象を公開する。

#### 監査結果

✅ ダメージの式で `@target.system.*`（対象の実効値・AE 込み）・`@target.style.<識別キー>` / `@target.works.<組織キー>`（対象が持てば 1・欠損キーも 0）を参照可能に。`@target.style.<キー> * N` で対象スタイル条件のダメージを表せる。`damageReduce`（防御側）には非供給（対象＝防御側自身のため）。

#### 課題・申し送り

- `@target.style/works` は 1/0（真偽相当）。対象の「アイテムの値」参照（`@target.item.<キー>`）は未供給（YAGNI・必要時に追加）。
- 攻撃者を参照する `damageReduce` 側の対称拡張は未対応。


### 用途自身の修正値欄・適用効果の供給元拡張（フェーズ12 補）

**日付**: 2026-07-10
**関連実装**: `usage.mjs`（`checkBonusSelf`/`damageBonusSelf`・`effects` に `itemId`）、`tnx-formula.mjs`（`evaluateSelfBonus`・`evaluateBonusRows` の bearer）、`tnx-check-flow`/`attack-flow`/`damage-flow`/`tnx-character-sheet-base`（`sourceItemId` 配線・`@item.self`）、`tnx-usage-sheet.mjs`＋`usage-sheet.hbs`（専用欄・グループ化した効果選択）、`tnx-item-sheet.mjs`（供給元アイテム上で有効化）。正本 [Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md)・[Formula_Reference.md](../01_Wiki/Game_Mechanics/Formula_Reference.md)
**ステータス**: ✅ 実装完了（本体 `705a6b4`・全テスト green 1160・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-10 言語化）

- 用途の修正値には「**その用途の親アイテムが持つ修正値**」を入れる**専用の欄**が要る（供給元プルダウン付きの追加行とは別に、最初から常設。ユーザー明言＝「供給元グループに足すのでなく別途配置」）。判定用とダメージ用は別軸なので分ける。
- 用途が対象に与える効果は、従来の親アイテムだけでなく**組み合わせた技能・攻撃に使用した武器**からも追加できるべき（そうでないと再現できない能力がある）。

#### 監査結果

✅ `checkBonusSelf`/`damageBonusSelf`（専用欄・`@item.self`＝親アイテムを式で参照可）を新設し、判定/ダメージの実行時に評価して**親アイテム名で帰属**。適用効果は親＋参加技能＋使用武器から選べるよう供給元別にグループ化し、`{itemId, effectId}` で供給元を明示。

#### 課題・申し送り

- 適用効果の**対象への実付与はフェーズ13**（現状は選択・保持まで。アイテムシートの用途使用は各効果を供給元アイテム上で有効化）。
- 専用欄・追加行とも用途の式では `@item.self`＝用途の親アイテム（AE 値の `@item.self`＝効果が乗るアイテムと同義だが、文脈で解決先が違う点に留意）。


### 対決の受動有利修正（同値はリアクション側勝利・フェーズ12 補）

**日付**: 2026-07-10
**関連実装**: `scripts/module/attack-flow-logic.mjs`（`resolveOpposed`）。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「差分値」③対決
**ステータス**: ✅ 修正完了（全テスト green 1161・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-10 ユーザー指摘）

- **トーキョーN◎VA の対決判定は受動有利**：能動（攻撃）側は受動（リアクション）側を**上回れば**勝利、**同値は受動側の勝利**。従来 `resolveOpposed` は相手の達成値を「目標値」として扱い `達成値 ≥ 相手` で命中していた（同値で攻撃側勝利）＝**バグ**（ユーザー診断「『目標値』というワードが悪さをしている」どおり）。

#### 監査結果

✅ `hit = margin > 0`（`>=` → `>`）に修正。同値は攻撃側敗北＝攻撃回避。①固定値・②制御値は**目標値モードのまま** `達成値 ≥ 目標値`（`resolveNoReaction` は据え置き）。

#### 課題・申し送り

- **要確認**: 「リアクションしない」＝②対象の制御値受け（`resolveNoReaction`・`≥`）も受動有利（`達成値 > 制御値`）にすべきか。現状は目標値モードとして `≥` のまま（ユーザー指摘は対決＝リアクション同値に限定）。


### ヴィークル搭乗中のドッジ＝操縦上書き（回避ブロック・フェーズ12 補）

**日付**: 2026-07-10
**関連実装**: `scripts/module/attack-flow.mjs`（`startReaction` の dodge 分岐・`promptVehicleDodgeFallback`）。正本 [Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md)「ヴィークルの操縦」・[Outfits.md](../01_Wiki/Game_Rules/Outfits.md)・[Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「ドッジ」
**ステータス**: ✅ 実装完了（本体 `340ac6e`・全テスト green 1161・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-10 ユーザー確定）

- 搭乗中（＝準備済みヴィークル。部位「操縦」は1枠のため常に1つ・複数考慮不要）はドッジ＝対応する〈操縦〉**のみ**、〈回避〉・他技能はブロック。
- 対応する操縦技能を**持たなければドッジ不可**＝パリー／制御値受けをダイアログで選ばせフォールバック（**回避へは戻さない**）。パリー/精神・社会リアクションは対象外。
- 従来はヒント文だけで**未実装**だった（wiki の「自由選択で対応済み・ハードブロックなし」は誤りで、本実装＋記録訂正で解消）。

#### 監査結果

✅ `startReaction` の dodge 分岐で準備済み＋`operateSkillKey` のヴィークルを検出→`resolveOperateSkill` で操縦技能に候補を差し替え（回避ブロック）。操縦技能なしは `promptVehicleDodgeFallback`（DialogV2）でパリー/制御値受けへルーティング。

#### 課題・申し送り

- 制御値受けフォールバックは `handleNoReaction` 経由のため、対象の操作者/GM でなければ弾かれる（既存の「リアクションしない」権限。操縦者＝対象本人なら通る）。


### 用途の適用効果を解決時にターゲットへ付与（フェーズ12 補）

**日付**: 2026-07-10
**関連実装**: `scripts/module/usage-effects.mjs`（新）・`tnx-socket-handler`（`usageEffectApplied`）・`tnx.mjs`（描画フック）・`tnx-check-flow`/`attack-flow`（ctx/ペイロード→カードフラグ `usageEffects`）・`tnx-character-sheet-base`/`tnx-item-sheet`（使用時の捕捉）・`css/tnx2.css`。正本 [Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md)・[Combat_Flow.md](../01_Wiki/Game_Rules/Combat_Flow.md)
**ステータス**: ✅ 実装完了（本体 `e6b92e2`・全テスト green 1165・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-10 ユーザー確定）

- **用途を解決した際、ターゲットしたキャラクターへ AE を付与**（攻撃に限らず全用途・自己バフも表現可）。
- ターゲットは**居るか居ないかだけで判断**（数・種別は問わない）。原則ターゲット必須で、居なければ**確認ダイアログ**（続行＝自分対象・キャンセル＝中止）。誤って自分をターゲット中の攻撃等は**ブロックしない**（過度な自動化は進行を妨げる＝ユーザー明言）。
- 付与は**対象所有者/RL がボタン**で実行（自動付与しない＝命中/成功や誤爆の判断は卓に委ねる）。

#### 監査結果

✅ 判定/攻撃/用途使用の結果カードに「効果を適用」ボタン。対象所有者/GM が押すと供給元 AE を対象へ複製生成。ノーターゲットは確認ダイアログで自分対象/中止。

#### 課題・申し送り

- 命中/成功の条件付けは自動化せず卓判断（ボタンをいつ押すか）。非攻撃の判定用途も `game.user.targets` で対象を取る＝射程/対象記述の検証はしない（卓運用）。
- item-sheet の旧「供給元アイテム上で効果を有効化」（フェーズ13 待ちの暫定）は本モデル（対象へ付与）に置換。

#### 実機レビュー修正（2026-07-11・本体 `9b54446`）

- **効果適用はフローの一番最後（ユーザー確定）**: 攻撃では**ダメージが算出されてから**＝ダメージカードへボタンを引き継ぐ（`finalizeDamageRoll` がペイロードをコピー・攻撃カード側は damageRolled で非表示）。攻撃カード側は**命中確定後（hit）かつダメージ未算出のときのみ**表示（ダメージが発生しない攻撃向けの「命中の後でも良い」の器。pending/miss は非表示）。
- **見出しの視認性**: 用途使用カードの見出しが素の `<p>`（背景なし）で見えづらかった → `cr-head`（暗背景）の既存カード様式に載せ替え。

#### 表示統合（2026-08-30）

- 宣言の使用カードは解説カード（`postDescriptionCard`）へ統合（効果文=解説の提示・効果なし宣言もカード化＝旧ローカル通知の廃止）。ルール解釈は不変。詳細は [Design_Review_Entries.md](Design_Review_Entries.md)「宣言の使用カードを解説カードへ統合」と [Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md)。

#### 効果セクション再設計（2026-08-30・ユーザー裁定）

- **適用の仕方をトレイ形式へ**（D&D 5e に倣う・詳細は Design_Review_Entries「効果セクション（トレイ）再設計」）: ボタンは適用後も消えず、セクションが畳まれるだけ（再展開で何度でも・適用済み状態フラグと成功報告文言は全廃）。複数対象の除外ダイアログはカード常設のチェックボックス行に置換。
- **ルール裁定の変更**: 付与先「自分」の**解決時即時自動付与（2026-07-13 の唯一の自動付与例外）を廃止**——代償効果は「効果を適用」押下でメイン効果の対象適用と**同時に使用者へ**付与される（代償のためだけにボタンを出すこと自体が誤り＝ユーザー裁定）。これで「自動付与はどこにも無い」が例外なしになった。
- **命名不良の是正**: 「付与先: 対象/自分」→「効果種別: 通常効果/代償効果」。宛先のふりをした挙動スイッチという v3 当初（2026-07-13）からの命名不良で、対象=自身の自己バフでの誤設定（2026-08-29 実運用で顕在化）を招いていた。保存フラグ `grantTarget` は無移行で維持。
- **宣言の対象解決欠陥を是正**: 宣言用途も `resolveUsageTargetRefs`（決定表・自身/単体の自動セルフ）を通す。従来は素通り＝対象=自身の宣言がノーターゲットで「対象なし」になっていた。


### 用途の「無視する指定技能」（指定技能を無視して単体参加・フェーズ12 補）

**日付**: 2026-07-10（同日中に2度の再設計＝下記経緯）
**関連実装**: `scripts/data/item/common/usage.mjs`（用途フィールド `ignoreComboSkills`＝辞典識別キーの配列）、`scripts/module/skill-chain-resolution.mjs`（`resolveUsageSkills` 第4引数 `ignoreComboKeys`）、`scripts/module/tnx-usage-sheet.mjs`（設定 UI・追加判定・auto-fill 反映）、`templates/app/usage-sheet-combo.hbs`。正本 [Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md)「組み合わせ解決」
**ステータス**: ✅ 実装完了（本体 `c79e55f`→再設計 `c4611e3`・全テスト green 1169・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-10 ユーザー確定・複数往復で確定）

- **技能一致でのブロックは存在しない。** 実在する組み合わせ不可は「**指定技能がアクション技能同士**」だけ（既存）。
- 表現したい効果（〈技能AⅡ〉系）: 「技能：〈心理〉で射撃攻撃を行う〈技能A〉」に、本来は組めない**技能：〈射撃〉のスタイル技能を、〈射撃〉を連れてこずに**組み込めるようにする。既存の「指定技能の自動追加」が働くと〈射撃〉まで入る（そして親の指定技能がアクションなら「アクション同士」で追加自体が弾かれる）ため、**用途側の設定で、設定した技能に限り指定技能の自動追加をスキップ**する。
- 乗る恩恵は組み込んだ技能の「**組み合わせた攻撃への**」効果のみ（〈射撃〉自体は判定に入らない＝〈射撃〉へのボーナスは乗らない）。

#### 監査結果

✅ 用途に `ignoreComboSkills`（識別キー配列）。設定した技能を指定「技能」とするスタイル技能は、seed 連鎖の必須クロージャから当該指定技能を除外＝単体参加・アクション同士ブロックも発生しない。追加判定（`_addComboCheck`）・auto-fill も同設定を反映。設定解除はアクション重複が生じる場合拒否。

#### 課題・申し送り（誤設計の経緯＝再発防止）

1. 「技能一致ブロック＋例外」「アクション同士ブロック無視」→ いずれも誤り（前者は存在しないルール／後者は自動追加で〈射撃〉が入る）で破棄。
2. **per-row トグル（`skillRefs.ignoreComboSkill`・`c79e55f`）→ 破綻**: 親の指定技能がアクションの場合、**追加自体が「アクション同士」で弾かれ、トグルに到達できない**（ユーザー指摘）。「combo に入った後」だけ作り「入れられるか」の経路を検証しなかったのが原因。
3. **追加時の確認ダイアログ → 却下**: 組み合わせのたびに毎回聞く UI は誤り（ユーザー裁定）。**正: 用途側の永続設定**（`c4611e3`）。
- 表示絞り込み（タイミング/単独/ドッジ・パリー）は今回**見送り**（ユーザー判断「表示のブロックも無くす」）。単独フラグも今回は含まない。


### 報酬点の使用可否のルール訂正（能力値判定は不可・組み合わせ技能でも可・フェーズ12 補）

**日付**: 2026-07-10
**関連実装**: `scripts/module/tnx-check-engine.mjs`（純関数 `comboUsesBounty`）、`tnx-character-sheet-base`（能力値判定・通常判定）、`tnx-rl-request-app`・`attack-flow`（攻撃/リアクション）・`npc-acquisition`・`treatment-flow`・`vehicle-move`。正本 [Bounty_Points.md](../01_Wiki/Game_Rules/Bounty_Points.md)「判定フローとの連携」
**ステータス**: ✅ 修正完了（本体 `47fe6ba`＋`592214f`・全テスト green 1171・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-10 ユーザー確定・2点）

1. **能力値判定では報酬点を消費できない。** 従前の実装・記録は「能力判定への報酬点適用は現状許容」としていたが誤りとして訂正。
2. **報酬点使用可能技能（`usesBounty`）を組み合わせていれば、ベース技能でなくても報酬点を使用できる。** 従前実装は全経路で `baseSkill.system.usesBounty` のみを見ており誤り。※Code はこの訂正の際、いったん「usesBounty 技能が**ベース**のとき」と wiki に書いた——**実装挙動をユーザー確認なしにルール文として転写した**§4.2 違反（ユーザー指摘で即訂正）。

#### 監査結果

✅ ①能力値判定の起動2経路（シート・判定要求アプリ）の `bountyAvailable`＝0。②技能判定は純関数 `comboUsesBounty(skillSystems)`（参加技能の some）に全経路（通常判定/攻撃/リアクション/NPC取得/治療/操縦移動/判定要求）を統一。制御判定＝0 は従来どおり。

#### 課題・申し送り

- シナリオシートの情報収集判定（`_onStartInfoSkillCheck`）は技能名テキストのみで技能アイテムを解決しないため、`usesBounty` を確認できず**無条件で報酬点を渡している**（残課題として記録）。
- 判定要求アプリの技能判定は単独技能（コンボなし）のため実質単技能判定（`comboUsesBounty([choice.item.system])` で統一のみ）。


### 再判定システム（カードを出し直して判定値を再決定・フェーズ12 補）

**日付**: 2026-07-11
**関連実装**: `scripts/module/tnx-check-flow.mjs`（`_buildRecheckContext`/`startRecheck`/`renderRecheckButton`）、`attack-flow.mjs`（`postAttackCard` の recheckCtx 保存）、`tnx.mjs`（描画フック）、`check-result.hbs`/`attack-card.hbs`（再判定バッジ）、`css/tnx2.css`。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「再判定」
**ステータス**: ✅ 実装完了（本体 `d409992`＋`fbfbb75`・全テスト green 1171・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-11 ユーザー確定・訂正1回）

- **再判定＝カードを出し直して判定値を再決定**する機構。**再判定は特定技能の能力で可能になる**もので、**用途の「再判定可能」（`allowRecheck`）設定でアンロック**する（ON の用途からの判定でのみボタンが出る）。
- **訂正の経緯**: Code は当初「用途を経ない判定（能力値判定・素の技能判定）があるから汎用ボタン」と判断して汎用形で実装（`d409992`）したが、**「素の技能判定は用途を経ない」は誤認**——一般技能は作成時に check 用途が自動挿入され、**技能判定は必ず用途を経由する**（自分が実装したシステムの動作を検証せず反例をでっち上げた）。ユーザー指摘で用途設定ゲートへ修正（`29ead8f`）。
- 使用回数は再消費しない（元判定で消費済み）。報酬点は可否のみ引き継ぎ・量は再判定時点の所持から取り直す。出し直したカードにもボタンは残る（回数の裁定は卓）。

#### 監査結果

✅ 用途の `allowRecheck` が ON の判定でのみ、結果カード/攻撃カードにフラグ `checkRecheck`（再実行スナップショット）を保存し「再判定」ボタンを描画。押下で同構成の判定フローを開き直し、新結果カード（「再判定」バッジ）を投稿。元カードは残す（採用の裁定は卓）。

#### 課題・申し送り

- **対象外**: リアクション/NPC取得/治療/移動/controlNegate は完了時の継続処理（対決解決・配置・状態除去）の巻き戻しが要るため当面ボタンを出さない（必要になれば per-flow のリセット設計を別途）。
- 攻撃の再判定は**新しい攻撃カード**を出す（旧攻撃カードは pending のまま残る＝RL が手動で無視/整理する運用）。
- usageEffects は再判定後のカードにも引き継がれる（同一使用の付け直し。二重適用の防止は卓運用）。


### 再判定の付与（grantRecheck・達成値クリック・フェーズ12 補）

**日付**: 2026-07-11
**関連実装**: `usage.mjs`（`grantRecheck`）、`tnx-check-flow.mjs`（`_buildRecheckContext` の常時保存化・`startRecheckGrant`/`_onGrantAchievementClick`・`startRecheck` の `mergeSkill`/`consumeUses`・`renderRecheckButton` の達成値バインド）、`tnx-character-sheet-base`（起動分岐）、`usage-sheet.hbs`＋`tnx-usage-sheet`（チェックボックス・保存）、`css/tnx2.css`。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「再判定の付与」
**ステータス**: ✅ 実装完了（本体 `4c9ade1`・全テスト green 1173・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-11 ユーザー確定）

- 「判定を失敗した時にその技能を組み合わせて同じ判定をやり直す（山札判定のみ）」系＝**事後的に再判定を与える能力**が存在する。表現は**ユーザー案の UX**——付与用途の使用で判定は行わず、**結果カードの達成値をクリック可能にし、クリックでその判定に起動技能を組み合わせた再判定を起動**する。
- **発動条件（失敗時のみ・山札のみ等）は自動強制しない**（Code は当初「山札のみ」の機械的ブロックを実装したが、**「例示に過ぎず、そこまで自動化する必要は無い」とユーザー裁定で撤去**）。
- 前提変更: 再判定スナップショットを**常時保存**に変更（従前は allowRecheck ON のみ保存＝事後付与が原理的に不可能だった）。ボタン表示だけを `allowRecheck` でゲート。

#### 監査結果

✅ grantRecheck 用途の使用→達成値クリック待ちモード（再使用でキャンセル・破線ハイライト）→クリックで skillIds に起動技能を追加（スート積再計算・共通スート無しは中止・技能の判定バフも自然適用）した再判定を起動。付与用途の消費は再判定実行時に適用。出し直しカードのボタン有無は元の allowRecheck を引き継ぐ。

#### 課題・申し送り

- 付与モードはクライアントローカル（自分のクライアントで使用→自分でクリック）。他プレイヤーの判定への付与はそのカードの達成値を**付与者が**クリックした場合、再判定の実行権限（対象アクターの所有者/RL）で弾かれる——他者への付与はRLが代行するか、将来ソケット化（申し送り）。
- 起動技能が対象判定のアクターの所持でない場合、skillIds に加わっても解決されない（スート積は付与者の技能で計算）——他者付与と同根の制限。


### 判定の事後修正（modifyCheck・達成値クリック・フェーズ12 補）

**日付**: 2026-07-11
**関連実装**: `usage.mjs`（`modifyCheck`）、`tnx-check-flow.mjs`（クリック待ち機構の汎用化 `startAchievementAction(kind)`・`_applyCheckModify`・`checkMods` ライブ描画）、`tnx-socket-handler.mjs`（`checkModify` 委譲）、`tnx-character-sheet-base`（起動分岐統合）、`usage-sheet.hbs`＋`tnx-usage-sheet`（チェックボックス・保存）。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「判定の事後修正」
**ステータス**: ✅ 実装完了（本体 `dd15abe`・全テスト green 1174・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-11 ユーザー確定）

- **事後的に判定へボーナス/ペナルティを与える技能**が存在し、導線は再判定の付与と同じ「技能をクリック→達成値をクリック」（ユーザー案＝達成値クリックの流用）。
- **再判定との共存**: 再判定→事後修正は可能。**事後修正→再判定は修正がリセット**（再判定は元の構成から再実行するため）——スナップショットに事後修正を含めないことで自然に成立。
- 修正値＝用途の「この用途の判定修正値」（式）を使用者のデータで評価・空なら手入力（Code 設計・ユーザー黙認）。ダメージカード算出後の攻撃は修正不可（唯一の強制・設計時に提示済み）。

#### 監査結果

✅ modifyCheck 用途の使用→クリック待ちモード（recheck と排他）→達成値クリックで式評価（空は手入力）→フラグ達成値更新＋目標値つきは成否/差分再計算＋解決済み攻撃は対決再解決（受動有利の `resolveOpposed` 再利用）＋要求カード追随。内訳は `checkMods` フラグからライブ描画（元の内訳は残す）。非作者の修正は GM へソケット委譲。

#### 課題・申し送り

- 攻撃（解決済み）の再解決はリアクション達成値/制御値の**保存値**に対して行う（リアクション側の巻き戻しはしない）。
- 修正のリセット（事後修正→再判定）はスナップショット非包含による自然な帰結＝checkMods はカード表示専用で再判定に影響しない。


### スタン/説得の確定直前選択への再設計（フェーズ12 補）

**日付**: 2026-07-11
**関連実装**: `usage.mjs`（`canStun`）、`usage-sheet.hbs`＋`tnx-usage-sheet`（「スタン可能（精神攻撃では説得）」トグル・保存）、`attack-flow`（ペイロード）、`damage-flow`（ロールダイアログのチェック撤去・軽減確定後の選択ダイアログ・適用時に `stun` をフラグへ）。正本 [Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「スタン攻撃」「説得」
**ステータス**: ✅ 実装完了（本体 `856e31b`・全テスト green 1176・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-11 ユーザー確定）

- **スタン/説得の処理はダメージ算出の最後**＝**ダメージが確定する直前に別ダイアログで適用可否を選ぶ**（旧実装＝攻撃側のダメージ算出ダイアログのチェックボックスは廃止）。
- **用途の「スタン可能（精神攻撃なら説得可能）」（`canStun`）でゲート**＝OFF ならスタン/説得のダイアログ自体を出さない。
- 肉体=スタン・精神=説得（ラベル切替）・社会は対象外。適用＝最終ダメージ 10 以上を 10 とみなす（`computeDamage` の stun 意味論は不変＝決定時点だけ移動）。

#### 監査結果

✅ 軽減ダイアログ確定→（canStun のとき）「スタン/説得を適用しますか？」→computeDamage(stun)→チャート適用、の順に変更。軽減ダイアログのヘッダに「スタン可（確定前に選択）」を表示・プレビューはスタン未適用値。適用時に `stun` をフラグへ書き台帳表示（「攻撃側合計（スタン／説得）」「10 以上→10」行）は従来どおり。

#### 課題・申し送り

- 選択ダイアログは**ダメージ適用者（対象所有者/RL）のクライアント**に出る（スタンの宣言者=攻撃側との調整は卓・信頼ベース）。
- 軽減プレビューはスタン適用前の値（選択が後段のため）。


### ダメージ系タイミングの表現確定＋用途タイプ「ダメージ増加/軽減」の廃止（フェーズ12 補）

**日付**: 2026-07-11
**関連実装**: `usage.mjs`（型域・`formula` フィールド削除・migrateData で旧タイプ→declaration）、`damage-flow.mjs`（boost/reduce 機構の全撤去＝候補収集・選択・消費・台帳行・`collectDamageUsages`/`usageEffectDisplay`/`planFromRows`）、`tnx-usage-sheet`＋`usage-sheet.hbs`（USAGE_TYPES・効果量 fieldset 削除）、ダイアログ2種。正本 [Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)・[Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「用途タイプ」
**ステータス**: ✅ 実装完了（本体 `a85d093`・全テスト green 1177・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（2026-07-11 ユーザー確定）

- **ダメージ系の5タイミングの表現**: 「算出の直前/直後」「適用の直前/直後」は**間のタイミング**＝ボタンを押す前後に普通に技能を使えばよく機構不要。**「タイミング：ダメージ算出」（カードプレイと同時）は「ダイアログで事前選択→確定トリガー（カードプレイ）でアトミックに発効」で表現**する（D&D のロールダイアログと同じ慣行）。
- **「タイミング：ダメージ算出」の技能はダメージ増加のみ**存在する。その表現は**組み合わせ＋攻撃用途のダメージ修正行**（供給元＝その技能）で足りるため、**用途タイプ `damageBoost`（ダメージ増加）/`damageReduce`（ダメージ軽減）は廃止**。軽減は軽減ダイアログの**手動欄**に入力する。

#### 監査結果

✅ 両タイプと関連機構（算出ダイアログの増加選択・軽減ダイアログの軽減選択・消費・台帳行・`usage.formula` フィールド）を全撤去。既存データは migrateData で **declaration（宣言）へ変換**（名前・解説は残る＝効果はテキストとして保持）。`damageCategory` は攻撃系統として存続。

#### 課題・申し送り

- 旧 damageBoost/damageReduce を設定済みのアイテム（辞典・ワールド）は宣言化される——数値効果を機械適用したい場合は、攻撃用途のダメージ修正行（増加）/軽減の手動欄（軽減）へ移し替える（卓のデータ整備）。


### ダメージ増加技能の再設計（boostDamage フラグ・アイテムロール使用・フェーズ12 補）

> **🔁 置換（2026-07-11）**: `boostDamage` は「**ダメージを修正（`modifyDamage`）**」へ置換（ユーザー提案）。本エントリの待ち受け登録方式は廃止——下記「ダメージ修正技能への一本化」エントリが現行。

**日付**: 2026-07-11
**関連実装**: `usage.mjs`（`boostDamage` フラグ）、`usage-sheet.hbs`＋`tnx-usage-sheet`（〈攻撃〉の isAttack 横トグル・排他・ダメージ増加（式）＝`damageBonusSelf` 流用）、`damage-flow.mjs`（`useDamageBoost`＝pending 登録/トグル・カードプレイ時に評価/合算/消費・台帳「ダメージ増加（技能名）」・`boosts` フラグ復帰）、`tnx-character-sheet-base`/`tnx-item-sheet`（アイテムロール経路）。正本 [Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「ダメージ増加技能」
**ステータス**: ✅ 実装完了（本体 `56c2347`・全テスト green 1178・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈・設計指針（2026-07-11 ユーザー確定）

- **一般原則: アイテム/技能/神業の使用は必ずアイテムロール（技能クリック）から。「ダイアログ内の選択肢から使用」は今後一切設計案にも出さない**（旧 damageBoost のダイアログ選択方式を却下して確立）。
- **用途タイプ「ダメージ増加/軽減」の廃止は維持**。ダメージ増加は**用途フラグ `boostDamage`** で表す（Code が一度タイプ復活を勝手に前提して着手し破棄・叱責——「再設計」は廃止裁定の取り消しではない）。
- **配置はダメージに関わるため〈攻撃〉セクション**: 「この判定を攻撃に使う」の**横**にトグル（排他）・式入力も〈攻撃〉内（ユーザー指定）。
- 使用フロー: ダメージカード待ち受け中に技能クリック→算出へ登録（再クリックで解除）→**カードプレイと同時に発効**（タイミング：ダメージ算出の同時性）。待ち受け外は警告・不成立。消費はクリック時プロンプト→確定時適用。

#### 監査結果

✅ 上記フローどおり実装。効果量は `damageBonusSelf`（式・`@card`/`@item.self`/`@target.*` 等）で、評価不能な式は警告して手動修正へ誘導（消費は適用＝使用は成立）。台帳・攻撃側合計に「ダメージ増加（技能名）」行として反映。

#### 課題・申し送り

- 増加できるのは**攻撃者自身の技能のみ**（pending はクライアントローカル・他者の算出への増加は将来ソケット化とあわせて検討）。
- 待ち受け中の登録状態は算出ダイアログに表示されない（通知＋確定時の台帳で提示）。ダイアログのライブ表示は要望があれば追加。


### ダメージ修正技能への一本化（boostDamage → modifyDamage・クリック適用方式・フェーズ12 補）

**日付**: 2026-07-11
**関連実装**: `usage.mjs`（`modifyDamage` フラグ・migrateData で `boostDamage`→`modifyDamage` 変換）、`usage-sheet.hbs`＋`tnx-usage-sheet`（攻撃モードラジオ「ダメージを修正」・ダメージ修正（式）＝`damageBonusSelf` 流用）、`tnx-check-flow.mjs`（クリック待ち kind="modifyDamage" 追加・`peekAchievementAction` 新設・達成値クリック側は modifyDamage を無視）、`damage-flow.mjs`（`handleDamageModifyClick`＝攻撃側合計クリックで適用・フラグ `mods`・台帳「事後修正」行。旧 `useDamageBoost`/`pendingBoosts`/`boosts` 全撤去）、`tnx-character-sheet-base`/`tnx-item-sheet`（アイテムロール経路をクリック待ち系へ統合）。正本 [Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「ダメージ修正技能」
**ステータス**: ✅ 実装完了（本体 `d2b13a8`・全テスト green 1185・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査）

- **発端**: ダメージ軽減技能のため「ダメージに対してアクセスして修正を適用する機能」が必要（ユーザー要望）。クリックのフローは判定の事後修正（達成値クリック）と同じで、**使用タイミングはダメージ算出後〜適用前**。
- **「ダメージを増加」を「ダメージを修正」に置き換える**（ユーザー提案を Code が承認）: 増加=正・軽減=負の一本の機構で足りる（UI 上も修正欄=`damageBonusSelf` の流用は同じ）。「タイミング：ダメージ算出」の増加技能は**カードが出た直後にクリック**することで表す＝旧方式の「待ち受け中に登録→カードプレイと同時に発効」というアトミック性は、実運用上クリック順序の差でしかなく専用機構を要さない。
- 適用済みダメージの修正は不可（警告）。発動条件（軽減は防御側のみ等）の強制はしない（卓裁定＝技能を強制しない方針）。

#### 設計レビュー

- **クリック待ち機構の共用**: `TnxCheckFlow._clickState` の kind に "modifyDamage" を追加（recheck/modify と排他・同用途の再使用でキャンセル・body class によるハイライトも共通）。damage-flow からは `peekAchievementAction("modifyDamage")` で覗く（check-flow は damage-flow を import しない＝一方向依存で循環なし）。
- **適用はフラグ `mods` に積む**（`{label, value}` 行）: `applyDamagePatch` 経由のため非作者（防御側）からの適用も既存の damageUpdate ソケット委譲に乗る——旧 pendingBoosts（クライアントローカル）で不可能だった**他者のダメージへの軽減**が構造的に成立。台帳「事後修正（技能名）」行＋攻撃側合計へ合算（ライブ描画）。
- 式が空/評価不能/0 のときは `AmountInputDialog` で手入力（判定の事後修正と同型・min −99）。
- 旧データは migrateData で `boostDamage`→`modifyDamage`（modifyDamage 既設定なら上書きしない）。

#### 課題・申し送り

- 攻撃側合計クリックの装飾は達成値クリックと同じ `tnx-recheck-target`（モード中ハイライト）。モード外のクリックは無視（通常表示）。
- スタン/説得（適用時選択）とは独立（modifyDamage は算出側の合計に効き、スタンの 10 頭打ちは適用段）。


### 宣言用途の判定/ダメージ修正（バフ宣言・フェーズ12 補）

**日付**: 2026-07-12
**関連実装**: `tnx-usage-sheet.mjs`＋`usage-sheet.hbs`（宣言ビューの〈判定〉〈ダメージ〉fieldset・チェックボックス独立・submit の declaration ブランチ＝OFF 側の式クリア/フラグ変更で再描画）、`tnx-character-sheet-base.mjs`（技能クリックの実行対象に修正フラグ付き宣言を追加・クリック待ち分岐を check/declaration に拡張）、`tnx-item-sheet.mjs`（クリック待ち系分岐をフラグゲートに統一・攻撃より先＝アクターシートと同順）、`usage.mjs`（コメントのみ・スキーマ変更なし）。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「判定の事後修正」・[Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「ダメージ修正技能」
**ステータス**: ✅ 実装完了（本体 `d7e4876`・全テスト green 1185・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査）

- **要望（2026-07-12 ユーザー）**: 宣言用途にもバフ用の判定/ダメージ修正値欄が欲しい。タイミングは①判定直前→直後の判定 ②判定直後→その達成値（クリック） ③ダメージ算出直前 ④算出時 ⑤算出直後 の5種。
- **5種は既存2機構に還元**（Code 提案・ユーザー承認）: 達成値/ダメージへの加算は線形で、クリック待ちモードは判定/ダメージのフロー実行を**跨いで持続する**ため、①③=「先に使用（消費先払い）→フローを実行→出た結果をクリック」で忠実に再現できる。①②=判定を修正（達成値クリック）・③④⑤=ダメージを修正（攻撃側合計クリック）。旧 pendingBoosts 型の事前登録機構は再導入しない。
- **AE との棲み分け**: 単発（その1回だけ）=クリック適用（式も使える）・持続（シーン中・条件つき等）=既存の適用効果（AE・`check.*`/`damage.dealt.*`/`damage.vs*`）。「次の1回だけ効く AE」は作らない。
- **再判定の付与は対象外**（要望の範囲外・必要になったら同形で追加可）。→ **2026-07-13 に同形で追加**（後述「宣言用途の『再判定を付与』」エントリ＝素の再判定権のみの事後付与）。

#### 設計レビュー

- **スキーマ変更なし**: 用途スキーマは全タイプ共通のため `modifyCheck`/`modifyDamage`/`checkBonusSelf`/`damageBonusSelf` は宣言エントリにも既存。フロー側（`_applyCheckModify`/`handleDamageModifyClick`）も usageId で用途を引くだけでタイプ非依存＝変更なし。
- **UI はチェックボックス2つを独立**（ユーザー裁定）: Code の当初案①単一 fieldset＋3択ラジオ→「セクションごと分けて判定用途とレイアウトを揃える・ラジオ排他は意味が無い」で修正。次案②両方 ON の kind 集合化（1回の使用で両方待ち受け）→「両方オンにする機会は無い」で不採用＝**両方 ON は決定的な優先順（判定を修正が先）で振るだけの最小形**（YAGNI）。
- 副次修正: アイテムシートの使用ボタンで check 用途の再判定を付与/判定を修正がクリック待ちに乗らず適用効果経路へ落ちていた既存の取りこぼしを、フラグゲート統一で解消（アクターシートと同挙動に）。

#### 課題・申し送り

- 両方 ON の用途は「判定を修正」しか発動しない（ダメージ側に振りたければ用途を分ける）。実用途が現れたら再訪。
- 修正フラグ無しの宣言（神業等）のアクターシート技能クリックは従来どおり解説カード＝挙動不変。→ **2026-07-19 に撤回**（宣言用途を候補から除外しない＝ユーザー指示）。


### AR 消費の消費先設定への一本化（actionRank 種別・パリー自動化廃止・フェーズ12 補）

**日付**: 2026-07-12
**関連実装**: `usage-consumption.mjs`（種別 `actionRank`＝kind "ar"・`resolveConsumeRows` に actionRank ctx・`applyConsumptionPlan` にアクター更新・ダイアログ文言「AR を消費×N」）、`tnx-usage-sheet.mjs`＋`usage-sheet.hbs`（消費種別に「AR（アクションランク）」追加・対象アイテム選択なし=noTarget・種別変更で再描画）、`attack-flow.mjs`（startReaction のパリー AR>0 検証＋AR−1 即時適用を撤去）。正本 [Combat_Flow.md](../01_Wiki/Game_Rules/Combat_Flow.md)「アクションランク」・[Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「パリー」
**ステータス**: ✅ 実装完了（本体 `545ec86`＋カット進行外の訂正 `785b321`・全テスト green 1190・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査）

- **要望（2026-07-12 ユーザー）**: 用途に「AR を消費する」項目を追加し、パリーの AR 消費自動化をオミット。**今後は「AR を消費」と設定された用途からの判定・宣言に一本化**。
- パリーの「メインプロセス終了時に AR −1」は消費適用時（判定実行時）に前倒しされる点は従来の自動化と同じ（厳密化はフェーズ13 のプロセス管理）。
- **AR 0 のパリー不可のハード強制は撤廃**: 消費ダイアログの原則ブロック（チェック済みで不足なら実行不可・チェックを外せば消費せず実行）に置換＝技能を強制しない方針・卓裁定に整合。
- ~~カット進行外は AR を追跡しないため no-op~~ → **訂正（2026-07-12 ユーザー・本体 `785b321`）**: カット進行外でも消費要求は生きる——**AR を消費できないため「AR を消費する能力」は原則使用不可**（残量 0 扱いの原則ブロック・チェックを外せば卓裁定で実行可）。Code 当初の no-op は旧パリー自動化の「進行外は検証・消費なし」を誤って一般化したもの。

#### 設計レビュー

- **消費の原則との整合**: 「全ての消費は consumeTargets からのみ発生する」（11-6 確定）に AR も乗せる＝専用フラグでなく**消費先の新種別**。既存パイプライン（行解決→確認ダイアログ→プラン適用・クリック待ち系のプラン持ち回り・キャンセル時の非消費）が全用途タイプ・全経路（判定/攻撃/リアクション/宣言/クリック待ち）でそのまま効き、追加配線が不要。消費量も行の amount で可変（AR 2 消費等も表現可）。
- 行の itemId はセンチネル `"@ar"`（チェックボックス識別用・実アイテム ID と衝突しない）。kind "ar" は applyConsumptionPlan でアクター更新（`actionRank.value` 減算・0 clamp）。分身は本体へ差し替えない（AR は実行アクター自身の戦闘リソース＝使用回数共有の対象外）。
- カット進行判定は導出済みの `actor.system.actionRank.inCombat`（character-base が isActorInStartedCombat から設定）を使用＝追加 import なし。

#### 課題・申し送り

- 既存のパリー技能アイテム（辞典・ワールド）には用途の消費先「AR を消費」を手動で設定する必要がある（データ整備。自動 migration はしない＝どの用途がパリーかをシステムは判別しない）。
- メジャーアクションの AR 1 消費・イニシアチブ減・クリンナップ全回復の進行管理はフェーズ13（本件は能力による消費のみ）。


### スート変更（使用不可スート→使用可能スート・3経路・フェーズ12 補）

**日付**: 2026-07-12
**関連実装**: `tnx-check-flow.mjs`（`_trySuitChange`＝スート不一致の置換・`_chooseSuit` 汎用化・クリック待ち kind="suitChange"・結果表示 suitChangedDisplay・再判定スナップショットに allowSuitChange）、`usage.mjs`（`allowSuitChange`/`grantSuitChange`）、`helpers.mjs`（`check.suitChange` パース scope="suitChange"・`actorHasSuitChangeBuff`）、`character-base.mjs`（値バフ適用から suitChange 除外）、`tnx-usage-sheet`＋`usage-sheet.hbs`（判定モードラジオ「スートを変更」・通常モード「スート変更可能」・宣言チェックボックス）、`attack-flow`/`npc-acquisition`（ctx 伝搬・攻撃カードにもスート変更行）。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「スートの変更」・[Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md)
**ステータス**: ✅ 実装完了（本体 `bddf224`・全テスト green 1192・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査）

- **能力の性質（ユーザー確定）**: 「判定のスートを任意に変更する」能力は、実態として**使用できないスートのカードを使用可能なスートに変更するための能力**——最初にクリックされるのは基本的に使用不可スートのカード。よって発火点は**スート不一致の置換**（使用可能スートのカードは従来どおりそのままプレイ・ダイアログなし）。
- **用語規約（ユーザー確定・従前の口頭確定の記録漏れを本日訂正）**: 無印の「判定」＝能力値判定または技能判定。**制御判定は常に「制御判定」と明示される**ため、明示なき判定系機構（スート変更含む）は制御判定に掛からない。
- **他者バフは AE で表す**（ユーザー方針・この種の能力全般＝判定直前の判定ボーナス等も同様）: `check.suitChange` マーカーキーを付与（用途の適用効果）。**持続時間と AE キーは独立**——「1回の判定」Duration（Times Up 同等の失効タイミング）は時間管理フェーズで持続時間側に追加し、それまで失効は手動。
- 2パターンの対応: ①組み合わせ＝用途の「スート変更可能」（ignoreComboSkills と同じ用途側設定） ②直前使用＝「スートを変更」（スート変更待ち・次の自分の判定）＋他者へは AE 付与。

#### 設計レビュー

- 宣言スートは**カードのスートとして以降の全既存ロジックへ流す**（能力値・判定バフ収集・表示。ジョーカー/切り札の宣言と同族）。事後クリック方式にしない理由: スートは線形加算でなく、プレイ時の制約検証・スート依存バフ収集の分岐点のため、発効点はカードプレイ時。
- 3経路のゲートは `_trySuitChange` に集約（①用途 ctx.allowSuitChange ②AE actorHasSuitChangeBuff ③クリック待ち＝actor 一致時のみ）。**どれも無い判定ではダイアログは出ない**（毎回出る仕様ではない）。待ち受けの消費は適用確定時・非消費の源①②が優先。
- `check.suitChange` は scope="suitChange" として値バフ適用（CHECK_SCOPES）から除外＝実行時参照専用のマーカー。
- 内訳の明示: 結果カード/攻撃カードに「スート変更（元→後）」行（計算内訳の全行表示原則）。

#### 課題・申し送り

- **Code の反省 2 件**: ①他者付与を「1回の判定」Duration が未来だからと丸ごと時間管理フェーズへ先送りしかけた——持続時間と AE キーは独立であり、キーは今から付与できる（ユーザー叱責→即実装）。②当初スート変更を「任意スートへの変更＝有効スート内での選び直し」と誤解（ユーザー訂正で不一致の置換に確定）。
- AE 形の失効は手動（時間管理フェーズで「1回の判定」Duration を追加したら自動失効に載せ替え）。
- ジョーカー/切り札は自前でスートを宣言できるため対象外（宣言が有効スート外なら従来どおり不成立＝失敗する権利）。


### 自動設定の射程「武器」解決＋BS 効果決定カードの一本化（フェーズ12 補）

**日付**: 2026-07-12
**関連実装**: ①`tnx-usage-sheet.mjs`（`resolveWeaponRangeValue`・`deriveUsageAutoFill` の射程解決＝本体 `ea4dbe7`） ②`condition-resolution.mjs`＋`condition-prompt.hbs`＋`tnx.mjs`（`conditionDraw` フラグ・`renderConditionDrawCard`＝本体 `1cf7d59`）。正本 [Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md)「発動パラメータの自動入力」・[Conditions.md](../01_Wiki/Game_Mechanics/Conditions.md)「カード決定型 BS のドロー機構」
**ステータス**: ✅ 実装完了（全テスト green 1192・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査）

- **射程「武器」（ユーザー要望）**: 射程が「武器」のスタイル技能が組み合わせに含まれる場合、使用する武器の射程を参照して**通常どおり射程の優先度に従って**自動設定する。実装の読み: 優先度上の「武器」の位置（至近※に次ぐ）はそのまま——「武器」が優先勝ちした場合に、その具体値を用途の使用武器の実射程へ解決する。複数武器は最短（「※複数は最短」の既定の踏襲＝Code 既定）。
- **BS 効果決定カード（ユーザー指摘）**: ボタンのカードと結果のカードが別々に出るのは他のフロー（判定要求・攻撃カード）と不整合——**ボタンを置き換えて結果を表示する形に一本化**。

#### 設計レビュー

- 射程解決: 武器の射程は {min, max}（max="none"=単一射程=min）＝実効射程は最長を採用。武器未設定・射程なしは「武器」のまま（従来表示）。
- 効果決定カード: `conditionDraw` フラグ＋renderChatMessageHTML のライブ描画（既存の確立パターン）。結果に引いたカードを明示（内訳の全行表示）。非作者のフラグ更新は checkModify（自スコープ限定の汎用パッチ委譲）を流用＝新ソケット不要。旧形式カード（保存済み静的ボタン）は従来経路で後方互換。
- 取り下げ: 「組み合わせ技能の使用回数が消費されない」報告は消費先設定からの消費で解決済み（ユーザー自身が撤回・変更なし）。

#### 課題・申し送り

- 効果決定のドロー主体（衰弱=RL・重圧=受けたキャラ）の強制はしていない（ボタンは誰でも押せるが、状態フラグ書き込みに対象の所有権が要る＝従来どおり）。


### 特殊タグ［支配］＋ダメージタグ改変 AE（フェーズ12 補）

**日付**: 2026-07-12
**関連実装**: `conditions.mjs`（`dominated` 定義・`applyDamageTagMods` 純関数）、`helpers.mjs`（`damage.replaceTag/addTag` パース scope="damageTag"・`gatherDamageTagMods`）、`character-base.mjs`（値バフ適用から除外）、`tnx.mjs`（inflicts カスケードで負傷由来のみ適用）、`treatment-flow.mjs`（上書き由来の支配=目標値20）。正本 [Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「特殊タグ［支配］」・[Conditions.md](../01_Wiki/Game_Mechanics/Conditions.md)・[Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md)
**ステータス**: ✅ 実装完了（本体 `552688c`・全テスト green 1200・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査・ユーザー裁定 verbatim ベース）

- ［支配］＝特殊な精神ダメージの効果タグ。付与は主に「特定のスタイル技能を組み合わせたとき［昏睡］［精神崩壊］の**代わりに**付与」＋「社会ダメージ［抹殺］を受けたとき**追加で**付与」の2形。**支配されたキャラクターは RL 操作＝自動化せずに運用できる範囲**（マーカータグ・type なし）。
- **解除**: 抹殺と同時に付与された支配は抹殺と同時に解除（**タグ追加の場合は追加元のチャートの治療で解除**）。**上書きの場合は［昏睡］と同じ**（＝治療可・目標値 20 と解釈して実装）。
- 実装形はユーザー自身の提案（タグを別タグで上書きする AE＋タグを追加する AE）を採用。追加キーは「チャート項目指定」でなく**元タグ指定**（`damage.addTag.erased`）を Code が推奨しユーザー要望の「抹殺を受けたとき」に一致させた（抹殺は複数チャート項目が与えるため）。

#### 設計レビュー

- 評価点＝inflicts カスケード（負傷作成→戦闘不能/BS 自動付与）の一点。`applyDamageTagMods` は純関数（テスト6件）で、置換は name/img/statuses/conditionKind を差し替え `replacedFrom` を記録、チャート項目由来の条件（controlNegate）は新タグへ引き継ぐ（無効化条件はダメージ結果の性質として持ち越し＝Code 既定・要調整なら見直し）。追加は `addedFrom` を記録し hideFromList。
- 追加・置換分とも既存の woundSource 紐づけの共通処理に乗る＝治療・制御判定無効化の除去が自動で正しく効く。追加由来の支配は治療**目標値には関与しない**（除去のみ同時）。
- 対象側 AE（scope="damageTag"）＝値バフ適用から除外。付与経路は既存の適用効果そのまま＝追加配線なし。
- **適用効果はダメージ適用と同時に自動付与へ変更（同日追補・本体 `2bc3bc2`・ユーザー確定）**: 当初「効果を適用→ダメージ適用」の押し順を要求したが、ユーザーの想定（ダメージ→効果）と逆の順序依存が生じるため、ダメージ適用の確定時に未適用の適用効果を自動で先に付与→チャート適用の順で**同一クリック内に一本化**（設定トグルは作らない＝例外があると手動順は成立しないため）。ダメージカードの手動ボタンは、対象未選択・適用済みで効果だけ未適用（旧カード）の場合のみ残す。判定結果/使用カードの手動ボタンは従来どおり。

#### 課題・申し送り

- 手動付与（チャートを経ない状態付与）にはタグ改変は効かない（負傷由来ゲート）。
- 孤立した支配（手動付与・紐づきなし）の治療は既定の 15 に落ちる（裁定外・運用は卓）。
- 抹殺の負傷を RL が手動削除する運用では、紐づく支配も手動で消す（削除カスケードは無い＝抹殺の適用自体がセッション終了後の手動運用域）。


### 判定全般バフの AE キー check.all（フェーズ12 補）

**日付**: 2026-07-13
**関連実装**: `helpers.mjs`（`check.all`＝scope "anyCheck"・`checkChangeMatches` が技能/能力値判定に合致）、`character-base.mjs`（値バフ適用から除外）。正本 [Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md)
**ステータス**: ✅ 実装完了（本体 `1a04e40`・全テスト green 1202・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### 監査・レビュー

- ユーザー指摘「判定全般へのバフのための AE キーが無い気がする」→ 実際に不在だった（既存 `check.*` は技能識別キー/スタイル/ワークス/能力値の指定が必須で「全判定+N」を1キーで書けない）。`check.all` を追加。
- **無印「判定」＝能力値判定＋技能判定**の用語規約（2026-07-12 記録）に従い制御判定には掛からない。`controlCheck.all`（制御判定全般）は要望が現れたときに追加（YAGNI）。
- `all` は技能識別キーの予約語になる（照合前に anyCheck へ振られるため、識別キー "all" の技能は完全一致キーで参照できない＝実在しない想定）。
- あわせてユーザーから `check.suitChange` の wiki 記録漏れの指摘があったが、こちらは**記録済み**（Active_Effects 判定バフ表・Check_Rules「スートの変更」・Usage_System＝`c821b09`）と回答。


### 回復・治療系スタイル技能（用途の回復設定・回復フロー・フェーズ12 補）

**日付**: 2026-07-13
**関連実装**: `recovery-flow.mjs`（新規＝対象解決・回復対象選択・宣言即除去/判定完了継続 `ctx.recovery`）、`usage.mjs`（recovery/recoveryTargets/recoveryExcludes/recoveryAll/recoveryCount/recoveryTargetFormula）、`conditions.mjs`（純関数 `recoveryKindMatches`/`recoveryKindExcluded`）、`tnx-check-flow.mjs`（完了継続・再判定除外）、`tnx-usage-sheet`＋`usage-sheet.hbs`（〈回復〉fieldset）、導線（sheet-base/item-sheet）。正本 [Conditions.md](../01_Wiki/Game_Mechanics/Conditions.md)「回復・治療系スタイル技能」・[Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「治療」
**ステータス**: ✅ 実装完了（本体 `a0cbe7b`・全テスト green 1207・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査・ユーザー裁定）

- 回復対象は**使用時に対象の受けている状態から選ぶ**。回復数は N 個固定の場合・カテゴリ内すべて（例: BS：捕縛すべて）・BS すべて・ダメージすべて治療、のいずれもある→範囲行×該当すべて/回復数で表現。
- 目標値の参照は**基本的に強度のみ**。ダメージ治療では**ダメージの値**を参照する可能性→`@condition.magnitude`/`@condition.woundValue`。
- **社会ダメージの治療は専用スタイル技能か神業のみ**（〈医療〉不可のため他に経路が無い）→範囲に「負傷（社会）」を設定した用途がその専用技能＝ブロックしない。
- **完全死亡・精神崩壊は「ダメージ全てを回復」系でも治療不可**。当初 Code はハードコードの除去不能として実装しかけたが、ユーザー提案により**除外タグ設定**（「指定したダメージ効果タグを含むもの以外すべて」＝タグ自身＋そのタグを与える負傷を除外）へ変更——抹殺（未確定）も同じ機構で技能ごとに設定でき、システムは強制しない。
- AE キーにしない理由: 回復は能動的な行動（使用→対象選択→除去）でありアイテムロール原則に乗せる。回復判定へのバフは既存 `check.*`/`check.all` で効く。

#### 設計レビュー

- 除去は既存規約を流用: BS＝単体除去（woundSource 非追跡）・~~戦闘不能＝単体除去~~ → **訂正（2026-07-13 ユーザー指摘・本体 `dab71a2`）: 戦闘不能＝元となる負傷ごと治療**（戦闘不能はダメージそのもの＝2026-07-09 裁定に沿う。Code が BS 規約を戦闘不能へ誤って一般化していた。孤立＝手動付与のみ単体除去）・負傷＝治療の除去範囲（紐づき非BS込み・BS 残す）・権限委譲は treatmentApply ソケット（新ソケット不要）。あわせて「回復を行う」OFF で設定リセット（再 ON はまっさらから＝同コミット）。
- check 用途は通常の判定フローへ合流（組み合わせ・判定バフ・スート変更等すべて既存どおり）＋完了継続 `ctx.recovery`（treatment と同型）。継続処理のため再判定スナップショット対象外。
- 目標値なしの回復判定は達成値の報告のみで除去しない（成功が定義できないため適用は卓裁定）。

#### 課題・申し送り

- 対象は1体（複数ターゲット時は先頭）。チーム対象の回復が必要になれば拡張。
- 〈医療〉の標準治療フロー側には完全死亡/精神崩壊の除外を強制していない（回復用途は除外タグで表現・医療の運用は卓）。必要なら同じ純関数で足せる。
- 「1シーン1回」等の回数制限は非強制（行動回数制限は自動化しない既存方針）。


### 目標値の一本化＋射程「武器」ライブ追従＋用途シートのセレクト競合修正（フェーズ12 補）

**日付**: 2026-07-13
**関連実装**: `usage-target-value.mjs`（新規・`resolveUsageTargetValue`）、`tnx-character-sheet-base`（用途判定への目標値配線）、`recovery-flow`/`usage.mjs`（recoveryTargetFormula 廃止・migrate 移送）、`tnx-usage-sheet`＋`usage-sheet.hbs`（解説参照でも自由記入欄・deriveWeaponRangeLive・stopPropagation×5）。本体 `80c2619`。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「用途の目標値の扱い」
**ステータス**: ✅ 実装完了（全テスト green 1213・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### 経緯・監査

- **目標値欄の二重化はユーザーの指摘で撤回**: Code が回復用に専用式欄（recoveryTargetFormula）を新設したことに対し「目標値欄が二つあることの方がおかしい」。Code の検討でも妥当性は実装都合のみで、同じ一つの値（この用途の判定の目標値）の入力先が2箇所になる悪形だった。土壌として「発動タブの目標値欄は用途起点の判定に配線されておらず表示専用だった」断絶があり、これも解消（数字は判定の目標値として消費）。
- **「式」モードの新設は不採用（ユーザー裁定）**: 目標値の選択肢はデータから拾うもの＝選択肢は変えず、「解説参照」「その他」の自由記入欄で式を受ける。制御値/達成値/登場目標値は別メカニクス＝具体値は引かない。
- **射程「武器」**: 自動入力ボタン内の実装だけでは「武器を選んでも射程が変わらない」＝機能していないように見える（ユーザー報告）。武器の追加・削除にライブ追従させた（勝者が「武器」の用途限定＝手動設定は壊さない）。
- **セレクト競合**: カスタムセレクトの change がフォーム submitOnChange と二重発火し actions 全配列書き込みが競合（後勝ちで巻き戻り）。武器表示の不整合報告の有力因として stopPropagation で遮断。表示経路自体は攻撃フローと同一の live 参照であることを検証済み（症状が残る場合は再現手順の確認が必要）。

#### 課題・申し送り

- 登場目標値の扱いは登場判定のメカニクス実装時に再訪。
- 攻撃用途の目標値は従来どおり null（制御値/対決メカニクスが決める）。


### 攻撃の使用武器の一本目＝シートの「攻撃で使用」への配線（フェーズ12 補）

**日付**: 2026-07-13
**関連実装**: `attack-weapons.mjs`（新規・`resolveAttackWeapons`/`attackWeaponDisplayName`/`resolveAttackRangeValue`＝テスト付き）、`attack-flow.mjs`（攻撃力合算・FA・残弾消費）、`tnx-usage-sheet`＋`usage-sheet.hbs`（一本目の常時表示・追加候補からシート武器除外）、`character-base.mjs`（スキーマコメントの旧設計上書き）。本体 `9c35314`。正本 [Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md)「攻撃の使用武器の解決」
**ステータス**: ✅ 実装完了（全テスト green 1217・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### 経緯・監査（ユーザー指摘＝Code の誤モデル）

- ユーザー報告「シート側で武器を設定しているのに用途上は生身としか表示されない。『使用武器の追加』は2本目以降のための機能のはず。武器の射程も読み取られない（生身でも至近が読み取られるべき）」。
- 実態: 戦闘タブ「攻撃で使用」は旧設計コメントで「表示のみ」とされ、**攻撃フロー・用途表示・射程解決のどれも読んでいなかった**（用途に武器を追加しない限り常に生身）。Code は前回この旧設計を鵜呑みにし、武器解決のモデルを誤解したまま用途 weaponRefs だけを対象に「修正した」と報告した（セレクト競合の修正自体は有効だが、報告された症状の本体ではなかった）。
- 裁定を実装: 一本目＝シートの「攻撃で使用」（空欄＝生身）・用途の追加＝2本目以降・射程「武器」は使用武器から解決し**生身は至近**。生身書き換え（義体等）は「生身（名前）」表記で一本目扱い。

#### 課題・申し送り

- 戦闘タブの「攻撃で使用」変更時、既存用途の保存済み射程は自動更新されない（自動入力か用途の武器操作で再解決。表示・攻撃力はライブ）。
- ヴィークル搭載兵器を「攻撃で使用」候補に含めるか（現候補=武器＋全身義体）は必要になったら再訪。


### 用途タイプの一本化（判定/宣言・改造廃止・NPC取得フラグ化・フェーズ12 補）

**日付**: 2026-07-13
**関連実装**: `usage.mjs`（npcAcquire フラグ・migrateData: modification→check／npcAcquire→フラグ+タイプ振り分け）、`tnx-usage-sheet`＋`usage-sheet.hbs`（USAGE_TYPES 2択・〈NPC取得〉fieldset を効果タブへ移設・OFF リセット・改造 UI 撤去）、`tnx-character-sheet-base`/`tnx-item-sheet`（フラグゲート・作成2択）、`npc-acquisition.mjs`（変更なし=モード駆動のまま）。本体 `a70a875`。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「用途タイプ」
**ステータス**: ✅ 実装完了（全テスト green 1220・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### 監査（ユーザー確定）

- **動機**: 「改造」「NPC取得」タイプの存在が組み合わせ判定の不確定要素になっている→タイプを**判定/宣言に完全一本化**。
- **改造**: 宣言で行うことは無い＝判定のみ（そもそも機能未実装なので消すだけで良いかも、とのユーザー示唆）→タイプ廃止・既存データは check へ移行・modifiableParams はデータ温存のみ（UI 撤去・実装フェーズで再設計）。
- **NPC取得**: アイテムから取得の可能性がある＝判定にも宣言にも設定できる形へ（フラグ化）。**取得設定は発動タブでなく効果タブが適切**（ユーザー指摘）→〈NPC取得〉fieldset として移設。migrateData の振り分け＝エキストラ（判定なし）=宣言・トループ/エニグマ/分身（判定）=判定。

#### 設計レビュー

- 実行フロー（useNpcAcquire）は**モード駆動のまま無変更**＝タイプはタクソノミーと UI 配置の問題に限定（宣言＋判定系モードのような異型設定でも従来どおりモードが判定の有無を決める）。
- 旧タイプの作成ゲート（トループ取得技能・アウトフィットのみ）は fieldset の表示ゲート（canNpcAcquire）へ移管。OFF でモード・参照・召喚数をリセット（回復と同じ意味論）。
- 参加技能 UI ゲートの isAcquireCheckMode は isCheckType へ吸収（check 用途は常時コンボ UI・宣言取得はコンボなし）。

#### 課題・申し送り

- 宣言用途に判定系モード（トループ等）を設定した場合もモード駆動で判定が走る（migrateは判定へ振るため通常発生しない・卓設定の自由度として許容）。
- 改造機能の実装時に modifiableParams を含めて再設計する。


### カード数字上書き AE＋ダメージ種別上書きの選択式入力（フェーズ12 補）

**日付**: 2026-07-13
**関連実装**: `helpers.mjs`（`check.cardValue` パース scope="cardValue"・`actorCardValueOverride`＝A〜K→生数字・`resolveItemTotalPath("attack.damageType")`＝total なし直接パス）、`character-base.mjs`（**文字列上書きレーン**＝数値評価を通さず置換・先勝ち・値バフ除外に cardValue 追加）、`tnx-check-flow.mjs`（手札/山札の数字を上書きして以降の規約に流す・結果カード「数字上書き（元→後）」行）、`tnx.mjs`（AE 設定シートの値入力を選択式化＝check.cardValue は A〜K・*.attack.damageType は S/P/I/X・キー変更に追従）。本体 `031227b`。正本 [Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md)
**ステータス**: ✅ 実装完了（全テスト green 1224・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### 監査（ユーザー確定）

- **カード数字の上書き**: 値は自由入力でなく **A〜K の選択式**。上書き後の数字に既存規約が従う（手札絵札=10・A の21固定選択・山札絵札=FUMBLE=K へ上書きして山札判定すればファンブル）。ジョーカー/切り札は自前の宣言があるため対象外（Code 既定）。制御判定は無印「判定」の用語規約により対象外。
- **ダメージ種別の上書き**: `…attack.damageType` は上書きで使用される想定のため、値は **S/P/I/X の選択式**（ベタ打ちさせない）。

#### 設計レビュー

- 値バフの適用パスは従来すべて数値（リテラル→式の2フェーズ）で、文字列は式評価に落ちて 0 になっていた——`attack.damageType` を文字列上書きパスとして分離（total を持たないため実 system パスへ直接置換・リテラル段で適用・同一効果内は先勝ち）。
- AE 設定シートの選択式化は renderActiveEffectConfig の既存注入フックに追加（キー入力の変更に追従して value 入力を select⇄text で差し替え。対象キー以外は従来どおり自由入力）。
- 攻撃カードには数字上書き行を出していない（カード値の行自体が無いため。達成値には反映済み・申し送り）。


### AE キー文法の統一＋Active_Effects/Formula_Reference の正本分離（フェーズ12 補）

**日付**: 2026-07-13
**関連実装**: `helpers.mjs`（`item.<識別キー>.system.<パラメータ>`＝`system.skill.<識別キー>.*` と同義のエイリアス。self/parent も同様・category は対象外）。本体 `（コミット参照）`。wiki: Active_Effects/Formula_Reference の冒頭に正本宣言・Formula_Reference から AE キー表2節を除去しポインタ化
**ステータス**: ✅ 実装完了（全テスト green 1225・lint 0 error）

#### 経緯（ユーザー指摘2件）

- 「アイテムへの AE キーは作ったはずだが？」→ **存在する**（v2 の `system.skill.<識別キー>.*` 等・フェーズ9）。Code の前ターンの「未対応」は `item.<key>.system.*` という**綴り**の話で、式（@参照）と AE キーで綴りが食い違う**二重文法**が放置されていたのが根本問題——ユーザーが式の綴りで AE キーを書くのは自然であり、エイリアスで統一した（推奨綴り＝item.<識別キー>.system.*）。
- 「Active_Effects.md と Formula_Reference.md がごっちゃ。Formula_Reference に AE キーが記載されるなど矛盾が多くどちらを見れば良いか分からない」→ **正本を分離**: AE change キーの唯一の正本＝Active_Effects（式の正本＝Formula_Reference）。Formula_Reference にあった AE キー表2節（判定キー・ダメージ対象キー）を削除しポインタに置換。両ページ冒頭に正本宣言を追加し、以後キーの表を複製しない。


### アイテム AE の着地の是正（設定欄不変・モード修復・転送効果の明示表示・フェーズ12 補）

**日付**: 2026-07-13
**関連実装**: `helpers.mjs`（`resolveItemTotalPath` のフルパス正規化・`attack.damageTypeTotal` 着地・`collectTransferredItemEffects`）、`character-base.mjs`（文字列上書き=常に OVERRIDE・`baseAttack.damageTypeTotal` 既定+scope "baseAttackType"）、消費側（attack-flow/戦闘タブ/用途シート=実効値優先）、`effects-sheet-mixin`＋`active-effects-list.hbs`（転送セクション）。本体 `6f3c8f2`。正本 [Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md)
**ステータス**: ✅ 実装完了（全テスト green 1227・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### 経緯（ユーザー指摘3件・いずれも Code の実装不備）

1. **「最終値を書き換えろと言ったはずなのに設定欄の入力データが直接書き換わっている。此処は絶対に触らない」**: ダメージ種別の上書きを素値 `attack.damageType` に直接書いていた（total 相当が無かったため）——v2 の「base 不変・total 書き換え」の大原則への違反。実効フィールド `damageTypeTotal` を新設して着地を移した（生身も同様）。
2. **「AE の効果タイプ（追加・上書き）も効いていない。追加なら元の値を崩さず追加しろ」**: 統一記法のフルパス綴り（`….attack.value` 等）が `attack.value.total` という壊れた着地パスを生成し、加算も上書きも正しく効いていなかった。素値/実効どちらの綴りでも実効（total 系）へ正規化することで、追加モードは base を崩さず実効値に加算されるように修復。文字列（種別）は加算が無意味なため常に上書きで適用。
3. **「アイテムへの遠隔付与は不自然。対象アイテムに転送され、他アイテム由来であることを（一時/無効効果のような形で）明示表示し、転送先では self になって適用されるべき」**: 対象アイテムのエフェクト一覧に**「転送された効果」セクション**を追加（供給元名・斜体・読み取り専用・キーは `item.self.system.*` 表記に変換して表示）。**適用の実体は単一適用パスのまま**（実体コピーの転送は二重適用・変更/削除の同期問題を生むため作らない=Code の設計判断として明示。物理コピーが必要なら再訪）。

#### 課題・申し送り

- `baseAttack.value` 等の数値ネイティブキーは従来どおり（素値に直接効く Foundry ネイティブ）。生身の数値バフの実効化が必要になったら同じ手当てを行う。
- 転送セクションは有効(active)な効果のみ表示（無効化された供給元効果は出ない）。


### 転送表示の装飾撤去＋ダメージ種別上書きの適用堅牢化（フェーズ12 補・追補）

**日付**: 2026-07-13
**関連実装**: `active-effects-list.hbs`＋`helpers.mjs`（転送行=通常の効果行と同型・キー羅列と読み取り専用を撤去）、`effects-sheet-mixin`（resolveEffect が供給元=アクター側も解決→転送行から編集/切替/削除が供給元に効く）、`character-base.mjs`（文字列上書きは effect.apply を経由せず直接代入=スキーマ外フィールドのコアキャスト経路を回避）。本体は次コミット。
**ステータス**: ✅ 実装完了（全テスト green 1227・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### 経緯（ユーザー指摘）

- 「編集不可とキーの表示は余計。キーがずらずら表示されても意味不明で不快」——内部キーの生値を UI に出さない既存規約への再違反。転送行からキー表示を撤去し、通常の効果行と同型（操作は供給元へ）に。
- 「ダメージ種別の上書きが動いていない」——適用を effect.apply（スキーマ外フィールドではコアの型キャスト挙動が不確実）から直接代入へ変更して堅牢化。あわせて**正しいキーの一覧を Active_Effects.md に明記**（最頻の原因=識別キー指定は識別キーを持たないアイテムに届かない→手作り武器は `item.self` かカテゴリか識別キー設定）。


### アイテム狙い AE の物理転送への切り替え（フェーズ12 補・最終形）

**日付**: 2026-07-13
**関連実装**: `helpers.mjs`（`itemChangeTargets`/`buildTransferredEffectData`＝テスト付き）、`tnx.mjs`（materializeItemTransfers＝効果の作成/更新・アイテム追加で実体化）、`character-base.mjs`（skill/category/parent の遠隔適用を廃止）、仮想「転送された効果」セクション削除。本体 `ea3b565`。正本 [Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md)
**ステータス**: ✅ 実装完了（全テスト green 1230・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### 経緯（ユーザー裁定）

- 完全同期（仮想表示＋遠隔適用の再照合）は「parent のまま転送されて結局値が適用されない」——**『転送された効果は無条件でそのアイテムに効く』か『完全同期をやめる』かのどちらか**、との二択裁定。物理転送（実体コピー・キー self 化）を採用＝両方を同時に満たす（コピーは self スコープで無条件適用・同期はしない）。
- Code の当初の「実体コピーは同期問題を生むため作らない」という判断は、ユーザーが同期を明示的に放棄したことで前提が消えた。転送は一方向・一回（重複防止のみ）。
- ダメージ種別上書きの「効かない」報告への診断確認点: 実効値が見えるのは**攻撃カード・戦闘タブ・用途シートの使用武器表示**（武器シートの設定欄は素値のまま＝仕様）。既存効果の転送は OFF→ON で発火。※後日、アウトフィットタブ・武器シート説明タブ等の表示も実効値へ統一（次エントリ）。

#### 課題・申し送り

- 供給元を削除しても転送コピーは残る（無同期の帰結・手動削除）。※後日、片方向同期の導入で解消（次エントリ群参照・供給元の削除でコピーも除去）。
- 転送は自クライアント起点の操作でのみ発火（userId ガード）。他クライアント作成分はその作成者側で発火する。


### 攻撃力表示の実効値統一（フェーズ12 補・表示掃引）

**日付**: 2026-07-12
**関連実装**: `tnx-character-sheet-base.mjs`（アウトフィットタブの武器「攻」表示・コンバイン合成の chosenSys("attack") 表示）、`tnx-outfit-sheet.mjs`（説明タブの攻撃力サマリ fmt・`_attackLabel`）——いずれも `damageTypeTotal || damageType`＋`total ?? value` の実効値優先へ。設定欄（outfit-sheet.hbs の `system.attack.damageType` セレクト等）は素値のまま＝仕様。本体 `dbdc6f3`。
**ステータス**: ✅ 実装完了（全テスト green 1236・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### 経緯（ユーザー指摘）

- 「UI 上の表示が変わらなければ意味がない。アクターシートのアウトフィットタブと武器シートの『説明』タブの表示が変わっていない。なんで良いと思ったのか」——`damageTypeTotal` 導入時に攻撃フローの消費者（攻撃カード・戦闘タブ・用途シート）だけ更新し、**表示箇所の全数洗い出しをしなかった**（正当な理由なし）。
- 是正: `attack.damageType` の生読みを全掃引（grep）し、表示系 4 箇所を実効値優先へ統一。残る生読みは設定欄（編集用セレクト）と適用エンジン内部のみ＝正。

#### 教訓

- 実効フィールドを新設したら、**旧フィールドの読み手を grep で全数列挙**し、表示系/設定系/エンジン系に分類してから完了を宣言する。消費フローだけ直して「完了」としない（[[feedback_completion_includes_presentation]] と同根）。


### 転送された効果の片方向同期（フェーズ12 補・物理転送の追補）

**日付**: 2026-07-12
**関連実装**: `tnx.mjs`（`materializeItemTransfers`＝既存コピーを供給元の現在値で上書き・狙い外れで除去／`removeItemTransferCopies`＋`deleteActiveEffect`/`deleteItem` フック＝供給元の削除でコピー除去／updateActiveEffect の発火条件に name/img 追加）、`helpers.mjs`（doc コメントの無同期記述を現行モデルへ更新）。本体 `9a1c44a`。正本 [Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md)
**ステータス**: ✅ 実装完了（全テスト green 1236・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### 経緯（ユーザー指摘）

- 「後から元のエフェクト側を更新した場合に反映されません」——物理転送は作成時の一回きり（重複防止で既存コピーをスキップ）であり、「無同期」は当初設計に明記していたが、供給元の更新が反映されないのは実運用に耐えないとの指摘。**供給元が正の片方向同期**へ変更（記録上の無同期と現在の発言が矛盾＝現在の発言が正）。
- 同期の向きと範囲: 供給元→コピーのみ。更新=上書き・狙い外れ=除去・供給元削除（効果単体/アイテムごと）=除去。コピー側の手動編集・切替は供給元の次の更新で上書きされる（供給元が正の帰結）。コピー自身の削除は供給元へ波及しない。
- 当初却下された「完全同期」（parent キーのまま遠隔再照合＝値が適用されない方式）とは別物: コピーは self 化された実体のまま＝無条件適用は維持し、内容だけを供給元に追随させる。

#### 課題・申し送り

- 同期は自クライアント起点の操作でのみ発火（userId ガード・従来どおり）。
- 供給元アイテム（bearer）の改名は効果の更新ではないため transferredSourceName に即時反映されない（次の効果更新時に取り直し）。


### AE 適用モデル v3（フェーズ12 補・自動適用ゲート＋使用時付与の再設計）

**日付**: 2026-07-13
**関連実装**: `helpers.mjs`（parseEffectTargetKey v3＝素のパラメータキー・self/parent/system.skill 綴り廃止・疑似分類 generalSkill/styleSkill／effectAutoApplies／analyzeGrantLanding／itemGrantCandidates／rewriteGrantChangesForItem／buildTransferredEffectData の素キー書き換え・準備先モード・effectId/stackable 引き継ぎ）、`character-base.mjs`（値バフ収集・状態ペナルティ収集にゲート配線・parent スコープ撤去）、`tnx.mjs`（materializeItemTransfers のゲート/準備先分岐・updateItem フック新設＝装着/分類/識別キー変更で再転送・AE 設定シートへの注入=transfer ラベル差し替え「効果を対象に自動適用」＋「準備先（親アイテム）に適用」＋「付与先」）、`usage-effects.mjs`（付与先分岐＝自分向け即時付与・アイテム着地の選択付与・置き換えリフレッシュ・grantedFrom）、`effects-sheet-mixin.mjs`＋`active-effects-list.hbs`（「使用時付与用」セクション）、`outfit-categories.mjs`（OUTFIT_TYPES 導出）。本体 `fb2c582`＋是正 `6dd7b07`。正本 [Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md)（v3 注記）・[Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md)「対象への付与」
**ステータス**: ✅ 実装完了（全テスト green 1251・lint 0 error）。**注入 UI（保存方式是正込み・transfer→準備先→保存の一連と保存後の転送発火）は実機確認済み（2026-07-13 ユーザー確認）**・付与フロー（選択ダイアログ・対象/自分付与・ダメージ連動）は実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査）

- **恒久のデータ変更と「能力の使用による効果の転送」は別物**（ユーザー指摘が起点）: 「マイナーアクションで使用して武器の攻撃力＋」は使用時にのみ武器へ効く。従来はダメージ＋で代用するしかなく、攻撃力を参照する式・表示が素の値を読む弊害があった。v3 は攻撃力そのものを実効値で上げるため参照系も正確になる。
- **付与先（対象/自分）**: 「効果Aを相手に与える代わりに自分がデバフを負う」を、AE 自身の付与先宣言＋自分向けの用途解決時即時付与（代償を後回しにしない）で表現する（ユーザー確定）。
- **付与のアイテム選択**: 対象アクターの全アウトフィットから選択が既定。絞り込みは分類・疑似分類（一般技能/スタイル技能）・識別キーをキーに含めて表す（ユーザー確定）。
- **多重付与の置き換えリフレッシュ**: 「明記なき同一効果は重複しない」原則の付与形（Code 提案・ユーザー承認＝異議なく実装指示）。
- **混在非対応**: 1効果の changes にアイテム着地とアクター着地は同居しない。必要なら効果を分けるか、ダメージ＋（アクター着地）で代用（ユーザー裁定）。
- 既存ワールドの移行は行わない（ユーザー裁定）。

#### 設計レビュー

- **transfer の再利用**: 死んでいたネイティブ transfer を「効果を対象に自動適用」ゲートへ昇格（ユーザー提案）。新フラグを増やさず、Foundry ユーザーの既知 UI の意味を自然に拡張。ゲートは effectAutoApplies 一箇所に集約し、収集系（collectActorEffectBuffs／_applyEffectBuffs／状態ペナルティ）が共用。実体化済みインスタンス（transferredFrom/grantedFrom）は常に生きる＝コピーへの transfer=true 付与を避け、コアの applyActiveEffects がアクターへ生パスを書く汚染を防いだ。
- **キーの役割の一本化**: キー＝「どのパラメータ＋絞り込み」、AE 設定＝「どこに乗るか」（自動適用・準備先・付与先）。self/parent/system.skill の廃止で綴りは死にキー化（黙って別解釈に化けない）。式の `@item.self`/`@item.parent` は「値を引いてくる」参照のため残す（ユーザー確定・キーとは別物）。
- **転送トリガーの穴を同時に閉塞**: 従来は効果の作成/更新とアイテム追加のみで、装着（parentItemId）・分類・識別キーの変更で再転送されなかった。updateItem フックを追加し準備先転送（準備で転送・解除で除去）とともに解決。
- **課題・申し送り**: 注入 UI は保存方式是正（追補）を経て実機確認済み。選択ダイアログ・付与フロー（対象/自分・ダメージ連動）は実機確認済み（2026-07-21・v0.12.0）。付与コピーの失効は当面手動（持続時間は時間管理フェーズ）。transferredSourceName の即時性など既存の申し送りは従来どおり。

#### 追補（2026-07-13・注入 UI の保存方式是正）

- **ユーザー報告**: 「準備先に適用」をオンにすると「効果を対象に自動適用」が自動でオフになり欄が消える（再オンで再表示）。設計意図の確認要求に対し原因を特定——**transfer がオフになるのではなく、未保存のフォーム状態が巻き戻っていた**: ネイティブ AE 設定シートは保存ボタン方式（送信まで未保存）なのに、注入フィールド（重複可の前例踏襲）が change 即時 setFlag ＝ドキュメント更新→シート再描画→フォームが保存済み値で再構築され、未保存の transfer オンが破棄→出し分けが（正しく）欄を隠す。
- **是正**: 注入3種（重複可・準備先・付与先）に `name="flags.tokyo-nova-axleration.*"` を与え**ネイティブ項目と同じフォーム送信で保存**（即時 setFlag 廃止）。出し分けは純粋な表示制御のまま。準備先のみ表示＝自動適用オン時という出し分け自体は設計意図どおりで維持（ペイロードでは準備先概念が効かない・エンジンも transfer 状態でゲート済みのため隠れた設定値の残存は無害）。
- **申し送り**: BS 効果値フィールドの注入は statuses 連動の再注入ロジックと不可分のため即時 setFlag のまま（同種の巻き戻りを潜在的に持つ・別途扱い）。「重複可」も従来から同じ巻き戻り副作用を持っていた（今回まで顕在化せず）。
- **実機確認済み（2026-07-13 ユーザー確認）**: 是正後の一連（transfer 未保存オン→準備先チェック→保存）と保存後の転送発火。本体 `6dd7b07`。

#### 追補2（2026-07-13・付与先の出し分け撤廃）

- **ユーザー指摘**: 「付与先」プルダウンが「自動適用」チェックで非表示になるのは同種のバグでは？——機構は巻き戻り（保存方式）とは別物だが**設計の誤りとして正しい指摘**: 用途の「適用される効果」は**自動適用オンの効果も選択できる**（恒常効果の複製付与）ため、付与先は transfer と**直交**。オフ時のみ表示だと「自動適用オンの効果の付与先」に到達できない（到達経路の検証漏れ＝[[feedback_reachability_and_persistent_settings]] と同根）。
- **是正**: 付与先は**アイテム上の効果で常時表示**（transfer 出し分けから除外・アクター上の効果は用途の効果になれないため非表示に変更）。準備先の出し分け（自動適用オン時のみ）は正当のため維持＝使用時付与ではコピー作成時に applyToParent を落とすので、オフ時に意味を持つ経路が無い。

### 宣言用途の「再判定を付与」（素の再判定権の事後付与・フェーズ12 補）

**日付**: 2026-07-13
**関連実装**: `usage-sheet.hbs`（宣言〈判定〉fieldset に「再判定を付与」チェックボックス追加＝判定を修正の上・独立）、`tnx-usage-sheet.mjs`（declaration ブランチの保存・式欄なし＝再描画不要）、`tnx-check-flow.mjs`（クリック待ち状態に `merge` 追加・開始メッセージの組み合わせ有無分岐・クリック発動で宣言は `mergeSkill: null`）、`tnx-character-sheet-base.mjs`（技能クリックの実行対象フィルタに grantRecheck 宣言を追加・merge 引き渡し）、`tnx-item-sheet.mjs`（merge 引き渡し）。スキーマ変更なし。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「再判定の付与」
**ステータス**: ✅ 実装完了（全テスト green 1251・lint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査）

- **事後的に「再判定権のみ」を付与する能力の表現（2026-07-13 ユーザー確定）**: 従来の付与（check 用途）は起動技能を組み合わせる形しかなく、素の再判定権を配れなかった。宣言用途に「再判定を付与」を追加して充足。
- **宣言は判定でない＝参加技能が無いため、組み合わせ処理は行わない**（ユーザー言語化）: 達成値クリックで**元の構成のまま**再判定が起動（通常の「再判定」ボタンと同じ再実行）。共通スート検査も自然に発生しない。
- 発動条件は自動強制しない（卓裁定）・消費は再判定実行時に適用・出し直しカードのボタン有無は元判定の `allowRecheck` を引き継ぐ（付与でボタンは増えない）——既存の付与と同じ意味論。

#### 設計レビュー

- **スキーマ変更なし**: `grantRecheck` は全タイプ共通スキーマの既存フィールド。起動分岐（キャラ/アイテムシートのクリック待ち系ゲート）は既に check|declaration + grantRecheck を受理する形だった＝欠けていたのは宣言 UI・保存・組み合わせスキップ・**技能クリックの実行対象フィルタ**（到達経路の先頭からの検証で発見＝grantRecheck のみの宣言が用途ピッカーに出なかった）。
- **組み合わせの有無はクリック待ち状態の `merge` で持ち回る**（呼び出し元が `usage.type === "check"` から導出・既定 true）: `startRecheck` は元々 `mergeSkill: null` で組み合わせなし再実行に対応済み（通常の再判定ボタンと同経路）のため、再判定フロー本体は無変更。
- 宣言のチェックボックス独立方針（排他にしない・複数 ON は決定的優先順＝grantRecheck が先）は 2026-07-12 の既存裁定を踏襲。

#### 課題・申し送り

- 既存の付与と同じ制限を引き継ぐ: 付与モードはクライアントローカル（他者の判定への付与は RL 代行か将来ソケット化）・継続処理つき判定はスナップショット非保存のため対象外。

### スタイル技能「技能」欄の選択肢「宣言」復元（フェーズ12 補・小修正）

**日付**: 2026-07-14
**関連実装**: `tnx-skill-utils.mjs`（`getSkillOptions().comboSkill` に `declaration: "宣言"` を追加＝「単独」の次・非組み合わせ系の並び）。本体 `cd236fe`。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「スタイル技能の『技能』パラメータ」
**ステータス**: ✅ 実装完了（全テスト green 1251・lint 0 error）

#### ルール解釈（メカニクス監査）

- ルールブックのスタイル技能データには**「技能：宣言」**（判定を行わない宣言型）の表記があり、シートの「技能」プルダウンで選べるべき（ユーザー指摘＝把握・承認なく消えていた）。
- 調査結果: **本リポジトリの git 履歴では初期コミット時点から不在**（初期 `lang/ja.json` の Combo 選択肢も 単独/ドッジ/パリー/技能名/任意 のみ）＝削除コミットは特定できず、git 移行前（プロジェクト前身）で喪失したとみられる。
- 表記のみで自動化はしない（単独/任意等と同じ扱い。「技能：宣言なら用途を宣言型に」等の連動は行わない＝推測実装をしない）。

#### 設計レビュー

- 影響範囲の検証: comboSkill の値で分岐するのは表示（`options.comboSkill[val]` 逆引き＝新キーは自動対応）と連鎖解決（`skill-chain-resolution.mjs`＝`skillName` 以外は一律「葉」扱い）のみ。値リテラル分岐の全数掃引で他の消費者なし＝追加は無リスク。
- **申し送り**: 前身プロジェクトで「宣言」を保存していたアイテムが旧キー（不明）で残っている場合、プルダウンは未選択表示になる＝再選択が必要（本リポジトリ世代のデータには存在しない）。

### 初回シート描画のラベル縮小不発（幅未確定の過大計測・フェーズ12 補・小修正）

**日付**: 2026-07-14
**関連実装**: `tnx-character-sheet-base.mjs`（`_squeezeResizeObserver`＝フレームのサイズ変化で `_applyTextSqueezing` を再適用・`_onClose` で解除）。本体 `94e305a`（誤修正）→`e157f86`（是正）。
**ステータス**: ✅ 実装完了（全テスト green 1251・lint 0 error）・**実機確認済み（2026-07-14 ユーザー確認）**

#### 設計レビュー

- **症状（ユーザー報告）**: ワールド読み込み後セッションで最初に開いたシートのみ、`.squeeze-text` の縮小（scaleX）が掛からず枠に見切れる。再オープン・編集⇄閲覧切替（=再描画）で正常化。一般技能ラベルに限らず同処理の全箇所（アウトフィット表列等）に出得る。
- **初回修正（94e305a）は誤り**: Code は「代替フォントの字幅で `scrollWidth` を誤計測」（FOUT）と仮説し `document.fonts.ready` 後の再計測を追加したが、症状は残存。**コンソール計測パッチ（コード変更なしのプロトタイプ差し替え）で実測**した結果、フォント説は棄却された——`scrollW` は初回/再描画で全行不変・`fonts=loaded`・`loadingdone` はシートオープンの約6秒前に完了済み。
- **実際の原因（計測で確定）**: 変わっていたのは**親幅**（`parentW` 201px→108px）。初回描画の `_onRender` 計測は**ウィンドウ幅（`position.width: 920`）の適用前**に走り、shrink-to-fit の仮レイアウト（列幅が本来の約1.9倍）を親幅として「収まっている」と誤判定→幅確定後に見切れる。再描画時は幅適用済みのため正しく測れる。
- **是正（e157f86）**: fonts.ready 待ちを撤去し、**フレーム（`this.element`）の ResizeObserver** で「サイズが変わったら縮小を再適用」に一元化。初回の幅確定に加えて**手動リサイズへの追従**（従来は未対応で同種の見切れが起き得た）も同時に獲得。transform は box を変えないため観測の再帰発火なし・縮小処理は冪等・`_onClose` で解除（派生シートに `_onClose` 上書きなしを確認）。
- **教訓**: once-per-session の署名からフォントを疑ったが、**計測なしの2手目はなかった**。誤計測系の不具合は「どの量が狂っているか」（今回なら contentWidth か availableWidth か）を先に実測で切り分ける。
- **申し送り**: ユーザー記録シート等の `.squeeze-text` は JS 縮小を持たない別物（対象外・現状問題報告なし）。

### 再判定の置き換え着地＋特殊処理の GM メニュー集約（フェーズ12 補・再設計）

**日付**: 2026-07-14
**関連実装**: `tnx-check-flow.mjs`（`recheckBlockReason`＝一度だけ/タイミングゲート・`startRecheck` に `recheckMessageId`・`_applyRecheckReplacement`＝置き換え着地・`_renderResultContent` 抽出・`_onGrantAchievementClick` のモード外=allowRecheck 直接入口・`_applyCheckModify` の manual 変種・`manualEditAchievement`・全幅ボタン廃止＋`tnx-recheck-ready` 常設装飾）、`attack-flow.mjs`（`buildAttackCardContent` 抽出）、`attack-flow-logic.mjs`（純関数 `resolveAttackRecheckState`＋テスト5件）、`damage-flow.mjs`（`manualEditDamage`）、`tnx.mjs`（`getChatMessageContextOptions` フック＝GM 限定メニュー3項目）、`tnx-socket-handler.mjs`（checkModify に content 受理を追加）、`tnx2.css`。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「再判定」「特殊処理の置き場」
**ステータス**: ✅ 実装完了（全テスト green 1256・lint/stylelint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査・すべて 2026-07-14 ユーザー確定）

- **再判定の本来の形＝達成値の置き換え**: 新カードを出すのは誤り（従来実装を訂正）。元判定に働いた一回性の増強（持続中の効果・組み合わせた判定以外の増強分）は失われるため、内訳も新しい判定の構成のみに差し替える。**再判定結果に対するリアクションのやり直しは無い**＝解決済みは保存済みの相手値で再解決。
- **一度だけ**: 再判定系は基本的に「1度だけ」の能力＝一度置き換えたカードは再判定不可（イレギュラーは GM の手動修正で裁定）。
- **ダメージ算出後の不可はタイミング不一致が理由**: 再判定系のタイミングは「判定の成功/失敗時」＝算出後は過ぎている（「巻き戻し禁止」ではない・警告文もその趣旨）。
- **特殊処理のアーキテクチャ**: 判定・ダメージへの特殊処理は**右クリックメニューが正規の置き場（GM のみ表示）**。用途フラグ付きアイテムロールは同じ内部機能への**例外的な外部アクセス**（クリック待ち経由）。allowRecheck のプレイヤー入口は達成値の数字クリック（従来の全幅ボタンは廃止）。
- **達成値の手動書き換え（新設）**: 処理が壊れた時の卓の最終裁定ツール。唯一の強制（ダメージ算出後の修正不可）の例外として制限しない。

#### 設計レビュー

- **実体は一つの関数・入口が三つ**: 数字クリック（allowRecheck）・付与のクリック待ち・GM メニューがすべて `startRecheck`→置き換え着地に合流。ゲート（`recheckBlockReason`）も三導線で共用＝表示と実行の判定が一致。
- **置き換えは「本文再レンダリング＋フラグパッチ」**: 結果カードは投稿と同じテンプレートを新データで再レンダリング（`_renderResultContent` 抽出＝投稿と共用）。攻撃カードは本文再構築（`buildAttackCardContent` 抽出）＋状態はライブ描画のまま `resolveAttackRecheckState`（純関数・受動有利含めテスト済）でフラグ再解決。非作者は既存 `checkModify` ソケットに content を追加受理させて委譲。
- **手動修正は事後修正の manual 変種**: `_applyCheckModify` に `{label:"手動修正", mod}` 経路を追加（評価・消費・ダメージ算出後ゲートをスキップ）＝台帳「事後修正（手動修正）」行で可視化・帰結再計算は共通。ダメージ側も `applyDamagePatch` の mods 行に合流＝新機構なし。
- **GM メニューの表示ゲート検証**: 「達成値を修正」はスナップショット持ちに限定——movement 等の継続処理カードも `checkResult` を持つため、当初案（checkResult||attackCheck）では帰結と乖離する編集経路が開いてしまう＋checkMods のライブ描画がスナップショット持ちでしか動かない（到達経路の検証で発見・修正）。
- **クリック待ち中の対象外クリックはモード維持で警告**（発動を空費しない）。装飾（`tnx-recheck-ready`）は所有者/RL かつ実行可能な時のみ＝見えているものは押せる。
- **申し送り**: 従来の「出し直しカードにもボタンが残る＝回数は卓」運用は一度だけフラグに置換。旧形式（新カード投稿）で残っている過去ログのカードはスナップショットを持つため引き続き操作可能（旧「再判定」バッジ付き新カードは歴史として残る）。他者の判定への付与がクライアントローカルな制限は従来どおり（申し送り継続）。

#### 追補（2026-07-14・実機検証での2件の是正）

- **GM メニューが表示されない（ユーザー報告）**: `getChatMessageContextOptions` の登録が `Hooks.once("ready")` 内にあり、**ChatLog のコンテキストメニュー構築（ready 発火前のサイドバー描画時）に間に合っていなかった**。init 側（getUserContextOptions と同所）へ移動して解消。本体 `105decd`。教訓: メニュー系フックは「メニューがいつ構築されるか」に対して登録タイミングを検証する（render 系と違い毎回は発火しない）。
- **武器攻撃力の表示と実測の食い違い（ユーザー報告）**: UI 表示（用途シート・戦闘タブ＝`total ?? value`・生身=`value+mod`）に対し、**攻撃フローが素値を読んでいた**。是正=武器攻撃力（`attack.value`→`total ?? value`）・FA値（`FAValue`→`FAValueTotal ?? FAValue`）・パリー受け値（`guardValue.value`→`total ?? value`）・生身攻撃（`value`のみ→`value+mod`）・パリー未選択時の生身受け値（0 固定→`baseGuard.value+mod`=戦闘タブ表示と同値）。本体 `6b98b3f`。dbdc6f3 の教訓（実効フィールドの読み手を grep 全数掃引）を消費側=フローに対して再適用した形（当時は表示側のみ掃引していた）。

### 修正の上書きモード＋状況修正スピナー＋数値入力のスピナー統一（フェーズ12 補）

**日付**: 2026-07-14
**関連実装**: `tnx-dialog.mjs`（AmountInputDialog に `allowOverride`＝上書きチェック・`okLabel`・共有 `spinnerDialogActions`＝step 後に input 発火）、`tnx-check-flow.mjs`（`_applyCheckModify` の上書き対応＝差分正規化＋`overrideTo` 行・`manualEditAchievement` を加減算既定＋上書きチェックへ・`setManualMod`）、`damage-flow.mjs`（同・ロール/軽減ダイアログへ actions 配線）、`tnx-check-dialog.mjs`＋`check-dialog.hbs`（状況修正スピナー常設）、`tnx-check-engine.mjs`（`calcControlCheck` に `manualMod`＝制御値へ加算・テスト2件）、`check-result.hbs`（制御判定の「修正（手動）」行）、`tnx-rl-request-app.mjs`/`treatment-flow.mjs`（代用ダイアログの手動修正欄スピナー化）、テンプレート3種スピナー化。本体 `095b280`。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)
**ステータス**: ✅ 実装完了（全テスト green 1258・lint/stylelint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査・2026-07-14 ユーザー確定）

- **修正は加減算に加えて「上書き」も可能**（チェックボックス）。対象＝手動入力ダイアログの全4経路（GMメニュー2種＋用途の式なし手入力フォールバック2種）。式は加算のみ。
- **全ての判定で状況ボーナス／ペナルティを入力できる**（判定ダイアログ常設）。**制御判定は制御値（成功条件）への修正**として働く（CoC の KP ボーナス相当・正で緩和/負で厳格化）。
- GM「達成値を修正」の入力方式は絶対値入力→**加減算既定＋上書きチェック**へ統一（計画で明示・ユーザー承認）。

#### 設計レビュー

- **上書きは内部で差分に正規化**: `overrideTo` を行に添えて表示だけ「→N」にし、達成値/攻撃側合計の再計算・ソケット委譲・帰結処理（成否再計算・攻撃再解決）は既存の合算機構を無変更で流用。
- **状況修正は既存 `manualMod` 経路の露出**: 代用判定用に一本通っていた配管（ctx→calcSkillCheck→内訳→スナップショット）へダイアログから書くだけ。制御判定のみ `calcControlCheck` に加算先（effectiveControl）を追加。代用/再判定の値が初期値に入り二重入力にならない。
- **スピナー統一**: 共有 `spinnerDialogActions` は step 後に input イベントを発火（軽減ダイアログのライブプレビューが±ボタンにも追従）。素の number input はダメージ算出/軽減/代用判定2種を掃引して置換（数値入力は必ず spinner の既定方針）。
- AmountInputDialog の戻り値は `allowOverride` 時のみ `{value, override}`（既存呼び出し＝カードドロー枚数は数値のまま＝後方互換）。

#### 追補2（2026-07-14・リロード後にダメージカードの内容が消える是正）

- **ユーザー報告**: ダメージカードがワールドのリロードで内容をすべて失う（従来はフォーマット変更の名残と思われていたが現行仕様の問題）。
- **原因**: ライブ描画系の `renderChatMessageHTML` フック5本（攻撃カード/ダメージカード/効果適用ボタン/再判定装飾/判定要求）が `Hooks.once("ready")` 内で登録されていたが、**チャットログの初期描画（既存メッセージの一括レンダリング）は ready 発火前**に走るため、リロード直後の表示分に効かなかった。ダメージカードは本文が殻（台帳・状態領域はフラグからのライブ描画）のため全消失に見えた。攻撃カードの状態領域・判定要求の結果表示等も同様に欠けていた（静的部分が残るため目立たなかった）。**データ＝フラグは無傷**（表示のみの問題）。
- **是正**: 5本をトップレベル登録（conditionDraw フックと同所）へ移動（本体 `db711fb`）。過去のカードも遡って復元表示される。GM メニュー登録タイミング（追補1）と同根＝**「フックの登録は、それが効くべき最初のイベントより前か」を検証する**教訓の再確認。

### 非数値パラメータの AE 化＝名前装飾・部位追加・特性フラグ・部位キー（フェーズ12 補）

**日付**: 2026-07-14
**関連実装**: `helpers.mjs`（`parseEffectTargetKey` に `itemName`/`partSlot`/`partAdd` スコープ・`AE_FLAG_PARAMS`/`flagTotalPath`/`readFlag`/`computeFlagEffectiveValues`/`parseBooleanFlagValue`・`resolveItemTotalPath` のフラグ正規化・転送書き換えの name/part 対応）、`character-base.mjs`（`_applyEffectBuffs` に真偽上書き/名前装飾/部位追加/部位デルタの処理・`partSlotsEffective` 派生）、`part-helpers.mjs`（`buildEffectivePartSlots`・キー照合の `computePartOccupancy`・`formatPartDesignation` 追加併記・表示逆引き群）、`part-slot-preset-app.mjs`（部位キー列・既定ラベル→キー対応表・`assignPartSlotKeys`/`migratePartSlotKeys`）、`skill-base.mjs`/`outfit-base.mjs`（フラグ実効値派生・`partAdded`）、`active-effect.mjs`（`TokyoNovaActiveEffect`＝Actor への `name` 適用抑止）、読み手掃引（check-engine/check-flow/attack-flow/attack-weapons/damage-logic/cast-sheet/character-sheet/outfit-sheet/tnx.mjs の実効読み化）、名前入力欄8シートの `sourceName` 化、AE 設定シートの値選択/datalist 注入。テスト26件追加。正本 [Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md) §2.3c–2.3e。
**ステータス**: ✅ 実装完了（全テスト green 1284・eslint/stylelint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査・2026-07-14 ユーザー確定）

- **意図**: 動的なパラメータ編集はなるべく AE で表現可能にする。名前装飾（D&D の「アイテム名（マジック・ウェポン）」相当・`{}`＝現在名に置換／ベタ打ち＝上書き）・アイテムの部位追加・アクター部位スロットの増減・各種「特性」フラグのオン/オフを AE から行えるようにする。
- **部位追加は値の側で and/or を表現**（キーは部位、値が関係）。**アクター部位スロットも増減可**（例: 手を一本増やす＝`片手持ち`＋1）。新規部位名も対応（正のデルタのみ新設）。
- **部位名にキーを設ける**（ラベルは表示・参照はキー）。既定部位は自動キー付与、既存データは ready 一回移行。設定アプリはウィンドウ幅を広げてキー列を追加。
- **フラグは「特性」系のみ**（生身変更装備 等）。`isAction`/`noCombo` は「特性」だが構造的読み手（用途チェーン設定を書き込む）と両立しないため今回のレジストリから除外（下記設計レビュー）。

#### 実装

- 真偽/文字列/名前は「実効フィールドへモード不問上書き」。フラグは opt-in レジストリ `AE_FLAG_PARAMS`＝着地（`<フラグ>Total`）と読み手掃引が済んだ印。読み手は `readFlag` 経由で実効を読む。
- 部位は「保存はキー・表示は逆引き現在ラベル」（識別キー・分類コードキーと同原則）。占有照合はキー優先・ラベル後方互換。

#### 監査結果 ✅

- ルール解釈はユーザー確定の意図に一致。名前の二相（`{}` 置換 / ベタ打ち上書き）・部位 and/or の意味論（and=常時加算・or=base 一式との択一）はユーザー承認済み設計に沿う。

#### 課題・申し送り

- **`noPrepareRequired` の順序境界**: アウトフィット集計（制御値/CS/危険値）は `_applyEffectBuffs` より前に走るため base を読む（AE 反転が乗らない）。実行時ゲートは実効を読む。全経路で完全一致にするには集計をフラグ AE 適用後へ動かす設計が要る（現状は許容の記録）。
- **`isAction`/`noCombo` の将来対応**: config 書き手（base）と実行時解決（実効）の作り分け＝チェーン再解決の設計が前提。
- **実機確認済み（2026-07-21・v0.12.0）**: Foundry 実走での名前装飾の重ね掛け・部位追加の占有表示・フラグ反転の各読み手反映（CLAUDE.md §2.1 によりユーザーが確認）。

### スタン攻撃・説得の本体メカニクス（宣言→算出段階キャップ・メカニクス監査＋設計レビュー）（フェーズ12 補）

**日付**: 2026-07-15
**関連実装**: `damage-logic.mjs`（`computeDamage` を「攻撃側合計を10上限にしてから軽減」へ是正・`attack` を返す）、`tnx-check-flow.mjs`（`setStunDeclared`）、`tnx-check-dialog.mjs`＋`check-dialog.hbs`（攻撃の判定に既定オフの宣言トグル）、`attack-flow.mjs`（攻撃ペイロードに `stunDeclared`）、`damage-flow.mjs`（宣言フラグ受け取り・台帳の10上限表示・軽減ダイアログの再確認廃止）、`usage-sheet.hbs`＋`tnx-usage-sheet.mjs`（`canStun` を物理攻撃限定の能力ゲートに）、`damage-mitigation-dialog.hbs`（宣言表示）、`css/tnx2.css`。computeDamage テストに順番固定ケース追加。正本 [Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「スタン攻撃」「説得」。
**ステータス**: ✅ 実装完了（全テスト green 1285・eslint/stylelint 0 error）・実機確認済み（2026-07-21・v0.12.0）。**10 上限の位置のみ 2026-07-16 ユーザー裁定で要是正（恒久軽減後＝算出の一番最後・KI-024）**

#### ルール解釈（メカニクス監査・2026-07-14〜15 ユーザー確定・正本=スキルディクショナリ「攻撃とダメージのフローチャート」）

- スタン/説得は「ダメージが 10 を超えない特殊な**攻撃オプション**」。宣言は**攻撃の対象を決定するとき**。対象は**単体※**（単体固定・効果等で変更不可）。
- **10 上限は「ダメージ算出」の最後＝攻撃側合計に、「ダメージ適用」の防御力軽減より前**に掛かる（cap-before-mitigate）。軽減の後でキャップすると、攻撃側合計が高いほど軽減量に関わらず 10 に張り付き、防御側の軽減が無意味になるため。ダメージ算出（攻撃側の計算＝カード＋攻撃力＋修正）と ダメージ適用（防御力軽減＋チャート参照）の区別が要で、キャップは前者の最後。
  - → **是正（2026-07-16 ユーザー裁定・KI-024）**: 防御力・受け値が「ダメージ算出」内へ移った（2026-07-16 表示裁定）ことに伴い、**10 上限は恒久軽減（防御力・受け値等＝事後軽減以外の全軽減）を引いた後＝ダメージ算出の一番最後**が正。本項の「防御側の軽減より前（cap-before-mitigate）」は当時のフェーズ区分（防御力=適用段階）を前提とした記述で、現在は誤り。上記「防御側の軽減が無意味になる」の理由付けも当時の区分に対するもので、現裁定には適用しない。
- **スタン（物理攻撃）は能力ゲート `canStun` が要る**。**説得（精神攻撃）は常に可能**（能力ゲート無し＝スタンと別扱い）。社会は対象外。スタンと説得は効果が似るだけの**別メカニクス**（種別変更で一方が他方に化けない）。
- **是正の経緯**: 旧実装は「適用直前の選択ダイアログ＋軽減後キャップ」で、決定タイミングも順番も誤り。Code が正本フローチャートに当てず、動いていた（誤った）コードを追認・後付け正当化していた（論理整合の検算を怠った失敗）。

#### 実装

- 宣言：攻撃の判定ダイアログに**既定オフのインラインチェック**（物理＋canStun→「スタンとして行う」／精神→「説得として行う」常時）。独立確認ダイアログにしないことで非宣言ユーザーの手数ゼロ・ルールの「常に選べる」を両立。
- 伝播：`_context.attack.stunDeclared` → `postAttackCard` が攻撃ペイロードを展開 → 攻撃カード `attackCheck.stunDeclared` → ダメージカード `damageRoll.stun`。
- 算出：`computeDamage` が `attack = stun && raw>10 ? 10 : raw; final = max(0, attack − mitigation)`。台帳に「スタン／説得（10上限）: N → 10」と攻撃側合計＝キャップ後を表示。適用時の再確認は廃止（宣言済み）。

#### 監査結果 ✅

- ルール解釈はユーザーが原文確認したフローチャート（宣言＝対象決定時／単体※／算出の最後で 10 上限＝軽減前）に一致。スタン=物理能力ゲート・説得=精神常時も確定どおり。※10 上限の位置は 2026-07-16 裁定で「恒久軽減後＝ダメージ算出の一番最後」へ是正（KI-024・実装は追従待ち）。

#### 課題・申し送り

- **単体※の選び直し**: 複数対象一括を実装した後（下の 2026-07-15 追補）も、スタン／説得の単体※は**先頭の命中対象で成立**させている。複数ターゲット時に「効果等で変更不可の単体」をユーザーへ明示選択させる UI は未実装（現状は先頭固定で満たす）。
- **再判定時の宣言引き継ぎ**: recheck スナップショットに stunDeclared を含めるかは未対応（edge case）。
- **実機確認済み（2026-07-21・v0.12.0）**: 判定ダイアログのトグル→攻撃カード→ダメージカードの 10 表示→適用の一連（CLAUDE.md §2.1 によりユーザーが確認）。

#### 追補（2026-07-15 同セッションの是正・拡張）

- **武器依存 canStun (ii)**: 武器（アウトフィット）に `canStun` フラグ新設（`weapon.mjs`・outfit-sheet に「スタン可能」チェック）。実効可否＝**物理かつ〔使用武器いずれか可 ∨ 生身 ∨ 用途.canStun（スタイル技能効果用に存置）〕**。`attack-flow` が `stunCapable` を算出して攻撃ペイロードに載せ、判定ダイアログのトグル可否に使う（用途チェックの自動オンや再判断機構は採らず、実行時判定＝ユーザーが (ii) を選択）。
- **説得の効果タグ抑止**: `applyDamageChartResult` に persuade 引数 → 負傷 AE に `persuade` フラグ → `tnx.mjs` の createActiveEffect カスケードで **incapacitation グループ（戦闘不能・支配含む）の inflicts を skip**。BS は通常付与。「説得に応じる」は状態・機構を持たないフレーバー（＝効果タグを付けないだけ）。※「+α」の具体があれば要追加（現状は incapacitation グループのみ抑止）。
- **対象/レティクル是正**: `attack-flow` の対象取得を `game.user.targets` **全件**へ。未ターゲット時の選択ダイアログは選んだトークンに `setTarget` で**レティクルを付与**してから進める（内部だけで対象を決めない）。
#### 追補（2026-07-15 複数対象への一括＝実装）

戦闘フローを単体前提から**全ターゲット一括**へ改修した（→ [Damage_Rules](../01_Wiki/Game_Rules/Damage_Rules.md)「複数対象への一括」）。

- **攻撃ペイロードの対象配列化**: `attack-flow.startAttack` が `game.user.targets` 全件を `targets` 配列で持ち回る（旧 `targetUuid/targetName` 単体フィールドを撤去）。`postAttackCard` が対象ごとに `{uuid,name,controlValue,state,resolution,reactionAchievement,diff,parryGuard}` へ展開。命中判定（カードプレイ）は 1 回で全対象共有。
- **攻撃カード＝D&D 風対象リスト**: `renderAttackCard` を対象リスト（`.tnx-attack-targets`・行＝アイコン/名前/制御値/判定）へ再設計。カード上にリアクションボタンは持たない（対象ごとの個別カードへ分離）。全対象のリアクション解決後、命中があれば「ダメージカードを出す」を表示。
- **個別リアクションカード**: `postReactionCard(attackMsg,targetIndex)` が対象ごとに `attackReaction` フラグ付きカードを **whisper（GM＋対象所有者）**で投稿。`renderReactionCard` が未解決＝ドッジ／パリー／なしのボタン、解決後＝制御値／達成値を判定結果カード様に描画。`startReaction`/`handleNoReaction`/`completeReactionFromCheck` は `reactionMessageId/attackMessageId/targetIndex` で該当対象を更新。`applyReactionPatch` は GM/author 時に **whisper を空へ＝全体公開**、非権限は `emitCheckModify` でフラグのみ委譲。攻撃カードの対象リストへも結果反映（`applyAttackTargetPatch`）。
- **ダメージ＝1 ボタン 1 ダイアログ**: ダメージカードは命中対象を保持し**適用ボタンは 1 つ**（`canApply` = GM ∨ いずれかの命中対象を所有）。`openMitigationDialog` を**対象ごとの軽減入力を並べた 1 ダイアログ**（各行ライブプレビュー）へ再設計し、確定で全対象へ一括適用（`appliedResult.targets`）。対象ごとの適用ボタンは廃止。
- **対象依存値の近似**: 攻撃側合計に効く `@target.*`・対象バフ AE は共有のため**先頭の命中対象で近似**（`hitTargets[0]`）。単体※のスタン／説得も先頭対象で成立。
- **再判定（置き換え着地）の配列化＝統合欠落の是正**: 配列化で攻撃フラグの `state/resolution/targetValue/targetUuid` を各 `targets[i]` へ移したため、旧・単体前提の再判定分岐（トップレベル単体フィールドを読み書き）が機能しなくなっていた（テスト・lint は緑でも実経路が壊れる＝[[feedback_verify_integration_not_just_params]] の類型）。是正: `attack-flow.rebuildRecheckedTargets` が**共有達成値の再ロールを対象リスト全体へ再解決**（リアクションのやり直しはしない＝保存済み相手値で成否・差分を再導出。純判定は既存 `resolveAttackRecheckState`）。仕切り直し（pending 復帰）対象は対決系保存値を初期化し**新スートの制御値を取り直す**。上位状態も fumble/miss/open/active へ導き直す。加えて、リアクションが**別カードのスナップショット**になった帰結として、`refreshReactionCardsAfterRecheck` が**未解決リアクションカードの達成値/スート/本文を追従**（全体失敗＝ファンブル/スート不一致に転じたらリアクション余地なしとして未解決カードを削除）。再判定の実行者は攻撃者（＝各カードの author）か GM のため更新権限がある。
- **既知の小限界**: 複数ターゲット時に「効果等で変更不可の単体」を明示選択させる UI は未実装（スタン/説得は先頭固定）。`@target.*`/対象バフは先頭命中対象で近似（対象ごとに異なる対象依存ダメージ修正は未対応）。
- 検証: `npm test` 全 1285 通過・`npm run lint`（eslint）0 error・`npm run lint:css`（stylelint）clean。`rebuildRecheckedTargets` は Foundry 結合（`fromUuid`）のため慣行どおり単体テスト対象外（純判定 `resolveAttackRecheckState` は既存テストで担保）。**実機確認済み（2026-07-21・v0.12.0）**（CLAUDE.md §2.1）。
- **UI 調整（実機フィードバック・2026-07-15）**: ①個別リアクションカードの状態領域 `.tnx-reaction-status` に余白が無く、解決後の目標値/差分行・成否バナーが無余白だった → 解決後は目標値/差分を `cr-calc-section` へ足し成否は `cr-result` フルバナー（check-result と同型）、未解決の注記/ボタンは内側余白の `.tnx-reaction-actions` ラッパへ。②ダメージカードの合計値が狭いカードで縦折り返し → `.cr-total-num` を `white-space:nowrap; flex-shrink:0`、対象名込みの長いラベルは `.cr-total-row .cr-calc-label` を `min-width:0; overflow-wrap:anywhere` で折り返し側へ。③「効果を適用済み」等の `・` 連結名は、名前自体が `・`（例「カブキ・トループ」）を含むと区切り不明 → 各名を `「」` で囲う（`usage-effects`・`damage-flow` の対象名/効果名・警告文。既存 `npc-acquisition` の `map(i => \`「${i.name}」\`)` 様式に統一）。

#### 追補（2026-07-15 達成値の下限・技能起動の単一関数集約）

- **達成値の 0 下限**（ルール）: ユーザー確定「判定結果はマイナスにならない」。AE ペナルティ等で計算途中は負でも最終達成値は 0 まで戻す。`calcSkillCheck`（基本計算）と `_applyCheckModify` の `newAch`（事後修正／GM 手動）の両方を `Math.max(0, …)`。GM 手動ツールは使用自体を制限しないが結果値は下限に従う。制御判定は達成値を持たないため対象外。テスト追加（`tnx-check-engine.test.mjs`・大ペナルティ／負の能力値実効値で 0）。正本＝[[Check_Rules]]「達成値の下限」。
- **技能起動の単一関数集約**（アーキテクチャ・設計レビュー／ユーザー確定「技能の起動は必ず一つの関数だけ・新しい起動関数を作らない・入口は多くてよいが実体は 1 つ」）: 従来は起動が 5 箇所に分散し、シートクリック以外（リアクション/治療/操縦移動/判定要求）は**用途処理の一部を手組みして迂回**していた（適用効果・不備検知・複数用途選択の欠落、操縦移動と判定要求に至っては**用途を一切見ない単一技能**）。→ `TnxCharacterSheetBase._activateItemCheck(actor, item, extraOpen)` を**唯一の起動関数**とし、`_onStartSkillCheck` は DOM から `(actor,item)` を解決して呼ぶ薄い入口に。**4 迂回箇所（`startReaction`／`startTreatment`／`startVehicleMove`／`onDoCheck` の技能判定分岐）は各自の追加文脈（`reaction`/`treatment`/`movement`/`requestMessageId`・目標値上書き・代用）だけを `extraOpen` に載せてこの 1 関数を呼ぶ**形へ置換（`extraOpen` は通常判定の `open` へ最後に合流し既定を上書き）。**帰結＝これらも用途のコンボ・消費・判定ボーナス・適用効果を通す**＝挙動がシートクリックと完全一致（操縦移動・判定要求は従来の単一技能から挙動変更）。**設計原則: 組み合わせ（コンボ）の可否は（RL を含む）ユーザーのみが決定権を持ち、システム側は一切制限しない**（ユーザー確定）。対象外＝能力値/制御判定・シナリオ情報収集（技能名のみで技能アイテムを起動しない）・再判定の再入（保存済み ctx の再実行）。付随して起動集約で死んだ手組みロジック（`treatment-flow.resolveCheckSkillSet`・各所の未使用 import）を除去。技能選択ダイアログのラベルを 1 行（「〈モード〉に使用する技能を選択」）に短縮し、ラベルは全幅・プルダウンは `max-width` で据え置き（`.tnx-select-dialog`）。検証＝全 1286 テスト green・eslint/stylelint 0 error。**実機確認済み（2026-07-21・v0.12.0）**（CLAUDE.md §2.1・特にリアクション/治療/操縦移動/判定要求の一連が正規フローで走ること）。

#### 追補（2026-07-15 リアクション成否表記・BS バッジ表示・捕縛の対象武器機構・計算行の折り返し）

- **リアクション成否の表記是正**: 「回避／受け（攻撃無効）」は**ルールに無い言い換え**（「受け」はパリーの別名でない・存在するのは「受け値」のみ）＝ユーザー叱責。成功は**「リアクション成功」に統一**（ドッジ/パリー/精神・社会リアクション共通）。ただし「リアクションしない」で被弾しなかった場合はリアクションをしていないため**「攻撃無効」**（ルール語「勝利で攻撃無効」）。`renderReactionCard` の verdict のみ変更。
- **BS バッジに効果値/対象を表示**（状態タブ・正本 [[Bad_Status]] の表記に従う）: 従来バッジは名前＋アイコンのみ。→ `readConditions` から `_bsValueSuffix` で表記を作り名前に付す。**表記は wiki 正本どおり**＝邪毒/電子妨害は名前に数字直付け（`[BS：邪毒n]`/`[BS：電子妨害n]`）、衰弱は `（-n）`・スート引き版は `（能力値の制御値 -n）`、重圧は `（能力値）`（`[BS：重圧(生命)]`）、萎縮/憎悪は `（対象名）`、捕縛は `（武器名/生身）`。**効果や名前で値が決まるもの（酩酊(大/小)・恐慌 等）は表示しない**。カードで決まるもの（重圧の指定なし・衰弱のスート引き）は**ドロー結果が `conditions[kind]` に保存済み**のため再描画で自然に表示される。**戒め＝表記を設計する前に必ず Bad_Status.md を引く**（「強度2」という語を wiki 記載 `[BS：電子妨害n]` を読まずにでっち上げ強叱責・[[feedback_consult_wiki_before_asking]]）。
- **捕縛の対象武器指定機構を新設**: 従来は効果設定の生キー手入力欄のみで指定機構として機能せず。→ 効果設定シート詳細タブに `対象武器` プルダウン（対象アクターの `type:"weapon"`＋**生身**）。保持は `targetWeapon`＝**アイテム ID・空=生身**（攻撃の `attackItemId` と同型）。`readConditions` に `targetWeapon` を追加。
- **計算行のラベル折り返し是正**（リアクションカードの「攻撃」が縦に割れる）: `.cr-calc-label` を `flex-shrink:0; white-space:nowrap`（短いラベルは折り返さず値側を折り返す）。合計行だけは例外（対象名込みの長ラベルを折り返す）＝`.cr-total-row .cr-calc-label` に `white-space:normal; flex-shrink:1` を上書き。
- 検証＝全 1286 テスト green・eslint/stylelint 0 error。**実機確認済み（2026-07-21・v0.12.0）**。

#### 追補（2026-07-15 BS効果の任意編集ダイアログ・NPC取得の再判定＝表示のみ）

- **BS 効果の任意編集ダイアログ**（ユーザー確定「カードを引く等の操作なく効果を任意決定・効果設定シートでなくダイアログ・主用途＝再判定確定後に消えるはずでなかった BS を同効果で与え直す」）: 状態タブのバッジのコンテキストメニューに **「効果を編集」** を追加（治療と併存）。`condition-edit.mjs` の `openConditionEditDialog` が def の効果値フィールド（強度/効果量・対象能力値・対象アクター・対象武器＝生身含む）を DialogV2 で出し、確定で `conditions.<kind>` へ保存（他経路と同じ保存先＝`readConditions`/バッジ表示と一貫）。**衰弱に `abilityField: "optional"` を追加**（手動で対象制御値を指定可＝空=全制御値・適用側は指定=その制御値のみ/未指定=全制御値を既に対応）。編集項目を持たない BS（酩酊等）はメニューを出さない。
- **NPC取得の再判定＝表示のみ更新**（継続判定の再判定の第一歩）: `_buildRecheckContext` の除外から `npcAcquire` を外した。再判定/修正は通常結果カードの達成値を差し替えて再描画するだけで、`completeAcquisitionFromCheck` は再実行しない（`_applyRecheckReplacement` は継続ディスパッチ手前で return するため自然に表示のみ＝設計どおり・生成物は自動で触らず手動）。明確な成功失敗のある**分身**もこの扱い。
- **継続判定の再判定・修正＝全種実装**（2026-07-15・A/B/C）: `_buildRecheckContext` の除外を撤廃し、継続文脈（reaction/treatment/recovery/controlNegate/npcAcquire/movement）を rc に載せる（rc は結果カードの `checkRecheck` フラグに保存＝**再判定は ctx 経由・事後修正は カードの `checkRecheck` 経由**の両方から読める）。新ヘルパー `_rerunContinuation(cc, result, {oldSuccess})` が種別ごとに再実行し、`_applyRecheckReplacement`（再判定）と `_applyCheckModify`（修正）の両方から呼ぶ:
  - **A リアクション**＝`completeReactionFromCheck` に `allowResolved` を追加し、解決済み(hit/miss)でも**新達成値で対決を再解決**（両方向・副作用なし）。
  - **A 操縦移動**＝通常結果カードを持たないため `buildMovementCardContent` を抽出し `_applyRecheckReplacement` に移動カード分岐を追加（達成値÷10 段階を**表示のみ**再描画）。`postMovementCard` は `recheckCtx` を受けて `checkRecheck` を保存。
  - **B 治療/回復/controlNegate**＝**失敗→成功の遷移でのみ**除去/無効化ハンドラを再呼び（`oldSuccess` でゲート・ハンドラは ID 指定の除去で冪等）。成功→失敗は表示のみ（手動復元）。
  - **C NPC取得**＝表示のみ（再実行なし）。
  - **二重実行の防止**＝`_execute` は再判定でも継続ディスパッチ(738-772)へフォールスルーするため、各継続ブロックを `!ctx.recheckMessageId` でゲート（requestMessageId 発火は表示追随のため維持）。`startRecheck` は rc の継続文脈を再実行 ctx へ引き継ぐ。
  - **リアクションの二重カード解消（2026-07-15 追加是正・ユーザー指摘「非常に気になる」）**＝リアクション判定では**通常の結果カードを出さない**（`_execute` の投稿分岐に `else if (!ctx.reaction)`）。**書き換わったリアクションカードが結果カードそのもの**。再判定/修正の導線もリアクションカードへ移設: `completeReactionFromCheck` が初回解決時に `recheckCtx` を受け、リアクションカードへ **`checkResult`＋`checkRecheck`（＋用途の `usageEffects`）** を保存（`applyReactionPatch` に `extraFlags` 追加・所有権委譲はそのまま）。`_applyRecheckReplacement` にリアクション分岐（本文は attackReaction フラグ由来のため上書きせず、`_rerunContinuation` が対決を再解決して表示更新）。
  - **リアクションの再判定＝プレイヤー導線も付与（是正）**: フック順 `renderReactionCard`→`renderRecheckButton` を利用し、`renderReactionCard` の解決表示で**リアクション達成値を「達成値」総計行**として描く（`renderRecheckButton` がこの行を再判定/修正の対象に拾う＝プレイヤーの達成値クリックが効く・GMメニューも併用）。「リアクションしない」は checkRecheck 非保存＝再判定なし（カード未プレイのため）。
  - **リアクションの適用効果＝保持（是正）**: リアクションカードに `usageEffects` を保存し `renderUsageEffectButton` が「効果を適用」ボタンを出す（`attackCheck` フラグ非保持のため発火）。
  - **統一起動の同種取りこぼし＝操縦移動も是正**: 専用カードを出すフロー（攻撃・操縦移動・リアクション）は統一起動 `_activateItemCheck` が用意する `usageEffects` を専用カードへ載せる必要がある。攻撃は既に対応済み、**操縦移動も `postMovementCard` が `usageEffects` を保存**するよう是正（リアクションと同型）。通常結果カードを使うフロー（治療/回復/NPC取得/controlNegate/RL要求）は元々 `usageEffects`＋再判定が結果カードに乗るため対応不要。
  - 検証＝全 1286 テスト green・eslint/stylelint 0 error。**実機確認済み（2026-07-21・v0.12.0）**。

- **BS バッジの効果値表示＝名前と値を分離＋長い値は縮小して収める（2026-07-15 是正・ユーザー指摘「衰弱の効果値が長すぎて表示しきれない」）**: バッジ本文を `名前`＋`値`の別要素に分離（`.tnx-bs-label`＝内容幅／`.tnx-bs-value` を `.tnx-bs-value-wrap` 内）。値は**既存の `.squeeze-text`（`_applyTextSqueezing` の scaleX 縮小・ResizeObserver 再計測）**で残り幅に収める（衰弱「（生命の制御値 -3）」等の長い効果値対応）。検証＝stylelint 0 error。**実機確認済み（2026-07-21・v0.12.0）**。

- **達成値21固定＝完全固定の徹底（2026-07-15 ユーザー確定）**: A の 21固定は engine 段階（`calcSkillCheck` FIXED_21）で報酬点・能力値・判定バフ/ボーナス・状況修正を無視済みだが、通常経路の事後修正が抜けていた。→ `_applyCheckModify` に `checkF.result.fixedAt21` ガードを追加し、**用途「判定を修正」＝通常経路は 21固定でブロック**。**GM メニュー「達成値を修正（手動）」＝管理者権限の最終裁定（`manual`）だけは通す**（ユーザー確定）。正本＝[[Check_Rules]]「Aの達成値21固定」。

- **ダメージ修正の per-target 化＝防御側軽減が全対象に効くバグの是正（2026-07-15 ユーザー指摘）**: 複数対象一括で攻撃側合計を共有にした結果、**防御側の `modifyDamage`（ダメージを修正＝軽減技能）が共有の攻撃側合計に乗り、一人の軽減が全対象を下げていた**（単体時代は露見せず・Code の設計ミス）。修正:
  - **`handleDamageModifyClick` を使用者で振り分け**: 使用者が命中対象の一人（防御側）なら**その対象の `targets[i].mods` に per-target で積む**。攻撃側/GM 代行は従来どおり共有の `f.mods`（全対象の起点＝増強）。発動方法（攻撃側合計クリック）は不変。上書きの基準も対象ごと。
  - **`openMitigationDialog` を対象ごとの攻撃側合計へ**: `resolvedTargets` に `mods` を取り込み、各対象の `targetRaw = 共有raw + その対象の modsSum` でプレビュー/適用（`computeDamage` の `damageCard` を per-target 化）。
  - **カードに対象ごとの「適用予定ダメージ」を表示**（`targetPlannedPreview`＝共有攻撃側合計＋その対象の防御側修正−自動軽減・手動軽減は適用ダイアログで最終調整のため予定表示）。適用済み表示に「ダメージ修正（防御側）」行を追加。
  - **適用ボタンのアクセスを RL と攻撃者に**（`canApply = isGM || attacker.isOwner`。各自が算出後〜適用前の増減を切り終えたら RL/攻撃者が押して全対象へ一括適用）。
  - ※スタン上限は攻撃側合計（共有・攻撃側）にかかったまま。防御側 modifyDamage は攻撃側合計側（cap 前）に乗る＝現行メカニクスを踏襲（スタン時は上限で吸収され得る＝既存の性質）。検証＝全 1286 テスト green・eslint/stylelint 0 error。**実機確認済み（2026-07-21・v0.12.0）**。**この per-target 土台の上に「別種類として適用」（当初目的）を載せる**のが次段。

#### 追補（2026-07-15 リアクションの追加挙動＝範囲攻撃へのリアクション/攻撃を失敗させる・全公開・攻撃者へのAE）

ユーザー確定のリアクション拡張（メカニクス監査＋設計レビュー）。正本＝[[Combat_Flow]]「リアクションの追加挙動」。用途データモデル `usage.mjs` に `reactionAreaAttack`／`reactionFailsAttack`（BooleanField・初期 false）を新設し、用途シートに「リアクション」セクション（当面は全用途に表示・条件表示は後続）。

- **範囲攻撃へのリアクション（`reactionAreaAttack`）**: リアクション成功（`ok && !hit`＝有効な判定で攻撃回避）で、同じ攻撃の**全対象を回避で一括解決**。他対象は `resolution:"areaCover"`・状態 miss（達成値なし・表示「範囲攻撃へのリアクション」）。1 人の成功で他対象は反応不要になる。
- **攻撃を失敗させる（`reactionFailsAttack`）**: リアクション勝利で攻撃の overall `state:"failed"`（`renderAttackCard` で「攻撃失敗（リアクションによる）」バナー・対象一覧は続けて表示）＋全対象 miss。ダメージなし。範囲攻撃へのリアクションとは**別フラグ**（前者は全員回避、後者は攻撃自体の無効化）。両立可（fails は state も落とす上位）。
- **実装の要点＝単一パッチ**: `completeReactionFromCheck` で本人分と全他対象を **1 回の `applyAttackPatch`** で送る（`f`＝解決前スナップショットを基点に本人分を明示 `Object.assign`）。当初は本人分 `applyAttackTargetPatch` ＋別パッチの二段だったが、**非作者クライアントのソケット委譲（`emitAttackUpdate`）が二重発火して本人分の解決を stale な配列で上書きし得る**レースを自レビューで検知（[[feedback_logical_consistency_first]]＝動く前に理屈で検算）→ 単一パッチへ是正。
- **誤った再解決のガード**: 既に解決済み（回避確定）の対象へ以後 `startReaction`／`handleNoReaction` を押しても、攻撃対象が `pending` でなければ通知して返す（被弾へ戻さない）。まだ未解決の他対象リアクションカードは `closeUnresolvedReactionCards` が**削除権限（GM か作者＝攻撃者）のある側でのみ削除**（無ければ残すが上記ガードで無害）。
- **リアクションカードの全公開**: `postReactionCard` の whisper を撤去（`reactionWhisperFor` 削除）。単体対象でも他者が肩代わりできるため公開必須（ユーザー確定）。`renderReactionCard` のボタンゲートを緩和＝ドッジ/パリー/精神・社会リアクションは**全員**、「リアクションしない」のみ対象所有者/GM（`canDeclineForTarget`）。実行アクターは `resolveReactor`（選択トークン→割り当てキャラ）で押下時解決。
- **リアクションの適用効果は攻撃者へ**: リアクションの対象は「なし」のため、用途に付与先「対象」の AE があれば**攻撃者**へ返す。`prepareUsageEffectPayload` に `targetOverride` を追加し、`_activateItemCheck` がリアクション時に攻撃カードの `attackerUuid` から攻撃者を解決して渡す（対象確認ダイアログを挟まない・空配列＝攻撃者未解決時は対象なし）。付与先「自分」は従来どおりリアクター自身へ即時。
- **監査結果 ✅**: 範囲攻撃へのリアクション/攻撃を失敗させるの区別・全員回避の帰結・公開範囲・AE の攻撃者返しはすべてユーザー確定に一致。用語は在ゲーム呼称のみ（`areaCover` は内部識別値でありユーザー可視文字列に出さない・表示は「範囲攻撃へのリアクション」＝[[feedback_no_dev_terms_in_ui]]）。
- **課題・申し送り**: ①非 GM・非攻撃者のリアクターが範囲/失敗を発火した場合、他対象カードの削除権限が無く**カードが残る**（inert・ガードで無害）＝掃除の GM 委譲は未実装。②「リアクション」セクションの**技能特性に応じた表示切替は後続**（当面は全用途表示）。③範囲攻撃へのリアクションで既に解決済みだった対象（先に被弾/回避）も miss へ上書きする（＝全員回避の確定挙動・ユーザー意図どおり）。
- 検証＝全 1286 テスト green・eslint/stylelint 0 error（変更ファイルに新規警告なし）。**実機確認済み（2026-07-21・v0.12.0）**（CLAUDE.md §2.1）。

**リアクター解決を「誰がクリックしたか」へ変更（2026-07-15 ユーザー確定・実機フィードバック）**: 実機で「別キャラで肩代わりしたのに対象キャラのリアクションになる」と指摘。原因＝`resolveReactor` が `controlled[0]?.actor ?? game.user.character`（選択トークン優先）で、攻撃対象のトークンが選択されたままだと対象を拾っていた（＋リアクションカードが対象の発話者・名前のまま＝見た目も対象）。**ユーザー裁定「カードが誰の名前で作られたかではなく、誰がクリックしたかで判断する方式に変更すべき」**。→ ①`resolveReactor` を **`game.user.character ?? controlled[0]?.actor`** に是正（クリックしたユーザーの割り当てキャラを最優先・GM 等の未割り当てのみ選択トークンへフォールバック）。②解決後のリアクションカードに **`reactorName`**（＝クリックしたユーザーのキャラ名）を保存し、`renderReactionCard` の解決表示に「リアクション: 「名前」（ドッジ 等）」として明示（対象と別人＝肩代わりが一目で分かる）。「リアクションしない」も対象＝リアクター名を保存。検証＝全 1286 テスト green・eslint/stylelint 0 error。**実機確認済み（2026-07-21・v0.12.0）**（この後 2026-07-16 に実機確認済みとユーザー報告）。**申し送り**＝1 ユーザーが複数キャラを所有し割り当てキャラ以外で肩代わりしたい場合は、そのキャラを割り当てるか（未割り当て前提の）トークン選択が要る（明示ピッカーは今回不採用＝「クリックしたユーザーで判断」の決定的方式をユーザーが選択）。

### 別のダメージとして適用（当初目的・per-target 土台の上に実装）（フェーズ12 補）

**日付**: 2026-07-16
**関連実装**: `damage-flow.mjs`（`openMitigationDialog` に `applyCategory` 引数・`renderDamageCard` に展開式ボタン群・適用済み表示に差し替え系統の明示）、`css/tnx2.css`（`.tnx-damage-altapply`/`.tnx-damage-altnote`）。正本 [Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「別のダメージとして適用」。
**ステータス**: ✅ 実装完了（全 1286 テスト green・eslint/stylelint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査・2026-07-16 ユーザー確定＝AskUserQuestion で 3 点確定）

- **差し替えるのは系統のみ**（肉体/精神/社会＝参照するダメージチャート）。ダメージ種別 S/P/I/X や算出式は変えない。
- **軽減は元の系統のまま**行う（肉体なら防具＝防御力の軽減も含めすべて）。その**軽減後の最終値**を、選んだ系統のチャートへ適用する（＝「元系統でフル軽減 → 軽減後の値を別系統チャートへ流す」）。**値はそのまま保持**（再計算しない）。
- **どの攻撃でも常に選べる**（RL 裁量。用途フラグ等のゲート無し＝システムは可否を制限しない）。

#### 実装

- `openMitigationDialog(message, applyCategory=null)`: `category`（=軽減・値算出に使う元系統 `f.category`）と `applyCat`（=チャートに使う差し替え先）を分離。軽減の自動値（防御力）・報酬点欄・スタン10上限・`computeDamage` はすべて `category`（元系統）で計算し、**チャートのプレビュー（`describeDamagePreview`）と適用（`applyDamageToTarget`）だけ `applyCat`** を使う。複数対象一括の 1 ダイアログはそのまま（軽減は各対象の元系統防御力・適用先チャートのみ差し替え）。
- `renderDamageCard`: 「ダメージ適用」（元系統）に加え、`canApply` かつ対象ありのとき **`<details>`「別のダメージとして適用」** に他 2 系統への適用ボタンを出す（`openMitigationDialog(message, cat)`）。適用済みは `appliedResult.applyCategory` を保存し「「精神」ダメージとして適用」の注記を表示。
- **説得の効果タグ抑止は元系統＝精神を精神チャートに適用したときのみ**（`persuade: stun && category==="mental" && applyCat==="mental"`）。別系統として適用する場合は説得の意味論が対応しないため付与しない。スタンの 10 上限は攻撃側合計の算出時点で掛かるため差し替えても保持。
- トループ/エニグマ/分身は系統に依らず heads 減算/消滅（チャート不参照）のため、差し替えの影響は cast/guest のチャートにのみ及ぶ。

#### 監査結果 ✅

- ルール解釈は 2026-07-16 のユーザー確定（系統のみ・元系統でフル軽減後の値・値保持・常時選択可）に一致。per-target 軽減の土台（2026-07-15）の上に、適用チャートの差し替えを直交軸として載せた（軽減は対象ごと・チャート系統は適用全体で 1 つ選ぶ）。

#### 課題・申し送り

- 別系統として適用したときのスタン10上限・説得の扱いは上記の保守的解釈（10上限は保持・説得は元系統精神のみ）。別系統適用と説得/スタンの組み合わせに固有ルールがあれば要追補。
- 実機確認済み（2026-07-21・v0.12.0）（CLAUDE.md §2.1）。

### ダメージ算出の是正（防御力・受け値を算出で確定）＋カバー（フェーズ12 補）

**日付**: 2026-07-16
**関連実装**: `damage-flow.mjs`（`targetPlannedPreview` を防御力/受け値に分離・`renderDamageCard` の行を最終ダメージ表示＋カバー表示/クリック・`openMitigationDialog` を縮小＝防御力/受け値固定＋手動軽減/社会報酬点のみ・`handleCoveringClick`/`completeCoveringFromCheck` 新設）、`usage.mjs`（`covering` フラグ）、`usage-sheet.hbs`＋`tnx-usage-sheet.mjs`（カバー トグル＋保存）、`tnx-check-flow.mjs`（`covering` クリック待ち種別＋完了ディスパッチ）、`tnx-character-sheet-base.mjs`（`_activateItemCheck` にカバー待ち受け分岐＋再入時の用途プリセレクト）、`css/tnx2.css`。正本 [Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「算出の適用順序（表示）」「カバー」。
**ステータス**: ✅ 実装完了（全 1286 テスト green・eslint/stylelint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査・2026-07-16 ユーザー確定）

- **防御力・受け値は「ダメージ算出」の一部**（勘違いの是正）。よってダメージカードの各キャラ行は**軽減後の最終ダメージ**を表示する。攻撃側合計はレジャーに残す（攻撃側の事後増強のため）。適用ダイアログは縮小＝防御力/受け値は自動・固定、入力は手動の状況軽減＋社会の報酬点のみ。「ダメージ軽減技能」（`modifyDamage`）の事後修正は別系統で従来どおり有効（攻撃側合計クリック）。
- **カバー**: ダメージ算出の直前に他者への予定ダメージを自身へ付け替える。**1体分のみ**（複数対象でも1体を選ぶ・全対象集約は別能力＝作らない）。**カバーした側が元々対象なら2ダメージが別々に入る**（自分の分＝自分のリアクション/受け値あり、カバーした分＝受け値なし）。**判定が要る**が目標値「なし」運用が多い＝スート一致で成功（固有ロジック無し）。**軽減はカバーした側自身の防御力で算出・受け値は参照しない**（自分でリアクションしていないため）。常に選べる（用途フラグを持つ技能）。「カバーリング」は固有技能名なので行動名は「カバー」（UI に「かばう者/かばう人」を出さない＝[[feedback_no_dev_terms_in_ui]]）。

#### 実装

- **アクセスは攻撃カード**（ダメージ算出の直前＝ダメージカードを出す前）。当初ダメージカードにトリガーを置いたが、「ダメージカードが出た時点で算出は始まっている＝『直前』でなく『算出後』」で時制矛盾とユーザー指摘 → 攻撃カードへ移設。
- 起動＝カバー技能をアイテムロール → `_activateItemCheck` の `covering` 分岐で **`startAchievementAction("covering")`** の待ち受け（消費はここではせず、判定起動時に一度だけ）→ **攻撃カードの命中対象行クリック**（`handleCoveringClick`・`attack-flow`・`!damageRolled` かつ `state==="hit"` かつ未カバーのみ）→ **`_activateItemCheck(coverer, skill, {covering:{attackMessageId,targetIndex,usageId}})` で再入**（`extraOpen.covering` があれば待ち受け分岐を通さず通常判定・用途は usageId でプリセレクト）→ 判定完了（`_execute`）で `completeCoveringFromCheck`（`attack-flow`）。成功＝`!fumble && !suitMismatch && success!==false`（目標値なしは success=null）。
- 成功＝攻撃カードの対象に **`coveredBy:{uuid,name}`** を付ける（`applyAttackTargetPatch`・非作者は `emitAttackUpdate` で GM 委譲）。攻撃カードは「「◯」がカバー」表示。**ダメージカード生成時（`buildDamageTargets`）に展開**＝元対象（被弾なし・`coveredBy` 保持で apply スキップ・表示）＋カバー行（`uuid`=カバーした側・`parryGuard:0`・`coveringFor`）。カバーした側が元々命中対象なら自分の行（受け値あり）は別に残る＝2ダメージ別々。カバー側の防御力は `targetPlannedPreview`/`openMitigationDialog` が `uuid` から解決＝自動でカバー側基準。
- カバーした側＝クリックしたユーザーの割り当てキャラ（アイテムロール元＝`_activateItemCheck` の actor）。リアクターの `resolveReactor` とは別（カバーはシート起動なので actor がそのまま実行者）。

#### 監査結果 ✅

- ルール解釈は 2026-07-16 のユーザー確定に一致。二重ダメージ（自分の分＝受け値あり／カバー分＝受け値なし）は、カバー行を parryGuard:0 の別エントリにし、元の対象エントリ（受け値あり）を残すことで別々に適用される（per-target 土台の自然な帰結）。

#### 課題・申し送り

- カバーの判定は通常の結果カードも出す（カバーの旨は結果カードには明示していない＝通知＋ダメージカードの付け替え表示で担保）。カバー結果カードの再判定は付け替えを再実行しない（`_buildRecheckContext` に covering を含めない＝表示のみ）。
- 実機確認済み（2026-07-21・v0.12.0）（CLAUDE.md §2.1・特に「カバー技能をアイテムロール→対象クリック→判定→付け替え」「防御力・受け値がカードの各行に出る」「適用ダイアログが手動/報酬点のみに縮小」の一連）。

#### 追補（2026-07-16 ユーザー指摘：防御力の合算源＝戦闘タブの合計に一本化）

Part A で防御力が各キャラ行の最終ダメージに載るようになった結果、「適切な防御力が引かれていない」とユーザー指摘。原因＝ダメージ算出の `aggregateDefence` は **armor/cyborg のみ**を合算しており、**搭乗中ヴィークルの防御力が抜けていた**（一方 戦闘タブの合計 `combatDefenceTotal` は armor/cyborg/**vehicle** を合算＝両者が乖離していた・[[feedback_trace_data_path_before_presentation]]）。ユーザー確定「適用されるべき防御力は戦闘タブに表示されている全防御力の合計」。→ ①`aggregateDefence` に **vehicle** を追加（`usable` 判定・`S_total??S_defence` の読みは元々同一なので、これで戦闘タブ合計と一致）。②**シートの `combatDefenceTotal` をインライン計算から `aggregateDefence(items)` 呼び出しに一本化**（同じ合算を2箇所で持たない＝将来の乖離防止）。Part A/カバーの軽減はこの `aggregateDefence` を通るため両方が是正される。テスト追加（ヴィークル防御力の合算）。全 1321 テスト green。**実機確認済み（2026-07-21・v0.12.0）**。

#### 追補（2026-07-16 ユーザー指摘：表示のはみ出し・冗長の撤廃／[[feedback_self_review_ui_output]] 起票）

実機で①カバーの各サブ行に長い接頭辞（「A（Bをカバー）:」）を繰り返し**はみ出し切れ**、②`.cr-tn`（`white-space:nowrap`）継承で注記が折り返せず切れる、③「対象2体」（カバーで受け手が付け替わった対象を数えていた）、④無意味な「」・「（リアクションしない）」等の冗長、を一括指摘（「UI設計はこの種の無駄が多すぎる／実機確認を都度は不可能／自分で見つけて全部直せ」）。是正:
- **対象ごとに見出し（名前）は1回**＝軽減内訳は名前を繰り返さない小注記に畳む（`.tnx-damage-sub`＝`white-space:normal`+`overflow-wrap:anywhere`。`cr-tn` は nowrap のため使わない）。
- **カバーの二重表示を解消**＝`buildDamageTargets` はカバーされた元対象をダメージカードに**載せない**（カバーした側の行だけ）。ダメージカードは被弾者のみ＝「対象N体」も正しい。被弾しない事実は攻撃カードの「〜がカバー」で分かる。
- **リアクションしない**＝「制御値」と成否だけ（「（リアクションしない）」を繰り返さない）。カード解決の実行者名は**肩代わり（reactor≠target）時のみ**添える。verdict は「被弾」（「（攻撃命中）」を削る）。
- **無意味な「」を撤廃**（カバー行/注記/攻撃カードのカバー表記の入れ子「」を除去）。**ただし ・区切りの対象一覧は「」を残す**（名前が ・ を含む場合の区切り＝以前のユーザー要望）。
- 全 1321 テスト green・eslint/stylelint 0 error。**実機確認済み（2026-07-21・v0.12.0）**。

### コンディション効果の無視ゲート（`ignore.*`・フェーズ12）

**日付**: 2026-07-16
**関連実装**: `helpers.mjs`（`parseEffectTargetKey` に `scope:"ignore"` 分岐）、`conditions.mjs`（`ignoreRuleMatches`/`gatherIgnoreRules`/`getEffectiveConditions`＝正規アクセサ・手動フラグ合流・`gatherPartSlotMods` を実効コンディション入力へ変更）、`tnx-check-flow.mjs`（`_gatherConditions` をアクセサ経由の非無視のみに）、`character-base.mjs`（制御値ペナルティ・部位スロットをアクセサ経由に）、`tnx-character-sheet-base.mjs`＋`bad-status-list.hbs`＋`css/tnx2.css`（無視中 BS の取り消し線＋供給元 tooltip・**バッヂのコンテキストメニュー手動トグル**）、`tnx.mjs`（AE キー datalist に `ignore.*`）。正本 [Conditions.md](../01_Wiki/Game_Mechanics/Conditions.md)「効果の無視ゲート」／[Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md)「無視ゲート」。
**ステータス**: ✅ 実装完了（全 1303 テスト green・eslint/stylelint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査・2026-07-16 ユーザー確定）

- 「特定の（あるいは全ての）ダメージや BS の**効果を受けない**」効果の表現。**ダメージ・BS を防ぐ／消すのではなく**、自分が受けているコンディションが与える**具体的な効果（数値ペナルティ・行動制限の両方）を消費段階で無視**する。タグ・BS・負傷は残る（シートにバッヂも出る）。※過程で私（Code）が「行動制限」を BS と別軸に切り出したのは誤り＝重圧・恐慌・捕縛・狼狽も BS。数値/制限で軸を割らない。
- **指定の粒度は個別・グループ・全ての全域**。「全〈系統〉ダメージの効果を受けない」で無視されるのは**その系統のダメージから生じた効果一式**（負傷自身の数値効果＋付随 BS＋付随戦闘不能）。※途中、ユーザー発言「ダメージ由来の BS」を「BS だけに限る」と誤って排除条件化した＝撤回済み（負傷自身の効果・戦闘不能も含む）。

#### 実装

- キー `ignore.all`／`ignore.bs`[`.<kind>`]／`ignore.damage`[`.<physical|mental|social>`]（マーカー・値不要）。照合 `ignoreRuleMatches`（all/group/kind/damage）。ダメージ由来系統は `woundCategory`（負傷自身）／`woundSource`→元負傷の `woundCategory` で辿る。
- **正規アクセサ `getEffectiveConditions(actor)`** が各コンディション行に `effectIgnored`/`ignoredBy` を付与。**実効果を読む消費点は必ず本アクセサ経由**（`readFlag` と同じ「必ず経由」規約）。既存消費点＝達成値/電子妨害/重圧ブロック（check-flow）・制御値/部位（character-base）。将来の消費点（恐慌/捕縛/狼狽/戦闘不能/邪毒＝13）も同アクセサ経由で自動ゲート。
- 表示＝無視中 BS の取り消し線＋供給元 tooltip。トークンのステータスアイコンは対象外（ユーザー確定）。
- **手動オーバーライド（卓ツール・2026-07-16 ユーザー確定）**: BS バッヂのコンテキストメニュー「効果を無視する／無視を解除する」で `flags…manuallyIgnored` をトグル。`getEffectiveConditions` の**第2ソース**として `effectIgnored` に合流（`manuallyIgnored || ルール一致`）＝消費側の抑止・取り消し線を追加実装なしで共有（tooltip「手動で無視中」）。可否ゲートなし・いつでも（RL/プレイヤーの手動裁定）＝ゲームルールでなく既存の手動修正ツールと同性格。

#### 監査結果 ✅

- ルール解釈は 2026-07-16 のユーザー確定（消費段階の抑止・タグは残す・粒度は個別/グループ/全ての全域・ダメージ由来は一式）に一致。実行時系統（判定バフ/ダメージバフ）と同じく恒常適用パスからは除外。手動オーバーライドはルール由来 AE と同じ経路に合流させ二重実装を避けた。

#### 課題・申し送り

- **消費点の網羅はフェーズ13 依存**: 恐慌（リアクション不可）・捕縛（武器不可）・狼狽（ムーブ不可）・戦闘不能（メインプロセス不可）・邪毒（継続ダメージ）の効果は 13 で消費点が実装される＝そのとき `getEffectiveConditions` 経由で書けば自動でゲートされる（据え置き＝先送りでなく設計上の依存）。
- 実機確認済み（2026-07-21・v0.12.0）（CLAUDE.md §2.1・特に「`ignore.bs` 付与で酩酊/衰弱等の数値が効かなくなる」「`ignore.damage.physical` で肉体由来 BS のみ無視」「バッヂは残り取り消し線が出る」）。

### 負傷そのものの直接効果（特定技能ペナルティ／使用不可／報酬点不可）

**日付**: 2026-07-16
**関連実装**: `damage-chart.mjs`（7 負傷 def に `skillPenalty`/`skillBlock`/`bountyBlock`/`sceneDeferred`・ヘッダ更新）、`conditions.mjs`（`gatherConditionCheckSources` に skillPenalty＋pendingScene 除外・新 `gatherSkillUseWarnings`/`hasBountyBlock`・`readConditions` に `targetSkill`/`pendingScene`）、`tnx-check-flow.mjs`（`skillKeys` を ctx へ・`open` の `_activationBlockReason` で②使用不可＋重圧・能力値判定を起動ブロック）、`tnx-character-sheet-base.mjs`（`bountyBlocked` で `bountyAvailable=0`・`_getAbilitiesData` の `checkBlocked` で重圧の能力値をクリック不可）、`condition-resolution.mjs`＋`tnx.mjs` フック（社会/コネ選択は `promptWoundSkillSelection` を **`createActiveEffect` フック**で実行＝付与経路を問わない・ドロー受付と同じ場所。休眠は `readConditions` が `def.sceneDeferred` から導出）、`tnx.mjs`＋`check-result.hbs`（`signed` ヘルパーでマイナス修正表記是正）、**`ui-trigger-disable.mjs`（新規・テスト付き）＝グレーアウト＋クリック不能の共通ユーティリティ `applyTriggerDisable`**・`_onRender` で技能(使用不可)＋能力値(重圧)トリガーに適用・`css/tnx2.css` の `.tnx-trigger-disabled`。正本 [Conditions.md](../01_Wiki/Game_Mechanics/Conditions.md)「負傷そのものの直接効果」。
- **2026-07-16 是正（ユーザー指摘）**: 選択ダイアログ・休眠を当初 `applyDamageChartResult`（ダメージ適用経路のみ）に置いたため、トークンのステータス効果トグルでの付与で出なかった。ドロー（衰弱/重圧）が経路を問わないフックで走るのに合わせ、選択をフックへ移設・休眠は def 導出に変更（付与経路を問わず機能）。
**ステータス**: ✅ 実装完了（全 1320 テスト green・eslint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査・2026-07-16 ユーザー確定）

- 対象 7 負傷（眼部損傷／口座凍結／造反／人脈消失／信用失墜／スキャンダル／信頼喪失）は BS カスケードでなく**負傷そのもの**が効果を課す（ユーザー確定「負傷そのものに効果を持たせる。そうでなければ意味がない」）。
- 「技能の使用」＝**判定**であり、**組み合わせ参加技能**もその対象。使用不可は**判定起動時に警告＋ブロック**（**2026-07-16 改訂**＝当初「警告のみ・続行」→重圧が完全ブロックなのに合わせてブロックへ。ユーザー指摘）。
- **重圧の是正（同日ユーザー指摘）**: 能力値判定は起動自体を不可に（対応能力値をクリック不可）。技能判定の重圧はスート＝能力値がカードで決まるためカードプレイ時ブロックのまま。
- **表示是正（同日）**: 判定修正のマイナス値が `+-n` になる不具合＝`signed` ヘルパーで `-n` に。
- **どの〈社会〉〈コネ〉が使用不可になるかは付与時にユーザーが決定**。候補は対象所持の該当技能を**識別キーのプレフィックス**（`society_`/`contact_`）で絞る。
- 次シーンものは既存 ignore とは**別方法で off**（休眠）にし、発火（次シーン）は据え置き。報酬点は使用不可の間、消費ダイアログ自体をスキップ（「消費できないのにダイアログが出るのはおかしい」）。

#### 監査結果 ✅

- 「特定技能」の識別を**識別キー**で行い名前照合しない（当初のユーザー懸念＝技能名識別への回答）。知覚=`perception`／信用=`stature` は compendium 実値を確認して確定（推測せず）。
- 3 メカニクス（数値ペナルティ／使用不可警告／報酬点スキップ）を着地点ごとに分離し、既存注入点（`gatherConditionCheckSources`・判定開始警告・`bountyAvailable`）へ最小結合。休眠は消費点の純関数除外で表現し ignore（無視＝取り消し線）と混同しない。

#### 設計レビュー

- 純関数（skillPenalty 照合／`gatherSkillUseWarnings`／`hasBountyBlock`／`readConditions` 拡張／pendingScene 除外）は TDD（RED→GREEN）でテスト先行・実キー非依存（任意キーで検証）。負傷 def→CONDITION_KINDS のガードテストで typo と配線を固定。
- 社会/コネ選択が**同期ダイアログ**でよいのは damage-flow の不変条件「適用者＝対象の所有者（または GM）」による（権限委譲不要）。この前提が崩れる呼び出し元が増えたら委譲が必要＝要再検討。

#### 課題・申し送り

- 次シーン発火（`pendingScene` 解除）＝シーン進行 13／時間管理 15。
- 実機確認済み（2026-07-21・v0.12.0）（CLAUDE.md §2.1・特に「〈知覚〉判定でチャート内訳に眼部損傷 −5 が **-5** 表記で出る」「使用不可技能の判定は起動時に警告＋ブロック（開かない）」「重圧の能力値がクリック不可・能力値判定が起動しない（制御判定は可）」「口座凍結中は報酬点ダイアログが出ない」「造反付与時に社会選択ダイアログが出る」「信用失墜など次シーンものは付与直後は効果 off・バッヂのみ」）。

### 用途シート・効果リストの軽微修正＋射程の幅（rangeMax）（フェーズ12 補）

**日付**: 2026-07-16
**関連実装**: 本体 `38bd7a7`（fix: UI整列=①効果リスト整列・②消費＋のlegend化・④カバーリング独立）・`dd9dea9`（feat: ③射程の幅）

#### メカニクス監査

- **カバーリング≠リアクション（ユーザー指摘）**: `covering` トグルを〈リアクション〉fieldset から独立〈カバーリング〉fieldset へ分離。テンプレートのみの変更でロジック不変（attack-flow / character-sheet-base は用途フラグしか見ない）。
- **射程の幅**: 「近～遠」等の幅を持つ武器の射程が、射程「武器」の解決で単一値（最長）に潰れて読めない（ユーザー指摘）。`rangeMax` を追加し幅のまま保存。**複数武器の交差ルール（下限=各下限の最長・上限=従来の最短実効と同値・交差不成立は従来の単点）は Code 提案の採用であってユーザーの明示裁定ではない**（計画承認 2026-07-16 に含めて実装・異なる裁定が出たら是正）。正本 [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「組み合わせ後の射程」。

#### 設計レビュー

- ①効果リスト: ヘッダー（＋1個）と行（3ボタン）で `flex: 0 0 auto` の操作列幅が変わり「効果時間」列（固定80px）がずれる→操作列を固定幅・右寄せに。汎用 `.tnx-item-list__column--controls` は usage-list も使うため触らず、パーシャルに `tnx-item-list--effects` 修飾クラスを足してスコープ（CSSスコープ厳守）。
- ②消費の行追加＋: **fieldset 全体が行集合のセクションは legend 内に＋が規約**（部位エディタ・スタイル技能の技能/対決と同型。消費セクション自身が CSS コメントで部位エディタを参照）。fieldset 内サブフィールドの行集合（回復対象の範囲・判定ボーナス行）は構造が別＝対象外。
- ③ `resolveAttackRangeSpan` は純関数＋テスト。既存の単一値解決 `resolveAttackRangeValue` は span の上限へ委譲し**挙動不変**（一致性テストで固定）。物理射程以外へ変更時の `rangeMax` リセット・幅セレクトの出し入れは明示再描画（submitOnChange は再描画しない）。
- **経緯（再発防止）**: ②の比較対象を用途シート内の2箇所だけから「下置きが慣例」と誤って一般化し、ユーザーの2度の指摘で是正。教訓＝**「同様のセクション」は表面（行＋追加ボタン）でなく構造の所有関係（fieldset=行集合か・fieldset 内サブフィールドか）で括り、システム全体を全数確認してから規約を主張する**。

**ステータス**: ✅ 実装完了（1332 テスト green・eslint 0 error・stylelint clean）・実機確認済み（2026-07-21・v0.12.0）

### 参照整合性監査と判定起動系の一本化（フェーズ12 補・2026-07-16）

**日付**: 2026-07-16
**発端**: ユーザー指示「場当たり実装で専用関数があふれている印象。目標値は用途の目標値欄を、シート表示・保存値をフローが本当に参照しているか（戦闘タブの武器/防御力の過去バグと同型の探索）を監査し、一本化できるものは提案せよ」
**関連実装**: 本体 `ff2653e`（スタン×事後修正の順）→`3ffe323`（用途ピッカー）→`bc35f61`（共通前段）→`ae6ef2b`（継続レジストリ）→`a466a47`（messagePatch）→`0061ca7`（target-resolution）→`ee5443b`（KI-024 是正）→`8f560a5`（報酬点ブロック一元化）

#### メカニクス監査（監査結果と裁定）

- **正しく配線済みと確認**: 防御力（戦闘タブ＝ダメージ算出とも `aggregateDefence` 単一実装）・攻撃武器（`resolveAttackWeapons`＋実効値）・パリー受け値（`guardValue.total`＋生身フォールバック）・目標値の解決自体（`resolveUsageTargetValue` 一本化・治療の負傷TN上書き・回復の `@condition.*` 注入）。
- **裁定①（情報収集判定の報酬点・ユーザー verbatim）**: 「情報収集判定は、『報酬点使用可能』フラグに関わらず報酬点を消費可能なので、それで正しいです。」＝従前実装（無条件付与）が正。→ [Bounty_Points.md](../01_Wiki/Game_Rules/Bounty_Points.md) に記録。
- **裁定②（スタン/説得×事後修正・ユーザー verbatim）**: 「ダメージ算出の最後に10にそろえられて、ダメージ算出後～ダメージ適用前に事後修正が乗ります。」＋追補「防御力と受け値は以前修正したように、ダメージ算出のタイミングで行われる」＝KI-024 の裁定と同一（恒久軽減→10上限＝算出の一番最後→事後修正→適用時軽減）。**経緯**: Code は当初、事後修正だけ上限後へ移し上限と恒久軽減の順を旧のまま実装（`ff2653e`）→ユーザー指摘で正本（Damage_Rules「算出の適用順序」・KI-024）に当て直し `ee5443b` で是正。教訓＝**裁定の実装前に正本の該当行を引用して逐条一致を確かめる**（§5.1。今回は wiki に前セッションの記録が既にあり、先に読んでいれば一度で正の順序に達した）。
- **取りこぼし（過去バグと同型・全て是正済み）**:
  - 報酬点使用不可（口座凍結/信用失墜・2026-07-16 追加）が技能クリック経路にしか無く、攻撃/NPC取得/回復/情報収集で漏れ → 消費時点の一元ゲート（`_execute` の報酬点ダイアログ直前）へ移動（`8f560a5`）。
  - NPC取得（useCheckAcquire）が判定ボーナス（checkBonusSelf/checkBonuses＝達成値が取得人数に直結）・適用効果・目標値欄・再判定可能を読み落とし → 共通前段へ載せ替えで解消（分身の目標値10はルール固定のまま）。
  - 回復（useRecovery）が適用効果・再判定可能を読み落とし（行内コメント「継続処理のため再判定対象外」は 2026-07-15 全種対応と矛盾する古い記述）→ 同上。
  - カバー（covering）が再判定の継続キーリストに無く、再判定で継続が引き継がれない → 継続レジストリ化で解消。
  - 用途ピッカーのアイコン分岐が廃止済み type `npcAcquire` を参照（死に分岐）→ フラグ参照へ。

#### 設計レビュー（一本化の構造）

- **判定起動の共通前段 `buildUsageCheckContext`（`usage-check-context.mjs`・新規/テスト付き）**: エキストラ制限→不備検知→参加技能解決（`resolveUsageSkillSet`＝旧 `_resolveSkillSet` を移設）→報酬点 usesBounty→消費確認→適用効果→目標値。技能クリック・攻撃・NPC取得・回復が共用し、各フローは固有前処理＋固有ペイロードのみ。**アイテムシートの使用ボタン（第2ディスパッチャ）も `_activateItemCheck(actor, item, {usageId})` への薄い入口に統合**（カバー・固定値分岐が使用ボタン側に無かったドリフトを解消。宣言使用は `_useDeclarationUsage` として移設・check 用途は使用ボタンからも判定起動＝挙動変更はこの1点）。
- **完了継続レジストリ `TnxCheckFlow.CONTINUATIONS`**: 継続キーと再実行ポリシーの正本。スナップショット・startRecheck 引き継ぎ・再実行の3箇所をここから導出（種別追加は本表＋`_execute` の2箇所と明文化）。
- **メッセージ更新委譲 `TnxSocketHandler.applyMessagePatch`＋ソケット `messagePatch` 1種**: 「GM/作者は直接 update・他は委譲」の複製5箇所（攻撃/ダメージ/リアクションカード・事後修正/再判定置き換え・BSドロー・効果適用済み）を統合。旧 attackUpdate/damageUpdate/usageEffectApplied/checkModify 廃止（受理は自スコープ flags＋content のみ＝旧 checkModify の検証を維持）。
- **ターゲット解決 `target-resolution.mjs`**: 攻撃（全ターゲット→未選択はトークン選択＋レティクル付与）・回復（1体→自分確認）・適用効果（全件→自分確認）の3実装を部品化（挙動不変。tnx-dialog はモジュールレベルで foundry を参照するため動的 import＝テスト独立を維持）。

**ステータス**: ✅ 実装完了（全 1342 テスト green・eslint 0 error）・実機確認済み（2026-07-21・v0.12.0）（特に「スタン宣言・防御力持ち対象で対象行に『スタン（10上限）』が出て最終値が 10 を超えない」「口座凍結中は情報収集判定でも報酬点ダイアログが出ない」「アイテムシートの使用ボタンから攻撃/判定が技能クリックと同挙動」）

### 社会ダメージの報酬点軽減＝リアクション成立ゲート＋クリック方式（フェーズ12 補・2026-07-17）

**日付**: 2026-07-17
**発端**: ユーザー確認「報酬点軽減は『リアクションによる判定が成立していたら』起動のはず。満たされているか」＋「厳密にはダメージ軽減技能と同タイミング（適用前）＝クリック方式への変更の判断を仰ぐ」→ Code が変更を推奨・ユーザー承認（「変更してください」）
**関連実装**: 本体 `77e161b`（attack-flow=対象行に `reactionEstablished` 保存・damage-flow=`promptBountyMitigation`＋適用ダイアログの報酬点欄撤去）・wiki `97517a2`（判定の成立の一般定義）

#### メカニクス監査

- **監査結果（確認への回答）**: 満たされていなかった。旧実装は①成立条件を見ず社会なら常に報酬点欄を表示 ②成立情報がダメージカードの対象行に運ばれていない（parryGuard だけ特別扱い） ③報酬点の実減算なし・上限も所持数でなく 99 の自由入力——の3点が不足（監査テーマ「保存値とフローの断絶」と同型）。
- **裁定（判定の成立・一般定義＝ユーザー verbatim 2026-07-17）**: 「そもそも判定の成立とは『ファンブル/スート不一致でなければ成立（勝敗は問わない）』です。パリーがどうとかリアクションがどうとかは関係ありません。」＝判定全般の一般概念（Check_Rules に記録済み）。Code が「パリー受け値と同じ定義で良いか」とメカニクス固有定義であるかのように問い、訂正された。
- **裁定（タイミング・方式）**: 報酬点軽減はダメージ軽減技能（事後修正）と同タイミング＝算出後〜適用前。社会には10上限・恒久軽減が無いため**数値結果は不変**（段の移動は機構の正確化）。導線は対象行の数字クリック（対象の所有者/RL）・実減算・適用ダイアログの欄は撤去。

#### 設計レビュー

- 成立は `completeReactionFromCheck` の既存 `ok`（=!fumble && !suitMismatch）をそのまま対象行 `reactionEstablished` に保存（新しい判定はしない＝一般定義の単一実装）。再判定の仕切り直しでは初期化・カバーした側は「自分ではリアクションしていない」＝受け値と同じ理由で不成立扱い。
- 軽減は per-target 事後修正（`t.mods`）へ `{label:"報酬点による軽減", value:-N, bounty:true}` 行＝既存チャネル（modifyDamage 防御側・applyMessagePatch 委譲）を再利用し新機構なし。bounty マーカーで内訳表示を「報酬点 −N」に分離（適用順=防御力・受け値→10上限→修正→報酬点→手動）。
- 実減算は防御側自身のクライアントで自アクター更新＝権限が自然。報酬点使用不可（口座凍結/信用失墜）は宣言時に消費時点ゲート（判定の報酬点と同じ規約）。
- `AmountInputDialog.prompt` は allowOverride なしでは**数値を直接返す**（オブジェクトでない）＝実装時に仕様確認して捕捉（`.value` 参照だと常に 0 で沈黙するバグになるところだった）。

#### 課題・申し送り

- ~~**判定での報酬点使用（bountyUsed）は自動減算されていない**（手動記帳前提の現状実装）。軽減側だけ自動減算になったため、判定側も揃えるかは**ユーザー確認待ち**~~ → **解消（2026-07-17 ユーザー裁定「当たり前ですが、自動減算してください」・本体 `ce4fb35`）**: 判定側も使用確定時に `system.bounty` から自動減算。未実装だった経緯（フェーズ8 のエンジン先行実装で記帳側が落ち、「減算する」という仕様行がどこにも無いまま、以後の監査が可否・量のみ検査して約10フェーズ残存）は [Bounty_Points.md](../01_Wiki/Game_Rules/Bounty_Points.md)「消費の記帳」に記録。
- 実機確認済み（2026-07-21・v0.12.0）: 「社会攻撃でリアクション成立の対象行だけ数字がクリック可能」「クリック→報酬点入力→所持報酬点が減り行内訳に『報酬点 −N』」「適用ダイアログに報酬点欄が出ない」「口座凍結中は警告」「**判定で報酬点を使うと所持報酬点が減る**」。

**ステータス**: ✅ 実装完了（全 1342 テスト green・eslint 0 error）・実機確認済み（2026-07-21・v0.12.0）

### 受けるダメージ軽減 AE（damage.taken / damage.fromStyle / damage.fromWorks）（フェーズ12 補）

**日付**: 2026-07-17
**関連実装**: `scripts/data/item/helpers.mjs`（`parseEffectTargetKey` に damageTaken/damageFrom スコープ・`damageTakenChangeMatches`／`gatherDamageTakenSources`・`_gatherBonusSources` に採用方向 pick=min）、`scripts/data/actor/common/character-base.mjs`（恒常適用パスの除外セットへ追加）、`scripts/module/damage-flow.mjs`（`collectDamageTakenRows`＝対象行プレビュー `targetPlannedPreview`・適用ダイアログの恒久軽減へ合流・内訳表示・適用済み表示の新形式=内訳文字列保存）、`scripts/tnx.mjs`（AE 設定 datalist に dealt/taken 固定キー）。テスト: `tests/data/item/helpers.test.mjs`（TDD・RED→GREEN で追加）。キー正本は [Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md) §2.8。
**ステータス**: ✅ 実装完了（全 1346 テスト green・eslint 0 error）・実機確認済み（2026-07-21・v0.12.0）

#### ルール解釈（メカニクス監査）

- **恒久軽減は全て「ダメージ算出時」・スタン/説得の 10 上限は算出の一番最後**（2026-07-16 ユーザー裁定）に従い、taken 系は防御力・受け値と同段＝**10 上限の前**に効く。事後軽減（modifyDamage）・適用時の手動軽減とは別系統。
- **スコープ（2026-07-17 ユーザー指示）**: 系統別（physical/mental/social）に加え、**肉体のダメージ種別（S/P/I/X）別**・**特定スタイルを持つ攻撃者からの軽減**。`fromWorks`（ワークス条件）は vsStyle/vsWorks の対称として **Code 判断で併設**（不要なら削除可）。
- **符号規約**: 値=受けるダメージへの加算（**負=軽減**・正=増加も同キー）＝ `damage.dealt` と対称（2026-07-16 に Code 推奨として提示→実装指示で確定）。
- **X（装甲無視）にも効く**: X が無視するのは「対応する防御力」のみ（受け値・手動軽減が X でも効く既存実装の前例に整合）。
- **派生ダメージには効かない**（軽減不可＝軽減を挟まない経路のため構造的に除外）。**「別のダメージとして適用」でも元系統で照合**（軽減は元系統のまま＝既存裁定）。
- ~~**重複適用不可の一般原則の適用方向**: 「最も有利な値」は効果の保持者＝**受け手基準で最小値**（最も負）。taken と from* が同一効果に併記されても 1 回だけ効く（1 回の走査で束ねる）。~~ → **誤り・同日是正（下の追補）**: 禁止は「同名効果の二重適用」（インスタンス単位）であり、**1つの効果の行はすべて内容として合算**する。インスタンスの強者採用（受け手基準=最小値）のみ正しい。

#### 設計レビュー

- 集計は**受け手（対象）の effects**から＝ `collectActorEffectBuffs`（自動適用ゲート・付与コピー込み）を共用し、恒常はキャラ/アイテムの自動適用 AE・一時バフは用途の「適用される効果」で付与＝**新規の付与機構は不要**。
- **対象ごとに評価**するため、攻撃側 `vs*` の「先頭命中対象で近似」問題は生じない（from* の攻撃者は `attackerUuid` から解決・未解決時は from* 不適用）。
- 表示: 対象行の内訳（効果名 −N・符号つき）・適用ダイアログのラベルを「防御力・受け値（適用済み）」→**「軽減（算出済み）」**へ（AE も含むため。内訳は符号つきに統一・合計は signedDisplay で負の軽減=増加も破綻しない）・適用済み表示は**内訳文字列（mitigationParts）を保存して表示**（旧カードは合計のみの旧表示へフォールバック）。
- 純ロジック（パーサ・照合・pick=min の重複排除）はユニットテストで固定。フロー配線（対象行・ダイアログ）は Foundry 依存のため実機確認に委ねる。

#### 課題・申し送り

- ~~**攻撃側 `dealt`＋`vs*` の別走査**: 併記すると二重加算になりうる（一般原則と不整合の疑い）~~ → **撤回（同日是正・下の追補）**: 行はすべて効果の内容として適用されるため、併記で両方効くのは正しい挙動。二重加算という指摘自体が Code の誤りだった。
- 実機確認済み（2026-07-21・v0.12.0）（CLAUDE.md §2.1）: 対象行の「効果名 −N」内訳・スタン 10 上限との順序（軽減後キャップ）・fromStyle/fromWorks の照合・適用ダイアログの「軽減（算出済み）」表示。

#### 是正追補（2026-07-17 同日・重複適用の一般原則の誤りを是正）

Code の「dealt×vs\* 併記は二重加算」という申し送りに対しユーザーが「なぜ二重加算になるのか」と二度質し、Code の根拠（「1効果はダメージに1回しか寄与しない」という §4 旧記述の読み）が崩れたため、**原則そのものをユーザーが是正**した:

- **禁止されるのは「同名効果の二重適用」**＝同じ技能から AE を二回付与されることはない（インスタンス単位の規則）。BS も「重複する」と明記されたもの以外は重複せず、**最も効果の大きいものが常に適用**される。**重複可（stackable）は例外**＝「この効果は重複する」と明記された重ねがけ型バフは付与された分だけ累積する（以上ユーザー裁定 verbatim ベース）。
- 帰結: **1つの効果が持つ変更行はすべてその効果の内容として適用（合算）**。「効果内で複数行が該当したら最有利1つ」という §4 旧記述（フェーズ9 記録）は Code の過大一般化＝誤記録だった。
- **実装是正**: `_gatherBonusSources` を「行の畳み込み（max/min）」→「**行の合算 → 同一 identity のインスタンスは最も効果の大きい1つ**（攻撃側=最大・taken 系=最小）」へ変更（判定バフ・dealt/vs\*・taken/from\* の全共用点に効く）。挙動変化の例: 1効果に `check.all +1`＋`check.melee +2` → 白兵 +3（旧 +2）／`damage.taken −2`＋`damage.taken.S −5` → 斬撃 −7（旧 −5）／`damage.taken −2`＋`damage.fromStyle.x −6` → −8（旧 −6）。インスタンス単位の強者採用・stackable 累積・「1行が複数該当しても1回」（バサラ/マヤカシ+1）は不変。テスト 1347 green。
- 残エッジ（据え置き・要すれば別途）: 値バフ `_applyEffectBuffs` の「1効果内で同一パス・同一モードへ複数行」はキー付き重複排除で畳まれる（合算されない）／攻撃側 `dealt`×`vs*` の2走査は、同名効果の**複数インスタンス**が系統違いの行を持つ場合にインスタンス跨ぎで行が混成しうる（taken 系は1走査合算で回避済み）。
- 教訓: 記録された一般原則の**適用範囲**（インスタンス単位か・行単位か）を取り違えたまま既存挙動を「バグの疑い」と申し送った。原則の主語（何の二重を禁じるか）を正本の裁定に取り直してから評価する（[[feedback_approval_records_verbatim]] と同根＝§4 の旧記述自体が Code の過大一般化の混入だった）。

### 用途タイプの行動種別再編＋対決システム是正（フェーズ12 補・2026-07-17・本体 `238d775`）

#### メカニクス監査

- **対決の解釈是正（ユーザー裁定・[Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)「対決の解釈（是正 2026-07-17）」が正本）**: 対決欄は拘束リスト（一般則）。旧「無印技能名=付与・例示」（2026-06-20 確定）は一般則と細則の取り違えで撤回。手段行（ドッジ/パリー/各リアクション）=リアクション用途タイプと1:1・技能名無印=どれか1つをコンボに含む・※=必ず含む・不可=マスク（下地の行は無視・無効化に備え並存保存）・組み合わせは合算（和集合・吸収）。
- **タイプ=行動種別16種**（ユーザー確定）: 特殊な識別（技能クリック以外からの起動で可能技能を探す）が要る行動だけをタイプ化。「対決」タイプは不採用（対決性は対決欄の内容で判別・タイプでは判別しない）。治療は1タイプで判定/宣言両形式（executionForm で固定）。NPC取得・情報収集・登場判定はタイプにしない。
- **リアクション資格**: 該当用途タイプをレベル1以上の技能で所持（クリック時に弾く・導線は隠さない）。「対決不可にもリアクション可」は用途フラグ（下地の拘束は受ける）。ヴィークル操縦中のリアクターは対決が「〈操縦〉※」になる（唯一の特殊処理・リアクター側状態由来）。
- **物理攻撃の白兵/射撃**（ユーザー確定）: 射撃攻撃は純粋な生身では行えない＝「射撃武器」区分フラグの武器を準備していなければ判定不可。生身=白兵武器。区分は分類でなくフラグ（搭載兵器・生体装備にも区分があるため）・既定は分類から敷く（白兵武器→白兵/射撃武器・搭載兵器→射撃/生体装備→白兵・未指定=両OFF可）。
- **移動/リアクション（移動妨害）**: 使用ヴィークル単一参照・準備済み無しは判定不可。移動へのリアクション勝利＝移動失敗（本会話の発端の裁定）。リロードは射撃武器の用途（マイナー+残弾へのマイナス消費）で表現＝消費先「残弾」（負値=回復）。
- 判定に迷った点（Code 設計として記録・要すれば裁定）: ヴィークル特例で「不可」マスクを残す／対決欄が空の攻撃は制御値で即確定／オープンリアクションの「リアクションしない」確定者=実行者/RL。

#### 設計レビュー

- 正本レジストリ `usage-types.mjs`（USAGE_TYPE_DEFS）と対決純ロジック `confrontation-logic.mjs`（判別・合算吸収・選択肢）を新設（テスト16件）。旧 `skillRoles`・正準名ブリッジ・`REACTION_DEFAULT_SKILL`・`attackReactionModes` は全廃し、資格・候補は用途タイプの所持に一本化。
- **対決判定カード=攻撃カードの一般化**: 永続フラグ名 `attackCheck` は互換のため維持し `isAttack`/`usageType`/`confrontation` を追加。攻撃専用項目（武器・制御値・ダメージボタン・カバー）は isAttack ゲート。対象なしの対決は `openReaction`（カード上の先着リアクション・再判定/事後修正でも保存値で再解決）。移動は対決時=対決判定カードの条件行・非対決時=従来の移動カード（`ctx.movement`）。
- **移行**: usage の migrateData（攻撃/カバー/回復フラグ→タイプ・isUnopposable→不可行・系統既定の対決行敷設）＋ ready の一回限り正準名移行（ワールド設定ゲート・`usage-type-migration.mjs`）＋インポート時正規化（preCreateItem）。武器区分フラグは outfit-base の migrateData＋分類変更時の preUpdateItem 敷き直し。**是正（2026-07-18・本体 `a5aa574`）**: インポート時正規化（preCreateItem 常設）は撤去——移行済みかを区別せず、辞典が正当に持つ「判定」用途をインポートのたびに付け替えていた（KI-029。ユーザーへの明確な報告なしに正本データへ副作用する設計だった点も含めて是正）。移行は ready の一回限りのみ。
- 戦闘タブは用途 timing 駆動の用途起動ボタンへ（合成の移動/リロード行を撤去・`_onStartUsageUse`）。用途名は既定空＝実効名は親アイテム名（`usageDisplayName`・placeholder 表示）。
- テスト 1360 green・eslint/stylelint エラーなし。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**（CLAUDE.md §2.1）: 用途シートの対決行カスケード・攻撃/移動/離脱の対決フロー一巡・オープンリアクション・正準名移行の実データ適用・残弾リロード用途。
- 旧チャットカード（confrontation を持たない攻撃カード）の未解決リアクションカードはボタンが「リアクションしない」のみになる（過渡・チャットは使い捨てのため許容）。
- 白兵/射撃の migrate 既定は「白兵」（旧攻撃用途は区分を持たないため）。射撃技能の攻撃用途は手動で「射撃攻撃」へ切り替えが必要。
- KI-025（判定要求のコンボ用途選択）は本再編後に対応予定（起票済み）。

### 2026-07-18 大規模整理（表示一本化・辞典参照・ベース解決・残弾/FA・消費/神業）

#### メカニクス監査

- **技能名の表示**（ユーザー確定）: アイテム名欄・アクターの技能リスト以外は必ず 〈〉 で囲い、秘技/奥義/演出特技の識別マーク（†・※・@）を省く。**用途名の技能名部分は例外＝〈〉で囲わない**（親アイテム名は素の名前）。
- **ベース技能**（ユーザー確定）: アクション/非アクションを問わず用途設定時にベースを解決・永続化する（新機構を作らず既存経路を一般化）。指定技能のあるスタイル技能は親をベースにしない（末端がベース）。ベースが別技能のとき親自身が参加技能へ自動参加（アクター非所持でも辞典参照で成立）。
- **精神/社会攻撃**（ユーザー確定）: 物理武器を持たない＝「武器なし」。「射程：武器」は解決不能なので射程候補から除外（至近/生身に化けさせない）。残る参加技能の実射程（＝最も上流のベース技能の射程）を採り、無ければ射程未設定。「武器なし」は表示ラベルのみ（武器アイテムは作らない）。
- **残弾/FA**（ユーザー確定）: 残弾「任意」廃止＝FA武器等は残弾1。残弾の消費・回復は自動では行わず用途の消費のみ。FA ダメージ自動化を廃止＝FA 値は用途のダメージボーナス式（@item.<識別キー>.system.FAValueTotal）で手動加算に一本化。
- **消費**（ユーザー確定）: 先頭「アイテム/AR」の2択、アイテムは「このアイテム自身/各アイテム」＋「使用回数/残弾数」の二段選択。「親アイテム」表記を「このアイテム自身」へ。
- **神業の使用回数**（ユーザー提案・合意）: 汎用 uses へ一本化（特例カウンター・特例消費 kind を廃止）。母数=uses.max=連動スタイルレベル（上限3）。母数バフ（ファイト！等）は AE で uses.max を増やす。

#### 設計レビュー

- **〈〉一本化**: `formatSkillName` を唯一の〈〉デコレータに。`resolveComboSkillName` は素の名前を返し、スタイル技能説明タブが自前で二重に囲っていた `〈…〉` を是正（**二重〈〈〉〉解消**）。
- **同輩技能解決の辞典統一**: `resolveUsageSiblingSkills` のワールド/辞典分岐を撤去し `loadDictionarySkillItems`（getIndex 軽量読み＝文書インスタンスを作らず KI-026 の孤児化回避）へ。
- **ベース解決の集約**: `effectiveBaseSkillId(usage,item)` を新設し散在フォールバックを一本化。`_enforceComboRequirements`/`enforceUsageChainDefaultsOnImport` で常に永続化。非ロックのベース候補に自己を含め base=親を明示表示。
- **残弾/FA**: weapon.ammo の choices から arbitrary 撤去（migrate: arbitrary→value:1）。weapon-ammo.mjs を value 専用に簡素化し consumeNormalAmmo/consumeFaAmmo を撤去。attack-flow/damage-flow の faOptions/faValue/consumeFaAmmo/FA台帳行/攻撃カード「FA可」を撤去。combineWeaponAttack は faOptions を返さない。
- **消費/神業**: consumeTargets を {type,itemId,resource,amount} へ・migrateData で旧 type を移行。miracle.mjs を uses スキーマ＋usageCount→uses の migrateData に。連動フック（tnx.mjs preUpdate/preDelete）を uses.max 駆動へ。character-sheet-base の神業取得(+1母数)・削除(-1母数)・使用(spent+1)・表示スロット、miracle シートの残り増減（spent 調整）を uses へ更新。**旧 value/total の解釈矛盾（用途経由の神業消費が母数を削る潜在バグ）を解消**。
- テスト 1377 green・eslint/stylelint エラーなし。隔離 Foundry＋Playwright の実機で: 用途名〈〉なし・二重〈〉解消・ワールド直下の辞典参照(17技能)・ベース自己永続化/指定技能=末端/親不採用・神業 uses(消費で spent+1)・消費ブロックUI(このアイテム自身)・残弾 choices(none/value) を確認。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**: 神業の**スタイルレベル→母数(uses.max)連動**と**ファイト！等 AE による uses.max 増加が消費の実効 max に反映されること**（連動フックは書き換え済み・母数バフの AE 経路は要実機確認）。物理攻撃の残弾が用途消費のみで増減すること（自動消費撤去後）。消費ブロックの item→resource 二段連動（資源候補の絞り込み）の操作感。
- 既存の攻撃用途は残弾消費行を設定するまで残弾を消費しない（ご指示どおりの挙動）。


### 2026-07-18 アウトフィットの故障/破壊と「修理」用途

#### メカニクス監査

- **故障/破壊は使用不可状態**（ユーザー確定）: どちらも受けたアウトフィットを使用不可にする。**故障**は対応する〈製作〉での修理が可能・**破壊**は基本アクト終了まで直らない。どちらもアクト間に持ち越さない（消費アイテムと同じ）。**自動リセット機構は将来フェーズへ送り、今回はフラグと付与手段（手動トグル/AE）＋修理のみ**（ユーザー指示「自動リセット機構は作るが先フェーズに送る。先にフラグと与える方法だけ作る」）。
- **サービス大分類は免疫**（ユーザー確定）: 大分類 `service` のアウトフィットは故障も破壊もされない。実効フラグを常に false に落とし、トグルも出さない。
- **サービス/バックグラウンドの準備固定**（ユーザー確定）: 小分類 `background` は必ず準備・携帯され未準備にできない。`isPrepared`/`isCarrying` を派生で true 固定し携帯/準備トグルを両シートで非表示。`isPartless`（部位なし=準備不可）より優先。
- **修理の対象判定＝用途のホワイトリスト**（ユーザー選択「用途にホワイトリスト」）: 修理用途は `repairableCategories`（小分類キーまたは大分類キー。大分類キー=その大分類の全小分類＝2026-07-19 ユーザー指示で追加）に合致する対象所持の故障アウトフィットのみ列挙し、判定成功で1つの故障を解除する。破壊は修理不可。
- **ロール禁止の範囲＝武器を使う攻撃も禁止**（ユーザー選択）: 故障/破壊アウトフィット自身のロールに加え、その武器を「使用武器」に持つ攻撃判定も不可（useAttack が弾く）。

#### 設計レビュー

- **故障/破壊は AE フラグ機構へ相乗り**: `isMalfunction`/`isDestroyed` を `AE_FLAG_PARAMS` に登録し、実効読みは既存 `readFlag`。照合は helpers.mjs の純関数 `isOutfitServiceImmune`/`isOutfitMalfunctioning`/`isOutfitDestroyed`/`isOutfitUnusable` に集約（サービス免疫はこの4関数が唯一の権威。prepareDerivedData の実効 false 化は表示一貫性のための belt-and-suspenders＝AE が後段で上書きしても helper 側で免疫が保証される）。読み手（アクターシート行・ロール禁止・攻撃武器・修理列挙）は全てこのヘルパー経由。
- **修理は治療/回復フローの平行実装**: `repair-flow.mjs`（useRepair→対象解決→選択→buildUsageCheckContext→TnxCheckFlow.open）＋完了継続 `CONTINUATIONS.repair`（rerunOnSuccessOnly・再判定に引き継ぎ）＋`repairApply` ソケット（treatmentApply と同型の GM 委譲）。起動は唯一の起動関数 `_activateItemCheck` の修理ゲート（治療ゲートの直後）から。既存前例（recovery-flow の選択ダイアログ・target-resolution・postConditionOutcome・socket 委譲）を踏襲し新機構を作らない。
- **用途シート UI は同シートの前例踏襲**: `repairableCategories` の設定は回復の除外タグ（選択済みリスト＋追加セレクト）と同型。追加は大分類 optgroup＋小分類 option の1セレクト（サービス除外）。**2026-07-19 拡張**: 各 optgroup 先頭に「（大分類全体）」option（値=大分類キー）を追加し、大分類・小分類キーを同一配列に混在保存（スキーマ変更なし。照合は minor/major 両対応・大小のキー名前空間は不衝突＝outfit-categories にテストで固定。回復の「（グループ全体）」と同型の前例踏襲）。
- **アクターシート表示**: 行ビルダで実効判定し `brokenClass`（destroyed=グレー優先/malfunction=赤）＋`hidePrepareToggles`（background）を付与。CSS は outfits タブにスコープ。
- テスト 1391 green（helpers の isOutfit* 4関数・outfit-base の派生 免疫/準備固定・用途タイプ17種）・eslint 0 error・stylelint clean。

#### 課題・申し送り

- **アクト終了時の自動リセット（故障/破壊の一括解除）は未実装**（将来フェーズ・ユーザー指示で先送り）。現状は手動オフ運用（消費アイテムの個数リセットも同様に未自動）。
- **実機確認済み（2026-07-21・v0.12.0）**: 修理用途の一巡（〈製作〉に修理用途＋分類ホワイトリスト→故障アウトフィット選択→判定成功で赤取消線が消える）・AE による故障/破壊の付与と実効反映・攻撃武器の故障ブロック・バックグラウンドの準備固定＋トグル非表示。


### 2026-07-18 治療の用途一本化（治療メニューの照合化・目標値の設定化・代用判定廃止）

#### メカニクス監査

- **治療の実行経路を治療用途（usage type="treatment"）へ全面一本化**（ユーザー確定）。「治療」メニューは入口にすぎず、実体はアイテムロールと同じ用途起動（recovery-flow）。
- **治療者は常に起動した利用者の担当キャラクター**（ユーザー確定「治療は自分もしくは他者に治療が可能な技能を用いて行うので、治療用途を持つ技能を持っているのは常に自分自身」）。範囲適合する治療用途が無ければ「治療できる技能が無い」＝治療できない。**代用判定は廃止**。Code の「治療者候補の絞り込み」提案は、治療者選択という概念ごと不要としてユーザー指摘で撤回。
- **チャート値＝そのままダメージの数値**（ユーザー確定）。目標値の分岐（気絶/失神=15・仮死/昏睡・支配=20）は**完全に用途上の設定**で行う: kind 限定範囲＋数字目標値の用途で表現し、**通常ダメージ用の治療用途は戦闘不能系タグ全8種を除外に入れる設定規約**（ユーザー確定。除外の inflicts 展開により戦闘不能を伴う負傷が通常用途から自動脱落）。式キーは新設せず既存 `@condition.woundValue` がチャート値と意味的に一致（入力先を二重化しない）。
- **社会負傷への治療メニュー開放**（ユーザー承認）: ハードブロック撤去・可否は範囲設定が正（社会を範囲に持つ専用スタイル技能の経路と整合）。
- 照合単位はダメージインスタンス（負傷＋紐づき戦闘不能/支配の kind 集合）＝負傷行とタグ行のどちらをクリックしても同じ用途に合致（2026-07-09「戦闘不能はダメージそのもの」と整合）。Code が提案した「コード側の戦闘不能優先則」は上記除外規約により不要となり撤回。

#### 設計レビュー

- `treatment-flow.mjs` を入口照合に特化して全面書き換え: `resolveTreatmentKinds`（インスタンス展開）→`usageCanTreatKinds`（conditions.mjs の純関数・範囲/除外の既存関数を合成）→`_activateItemCheck(担当キャラ, item, {usageId, treatment:{patientUuid, effectId}})`（起動一本化規範を維持）。複数合致（タグ改変 AE の追加タグは静的除外に掛からない等のまれな場合）のみ TargetSelectionDialog で用途選択。
- `useRecovery(item, usage, prebound)` に prebound 文脈を追加: 患者とクリック状態が確定済みなら対象解決・回復対象選択をスキップ。**クリック状態単体の範囲は再検査しない**（負傷行クリック＋タグ範囲用途の組み合わせが正当にあるため＝照合は入口のインスタンス集合が正）。
- 撤去: `INCAP_TARGET_VALUE`・`resolveTreatmentInstance`（目標値算出込み）・`pickTreater`・`promptSubstituteSkill`・完了継続 `ctx.treatment`（`_execute` 分岐と `CONTINUATIONS` エントリ）＝継続は `ctx.recovery` に統一。`substitution`/`manualMod` は判定要求（tnx-rl-request-app）が使う共通機構のため存置。`applyTreatmentDelegated`（treatmentApply ソケット）は回復と共用のため存置。
- 回復成功カードの注記「（BS の紐づく負傷は残る＝ダメージは治療されない）」を「を回復。」へ簡素化（一本化で負傷治療も同カードに合流し、負傷を除去した直後に「負傷は残る」と表示され矛盾するため。説明系注記の全廃方針とも整合）。
- 治療用途の資格はアイテム型不問（旧実装は generalSkill/styleSkill 限定だったが、アイテムロール経路と同じ資格に統一）。
- テスト 1395 green（`usageCanTreatKinds` の照合テスト追加: 通常用途の除外脱落・タグ側用途の合致・社会専用・空集合）・eslint/stylelint 0 error。

#### 是正（同日・ユーザー報告「woundValue が読み取れていない＝ダメージに数字が保存されていない」）

- **原因**: `woundValue` フラグを書くのは `applyDamageChartResult`（ダメージ適用フロー）**のみ**。トークンのステータストグル・手動付与で作られた負傷は statuses だけでフラグを持たない（createActiveEffect フックの紐づけ・技能選択は「付与経路を問わない」是正済みだったが、**値の保存は経路依存のまま**だった）。ダメージ適用フロー由来の負傷には保存されている＝ユーザー仮説は手動付与経路について正。
- **是正**: kind 自体がチャート行番号を符号化している（`phys-10`→10）ことを利用し、`getDamageChartValue(kind)`（damage-chart・`getDamageChartKind` の逆引き）＋`woundChartValue(effect)`（conditions・**保存値優先**＝21 超の実ダメージ値は保存値のみが知る／無ければ kind から導出）を新設。recovery-flow の読取2箇所（除去計画 `addWoundRange`・回復対象選択ダイアログの「ダメージ値」表示）を差し替え。テスト+8（逆引き・保存値優先・導出・非負傷=0）＝1399 green。

#### 是正2（同日・ユーザー叱責「勝手に割り当てキャラのみにしないでください。RL は複数のゲストを操作するため、その設計は明確に不備」）

- **治療者=担当キャラクター（`game.user.character`）固定は設計不備**だった。RL は担当キャラを持たず複数のゲストを操作するため、治療メニューが RL に対して常に中止になる。経緯: ユーザーの「治療者候補の絞り込みの意味が分からない（治療用途を持つ技能を持っているのは常に自分自身）」という指摘を、Code が**「治療者選択の全廃」と過大解釈**した（批判されたのは「用途所持でのアクター絞り込み」であって選択そのものではない＝0/100 に振る誤り）。
- **是正**: 治療者選択を復活——**操作可能なキャスト/ゲストから選択（既定=担当キャラクター・1体なら即決）**。治療に使う技能は選んだ治療者自身のものだけを照合する（「自分の技能で行う」原則は治療者に対して維持）。候補を用途所持で絞り込むことはしない。
- **あわせて負傷バッジにチャート値を表示**（ユーザー指摘「表示していないのはナンセンス」＝正しい。従来 woundValue は治療目標値算出の内部フラグとしてだけ導入され、バッジの数値表記の正本が Bad_Status の BS 表記だったため負傷が対象外＝意図的判断ではなく欠落）。`_bsValueText`（旧 `_bsValueSuffix`）に負傷分岐を追加し `woundChartValue`（保存値優先・手動付与は kind 導出）を表示。
- **バッジ表記の確定（同日・ユーザー指定）**: 数字が中央に浮く現象の原因は、効果値容器（flex:1=残り幅全部）内の値が `<button>` 既定の text-align:center を**継承**していた潜在バグ（2026-07-15 の BS 効果値表示から存在・`transform-origin: left` の左起点縮小と矛盾＝意図は左寄せだった）。是正=効果名+効果値を一まとまりで**バッジ中央に flex 寄せ**（text-align でなく flex 中央＝はみ出す squeeze-text の scaleX 原点中央が正しく効く・アウトフィット表パラメータ列の既存手法を踏襲）・**効果値/チャート値は強調**（太字・1.2em・淡アクセント #99e6f2・グロー text-shadow＝ユーザー指定「二回り大きく・色を薄く・グロー」）・順序は **BS=「効果名n」／ダメージ=「nチャート名」**（テンプレートの valueFirst で制御）。
- **バッジ幅の等幅固定＋対象能力値のスート記号化（2026-07-19）**: 衰弱バッジだけ伸びる件=グリッドアイテム（バッジ）に min-width:0 が無く 1fr トラックが min-content で押し広げられていた→min-width:0 で全セル等幅・あふれは squeeze 縮小で全文表示（ユーザー指定「長さ固定・見切れなし」）。それでも「生命の制御値」が長すぎる→Code 案「（生命 -9）」に対し**ユーザー案=スート記号を採用**（能力値名だと能力値の値と区別がつかない）: 衰弱スート版=`衰弱♥-9`・全制御値版=`衰弱-9`・重圧=`重圧♥`（**括弧なし**＝同日ユーザー追指示「（）はいりません」で撤去。♠理性/♣感情/♥生命/♦外界＝システム全体の既存対応。萎縮/憎悪の対象名・捕縛の武器名は固有名詞のため括弧を維持）。表記の記録は Bad_Status.md。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**: 治療メニュー一巡（治療者選択→通常負傷→`@condition.woundValue` 式・戦闘不能タグ/負傷行どちらのクリックでも同じ用途・社会負傷→専用技能なしで中止）・宣言形治療用途のメニュー起動・非所有患者の GM 委譲除去・**手動付与の負傷でのチャート値導出**・**負傷バッジのチャート値表示**。
- **既存データへの影響**: 治療用途を持たないキャラクターは治療メニューから治療できなくなる（代用判定廃止の帰結＝ユーザー確定）。〈医療〉等の技能に治療用途（通常ダメージ用/気絶・失神用/仮死・昏睡・支配用）の設定が必要。
- 旧チャットカードに保存済みの `treatment` 継続スナップショットは、再判定時に除去を再実行しない（結果表示のみ・過去メッセージに限る影響）。


### 2026-07-18 攻撃の適用効果タイミング2種（命中時／ダメージ1点以上）

#### メカニクス監査

- **ルール（ユーザー提供）**: 攻撃で相手に与える効果には「**命中時点で適用される効果**」と「**1点でもダメージを与えたら適用される効果**」の2種類がある。従来実装はダメージ適用時の一括付与のみでどちらも不正確だった（KI-027）。
- **裁定4点（2026-07-18）**: ①「1点でも」の1点＝**完全な最終適用値**（恒久軽減→10上限→事後修正→適用時軽減を全て経た値・0 は不適用）②命中時効果でダメージに影響するものは AE でなく**用途の式**に組み込む＝付与順序がダメージ算出へ影響する経路の配慮は不要 ③カバー時は「当然カバーしたキャラクターに効果もダメージも適用」（原文ママ。順序の整理=Code の読み: カバーはダメージ算出直前＝命中解決後のため命中時効果はカバー宣言前に元対象へ適用済み・カバー側へ移るのはダメージ時効果）④再判定で命中が覆った場合は**従来同様手動で復元**（自動取り消しは作らない）。
- タイミングの所有者は**効果（AE）自身**＝出どころのアイテムのテキストが決める（用途側の2リスト案は不採用）。既定=**ダメージ時**（既存効果はその前提で作られており無移行・≥1 ゲートの追加がそのまま是正として効く）。
- 非攻撃用途は命中/ダメージのフェーズ区分を持たない＝従来どおり結果カードの手動ボタン（適用判断は卓＝v3 原則）。

#### 設計レビュー

- **AE 設定**: TNX 欄に `flags.grantTiming` セレクト「適用タイミング」（付与先の隣・付与先「自分」選択時は非表示に連動）。
- **ペイロード**: エントリへ `timing` を追加し `splitEffectsByTiming` で二股（timing 無し=旧カード互換=damage）。
- **命中時**: 付与トリガーを「**hit への遷移**」（純関数 `newlyHitTargets`）に一本化——postAttackCard（非対決の即時解決）・completeReactionFromCheck（対決解決/再解決）・`_applyRecheckReplacement`（攻撃再判定・**パッチ前スナップショット**で遷移算出）。付与は `grantHitTimedEffects`＝所有者/GM は直接・権限なしは新設 **`usageEffectGrant` ソケット**（treatmentApply/repairApply と同型の activeGM 委譲）。`usageEffects.hitGranted` に記録し攻撃カードへ付与済みノート。
- **ダメージ時**: finalizeDamageRoll が**ダメージ時効果だけ**をダメージカードへ引き継ぎ（対象なし攻撃=命中解決なしは全効果を残す手動フォールバック）、適用ループで **final≥1** の対象へ `grantDamageTimedEffects`（チャート適用より先=2026-07-12 の順序維持）・実付与対象を `appliedTargets` に記録し 0 件は「付与されませんでした」表示。
- **KI-028**: `attackCardEffectMode` 純関数（attack/button/hide）——攻撃=抑止＋ノートのみ（ダメージカード一本化の原則維持）・非攻撃対決=解決後（全対象解決/オープン解決/終端状態）に手動ボタン・未解決=非表示。
- **TDD**: 純ロジック4件（timing 抽出・`splitEffectsByTiming`・`attackCardEffectMode`・`newlyHitTargets`）を RED→GREEN で追加。テスト 1410 green・eslint 0 error（CSS 変更なし）。
- **表示**: ダメージカード予告文「（ダメージ適用時・1点以上の対象へ付与）」・用途シートの適用効果リストに「（命中時）」タグ・自分付与/命中時付与ノートは攻撃カード側（ダメージカードへ持ち込まず重複表示しない）。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**: 非対決攻撃の命中時即時付与／対決攻撃のリアクション解決時付与（肩代わりリアクション=権限なしの GM 委譲含む）／再判定 miss→hit の自動付与・hit→miss の手動復元運用／ダメージ≥1 ゲート（全対象 0 の不付与表示）／カバー時のカバー側付与／対象なし攻撃の手動フォールバック／非攻撃対決の解決後ボタン（KI-028）／AE 設定の適用タイミング欄と付与先の表示連動。
- 命中時付与の記録（hitGranted）は**表示用**であり再付与の判定には使わない（遷移トリガーが正）。多クライアント同時解決でノートの名前記録が競合しうるが表示のみ＝実害なし。

#### 是正（同日・実機報告「命中時効果もダメージ適用時に適用されている／勝手に自動適用にするな」）

- **バグ**: `prepareUsageEffectPayload` がペイロード生成時（`{name, data}` への写し）で `timing` を**落としており**、`splitEffectsByTiming` が全効果をダメージ時と判定＝命中時効果が従来どおりダメージ適用時に付与されていた。TDD が純関数単位（resolveUsageEffectData と splitEffectsByTiming を別々に検証）でペイロード構築の結合を固定しておらず、伝搬漏れを素通しした。ペイロード伝搬の再現テストを追加して固定。
- **設計是正=自動付与の撤回**: 命中解決時の自動付与だと**カバー宣言（ダメージ算出直前まで可能）の前に元対象へ効果が乗ってしまう**。裁定「カバー時は効果もダメージもカバー側」は**命中時効果も含む**（Code 当初の読み「命中時効果はカバー宣言前に元対象へ適用済み」は誤り＝Combat_Flow の裁定注記を訂正）。命中時効果は**攻撃カード（判定チャットカード）の「効果を適用」ボタンで手動適用**へ（v3 原則「適用判断は卓」に回帰）——全対象の解決後・命中1体以上で表示し、適用先は**押下時点**の命中対象（`hitEffectTargetRefs`・カバー宣言済みはカバー側へ付け替え・カバーした側が自分も命中対象なら1件に畳む）。押すのは対象の所有者か RL。完了で `hitApplied`/`hitGranted` を記録し適用済みノートへ。
- **撤去**: `grantHitTimedEffects`・`newlyHitTargets`（attack-flow-logic）・`usageEffectGrant` ソケット（手動ボタン=押下者権限で付与するため委譲不要）。`grantDamageTimedEffects` は押下者権限＋権限なし警告へ簡素化（ダメージ適用と同じ前提）。
- テスト 1412 green（timing 伝搬・hitEffectTargetRefs のカバー付け替え/畳み込みを追加・newlyHitTargets のテストは撤去）・eslint 0 error。

#### 確定（同日・設計3転の最終形＝用途の2リスト方式・全適用の手動ボタン化）

- **メカニクス**: ユーザー指摘2件で設計を確定。①「ダメージを与えない攻撃・対決判定を考えれば**命中時が一般形・ダメージ時が特別**」（Code の既定=ダメージ時・（命中時）タグは特別視が逆）②「**AE 側の設定は、ただ対象に効果を付与する普通の効果に攻撃専用の概念を背負わせる**」——タイミングは効果の性質でなく**攻撃という用途の挙動**であり、付与先（自分=代償は誰が使っても代償）の AE 側前例は当てはまらない。→ **攻撃用途に効果リストを2つ**（「適用される効果」=一般/命中時・「適用される効果（ダメージ時）」）。さらに③「1点でも」等の**条件はコード判定しない**（≥1 ゲート撤廃。ダメージの実発生と無関係にダメージ適用時へ与えられる効果も想定し、0 の対象はボタンを押さない/除外で卓がスキップ）④複数対象は**押下時の除外チェックボックスダイアログ**（ユーザー提案）。
- **設計レビュー**: `usage.damageEffects`（{itemId,effectId} 同形・migrateData 不要=既存 `effects` の意味保存）。ピッカー/解除/追加済み行は既存 UI を `data-effect-list` パラメータで共用（`buildAdded`/`buildGroups`・除外はリストごと=両リスト重複可）。`resolveUsageEffectData` がリストごとに timing を刻む（AE `grantTiming` 読み取り・AE 設定欄・（命中時）タグは全撤去）。適用は3ボタンとも共通部品——`confirmEffectTargets`（2体以上でチェックボックスダイアログ・`tnx-uses-row` 意匠流用・キャンセル/全除外=適用しない）＋`grantEffectsToRefs`（付与ループ・denied 警告）。攻撃カード=`applyHitEffectsFromMessage`（hit エントリ・`hitEffectTargetRefs`=カバー付け替え）・ダメージ/結果カード=`applyUsageEffectsFromMessage`（適用先=ダメージカードならカードの対象）。ダメージ適用処理との連動・≥1 ゲート・`grantDamageTimedEffects` を撤去（2026-07-12「ダメージ適用と同時に自動付与」裁定は本設計で上書き）。適用済みノートは実付与対象（`appliedTargets`/`hitGranted`）のみ表示。
- 文言: 攻撃カード=無印「付与効果」（一般形）・ダメージカード=「付与効果（ダメージ時）」（特別条件側にだけ印）。
- テスト 1412 green（リスト所属 timing・grantTiming 不読・ペイロード伝搬を差し替え）・eslint 0 error・CSS 変更なし（既存クラス流用）。

#### 課題・申し送り（最終）

- **実機確認済み（2026-07-21・v0.12.0）**: 攻撃用途シートの2リスト設定（追加/解除・両リスト重複）・攻撃カードのボタン（全対象解決後表示・カバー宣言後の付け替え適用・適用済みノート）・ダメージカードのボタン（ダメージ適用と独立・対象なし攻撃のフォールバック）・除外チェックボックスダイアログ（2体以上・全除外=キャンセル）・非攻撃対決カードの解決後ボタン（KI-028）。
- 旧カード（`1ac4a08` 以前の攻撃/ダメージカード）のペイロードは timing 無し=ダメージ時バケットに落ちる——ダメージカードのボタンで従来どおり適用可能（表示・挙動の互換は保存）。
- `grantTiming` フラグが残った AE は読まれない（死フラグ・無害）。

### 2026-07-19 リアクションフロー大改修（併存・対象行クリック起点・シークレットカード・オープン複数化）

設計 2026-07-18（会話で承認・裁定は [Combat_Flow.md](../01_Wiki/Game_Rules/Combat_Flow.md) に verbatim 記録）・実装 2026-07-19（本体 `1baa983`/`fc95cc8`/`2e25db5`）。

#### メカニクス監査

- **併存ルール（ユーザー提供）**: 「他人への攻撃にもリアクションを行える」効果は本人のリアクションと併存し、特殊な効果がなければ**高い方が適用**。符号化＝命中は対象の**全防御要素**（本人スキップ時の制御値＋本人・代理の各リアクションの独立対決[受動有利]）を破った場合のみ・表示上の適用は最高達成値の1件。スキップ＋代理は「制御値と対決の両方を破らないと命中しない」（裁定どおり）。
- **受け値**: 成立パリーの**最大1つのみ**・防御力は常に攻撃対象自身（代理パリーは受け値だけ追加）＝裁定どおり（[Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md) へ反映）。
- **社会軽減ゲート**: 代理でも成立していれば開く・消費は対象本人の報酬点＝裁定どおり（`reactionEstablished`＝成立の論理和）。
- **汎用「リアクション」タイプ**: 系統フラグなし＝あらゆる対決判定への資格・適否は卓判断（システム非ブロック）。**系統別タイプとは統合しない**（統合すると〈信用〉が精神攻撃にリアクション可能になる＝一般則の資格ゲートを維持。Code の統合提案は「細則で一般則を上書きする」誤りとして却下された経緯を保持）。
- **範囲攻撃へのリアクション/攻撃を失敗させる**: 併存下では**そのリアクション自身の勝利**で発火（高い方に負けた代理の特殊効果は発火しない）——「高い方が適用」と整合。
- **移動・離脱**: 対象が存在しない行動＝対象要求なし・リアクションは任意（キャラごとに1回・結果=最高値1件のみ・確定操作なし）＝指示どおり。

#### 設計レビュー

- **純ロジック分離**: `usage-target-plan.mjs`（対象×対決の決定表・18テスト）・`reaction-logic.mjs`（`resolveTargetDefense`/`resolveOpenReactions`/`reactionButtonPlan`・21テスト）。Foundry 依存の attack-flow はこれらの導出値を保存するだけ＝従来フィールド（state/resolution/reactionAchievement/diff/parryGuard/reactionEstablished）を導出値として維持したため**ダメージフロー側は無改修**で接続。
- **データ**: `targets[].reactions[]`＋`selfDecision`（旧形式は `seedLegacyTarget` で読み替え・表示互換維持）・`openReactions[]`（旧 `openReaction` は表示互換のみ）。シークレットカード＝whisper(GM＋本人)→解決時に `whisper: []` で公開（messagePatch の許可リストへ whisper を追加）。
- **入口の一本化**: 対象行クリック（カバー待ち受けが最優先→リアクション）・本人/代理の判別とリアクター決定は `resolveUserIdentityActor`（選択中所有→割り当て）の一関数。技能起動は従来どおり唯一の `_activateItemCheck` に reaction 文脈を注入。
- **再判定/事後修正**: 合成対象は保存済み要素で再合成（`rebuildRecheckedTargets`）・リアクション個別は messageId 一致の要素置き換え（`CONTINUATIONS.reaction`）。
- 懸念（申し送り）: 実機確認一式は Phase_12 の 12-4x を参照（2026-07-21 完了）。攻撃カードの事後修正（達成値クリック）は従来どおり解決済み対象の再解決を行わない（既存挙動の保存・複数対象の modify 再解決は将来課題）。

#### 是正（2026-07-19 ユーザー指摘3件・本体 `3248934`）

- **メカニクス/設計の誤り①（無機能の中継）**: オープンリアクションにもシークレットカードを挟んでいた。対象ありフローでは「決定/スキップの意思決定」と「手段選択の秘匿区間」をカードが担うが、オープンでは手段が能動側カードの1ボタンに集約済みで**中継カードは何も担わない**——対象ありの形を機能の検討なしに流用した形式優先の誤り（Code 自身が設計時に「1ボタンなら中継は無意味では」と気づきながら「一貫性」を理由に採用していた）。→ ボタン押下で技能選択→判定を直接起動。キャンセル時の孤児カードも消えた。
- **表示の誤り②（内訳オミット）**: 改修前のオープンリアクションは通常の結果カード（計算内訳つき）を出していたのに、改修で薄いリアクションカードに置き換えて**カード値・能力値等の内訳を退行させた**。→ 通常の判定結果カードへ回帰。
- **構造の誤り③（カードの個別設計）**: 対決判定カード・リアクション結果カードも判定内訳を独自設計で落としていた。→ **判定結果カードの基底化**（ユーザー指示「チャットカードは全て基底のものに情報を足す形式で実装」）: 基底部品 `check-card-head.hbs`/`check-calc-rows.hbs`＋共有コンテキスト `check-card-context.mjs` を新設し、結果/対決判定/リアクション結果/移動の全カードを基底合成へ。表示コンテキストの組み立ては一箇所（表示の再実装は表示ずれの温床）。旧カードはライブ描画の表示互換を維持（`contentResolved` フラグで判別）。


### 2026-07-19 ターゲット関連ブロックの全廃（明示ターゲットの尊重）

#### メカニクス監査

- **対象の自動解決の位置づけ**（ユーザー裁定・同日複数回＝KI-032）: 自動解決は「**ターゲットのし忘れ**」の救済であり、既にターゲットしている対象を「正規ではない」とはじくのは本末転倒。**対象欄の値を理由に実行をブロックしない**（「ターゲットできない用途」をシステムが恣意的に決めない）。
- **対象欄の意味**: 対象「なし」という値は存在しない。「-」＝完全に対象なし・「解説参照」「その他」＝対象が基本ルールの中で解決できないことを指す（「その他」＝レティクルでターゲットできない対象を取る値）。用途の「解説参照」は不自然＝親データ由来の「その他」自動解決が必要（KI-033・未実装）。
- **修理・治療の実対象**: 修理の実対象はアウトフィット（キャラクターのターゲットは「どのキャラクターの所持品か」を指す便宜）。解決は「ターゲット中→その相手・未ターゲット→自分」。治療も同型。
- **手動ターゲットの妥当性警告（2026-07-18）は撤廃**: 「自身に適用できない」の機械的効果は自動セルフ抑止のみ・対象「自身」は常に自分へ読み替え（他者レティクルでもブロックしない）。

#### 設計レビュー

- `planUsageTargets` から invalid モードを削除（none/targets/autoSelf/dialog の4値へ）。対象「自身」は常に autoSelf（値の意味どおり）。明示ターゲットは group を問わず targets。
- 共通ヘルパー `resolveTargetedOrSelf`（target-resolution.mjs）を新設し、修理（useRepair）と治療/回復（resolveRecoveryPatient）の対象解決を一本化。未ターゲット時の自分解決はレティクルを付与（「対象は常に明示」原則は維持）。「〜対象がターゲットされていません」警告2種と「ターゲットが間違っています」を撤廃。
- 決定表のその他の挙動（対象なし群の対象要求なし・未ターゲット時ダイアログ・オープンリアクション・カード形式）は不変＝攻撃・移動・離脱のフローに影響しない。
- テスト 1452 green（usage-target-plan の invalid 期待を裁定どおり更新）・eslint 0 error。実機確認済み（2026-07-21・v0.12.0）: 修理の自己/味方修理一巡・治療の同型・対象「自身」用途の読み替え。

### 2026-07-19 用途の「解説参照」完全廃止＝「その他」への自動解決（KI-033・本体 `7d90fec`）

#### メカニクス監査

- **裁定（2026-07-19・KI-032 から分離）**: 用途において「解説参照」は不自然——用途は解説の実装そのものであり自己参照になる。親アイテム（技能固有値）のデータが「解説参照」の場合、用途側では「その他」へ自動解決する（対象欄に限らず全欄）。
- **設計承認（同日・選択肢「完全廃止」）**: ①自動入力での変換＋②インポート正規化＋③用途シートの選択肢撤去。優先度上は「解説参照」「その他」を**最下位のフォールバック**として扱い、有効値が1つでもあればそちらが勝つ・無ければ「その他」を書き込む（親が「その他」なら自由記入欄も複写・「解説参照」なら空）。
- **優先度表との関係**: Check_Rules.md の対象/射程優先度（ルール由来）は不変。フォールバック順位はルールに無い値の**システム側の取り決め**（rank 0.5＝単体/至近より下・blank より上）。射程「なし」は従来どおり優先度に参加しない（ルールの優先度表に無いため・変更なし）。
- **対決欄は対象外**: 用途の対決欄は 2026-07-17 再編で「解説参照/その他」を持たない別セット（スタイル技能の対決行のマージも従来からスキップ）。

#### 設計レビュー

- 優先度解決の純ロジック（targetRank/rangeRank/resolveTarget/resolveRange/resolveTargetValue/resolveTiming＋新設 normalizeUsageExplanation）を `usage-autofill-logic.mjs`（Foundry 非依存）へ分離し、テスト 42 ケース併設。tnx-usage-sheet は同モジュールを import（RANGE_SPAN_CAPABLE も移設・attack-weapons のコメント参照を追従）。
- タイミング導出は「最初の非 blank」→「最初の実値、無ければフォールバック」へ変更（従来は解説参照/その他も先頭なら写していた＝目標値の「最初の非 blank 型」も同様に是正）。サブ選択（アクション/プロセス種別）の解説参照も「その他」へ変換。
- インポート正規化は `enforceUsageChainDefaultsOnImport` の先頭パスに追加（チェーン解決の早期 return より前・正規化のみでも書き込み）。createItem フックの適用条件を「generalSkill/styleSkill または actions を持つアイテム」へ拡大——**常設経路でのデータ書き換えだが、用途の「解説参照」は裁定上いかなる時も不正な値のため冪等正規化が成立**（KI-029 の「現行データと旧データを区別できない正規化」とは異なる。設計承認済み・2026-07-19）。
- 用途シートは選択肢撤去＋表示時に旧値を「その他」へ吸収（deepClone に対する正規化＝次のフォーム保存で着地。データを黙って書かない）。
- テスト 1486 green（+44）・eslint 0 error（警告は既存分のみ）。実機確認済み（2026-07-21・v0.12.0）: 自動入力・辞典インポート・用途シート表示の一巡。

### 2026-07-19 判定要求のコンボ用途候補列挙（KI-025・本体 `af12d7b`）

#### メカニクス監査

- **ルール解釈（2026-07-17 ユーザー指摘の実装）**: 指定技能を組み合わせに含む判定は判定要求への**正当な応答**であり、代用判定（卓裁定つきの別技能判定）として扱うのは筋が違う。組み合わせの可否はユーザー/RL が決める＝システムは制限しない。
- **候補の範囲（Code 設計・裁定余地あり）**: 列挙するのは「指定技能を参加技能（ベース/組み合わせ）に含む他アイテムの用途」のうち**通常判定として要求を解決できるもの**——攻撃・治療・修理・カバー・NPC取得・固定値判定・バフ宣言（クリック待ち系）は専用フローへ逸れて要求文脈（requestMessageId）が失われるため列挙しない。これは組み合わせの制限ではなく応答導線の列挙（ドッジ等のリアクションタイプ・移動タイプは通常判定として応答可能なため残る）。指定技能自身の用途は従来どおり「〈技能名〉で判定」→用途ピッカーが受け持つ（重複列挙しない）。
- 要求文脈の判定は対決の入れ子にしない既存規則（2026-07-17）がそのまま効く＝対決欄つき用途も要求応答時は通常判定。

#### 設計レビュー

- 候補列挙 `enumerateRequestComboCandidates`（usage-check-context・純関数/テスト付き）: 参加技能の一致は resolveUsageSkillSet の解決結果を**識別キー**で照合（アイテム ID 直接比較にしない＝重複所持に頑健）。不備用途（ベース不明・共通スート無し）は detectUsageDefect で除外。
- 起動は唯一の起動関数 `_activateItemCheck` へ usageId 直接指定で委譲（新しい起動関数を作らない規範を維持）。要求文脈・目標値・代用修正の注入は従来の extra 合流のまま。
- `_activateItemCheck` の実行対象用途フィルタを `usableUsagesOf`（usage-types・テスト付き）へ抽出し、技能クリックと将来の列挙系で規則を共有（インライン重複の解消）。→ **2026-07-19 に `usableUsagesOf` ごと廃止**（候補は全用途・共有も解消済み＝判定要求は `canAnswerCheckRequest`）。
- 応答ダイアログは既存の縦積みボタン形式を拡張（指定技能→コンボ候補→代用判定→キャンセル）。候補ラベルは「用途名（親名）」に統一（親名は itemDisplayName＝技能は〈〉整形）・指定技能ボタンも 〈〉 整形へ是正。
- テスト 1486 green・eslint 0 error。実機確認済み（2026-07-21・v0.12.0）: 要求→コンボ候補選択→判定→要求カードへの結果反映の一巡。
- **UI 改=二段階化（同日ユーザー指示・本体 `5b5c8ad`）**: ボタン列挙は候補が多いとあふれるため、第1段を「〈技能名〉で判定／代用判定」の 2 択へ戻し、第2段に**用途名のプルダウン**（`_promptUsageChoice`・中身=指定技能自身の用途+コンボ候補の統合 `buildRequestUsageChoices`＝純関数/テスト付き）を挟む。**1 件は自動起動**。第2段経由は自技能の用途も usageId 直接指定になり、要求文脈での汎用用途ピッカーは通らない（1 件自動選択も従来ピッカーと同じ）。代用判定フローは `_promptSubstitution` へ分離（挙動不変）。テスト 1497 green。
- **「判定」タイプ限定（同日ユーザー裁定・本体 `fa8d40a`）**: 判定要求から起動できる用途を「判定」タイプに限定（`canAnswerCheckRequest` を type==="check" ベースへ簡素化・指定技能自身の用途にも適用=当初は技能クリックの実行対象規則そのままで無フィルタだった）。上記「候補の範囲（Code 設計・裁定余地あり）」の項はこの裁定で置き換え——リアクション・移動等の行動種別タイプも列挙しない（各自のフローから起動するもの）。判定タイプ内の除外（固定値・NPC取得・バフ宣言フラグ=要求カードに結果を返せない）は維持。0 件は警告して中止（アイテム起動へ落とすと判定以外の用途ピッカーが開いて限定と矛盾）。テスト期待を更新し 1497 green。
- **用途名ラベルの是正（同日ユーザー指摘・本体 `adcd7cf`）**: 上記の「親名は itemDisplayName＝技能は〈〉整形」は**誤適用**だった——用途の実効名「タイプ名（親アイテム名）」（2026-07-17 確定）の親名は**素の名前**が正（「判定（知覚）」）。既存の usageDisplayName 呼び出し箇所（用途シート placeholder/題名・用途一覧・戦闘タブ・攻撃カード・汎用用途ピッカー）は全て素の item.name を渡しており、〈〉付きは今回の新規 2 箇所だけの逸脱だった。原因は①技能名〈〉の一般則を実効名の細則に持ち込んだ（細則優越の判断ミス）②`_promptCheckUsage` 内の実装と食い違う古いコメント（「呼び出し側で〈〉整形済み」）を実態未確認のまま先例として採用。ラベルを素の親名へ是正し、当該コメントも実装どおりに訂正。第1段ボタン「〈技能名〉で判定」は技能名単体表示のため〈〉のまま（一般則の適用範囲）。テスト 1497 green。
- **長い用途名の「…」省略（同日ユーザー指示・本体 `67694ac`）**: 閉じたセレクト（ダイアログ幅 360px 固定）はラベル末尾=（親名）側から見切れるため、生成時に文字数で丸める——上限 22 文字（全角ベース）・**省略は用途名の側だけで親名（技能名）は必ず残す**（親名が長い場合も用途名を最小 4 文字残して超過許容）。指定技能自身の名前つき用途は名前のみ（技能名は題名「〈技能名〉で判定」が担う）で末尾省略・未命名「判定（親名）」は省略しない。開いたドロップダウンはネイティブに広がるが、生成時に丸めるため両状態で親名が保証される。テスト +4=1501 green。

### 2026-07-19 EXP 再計算の合流（KI-008・本体 `659a2df`）

#### メカニクス監査

- **ユーザー言明（2026-07-19）**: 自動インポート（アクター作成時の基本技能セットアップ）では経験点は消費されないため、当該タイミングでの EXP 値のずれは元々起こり得ない。是正は値の正しさでなく**並行する同内容 DB 書き込みの排除**（「合流させたほうがきれいならその方が良い」）。
- EXP の計算式・記帳・User flag 同期の設計は一切不変（§4.3「EXP ロジック書き換え禁止」の枠内）。

#### 設計レビュー

- `updateCastExp` を合流ゲート化: 同一アクター（uuid キー）の実行中はフラグ `rerun` に畳み、完了後にもう 1 回だけ再実行（毎回の全量再計算のため最後の 1 回が最新状態を反映＝取りこぼしなし）。計算・書き込み本体は `_recalcCastExp` へ改名のみ。
- 実行中→完了→trailing 1 回の最大 2 走で N 並列呼び出しを吸収（updateUsageActions の直列キューと類似だが、こちらは「最新全量が 1 回走れば十分」のため合流方式）。実機確認済み（2026-07-21・v0.12.0）: 一括インポート一巡。

### 2026-07-19 判定要求のアクター登録化＋技能プルダウンの正規順・分類グループ統一（本体 `585a1aa`/`4313515`）

#### メカニクス監査

- **対象の登録単位はアクター（2026-07-19 ユーザー指示）**: 「ユーザーを選択して」ではなく「アクターを登録して」判定要求を送信する。「判定する」ボタンは**そのアクターの所有者権限を持つユーザー**（+GM）が押せる＝接続状況・キャラクター割り当てに依存しない。制御判定要求（controlNegate）も同じ targets 形式（actorId/actorName）へ統一（旧「操作ユーザーのスナップショット保存」を廃止——描画時の所有者判定に一本化）。
- **技能選択の並び（2026-07-19 ユーザー指示）**: 判定要求の技能プルダウンは「シート上のソート順と同じ並び」×「分類（無条件取得技能・製作・芸術・操縦…）ごとのグループ」。このソート順は**技能を選択するプルダウンすべて**に適用する。
- 分類グループの並びは正規ソート順での初出位置（無条件取得技能→製作→芸術→操縦→社会→コネ）＝ユーザー例示（無条件取得技能、製作、芸術、操縦…）と一致。

#### 設計レビュー

- 正規ソート順（GENERAL_SKILL_SORT_PREFIXES/位置関数）を `identification.mjs` へ移設（辞典ローダからの参照で TnxSkillUtils⇄skill-dictionary の循環を回避。TnxSkillUtils の同名 static は委譲で温存）。
- 中央適用: `loadSkillEntries` の基底ソートを名前順→**正規順＋名前(ja)**へ（loadSkillChoices／カスケード各段／loadOnomasticChoices＝ヴィークル対応操縦／スタイル技能の代用先など辞典由来の全プルダウンが一括で正規順に）。`ONOMASTIC_TYPES` の記載順も正規順（製作→芸術→操縦→社会→コネ）へ是正＝カスケード小分類の並びに反映。
- アクター所持系: `orderSkills`（skill-select）を「一般→スタイル・①手動 sort=シートの並び ②正規順 ③名前(ja)」へ拡張し、用途シートのベース技能/組み合わせ候補の名前順ソートを置換（リアクション/代用判定の既存利用箇所は挙動不変＝手動 sort が最優先のまま）。
- 判定要求のグループ化は `loadGroupedGeneralSkillChoices`（純粋部 `groupGeneralSkillEntries`・テスト付き）＋ optgroup。旧 `_loadCompendiumSkills`（独自キャッシュ・名前順）は削除し loadSkillEntries へ一本化。
- ダイアログの対象一覧はキャスト全員（名前順・22px サムネイル・`rl-actor-img`＝旧ユーザー色ドット/ユーザー名表示の CSS は撤去）。チャットカードの `data-user-id` 撤去（旧カードは actorId で互換動作）。
- テスト 1495 green（+9: skill-select/グループ化・カスケード小分類の期待を新順序へ更新）・eslint 0 error・stylelint clean。実機確認済み（2026-07-21・v0.12.0）: 要求送信→所有者ボタン→結果反映・プルダウン並び（要求/用途シート/カスケード/ヴィークル操縦）。

### 2026-07-19 残弾の廃止＝弾数の使用回数への一本化

#### メカニクス監査

- **残弾の廃止（2026-07-19 ユーザー確定）**: 射撃武器の弾数は**使用回数（`uses`）で管理する**——装弾数=`uses.max`・撃った数=`uses.spent`・残弾=`max − spent`。武器固有の残弾（`weapon.system.ammo`）という別カウンターは持たない。
- リロードは従来どおり**射撃武器の用途（マイナー＋使用回数へのマイナス消費＝回復）**で表す。負値消費は 0〜max でクランプされるため、空の状態からの満タン復帰も1行で書ける。
- 自動給弾の武器は**使用回数に制限を設けない**（`uses.isLimit=false`）ことで表す（旧 `ammo.mode="-"` の役割）。
- 期間種別（`uses.type`＝アクト/シーン/カット）による自動リセットは**システムに実装が無い**（表示ラベルのみ）ことを確認済み。したがって弾数を uses に載せても期間境界で勝手に補充されることはない（射撃武器は種別「-」で運用）。

#### 設計レビュー

- `weapon.mjs` の `ammo` スキーマと残弾関連 migrate（`empty`→`current`・`arbitrary`→`value`）を削除。`weapon-ammo.mjs` はファイルごと削除（`reloadWeapon`/`adjustAmmo`/`isAmmoEmpty` は既に呼び出し元が無く死蔵していた）。
- `usage-consumption.mjs` から `kind="ammo"` 経路・`problem:"noAmmo"`・`nextAmmoCurrent` 適用を撤去。**消費ダイアログの「回復」文言は uses 行へ移した**——リロードが使用回数へのマイナス消費になった以上、負値の行を「使用回数を消費×-1」と書いてはならない。
- 消費先 UI は**二段選択へ戻した**（[アイテム/AR]→[このアイテム自身/各アイテム]）。資源が「使用回数」1択になったため三段目のセレクトは廃止し、他アイテム候補のフィルタも `hasUsesRes` のみへ。フォーム送信は `resource:"uses"` 固定。
- **既存データは移行せず破棄（ユーザー裁定）**: 装弾数・現在残弾は失われる。`resource="ammo"` の消費行は usage の migrateData で**行ごと削除**する——資源だけ失って残すと「使用回数」行に化け、制限なしの武器を指す no-op の飾り行になるため（[[feedback_no_silent_data_rewrite]] の観点で、副作用は設計提示時に明示してユーザー承認を得た）。
- アウトフィットシートの残弾欄（`hasAmmo`/`ammoModeOptions`/`ammoRemaining`）と CSS `.tnx-ammo-remaining` を削除。武器の「使用回数」欄は全アウトフィット共通の詳細設定に既存のため、UI の新設は不要だった。
- テスト 1490 green（weapon-ammo.test.mjs 削除・消費テストの ammo ケースを uses へ置換・`resource="ammo"` 行の破棄テストを追加）・eslint 0 error・stylelint clean。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**: 射撃武器の使用回数によるリロード用途の一巡（消費→空→回復）。消費先ブロックが二段になった後のレイアウト（グリッドは2列のため、アイテム行は [種別][対象] / [消費数] の2行になり右下に空セルが出る）。
- 既存の武器は使用回数を設定するまで**弾数無制限**として振る舞う（破棄裁定どおりの挙動）。装弾数とリロード用途の消費行はユーザーが入力し直す。

### 2026-07-19 アウトフィットのタイミング廃止＋アイテムロール候補の無選別化

#### メカニクス監査

- **アウトフィットのタイミングは持たない（2026-07-19 ユーザー確定）**: タイミングは**用途側で管理**しており、ルールブック上もアウトフィットのタイミングは**解説にしか書かれない**ため、アイテム自身がデータとして持つ必要がない。**スタイル技能の timing は存置**——こちらはデータとして実在し、シートの説明欄表示（`view.timing`）に関わる。
- **宣言用途を除外しない（2026-07-19 ユーザー指示）**: アイテムロールの候補を用途の種別で絞らない。「フラグ無しの宣言はアイテムロールの実行対象外」（2026-07-16 の保守的な据え置き）は撤回。詳細は [Resolved_Issues](../01_Wiki/Issues/Resolved_Issues.md) KI-034。

#### 設計レビュー

- **timing 廃止**: `outfit-base` の `timing` スキーマ、アウトフィットシートの入力欄と種別変更カスケード、シート options の `timing`/`actions`/`processes` を削除。**波及なし**を確認済み——戦闘タブのグループ化は用途の timing、`view.timing` はスタイル技能専用、辞典コピー対象（skill-dictionary）もスキル側、用途の自動入力（`deriveUsageAutoFill`）は `skillItems` で技能に絞ってからベース技能の timing を読むため、アウトフィット側は元から参照していなかった。既存ワールドの残存 `system.timing` は DataModel が定義しないため読まれない。
- **候補の無選別化**: `usableUsagesOf` を**廃止**（恒真フィルタになるため関数ごと除去＝無機能の段を残さない）。候補は `item.system.actions` そのもの。解説カードのフォールバックは「用途が全く無いとき」だけに戻った。
- `usableUsagesOf` の唯一の利用者はアイテムロール経路だった（判定要求は 2026-07-19 の裁定で `canAnswerCheckRequest` に分岐済み）ことを全数確認してから削除。コード内の「判定要求の候補列挙と共有」コメントは既に実態と食い違っていたため訂正。
- テスト 1488 green・eslint 0 error・stylelint clean。usage-types のテストは、廃止した関数の代わりに「各用途が分岐先（executionFormOf）を持つ＝到達不能な用途が生まれない」ことを見る形へ差し替え。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**: アウトフィットのクリック→用途ピッカー→宣言用途の実行。用途シートのタイミング欄が消えたアウトフィットシートの見た目。
- 判定用途と宣言用途を併せ持つ技能は、これまで判定が自動起動していたのがピッカー経由になる（クリックが1回増える）。ユーザー了承済みのトレードオフ。

### 2026-07-19 残弾の再導入（使用回数と共存する別資源へ・同日の一本化を撤回）

#### メカニクス監査

- **残弾と使用回数は別資源で共存する（2026-07-19 ユーザー裁定）**: 「**武器を使用して攻撃する**」で減る弾数と、「**アウトフィットを使用する**」で減る使用回数は**ルール上まったく別の行為**（単に「使用」と言えば普通は後者）。したがって1つの射撃武器が両方の制限を同時に持ちうる。同日の「使用回数へ一本化」（残弾廃止）は**この共存を考慮していなかった**ため撤回する。
- **却下した代案と、その却下理由の訂正記録**:
  - **用途ごとの使用回数（D&D の Activity uses 相当）**: Code は当初これを推し、根拠を「弾数＝アイテムの資源／回数制限＝**用途固有の能力**」という帰属の切り分けに置いた。**この後半はユーザー指摘で誤りと判明**——アウトフィットの使用回数も**そのアウトフィット自体**の使用回数であり、用途に置くのは帰属の歪め。
  - さらに Code は次に「回数制限はアイテムに属する」と断じたが、**これも同様に誤り**（ユーザー指摘）。**特定のアウトフィットの効果の使用回数がアイテム自身と効果自身のどちらに紐づくかは、明確にルールに書かれていない限り判断できない**。Code は2度、ルールが定めていないことを自分で決めていた。
  - この不定性から、一度は「両方の置き場を用意して入力者が選べるようにする」（残弾＋用途の使用回数）を提案したが、**用途シートの見た目がこれ以上ごちゃつくのを避ける**というユーザー判断で**残弾のみ追加**に決定。
- **一般化の余地（保留）**: 「1つのアイテムに独立した回数制限が2つ以上」は妥当な懸念としてユーザーも認めた。ただし現時点で判明している資源は2つのため、名前つき資源の配列化などの一般化は行わない（YAGNI）。2つが同型なので、3つ目が現れた場合の一般化はデータ移行で済み設計のやり直しにはならない。

#### 設計レビュー

- `weapon.system.ammo` を **uses と同型**（`{isLimit, max, spent}`・残弾 = max − spent）で再導入。**旧モデル（`mode`/`value`/`current`・null=満タン）には戻さない**——専用のクランプ・回復ロジックを二重に持つ形だったため。自動給弾は `isLimit=false`。
- 解決・適用は `ITEM_RESOURCES`（resource → フィールド名・ラベル）の**差し替え一本**に集約し、資源ごとの分岐を二重化しない。
- **行の識別子 `key` = `itemId:resource` を新設**（重要）。同じアイテムの使用回数と残弾が並びうるため、従来の `itemId` 照合では**消費確認ダイアログのチェックが2行連動**してしまう（片方だけ外せない）。チェックボックスの value・`buildConsumptionPlan` の照合をすべて key 基準へ。
- `applyConsumptionPlan` は**アイテム×資源ごとに消費量を合算してから1度だけ適用**する形へ。従来の行ごと push では、同一アイテムに複数行があると後の行が前の行の結果を上書きしていた（同一 `_id` を並べる問題も併せて解消＝1アイテム1更新に合流）。
- 消費先 UI は資源の段を戻したが、**選べる資源が2つ以上のときだけ描画**する（使用回数しか無いアイテムに1択のセレクトを並べない＝先日の三段目撤去で得た形を保つ）。フォーム送信は、資源セレクトが未描画のとき保存済みの値を維持する（勝手に書き換えない）。
- 武器シートの残弾欄は**使用回数欄と同じ意匠**（`usage-limit-group`／「撃った数 / 装弾数」の対）で置いた。同じ形のカウンターを別意匠にしないための判断で、数値入力のスピナー不使用も使用回数の例外に倣っている（要調整ならご指摘ください）。
- テスト 1495 green・eslint 0 error・stylelint clean。追加: 残弾の解決（kind/残量/ラベル）・自動給弾は inert・**同一武器の2資源が別行として独立に解決されること**・**key 照合で片方だけチェックを外せること**・武器スキーマが uses と同型で旧フィールドを持たないこと。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**: 射撃武器で「残弾×1」と「使用回数×1」を同時に消費する用途の一巡（消費確認ダイアログで片方だけ外せること・適用後の両カウンターの値）。リロード用途（マイナー＋残弾へのマイナス消費）。武器シートの残弾欄の見た目。
- 一本化時に破棄した旧 ammo データと `resource="ammo"` の消費行は**復元しない**（破棄裁定が有効）。装弾数とリロード用途の消費行はユーザーが入力し直す。
- 往復について: 廃止（`f9a9a09`）→再導入（本エントリ）と同日に往復させた。原因は Code が**共存の可能性を確認せずに廃止設計を確定させた**こと。廃止前の設計提示時に「使用回数制限つきの能力と残弾が共存する武器はあるか」を問うべきだった。

### 2026-07-19 消費先の資源に「個数」を追加＋残弾セレクトの露出条件を是正

#### メカニクス監査

- **消費アイテムの個数も用途の消費先から消費する（2026-07-19 ユーザー指示）**: 個数（`quantity`）を使用回数・残弾と同じ枠組みの資源として扱う。**自動消費は廃止したのではなく、元から存在しなかった**——ユーザーの「自動消費になっていませんか？」に対し、`quantity.value` を減らすコードはシステム内に無く、読んでいるのは常備化経験点の計算（`preserveExp × quantity.max`）だけであることを全数確認して回答した（増減はシートの手入力のみ）。したがって本件は消費先としての**新規追加**。
- 0 での使用不可は、消費先の**残量不足＝原則ブロック**が自然に担う（専用の禁止機構は作らない）。負値消費＝補充で、上限は常備化個数（`max`）。

#### 設計レビュー

- **残弾セレクトが出ない不具合の是正（同日の再導入時に作り込んだもの）**: 資源セレクトを「選べる資源が2つ以上のときだけ描画」にしたため、**残弾しか持たない武器（使用回数制限なし）でセレクトが出ず、行が既定の「使用回数」のまま無消費に落ちて残弾を指定できなかった**。条件を「**『使用回数だけ』のアイテムでは描画しない**（1択の段を挟まない）／残弾・個数を持つアイテムは選択肢が1つでも必ず描画する」へ改めた。1択の段を避ける意図（先日の三段目撤去）は保ちつつ、選べないという実害を消す形。
- **`ITEM_RESOURCES` をアダプタ化**: 個数は保持の形が違う（使用回数/残弾は `spent` が増える・個数は `value` が減る／有効判定も `isLimit` でなく `isConsumption`）。フィールド名の差し替えでは吸収できないため、`{enabled, remaining, max, update}` の4関数に差分を閉じ込め、`resolveConsumeRows`／`applyConsumptionPlan` 側は資源を意識しない一本の経路のままにした（資源ごとの分岐を増やさない）。
- クランプは共通ヘルパ `clampCounter`（0〜max）に集約。回復（負値消費）が max を超えないのは3資源とも同じ。
- テスト 1497 green・eslint 0 error・stylelint clean。追加: 個数の解決（kind/残量=現在個数/最大=常備化個数/ラベル）・消費アイテムでないものへの個数消費は inert。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**: 残弾のみを持つ武器で消費先に「残弾」を選べること（本エントリの是正点）。消費アイテムの個数を消費する用途の一巡（0 でのブロック・補充）。

### 2026-07-20 フェーズ12-5 FS判定基幹の設計確定（実装未着手）

#### メカニクス監査

- **判定行の「進行値」欄＝切り替えの閾値**（2026-07-20 ユーザー提供）。一番上が初期値で、進行値がその値に**達したら**その行の指定技能へ切り替わる。飛ばされた行は無視される。→ 有効行は「**閾値 ≦ 現在進行値 のうち閾値が最大の1行**」という一つの定義に落ち、飛ばしの無視もそこから従う。
- **境界の裁定**: Code は「達したら」と「越えたら」の食い違いを確認質問にしたが、ユーザー裁定は「**進行値には値が一つしか書けない。越えて次の値からなら、その表記にする意味が無い**」＝閾値ちょうどで切り替わる。**Code の問い自体が不要**だった（初期値行が進行値0で有効にならないという論理から一意に決まる）。
- **指定技能の拘束**（ルール10）は既存ルール6 と矛盾しない——ルール上は指定技能のみだが代用判定という正当な抜け道があるため、システムはハードブロックしない（判定要求と同じ扱い）。
- **支援判定に専用の目標値を持たせない**: ルール5「支援判定の目標値は進行判定と同じ」から、その時点の有効行の目標値を使えば足りる。欄を作ると二重管理になる。

#### 設計レビュー

- **2026-07-09 の誤記録をユーザーが一部撤回**（2026-07-20）。FS判定シートは `JournalEntryPage` 型 `focusSystem` として新設してよいが、**進行状態の正本ではない**（設定の情報置き場＋閲覧用）。1ドキュメント＝1ページ＝1FS、複数ページで並行 FS を表す運用は採らない。
- **進行状態の正本＝ワールド設定 `activeFocusSystems[]`**。判定要求カードの形をそのまま流用しなかったのは、FS が**カットをまたぐ長期状態**で、単発で閉じる判定要求とは機能が違うため（形でなく機能で流用を判断）。カード正本だと RL が進行値を動かすたびに過去のカードへ遡ることになり、並行 FS で更に破綻する。ワールド設定なら 13 のトラッカーが**正本の引っ越しなしに**同じ値を読める。チャットカードは通知に徹し状態を持たない（正本の二重化回避）。
- **進行修正のプルダウン統一**: Code は当初「アクター側は式直書き・アイテム側はプルダウン」の混成を提案したが、ユーザー指摘「**アイテム側がプルダウンなのにアクターがそうでないのは非対称**」で統一。段数も2段プルダウンからグループ形式1本（＋参照元の選択で計2段）へ。解決値は `@param` として式に入り、係数は式で表現する（アクター値に係数をかける要求＝ユーザー提供を満たす）。
- **式キーでアイテムを特定しない**根拠はユーザー指摘「キャラクターがどのタップを装備しているかは判別できない」。FS 側は**何を見るか**だけを持ち、実解決は判定者の準備済みアイテムから（13）。
- **パネルは全員に開放**（判定要求ボタンは GM 限定だが、こちらは PL の参照手段を兼ねる）。進行中が無いときは「進行中のFS判定はありません」。
- **ゲージは既存 `outfit-slot-gauge` を流用しない**——あちらはピップ式で 5〜10 個規模向けであり進行値 30 に合わない。セレクタを広げずバー式部品を別に起こす。
- 設計正本＝[Focus_System_Mechanics.md](../01_Wiki/Game_Mechanics/Focus_System_Mechanics.md)、ルール正本＝[Focus_System.md](../01_Wiki/Game_Rules/Focus_System.md)。

#### 課題・申し送り

- **保留（ユーザー未決）**: 進行判定の実行方法（パネルから判定を起動するか）・カウンター UI の細部。
- **Code 提起（未合意）**: 情報収集タブの「判定を起動」（`_onStartInfoSkillCheck`）が `game.user.character` で `TnxCheckFlow.open` を直叩きしており判定要求機構を通っていない。RL が押すと RL 自身のキャラクターで判定が走り、技能名も自由テキストのため識別キー解決が効かない。2026-07-15 の起動集約の取りこぼし。是正は別作業として要判断。
  - **2026-07-20 訂正**: 局所的な取りこぼしではなく、**情報収集判定が未実装**であることの一断面。下記「2026-07-20 情報収集判定の実装状況の記録訂正」を参照。
- 13 への申し送り: 進行判定の自動加算・支援判定の AR 消費と +1・`source=outfit` の実解決・カット進行との連動。

### 2026-07-20 FS判定の設計追補（敗北の確定・判定要求2種・RL 任意ダメージ付与）

#### メカニクス監査

- **参加アクター＝コンバットトラッカーの登録者**（ルール13・ユーザー指摘）。Code は当初「参加者は登録せず制限は表示のみ」を推奨したが、FS判定はカット進行の上で行われるため参加者は既にトラッカーが持っている。**FS 側に参加者リストを持たせない**のが正。
- **プロセスの区別**（ルール14/15）: 進行判定は**メインプロセス**＝行えるのは**その手番のキャスト1名**。「AR の残った参加者全員」は誤り（Code の当初案）。支援判定は**イニシアチブプロセス**で、AR の残ったキャストが自由に、**1 イニシアチブプロセスに複数人**行える。
- **敗北条件はカット経過に限らない**（ルール16）。「敗北条件までの残りカット数」はカット条件限定の表現であり、自由文一本でも、カット固定でもない → 種別を持たせる。
- **軽減の可否は完全に RL 判断**（ルール18相当・Damage_Rules へ記録）。Code は「軽減を通すかを RL に選ばせる」設計を提案したが、ユーザー裁定は**選択肢自体が不要**——口頭で伝えれば足り、防護点の可否は**種別 X** で既に表現できる。既存の表現力で足りるところに機構を足そうとしていた。

#### 設計レビュー

- **RL 任意ダメージ付与はユーザー提起・Code 賛成**。ダメージの発生源が「攻撃用途 → ダメージカード」しかないのは RL の手札が狭い。かつ 12-3/12-4 で `applyDamageToTarget` の2経路・権限委譲・トループ/エキストラ例外が揃っており、**新しい計算機構を一つも必要としない**。依存する未実装機構が無いので後続フェーズへ送らず 12 で実施する。軽減の裁定（上記）により RL 専用の分岐はゼロで、既存フローに合流するだけになった。
- **判定要求2種は 12 で作り切れる**。Code は当初「参加者の解決が 13 依存」と考えかけたが、**`Combat`/`Combatant` は Foundry コアのドキュメントで 12 でも `game.combat.combatants` を読める**（13 で作るのはトラッカー UI の上書きとプロセス進行であって、参加者という概念自体ではない）。進行判定の宛先も、**手番の自動判別を 13 に回し 12 は RL の手動選択**とすれば機能する（RL は誰の手番か分かっている）。13 での差し替えはカード構造・見た目を変えない。→ ユーザーの「このプロセスで行えることはほとんど無いかもしれない」に対し、両方 12 で完成できると回答。
- **敗北時の処理は自動化しない**（内容が FS ごとに異なる）。システムは**敗北の確定**までを担い、結果カードから RL 任意ダメージ付与へ導線を繋ぐ。役割の分離が保たれる。
- ゲージは最大値・現在値に加え**判定行の閾値をマーカーとして刻む**（切り替わりポイントの可視化・ユーザー指示）。カットゲージは `defeatCondition.type === "cut"` のときのみ。

### 2026-07-20 RL 任意付与の範囲確定（状態・報酬点の追加／経験点・手札の見送り）

#### メカニクス監査

- **報酬点の RL 配布はルール上の「前金」**（Bounty_Points「入手方法」に既記載＝RL から依頼の対価として付与される）。新しい概念ではなく、既存ルールの卓上実装。
- **経験点の減算は原作に存在しない**（2026-07-20 ユーザー裁定）。「経験点はそもそも没収ができるようなデータではない」——累積の記録であって消費可能な残高ではない。Code が RL ペナルティの候補として挙げたが**ルールの裏づけが無く、実装しない**。
- **手札の没収＝正確には強制全交換**。ありえなくはないが**手動でできる範囲**のため機能化しない（2026-07-20 ユーザー裁定）。
- ルール上の裏づけを確認せずに候補を実装しないため、機構として可能なもの（手札操作・使用回数の強制消費・経験点）は**一旦ユーザーに実在を問うてから**採否を決めた。3件中2件が「ルール上存在しない／手動で足りる」となり、問わずに作っていれば不要機能を作り込んでいた。

#### 設計レビュー

- **「AE には親ドキュメントが必要」という懸念は、この用途では成立しない**。`condition-resolution.mjs` が負傷・戦闘不能・BS を `actor.createEmbeddedDocuments("ActiveEffect", ...)` で**アクター直付け**で作っており、親はアクター自身。コード上も「ダメージ由来=供給元が浮く」と明記されている＝**供給元アイテムを持たない AE は既に正常な状態**。ユーザーの懸念に対し、手当てが要るのは親ではなく「何を付与するかの選び方」だと回答した。
- **ゼロからの AE 自作はシステム側で作らない**——Foundry 標準のアクター効果タブで既にできる。標準機能で足りるところに独自 UI を作らない。
- **RL 付与の効果は供給元と切り離す**（ユーザー確定）。用途由来の片方向同期（供給元が正）を適用すると、元アイテムを編集したときに過去に配ったデバフが書き換わる。一回限りのギミックには同期の前提が無い。
- **報酬点は負数も受け付ける**ことで配布と没収を一機構に畳んだ（別々の仕組みを作らない）。着地は `system.bounty` でシートの ±1 ボタン・判定消費・社会ダメージ軽減と同じ。所有者クリック方式により**権限委譲が不要**になり、判定要求のアクター登録方式とも揃う。
- **スコープ**: 報酬点配布はフェーズ12 の主題（攻撃判定・ダメージ・FS基幹）から外れるが、依存する未実装機構が無く FS 敗北処理と前金運用の両方で要るため先送りしない。12-4 追補を「RL 任意付与（ダメージ／状態／報酬点）」に広げ、コミットは3分割。

### 2026-07-20 情報収集判定の実装状況の記録訂正

ユーザー指摘「情報収集判定は雛形だけで実際にはほぼ何もできていないはずだが、wiki の記録と矛盾していないか」を受けてコードと突き合わせた結果、**wiki が誤って「実装済み」と記録していた**ことを確認し、該当記述をすべて実装に合わせて訂正した（コード変更なし・記録のみ）。

#### 確認した実装の実態

- 実装されているのは、アクトシート「情報」タブの**項目 CRUD**（情報名／公開フラグ／内容／調査済みフラグ／技能名テキスト＋TN／公開テキスト）と、公開済み内容のチャット送信、および技能行の ◆ ボタンから **TN 付きで判定ダイアログを開く配線**（`_onStartInfoSkillCheck`）まで。
- 未実装＝**技能アイテムを解決しない**（`skillIds: []` のため `_computeCheckBonus` の criteria が `ability` に落ち、技能レベル・技能バフ・組み合わせが一切乗らない＝実質は能力値判定に技能名ラベルを貼ったもの）／**PL 側の起動導線が無い**（RL のシート内のみ・`game.user.character` で走る）／**結果が情報項目に還らない**（`isDisclosed` は RL の手動トグルのみ）／**舞台裏・1シーン1回が無い**（シーン管理がフェーズ14以降）。

#### 誤記録の構造（再発防止）

- フェーズ7で**データ構造**が、フェーズ8で**ダイアログ起動の配線**が入り、その2つが実装状況表で「実装済み」「✅」とだけ書かれた。以降それが**メカニクスとして実装済み**の意味で参照され続けた。Phase_8 の完了条件「情報収集判定・RL 要求判定が実機で動作する」も、ダイアログが開くことをもって達成扱いになっていた。
- 一方 [Scenario_Progress](../01_Wiki/Game_Rules/Scenario_Progress.md) の申し送りは情報収集判定を「フェーズ16-17」に置いており、**同じ wiki 内で正面から矛盾**していた。矛盾に気づく機会はあったが、実装状況表だけを参照して発言していたため通過した。
  - **解消（2026-08-06 ロードマップ再編）**: 情報収集判定を**フェーズ16 の単独フェーズ**として確定し、両ページの指し先を一本化した。この矛盾が残っていた原因は、そもそも当時のロードマップに情報収集判定の**席が無かった**こと（16＝サブシーン／17＝辞典ブラウザで、どちらにも中身が無い番号を指していた）。
- **教訓**: 実装状況欄に「実装済み」とだけ書かない。**何が実装され何が未実装かを列挙する**（部分実装を単一のフラグで表さない）。データ構造・UI 配線が入った段階を、メカニクスの完成として記録しない。

#### 訂正した箇所

- [Check_Rules](../01_Wiki/Game_Rules/Check_Rules.md)「判定種別とTNの出どころ」の情報収集判定行
- [Check_System](../01_Wiki/Game_Mechanics/Check_System.md)「情報収集項目（アクトシート連動）」＝冒頭に実装状況の但し書き、「判定トリガー」を**未実装**と明示、「現状の実装」「未実装の項目」を新設、フェーズ8スコープ表の `✅ → △（配線のみ）`
- [Phase_8_Tasks_Detail](../01_Wiki/Phases/Phase_8_Tasks_Detail.md) 8-4 の該当項目と完了条件
- [Phase_12_Tasks_Detail](../01_Wiki/Phases/Phase_12_Tasks_Detail.md) 12-5 の「Code 提起（未合意）」＝局所不備ではなくメカニクス未実装の一断面である旨を追記

#### 着手時期（2026-07-20 ユーザー裁定）

> 「少なくとも、実セッションに大きく関わる部分なので、実セッションを行う土台がそろってからが良いと思います。」

- **セッション進行の土台（セッション進行＝フェーズ14／時間単位管理＝フェーズ15）が揃ってから着手する。**（着手先は 2026-08-06 の再編で**フェーズ16＝情報収集判定の単独フェーズ**に確定。当時の記述「舞台裏判定 → フェーズ16-17」は席の無い番号を指していた。舞台裏判定の**枠組み**は 14、情報収集判定は 16、購入判定は 17。）
- **技能解決・PL 導線だけを先に是正する案は採らない**（Code が選択肢として挙げたが不採用）。情報収集判定は舞台裏・基本1シーン1回というシーン単位の制約と不可分であり、土台の無いまま起動経路だけ整えても、シーン管理の実装時に作り直しになる。
- したがって**フェーズ12 では着手しない**。12-5 の「Code 提起」も是正せず現状のまま残す（記録上は未実装として正しく表示されている）。

#### KI 起票の要否（Code 判断）

- **起票しない。** これは動作の壊れた既存機能ではなく、**未着手の機能**である（KI＝既知不具合の正本は Issues だが、未実装機能の申し送りは実装状況記録とロードマップが担う）。実装状況の正本は [Check_System](../01_Wiki/Game_Mechanics/Check_System.md)「未実装の項目」、着手時期の正本は上記裁定。
- ただし「情報収集タブの判定起動ボタンが RL のキャラクターで走る」点は、**ボタンが現に存在して押せる**以上、着手までの間ユーザーの目に触れる。挙動として誤解を招くなら別途扱いを検討する（現時点では未提起）。

### 2026-07-20 RL 任意ダメージ付与の実装（T1・本体 `2c092ff`）

#### メカニクス監査

- 判定を経由しないダメージの発生源を追加した。ダメージは攻撃判定の帰結だけではない（罠・落下・爆発・環境ダメージ・FS判定の敗北時処理）→ [Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「RL による任意のダメージ付与」。
- **軽減の可否の分岐を持たない**（2026-07-20 裁定どおり）。防護点は種別 X、軽減技能の可否は自由記述ないし口頭。実装にゲートを一つも足していない。
- **リアクションが無いため受け値0・リアクション成立なし**。社会ダメージの報酬点軽減はリアクション成立がゲート（2026-07-17 規約）なので、RL 付与の社会ダメージでは起動しない。これは既存規約からの帰結であり、必要なら RL が卓で裁定する。

#### 設計レビュー

- **既存フローへの合流で実装した**。`damageRoll` フラグを攻撃由来のダメージカードと同じ形で組み立てることで、描画・軽減ダイアログ・チャート適用・トループ/エキストラ例外・権限委譲をすべて既存の経路が担う。新しい計算機構・新しい適用経路はゼロ。
- **値の置き場**: カードでも攻撃力でもないため `rlGrant {value, note}` を新設し、合計では `modifier` の段に乗せた（`damageRollTotals`）。台帳には**自由記述をラベルにした行**（`ダメージ（落下）` / 記述なしは `ダメージ`）として出し、攻撃力の段は出さない。「手動修正」に相乗りさせると主たる値が修正として表示されてしまうため避けた。
- **カードの追加ボタンを抑止**（値の直接指定でカードを出さないため）。
- 対象は**レティクルで明示**する（2026-07-18 の対象解決の規約どおり・ダメージの対象は常に明示される）。未ターゲットでは開かず警告する。
- 純ロジック `rl-grant-logic.mjs` を TDD（`buildRlDamageRollFlag` / `rlGrantAmount` / `rlGrantLedgerRow`・12件）。テスト 1509 green・eslint 0 error（警告数は変更前と同じ15＝新規警告なし）。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**: シーンコントロール「ダメージ付与」の一巡（未ターゲット時の警告・系統別・台帳の行と攻撃側合計・軽減ダイアログ・チャート適用）。社会で報酬点軽減が起動しないこと。トループ/エキストラへの付与。

### 2026-07-20 RL 任意の状態・効果付与の実装（T2・本体 `1b58549`）

#### メカニクス監査

- **効果値を RL に選ばせない**（重要）。衰弱の数字・対象制御値、重圧の対象能力値は**引いて決まる**もので選択ではない（→ [Bad_Status.md](../01_Wiki/Game_Rules/Bad_Status.md)）。付与データは状態のみ（changes なし・値なし）とし、決定は既存の `createActiveEffect` フック（`conditionNeedsDraw` → `postDrawPrompt`）に委ねた。**このフックは付与経路を問わない**（2026-07-16 の是正でそう作られている）ため、RL 付与でもカード決定ドロー・カスケード（inflicts）・制御判定による無効化・選択型負傷の技能選択がそのまま働く。入口を足すだけで既存のメカニクスが全部乗る。
- **負傷は付与対象から除いた（Code 判断・要確認）**。負傷はダメージチャートの出力であり、治療の目標値になる `woundValue` を伴う。裸の負傷を作ると治療目標値を持たない負傷が生まれるため、RL が負傷を与えたい場合は**ダメージ付与（T1）でチャートを通す**のが正しいと判断した。付与できるのは BS と戦闘不能の2群。

#### 設計レビュー

- **「AE には親ドキュメントが必要」問題は実装上も発生しなかった**。対象アクターへ直接 `createEmbeddedDocuments("ActiveEffect", ...)` するだけで親はアクター自身になる（BS が既にそう作られている）。設計時の読みどおり。
- **付与コピーの組み立てを一本化した**。`resolveUsageEffectData` に埋まっていた複製処理を `buildGrantedEffectData(eff)` として抽出し、用途の適用効果と RL 任意付与が同じ関数を通るようにした（同じ意味のコピーを二箇所で書かない）。抽出後も `usage-effects` の既存テストは green。
- **切り離しは新しい機構を必要としなかった**。片方向同期は `transferredFrom` をキーにしており、付与コピーは `grantedFrom` しか持たない（`materializeItemTransfers` / `removeItemTransferCopies` とも `transferredFrom` のみを見る）。つまり付与コピーは**既に供給元と切り離されている**。設計で決めた「切り離す」は、既存の性質をテストで固定する形で満たした。
- **ゼロからの効果自作は用意しない**（Foundry 標準のアクター効果タブで可能）。標準機能で足りるところに独自 UI を作らない。
- 供給元は「効果を持つワールドのアイテム」→ その効果、の2段プルダウン。表示は効果名・アイテム名で、内部 ID は出さない。
- テスト 1523 green（73ファイル）・eslint 0 error（警告数は変更前と同じ15＝新規警告なし）。

#### 課題・申し送り

- **要確認**: 負傷を RL 付与の対象から除いた判断（上記）。RL が負傷を直接与えたい場面があるなら再考する。
- **実機確認済み（2026-07-21・v0.12.0）**: 状態付与の一巡（衰弱・重圧でカード決定ドローのプロンプトが出ること・カスケードを持つ状態・非重複の状態を重ねたとき）。効果複製の一巡（供給元プルダウン→効果プルダウンの連動・複数対象への付与・供給元アイテムを削除してもコピーが残ること）。

### 2026-07-20 報酬点の配布カードの実装（T3・本体 `f614b39`）

#### メカニクス監査

- **ルール上の「前金」の実装**。Bounty_Points「入手方法」に既記載（RL から依頼の対価として付与される）であり、新しい概念を足していない。
- **着地は `system.bounty`・`bountyBase` は変えない**。シートの ±1 ボタン・判定での消費・社会ダメージの報酬点軽減と同じ着地に揃えた。
- **没収の下限（Code 判断・要確認）**: 負数配布は**有効報酬点（`bountyBase + bounty`）が0を下回らないところで止まる**。ルール文に明記は無いが、判定での消費が「所持数上限」で止まる（2026-07-17 の実減算）のと同じ考え方を適用した。持っている以上は没収できない。

#### 設計レビュー

- **一機構で配布と没収を担う**。負数を受け付けることで、別々の仕組みを作らずに済んだ（ボタン文言だけ「受け取る」/「支払う」、済み表示も「受け取り済み」/「支払い済み」で切り替える）。
- **権限委譲が要らない**。判定要求と同じアクター登録方式にし、受け取りは**対象の所有者**が押す＝所有者が自分のアクターを更新する。委譲が要るのはカードのフラグ更新だけで、既存の `applyMessagePatch` 一本に乗せた。
- **二重受け取りの防止**をフラグ（`received`）で行い、純ロジック側は元データを書き換えない形（`markBountyReceived` が新しいオブジェクトを返す）にした。
- **未定義の CSS クラスを作りかけたのを是正**: 受け取り済み表示に `cr-req-done` という新クラスを書いたが、CSS に存在せず無スタイルだった。判定要求の対象行が使う `cr-req-result` + `cr-inline-success` に載せ替え、CSS の追加はゼロ。
- テスト 1534 green（74ファイル）・eslint 0 error・stylelint clean。純ロジック11件（配布データ・着地値と下限・二重受け取り）。

#### 課題・申し送り

- **要確認**: 没収の下限（有効報酬点0で止まる）。
- **実機確認済み（2026-07-21・v0.12.0）**: 配布カードの一巡（対象所有者だけにボタンが出ること・受け取りで `system.bounty` が増減しシートの表示が変わること・受け取り済みのライブ書き換えが全クライアントに乗ること・負数配布の文言・所持を超える没収が0で止まること）。

### 2026-07-20 FS判定基幹の実装（T4〜T9・本体 `a01efa2` / `434cd6b` / `0d76b0f` / `21e5c69` / `d21a53f` / `78ea97d` / `b73857c`）

#### メカニクス監査

- **判定行の有効判定を一つの純関数に落とした**（`activeProgressRow`）。「閾値 ≦ 現在進行値 のうち閾値が最大の1行」で、ルール8（達したら切り替わる）・ルール9（飛ばされた行は無視）・ルール12（閾値ちょうど）が**すべてこの定義から従う**。テストは初期値行が進行値0で有効・閾値ちょうどで切り替わる・3を飛ばして5へ進むと3の行が使われない・並び順に依存しない、を固定した。
- **支援判定に専用の目標値を持たせない**（ルール5＝進行判定と同じ）。`buildSupportRequest` は有効行の `targetValue` を使うため、進行値が進むと支援判定の目標値も追随する。二重管理が生じない。
- **進行修正はアイテムを式キーで特定しない**（ルール4・2026-07-20 ユーザー指摘）。FS は「何を見るか」だけを持ち、解決値を `@param` として式に注入する。係数は式で表す。アウトフィットの実解決（判定者の準備済みアイテムから探す）は**フェーズ13**。
- **敗北条件はカット経過に限らない**（ルール16）ため種別を持たせ、`cut` のときだけカットゲージと「残りカット数」を出す。`other` では出さない＝「残りカット数」がカット条件限定の表現になる。
- **敗北時の処理は自動化しない**（ルール17）。結果カードに掲示し、適用は RL 任意付与（T1〜T3）で行う。
- **参加アクター＝コンバットトラッカーの登録者**（ルール13）。FS 側に参加者リストを持たない。

#### 設計レビュー

- **進行状態の正本はワールド設定**（`activeFocusSystems`）。FS はカットをまたぐ長期状態で、単発で閉じる判定要求カードとは機能が違うため、カードのフラグを正本にしない（形でなく機能で流用を判断）。開始カード・結果カードは**通知に徹し状態を持たない**。13 のトラッカーは同じ正本を読むだけで済み、正本の引っ越しが起きない。
- **スナップショット方式**（`buildFocusSystemSnapshot`）。起動後に FS判定シートを編集しても実行中の FS は変わらない。テストで「ページを編集しても実行中 FS が変わらない」を固定した。
- **`Combat` / `Combatant` は Foundry コアのため 12 で読める**という設計時の読みが実装でも当たった。13 で作るのはトラッカー UI の上書きとプロセス進行であって参加者の概念ではない。進行判定の宛先は RL の手動選択（手番の自動判別は 13）、支援判定は参加者全員（AR 絞り込みは 13）。**先送りは絞り込みの自動化だけ**で、カード構造・見た目は 13 で変わらない。
- **判定要求カードの投稿を `postCheckRequest` に切り出した**。判定要求ダイアログと FS の要求2種が同じ関数を通る（同じ意味のカード生成を二箇所で書かない）。FS の文脈は `extra` でフラグに載せ、13 が結果を受け取れるようにした。
- **ゲージは既存の `outfit-slot-gauge` を流用せずバー式部品を新設**（ピップ式は 5〜10 個規模向けで進行値 30 に合わない）。セレクタは `.tnx-fs-panel` 配下にスコープした。
- **パネルは全員に表示**（PL の参照手段を兼ねる）。編集は RL のみで、書き込みは `focus-system-state.mjs` の一点で権限を確認する。
- **プリセットは対象アクターを含めない**（2026-07-20 確定）。`checkRequestPresetToForm` / `bountyPresetToForm` が対象を写像しないことをテストで固定した。
- テスト 1593 green（79ファイル）・eslint 0 error・stylelint clean。純ロジック追加＝進行修正13件／FS状態17件／要求7件／プリセット14件／FSスキーマ8件。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**（FS判定 一式）: FS判定シートの作成と編集（敗北条件の種別切り替え・判定行の追加/削除/並び替え・進行修正の参照元→パラメータ連動）。パネルの一巡（未起動時の文言・読み込み元プルダウンからの起動・ゲージと閾値マーカー・進行値/カットの増減・有効行の切り替わり・達成/敗北の確定と結果カード）。進行判定/支援判定の要求カード（宛先の単一/複数・目標値が有効行に追随すること）。アクトシートの判定要求タブとプリセット読み込み（判定要求・報酬点の両方）。
- **フェーズ13 へ**: 進行値の自動加算（`floor(@diff/10) + 進行修正`）・支援判定の AR 消費と +1・宛先の自動絞り込み（AR 残量／手番判別）・`progressMod.source === "outfit"` の実解決・カット進行と `cut` の連動。

### 2026-07-20 宣言（declaration）の実行時棚卸し（T10・コード変更なし）

フェーズ12 スコープ #6「declaration の実行時棚卸し（薄い配線のみ・本格実装は神業フェーズ＝現18・旧20）」。`_activateItemCheck` の分岐を先頭から順に追い、宣言用途が到達しうる経路をすべて確認した。**不足は見つからず、コード変更は行っていない。**

- 分岐順は NPC取得 → クリック待ち系 → 治療/回復 → 攻撃 → 対決判定 → 固定達成値 → 宣言 → 通常判定。
- **NPC取得は宣言より前**に分岐するため、宣言タイプ＋NPC取得の用途が消費だけで終わることはない。
- **クリック待ち系（再判定を付与／判定を修正／ダメージを修正／スートを変更）は `check` と `declaration` の両方**を受けており、バフ宣言が素通りしない。
- **対決判定の分岐は `executionFormOf === "check"` で閉じている**ため、宣言が対決カードへ流れない。
- **`executionFormOf` は `selectableForm` を持つタイプでのみ `executionForm` を読む**。したがって、他タイプに古い `executionForm: "declaration"` が残っていても誤経路に落ちない（当初この点を取りこぼしの候補として疑ったが、ゲートが既にあった）。
- **固定達成値は `isFixedCheck` のときだけ入力欄が出る**ため、宣言タイプに固定達成値が入って判定結果カードが出ることはない。
- 宣言本体（`_useDeclarationUsage`）は 消費 → 適用効果ペイロード → 使用カード（効果があれば「効果を適用」ボタン）／無ければ通知、まで配線済み。

**本格実装（割り込み機構としての宣言）は神業フェーズのまま**（2026-08-06 再編で番号のみ 20 → **18**）。

### 2026-07-21 実機報告への是正（ダメージ種別・S/P/I/X・対象選択の統一）

#### メカニクス監査

- **ダメージ種別の表記は S/P/I/X そのものが正式**（2026-07-21 ユーザー指摘）。「斬撃／貫通／衝撃／装甲無視」は**正式なルール用語ではない**。UI に出していた3箇所（ラベル表 `ATTACK_DAMAGE_TYPES`・アウトフィットシート・用途シート）を是正し、コメントの言い回しも直した。
  - **wiki の記録も誤っていた**: Outfits.md に「S（Slashing=斬撃）…」と Code の付け足しをルール記録として書いていた（Damage_Rules.md にも併記）。UI だけ直して記録を残すと再発するため両方撤回した。
- **プレイヤーキャラクター＝ユーザー設定で割り当てられたアクター**（2026-07-21 ユーザー確認）。対象選択の「プレイヤーキャラクターをすべて追加」はこの定義で収集する。

#### 設計レビュー

- **用途のダメージ種別に X が無かった**（フェーズ8 の新設時＝2026-06-17 から欠落）。原因は**既にあるラベル表を使わず選択肢をテンプレートへ書き写したこと**。アウトフィットシートは同じ表から組み立てていたため4種そろっており、写した側だけが欠けたまま1か月以上残っていた。
  - **選択肢を1つ足すのではなく、表から組み立てる形へ直した**（同じ欠落が再発しない構造にする）。
  - **同種の重複を全テンプレートで掃引**し2件解消: 判定種別（`CHECK_TYPE_LABELS` ＋ 判定要求ダイアログ ＋ アクトシート判定要求タブ＝**3箇所**）／敗北条件の種別（FS判定シート ＋ 起動フォーム＝**2箇所**）。後者2件はいずれも 2026-07-20 に Code 自身が作った重複。
  - **Code の初動が誤り**: アウトフィット側と揃っていない時点で写し漏れを疑うべきところを「ルールの判断が要る」としてユーザーに判断を仰いだ。判断を求める事柄ではなく単なる欠落だった。
- **RL ダメージ付与にダメージ種別の入力が無かった**。「防護点で軽減できないダメージは種別 X で表現できるから軽減の有無を選ぶ UI は不要」という 2026-07-20 の裁定を記録しながら、その X を選ぶ手段を実装していなかった＝**裁定の前提が成立していなかった**。物理のときだけ種別を選べるようにし、選んだ種別は台帳にも出す（どの防御力で軽減されるかの根拠）。
- **対象選択を4つのダイアログで統一**（2026-07-21 ユーザー指示）。判定要求・報酬点の配布・RL 任意ダメージ付与・RL 任意の状態/効果付与で、チェックボックス形式とレティクル必須をどちらも廃止し、共通部品 `target-picker` に集約した（ダイアログごとに実装を書かない）。追加は選択/ターゲット/キャスト全員/PC全員の4通りで重複を畳む。行はグリップのドラッグで並び替え（専用グリップのみ draggable＝部位プリセットと同方式）。
  - **判定要求はアクター登録方式（ID で引く）**なので、トークン由来の uuid の末尾をアクター ID と見なすと非リンクトークンで解決に失敗する。収集時のアクター ID をリストに持たせて uuid の末尾に頼らない形にした。
  - **UI 是正（同日・実機報告）**: ラベル付きの2列グリッドが幅 440px のダイアログ（判定要求だけ 520px）から溢れていた。**~250px 幅で収まるかの確認を怠った**もの。ユーザー提案の Monks Token Bar 踏襲（表形式＋アイコンのみボタン・ラベルは title＝操作アフォーダンス）を採用し、ボタンが文字幅に依存しない形にした。名前は `min-width: 0` ＋省略記号で溢れさせない。対象選択はどのダイアログでも**一番下**に統一。
- **対象行が縦積みになっていた**（同日・実機報告）。行の横並びが `.rl-check-label`（チェックボックス行用）にしか無く、読み取り専用の行では効いていなかった。**レイアウトは行自身が持つ**形へ移した。
- テスト 1620 green（80ファイル）・eslint 0 error・stylelint clean。純ロジック追加＝対象選択18件・ダメージ種別9件。

#### 課題・申し送り

- **FS判定の進行判定/支援判定の要求は今回の統一に含めていない**。宛先がルールで決まる（進行判定＝メインプロセスの手番1名／支援判定＝参加者全員）ため、自由に対象を組み立てる形式とは意味が違うと判断した。揃えるべきかは要判断。
- **実機確認済み（2026-07-21・v0.12.0）**: 対象選択の一巡（4ボタンの収集・重複の畳み込み・グリップ並び替え・除去・すべて外す・狭いダイアログでの収まり）。用途シート／アウトフィットシートのダメージ種別表示。RL ダメージ付与の種別と台帳の種別行。

### 2026-07-21 効果の付与が実質できなかった問題の是正（付与元の拡大＋事前設定）

#### メカニクス監査

- ルール上の新規判断は無い。RL がギミックとして与える効果の**中身**は既存の AE 機構（[Active_Effects](../01_Wiki/Game_Mechanics/Active_Effects.md)）で表現され、本件は「どこから選ぶか／いつ組むか」の問題に閉じる。
- **AE の設定はアクト中に行うには重い作業である**（2026-07-21 ユーザー指摘）。卓の最中に組ませる前提の設計は成立しない、という運用上の制約を記録する。

#### 設計レビュー

- **効果の付与分岐が最初から死んでいた。** 付与元の母集団を「効果を持つ**ワールドアイテム**」に限っていたが、TNX では効果はアクターの所持アイテムに乗るのが通常で、同梱パックにも効果を持つアイテムが無い。したがって母集団は常に空で、ダイアログは「効果を持つアイテムがありません」を出すだけだった。
  - **Code の誤りは二重**。①母集団が空になることを実装時に確かめていない（`game.items` に効果が載る運用が TNX に無いことは既存コードから分かる）。②「ゼロからの自作は Foundry 標準の効果タブでできるからシステム側では作らない」と設計判断として記録したが、それは**対象ごとに各アクターのシートを手で編集する**ことであって付与機能ではない。
  - **ユーザーの当初の懸念（「AE は親ドキュメントが必要なのでどうするか考えたい」）に正面から答えていなかった。** 本旨は Foundry の親の要件ではなく**持ち主のいない効果の定義をどこに置くか**であり、Code は狭い技術的問いにだけ答えて「問題は存在しない」と結論した。**問われていない問いに答えて解決を宣言する**形の誤り。
- **是正**: 付与元を3通りに拡大（アクトのプリセット／**任意のアクターの所持アイテム**とワールドアイテムの効果／新規作成）。選択値は種別と参照先を1つの文字列に畳む（`effect-source-logic.mjs`）。uuid に `.` が含まれるため区切りは `|` とし、後半を分割しないことでトークン由来の uuid でも壊れない。
  - 付与コピー（`grantedFrom`）・転送コピー（`transferredFrom`）は既に誰かに乗っている**実体**であって定義ではないため、付与元の一覧から除外する。
- **事前設定**: アクトシートに効果プリセットを新設し、**効果を実体で持たせた**（参照にしない）。アクトのページを持ち出せば効果も付いてくること・供給元の削除で孤児が出ないことによる。実体である以上どこにも紐づかないため、付与コピーに `grantedFrom` は刻まれない（`buildGrantedEffectDataFrom` に由来 uuid が無い場合の分岐を追加）。
- **独自の効果エディタは作らない。** 変更キーの入力補助（注入 UI）は標準シートの `renderActiveEffectConfig` に乗っており、二重に作れば必ず乖離する。ActiveEffect は Actor か Item の中にしか存在できないため、持ち主のいない効果を標準シートで組ませるには置き場が要る——システムが**下書きアイテム**を1つ持ち、その中で組ませてデータだけ取り出す（アイテムディレクトリからは隠す・中身は毎回空に戻す）。確定は**フォーム送信**、押さずに閉じたら取り消し。
- **タブ名の是正**: 「判定要求」タブに報酬点・ダメージ・効果まで載ると名前が実態と合わない（2026-07-21 ユーザー指摘）。**タブを分けず「RL プリセット」へ改名**した。4種とも性格が同じでセクション見出しで分かれており、アクトシートのタブを増やす方が損。
- **重複の予防**: ダメージの系統の選択肢が付与ダイアログ内に閉じていたため、アクトシートからも使う時点で `RL_DAMAGE_CATEGORIES` として純ロジック側へ移した（2026-07-21 に是正した「選択肢の書き写し」を繰り返さない）。
- テスト 1648 green（81ファイル）・eslint 0 error・stylelint clean。純ロジック追加＝付与元14件・プリセット8件・素データからの付与コピー6件。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**（一式）。特に: 効果の「組む」で標準シートが開き、送信→閉じるでプリセット/その場作成に反映されるか。下書きアイテムがディレクトリに出ないか。アクター所持アイテムの効果が付与元に並ぶか（世界が大きいと一覧が長くなる可能性）。
- **効果を1つも変更せず「更新」を押して閉じた場合**、Foundry が差分なしで更新を発火しないため取り消し扱いになる。結果は「変更なし」で一致するため実害は無いが、新規作成では「未作成」のままとなる。

### 2026-07-21 効果プリセットの表示是正（用語・寸法・体裁）と hidden 不発の発見

#### 設計レビュー

- **「組む」は日本語として成立していない**（2026-07-21 ユーザー指摘）。Code が「組み立てる」から作った語で、UI の操作ラベルの語彙ではない。正しくは**「編集」**（未作成時は「作成」）。
- **ボタンが他の UI に被っていた。** Foundry の `button` は既定で全幅に広がるため、文字ボタンを置いた時点で欄いっぱいに伸びる。**既存の効果リストがアイコン操作（`tnx-icon-ctrl`・title に一語）で統一されているのに、そこを見ずに文字ボタンを新造したことが原因**。文字ボタンをやめて既存の操作様式に揃えた（寸法問題も同時に消える）。
- **アクトシートの効果セクションが名前とボタンだけだった。** 効果の表示にはシステム標準の様式（`tnx-item-list--effects`＝アイコン＋名前＋操作）が既にあり、それを使わず一行を自作したため、他のプリセット節と比べて明らかに未完成に見えていた。標準様式へ載せ替え、**要約列**（状態名・変更の件数）を足して開かずに見分けられるようにした。要約に**変更キーの生値は出さない**（表示ラベルを引けない状態は件数へ丸める）。
  - 教訓の再確認: 新しい表示を作る前に**同じものを既に表示している場所を探す**。効果は複数箇所で表示されており、様式は確立していた。

#### 不具合（同時に発見・是正）

- **`hidden` 属性による出し分けが全て効いていなかった。** `.rl-form-group` などに `display: grid` / `display: flex` を指定しているため、UA スタイルシートの `[hidden] { display: none }` が著者スタイルに負ける。`.tokyo-nova [hidden] { display: none !important; }` を追加して打ち消した。
  - 影響範囲は 14 箇所。**2026-07-21 に入れた「ダメージ種別は物理のみ」も実際には隠れていなかった**（精神・社会でも種別欄が出たまま）。FS判定シート／起動フォームの敗北条件の出し分けも同様に不発だった（2026-07-20 実装分）。
  - 素の `div`（`.effect-section` 等）では効いていたため、部分的に動いて見えていた。**出し分けを入れたら、対象要素に display 指定があるかを確かめる。**
- テスト 1654 green（81ファイル）・eslint 0 error・stylelint clean。要約の純ロジック6件追加。

### 2026-07-21 FS判定の作り直し（ドキュメント階層の誤り・起動フォームの欠落・支援判定の単数化）

#### メカニクス監査

- **支援判定の指定技能は複数持てる**（2026-07-21・現物のシート PDF をユーザーが提示して確定）。シートの「支援判定」欄は**ブロックのヘッダにある自由記入欄**（敗北条件と同じ行）であって、判定行の列ではない。したがって帰属先は**FS判定（ブロック）単位**で、判定行ごとではない。自由記入である以上、複数の技能を書ける。
  - Code は `supportSkillKey`（単数の文字列）として実装していた。**単複が未確定であることを自覚しながら、確認も現物確認もせず単数と決めた**もので、根拠は記録のどこにも無い。
- 紙面上の上限（進行値カウンター 1〜30／カットカウンター 1〜10／判定行 5行）は**シートで表現できる上限であってルール上の上限ではない**（2026-07-21 ユーザー明言）。実装で上限を設けない。
- シート冒頭の「ルーラー」「アクトタイトル」欄は実装しない（2026-07-21 ユーザー指示）。

#### 設計レビュー

- **FS判定シートを JournalEntryPage のサブタイプとして登録したのは誤り。** ユーザーの指示は一貫して「ジャーナルエントリーのタイプ」であり、手本として**アクトシート**が名指しされていた。Code は「1ドキュメント＝1ページ＝1FS判定」という**粒度**の発言を**登録先**の指定として読み、さらに「JournalEntryPage は DataModel のサブタイプを持てる」という技術的な都合が誤読と噛み合ったため、都合の良さを読みの正しさの証拠として扱った。
  - **アクトシートを一度も見ていない。** このシステムで唯一の既存カスタムジャーナルシートで、`Journal.registerSheet` ＋ flags ＋ ディレクトリの作成ボタンという答えがそこにあった。
  - JournalEntry はサブタイプを持てない（`hasTypeData` = false）ため「タイプ」は sheet+flags で実現するしかない。**これは判断の要る分岐だったが、ユーザーに出さず、黙ってドキュメントの階層を変えることで回避した。**
  - 結果として**作成導線が存在しないシート**になっていた。Code は登録コードの存在だけを見て「存在しています」と回答しており、**到達経路を先頭から辿る検証をしていない**（到達可能性の確認は既知の規約）。
- **起動フォームに判定行が無かった。** 「判定要求の UI を踏襲」を画面の形のコピーと読み、スカラー項目だけを並べた。判定行は繰り返し構造でエディタが要るため省き、**「判定行は読み込み元の FS判定シートから取り込まれます」という一文を UI に書いて欠落を仕様のように見せかけた**（解説文を UI に置かない規約にも違反）。
  - 実害: `buildProgressRequest` / `buildSupportRequest` は有効行が無いと null を返すため、**手動設定で開始した FS判定は進行判定も支援判定も要求できない抜け殻**だった。効果の付与と同じ「分岐そのものが死んでいる」失敗。
  - 是正: **FS判定シートと起動フォームで同じエディタ部品を使う**（`parts/focus-system-editor.hbs` ＋ `focus-system-form.mjs` ＋ `focus-system-editor.mjs`）。別々に作れる限り同じ乖離が再発するため、構造で塞ぐ。読み込み元の内容は**エディタへ流し込んでそのまま直せる**（以前は見えない変数に保持していた）。
- **配置は紙のシートの位置ではなくコントロールの性質で決めた**（2026-07-21・「シートをそのまま踏襲するのは賛成しない」を受けて）。①概要＝ラベル+入力の対→2カラム ②支援判定＝可変長リスト→独立ブロック（既存の「無視する指定技能」と同じ作法を流用） ③判定行＝互いに比較するレコード→全幅の表 ④メモ＝全幅。窓幅は③の列から逆算した（先に数字を決めない）。
  - 判定行を表にできる根拠: 進行修正は参照元＝なしのときパラメータも式も出ないため、多くの行でセルはプルダウン1つに収まる。
- 複数の指定技能を持つ要求は、応答時に**どの技能で判定するかを先に選ぶ**1段を足した。KI-025 の二段階化（用途プルダウン）には手を入れていない。
- テスト 1661 green（81ファイル）・eslint 0 error・stylelint clean。純ロジック追加＝FS設定の正規化14件。

#### 根本原因（同日、ユーザーが指摘）

**未確定に出会ったとき、黙って埋めて先に進む**動作が、この一連の失敗すべてに共通している（効果の付与元／支援判定の単数／ドキュメント階層／用途のダメージ種別 X／起動フォームの判定行）。さらに**決めた事実が残らない**——wiki には「設計」として書くため、未確定だった痕跡が消え、後から検算されない。発覚は常に実機で、ユーザーが見つけたときになる。

**是正**: 未確定だと気づいた時点で決めない。現物（シート・コード）かユーザーの発言に当たる。当たれないなら決めずに出す。今回の FS判定シート PDF は、Code が最初に提示を求めるべきものだった。

#### 課題・申し送り

- **実機確認済み（2026-07-21・v0.12.0）**（一式）。特に: 作成ボタンからシートが作れるか、エディタの保存と再描画、起動フォームの読み込みと開始、複数指定技能の要求と応答。
- **判定行の技能は単数のまま**。シートの「技能」列も自由記入欄なので複数を許すべきかは未確認（支援判定と同じ論法が成り立つ）。**決めずに残す。**
- FS判定の進行/支援判定の宛先選択は、他の4ダイアログで統一した対象選択部品を使っていない（宛先がルールで決まるため別扱いとした 2026-07-21 の判断のまま）。

### 2026-07-21 フェーズ13-2 カット進行の器（TnxCombat/TnxCombatant・登録・シード集約・手番順ロジック）

**2026-07-21** / **関連実装** `scripts/combat/tnx-combat.mjs`・`tnx-combatant.mjs`・`scripts/module/combat-turn-order.mjs`・`combat-seed-logic.mjs`・`scripts/tnx.mjs`・`eslint.config.js` / **ステータス**: 実装済（実機確認は 13-4 以降）

#### メカニクス監査

- 手番順＝**CSカレント降順**。同値時の優先順位は ①CSベース降順 ②ゲスト→キャスト→トループ ③話し合い（自動化不可・卓判断） ④ユーザー順（Combat_Flow §3）。③は扱わず、②④＋最終タイブレーク（id）で決定的に並べる。
- **エキストラは手番を持たない**（2026-07-21 ユーザー裁定）。行動できず、戦闘に巻き込まれ攻撃されればその場で死ぬだけの存在（護衛系 HO で護りきる対象にはなりうるが手番はない）。手番順・次のメインプロセスから**種別で除外**（トラッカーには対象として残りうる）。Code は当初「原文の②に無いので最下位へ仮置き」としたが誤り＝行動しないので順序に現れない。→ Combat_Flow §3 に記録。
- 次のメインプロセス＝**CSカレント最大かつ AR≥1** の1体（Combat_Flow §4）。行動不能による AR−1 の確認はこのロジックでは扱わない（進行管理＝13-3）。

#### 設計レビュー

- 13-2 の範囲は「器」＝クラス新設・CONFIG 登録・カット開始シードのロジック集約・純ロジック（手番順/シード算出）まで。プロセス状態機械・CS/AR 自動記帳・トラッカー UI・境界イベント・FS 接続は 13-3 以降。
- 純ロジックを Foundry 非依存モジュールへ分離しテスト（`combat-turn-order.mjs`＝13件、`combat-seed-logic.mjs`＝4件）。サブクラスは薄いグルー。
- **シード統合の範囲を一部 13-3 へ**（明示）: 13-2 ではシードのロジックを `TnxCombat` の static（`seedStartValues`/`refreshDisplays`）へ集約し、**トリガーは既存フックのまま（挙動不変・ゼロ回帰）**とした。ライフサイクル（`startCombat`/`_onEnter`）への完全移行は、CS/AR 自動記帳を一箇所で組む 13-3 に回す（半端な二重管理・二重発火の回避）。
- `eslint.config.js` の globals に `Combat`/`Combatant` を追加（`Item`/`ActiveEffect` と同様）。
- テスト 1682 green（83ファイル）・eslint 0 error。

#### 課題・申し送り

- **実機確認は未実施**。トラッカー UI が入る 13-4 以降でないと目視できないため、それまではロジックの正しさをテストで担保する（ユーザー了承 2026-07-21）。CONFIG 登録・シード回帰の実機スモークは後続で。
- シードのライフサイクル移行・プロセス状態機械・CS/AR 自動記帳・`_sortCombatants` 適用・手番判定の写像（combatant→素データ）は 13-3/13-4。

### 2026-07-22 フェーズ13-3 プロセス状態機械＋CS/AR 自動記帳＋CSカレントの凍結モデル改修

**2026-07-22** / **関連実装** `scripts/module/combat-progression.mjs`・`scripts/combat/tnx-combat.mjs`・`scripts/tnx.mjs`・`scripts/data/actor/common/character-base.mjs`・`attributes.mjs` / **ステータス**: 実装済（トラッカー UI＝13-4・実機確認は 13-4 以降）

#### メカニクス監査

- プロセス進行（戦闘準備→セットアップ→(イニシアチブ→メイン)×n→クリンナップ）を状態機械で表現。遷移は RL 操作、記帳（CS/AR）は自動（承認済み方針）。
- 記帳（Combat_Flow §4-6）: セットアップ開始で `current←CS実効値`・AR←付与値／セットアップ末で `current←valueTotal+currentBuff`（確定）／メジャー後 AR−1・CSカレント0／待機 CSカレント1／イニシアチブ行動不能 AR−1／クリンナップ AR 全回復。
- **CSカレントの「一度計算して凍結」モデル（2026-07-21〜22 ユーザー確定）**: CSカレント＝CS＋CSカレントへの AE を決定時（セットアップ末）に一度計算して保存値へ焼き込み、以後再計算しない。**メジャー後0/待機1 は AE でも変更されない（ハード）**。CS を変える効果は次セットアップの再計算まで CSカレントに反映されない。→ Combat_Flow「CSカレントの一度計算して凍結モデル」。
- エキストラは手番を持たない（13-2 裁定）を手番順・次の手番から除外済み。

#### 設計レビュー

- 純ロジック（TDD）`combat-progression.mjs`（遷移妥当性・AR−1・各記帳の更新オブジェクト・セットアップ末確定 `buildSetupConfirmUpdate`）。`TnxCombat` は薄い駆動メソッド（GM ガード・純ロジックを適用）。`combatStart→enterSetup` 配線でシードをライフサイクル統合（13-2 で 13-3 送りにした分を完了）。
- **CS 派生の改修（§4.3・ユーザー承認）**: CSカレントへの AE の着地先を実効 `currentTotal` から蓄積用の派生値 `currentBuff` へ変更（`_resolveBuffApplications` の cs.current マップ）。`currentTotal = current`（凍結）。決定時に `current ← valueTotal + currentBuff` を焼き込む。ユーザーが「AE は CSカレントに乗るが、CSベース/CS/CSカレントは一度だけ計算して再計算条件まで再計算しない」と明示したため、旧「決定値＋修正値（ライブ）」を CSカレントについて精密化。当初 Code は「cs.current の AE 廃止」「value 層へ移す」を提案したが、いずれもユーザーが否定（AE は CSカレントに乗る／前段階ではない）→ 現形に確定。
- 検証: cs.current の AE が currentBuff に着地し currentTotal に乗らないことを `_applyEffectBuffs` 実走テストで裏取り。テスト 1696 green・eslint 0 error。**トラッカー UI が無いため実機確認は 13-4 以降**（ユーザー了承）。

#### 課題・申し送り

- RL 操作メソッド（enterInitiative/assignMain/endMain/declareWait/declareCantAct/enterCleanup）の UI 起動＝13-4。手番順の `_sortCombatants` 適用・現在プロセス表示も 13-4。
- 実機確認（カット開始→各プロセス→記帳→凍結の一巡・setup バフの焼き込み・メジャー後0のハード）は 13-4 UI 後。
- 待機/行動不能/カット終了など RL 裁定箇所は強制しない（表示・補助に留める）。
- **注記（2026-07-22）**: 本エントリの離散プロセス遷移・assignMain・endMain(didMajor) は 13-4 のサブターンモデル再設計で棄却・置換された（下記エントリ）。

### 2026-07-24〜25 フェーズ13-7 FS実接続（①〜⑥・実機確認済 2026-08-06・v0.13.0）

**2026-07-24〜25** / **関連実装** `focus-system-logic.mjs`（`computeProgressGain`）・`progress-mod.mjs`（`resolveProgressModForActor`/`outfitFieldNumber`）・`focus-system-result.mjs`（新規・①手動/②自動）・`focus-system-request.mjs`（③④宛先/自動送信）・`focus-system-state.mjs`（`advanceFocusCuts`）・`tnx-focus-system-panel.mjs`＋`focus-system-panel.hbs`（⑥表示）・`tnx.mjs`＋`tnx-socket-handler.mjs`（フック配線） / **ステータス**: 実装済・**実機確認済（2026-08-06・v0.13.0）**。②は 2026-08-05 再設計で AE 化・下記改訂参照

#### 追加メカニクス監査（②〜⑥・2026-07-25）

- **②支援＝結果確定で自動適用**（2026-07-24 ユーザー確定＝ダメージと性格が違う）: `_onCheckResult` で AR−1（成否問わず）＋成功なら FS 全体の `pendingSupport` +1。①の進行適用が読み取り＋消費。**「次に進行判定を行うキャスト」が受け取る**ため対象別でなく FS 全体の1カウンタ（順番が入れ替わっても「次」は「次」・2026-07-25 ユーザー確定）。二重適用ガード。
  - **〔2026-08-05 改訂〕**②は下記「AR・メジャー・割り込み・支援の再設計」で置換。**AR−1 の直接引きは廃止**（メジャー記帳＝一般則がイニシアチブ終了時に AR−1＋CS0）・**進行ボーナスは対象キャラに乗る ActiveEffect**（支援者が判定時に普通にターゲット＝レティクルで対象1体を指定・成功で対象へ支援 AE(進行+1) 付与・対象の次回進行判定で合算して消費＝AE 除去）。専用 `pendingSupport` カウンタは廃止。「次に進行判定を行う者が受け取る」「FS 全体1カウンタ」は撤回。AE は対象に乗る＝FS 非スコープ。
- **③④＝カット進行への合流（自動送信）**: `tnxProcessStart` 購読で、メイン開始→手番キャスト（ルール14）へ進行判定・イニシアチブ開始→AR残の参加者（ルール15）へ支援判定を各実行中 FS に自動送信。手動ダイアログも既定を自動判別。
- **⑥cut連動＋カウントダウン**: `tnxCutEnd` で cut型 FS の経過カット +1。パネル表示は `cutLimit−経過` のカウントダウン（内部は経過カット＝round と同じ向き・2026-07-24 ユーザー確定）。敗北確定は RL 操作のまま（ルール17）。

#### メカニクス監査（①）

- **獲得進行値 ＝ `floor(差分値÷10 ＋ 進行修正) ＋ 支援ボーナス`**（ルール3＋5）。端数切り捨ては和全体（進行修正は floor 内）・支援は獲得進行値への加算（floor 外・ルール5訂正=2026-07-23）。差分値 null（失敗/ファンブル）は進行なし。
- **進行修正の解決**（ルール4）: none=固定/式・actor=アクター system 値（`reason.total` 等の直下パス）・outfit=判定者の**準備済み**アイテムの値（`{mode,value}`/直値/attackField を `total→value→数値` で吸収・準備済み複数は選択）。
- **適用はダメージと同じ手動ボタン**（2026-07-23 ユーザー確定＝自動加算でなく RL がボタンで反映）。進行判定要求カード（`extra.focusSystemKind==="progress"`）の**成功**対象行にボタン、押下で FS 進行値へ加算・カードは反映済み（再適用防止）。

#### 設計レビュー（①）

- 純ロジック `computeProgressGain`（TDD・null 差分の Number 化ガード込み）。`outfitFieldNumber`（フィールド形の吸収・TDD）。Foundry グルー `resolveProgressModForActor`（source 別 paramValue 解決→既存 `resolveProgressMod` の式評価へ）・結果カードのボタン描画は checkRequest 描画の**後**に登録して結果表示へ追記。更新は RL(=GM) のみ（world 設定）。
- テスト 1795 green・eslint 0・stylelint clean。

#### 申し送り・実機確認時の要点

- outfit の参照アイテム選択ダイアログは進行適用時＝RL 側に出る（判定者本人でなく RL が選ぶ）。
- **自動送信は各実行中 FS ごとに送る**（並行 FS が複数なら手番/イニシアチブ開始で複数の要求カードが出る）。**割り込みメイン（追加行動/イニシアチブ割り込み）でも `tnxProcessStart(main)` が出るため進行判定が自動送信される**（割り込み行動者が対象）。実機で過多／不要なら送信条件を絞る。
- カットのカウントダウン表示は現在値＝`cutLimit−経過`。パネルの手動 +/− は**表示（残りカット）**に合わせる（＋＝残りを増やす＝経過 cut −1／−＝残りを減らす＝cut +1）。〔2026-08-06 是正〕当初は経過カットを直接増減し「＋で表示が減る」直感逆だったのをユーザー指摘で修正。
- 〔2026-08-05 改訂〕支援ボーナスは対象キャラに乗る **ActiveEffect**（進行 +1）へ移行。進行判定を行う本人に乗った支援 AE を合算・消費（除去）する。専用 `pendingSupport` カウンタは廃止。

### 2026-07-23 フェーズ13-6 カット進行の境界イベント発火＋シーン終了確認（案1・実機確認済）

**2026-07-23** / **関連実装** `scripts/module/combat-events.mjs`（新規・`TNX_HOOKS`/`planPhaseEvents`）・`scripts/combat/tnx-combat.mjs`（startCombat/advancePhase/startInterrupt/endInterrupt の発火・`endCombat` override）・`css/tnx2.css`（ダイアログ縦積み） / **ステータス**: 実装済・実機確認済（2026-07-23 ユーザー）

#### メカニクス監査

- **境界イベントを発火するだけ**（正本 Combat_Flow「境界イベント」）: カット進行（＝シーン）開始/終了・カット境界（クリンナップ完了→次セットアップ）・プロセス遷移で `Hooks.callAll` する。**適用（効果の失効・回復・カウンタのリセット）はフェーズ15 が購読**——13-6 には適用ロジックを書かない。
- **開始はシーンと連動・終了は非連動**（Combat_Flow 裁定）: `tnxCutProgressionStart`＝シーン開始。終了は案1ダイアログでシーンも終わるか選ぶ（`tnxCutProgressionEnd` は常に・`tnxSceneEnd` は選択時のみ）。RL がシーン内で治療まで認める運用のため。
- **プロセスイベントは開始／終了の2エッジ**（2026-07-23 ユーザー指摘）: from→to の1本では「メインA終了＋メインB開始（割り込み）」を表せない。**挿入メイン（割り込み）も「メインプロセス」**として、その開始・終了、および割り込みで打ち切られた元メインの終了で `tnxProcessEnd(phase:"main")` を発火する＝「メインプロセス中」効果が挿入メイン・打ち切られた元メインでも失効できる。サブターンは割り込みで中断のみ（開始/終了を再発火しない＝サブターン中効果は割り込みを跨いで持続）。

#### 設計レビュー

- 純ロジック（TDD）: `planPhaseEvents({fromPhase, fromMainId, toPhase, toMainId, nextCut, round})`＝離脱プロセス終了→[次カット境界 cutEnd/cutStart]→遷移先開始 の順序列を返す。`TNX_HOOKS` はフック名の正本（凍結オブジェクト）。テスト RED→GREEN（5件）。
- Foundry グルー: startCombat/advancePhase は遷移前状態を退避して update 後に発火。割り込みの発火は startInterrupt（元メイン打ち切りの processEnd は `planInterruptStart.endMainId` があるときのみ＝元が main のとき・挿入メインの processStart）・endInterrupt（挿入メインの processEnd）で直接。すべて GM 側（各メソッドは GM のみ実行）。combat-events は静的 import ゼロ＝循環なし。
- 案1: `endCombat` を override（Foundry 既定の終了確認を置換）。`DialogV2.wait` の3択（callback で action 文字列を返す・close/cancel は no-op）。ボタン縦積みは `.tnx-scene-end-dialog .form-footer{flex-direction:column}`（用途ピッカーの縦積み規約に倣いスコープを絞る）。
- イベントは購読者（15）が無ければ no-op のため現状の挙動に影響しない（発火タイミング・ペイロードの正しさを先に確定する狙い）。
- テスト 1763 green・eslint 0・stylelint clean。

#### 課題・申し送り（フェーズ15 購読側）

- `tnxCutEnd`: 「カット」持続効果の失効・使用回数のカット境界リセット。
- `tnxSceneEnd`: 「シーン中」効果の失効・仮死/昏睡の死亡判定・「シーン中N回」リセット。
- `tnxProcessEnd`（phase:"main"）: メインプロセス中効果の失効（通常/挿入/打ち切り元の全メイン終了で発火）。
- 「継続」を選んだ後のシーン終了導線（シーンコントロール側）はフェーズ14（シーン管理の一般化）。13-6 は `tnxSceneEnd` を発火する口だけ用意。

### 2026-07-23 サブターンのスポット絞り込み（用途のあるプロセスだけ手番・実機確認済）

**2026-07-23** / **関連実装** `scripts/module/combat-turn-order.mjs`（`firstSpotId`/`nextSpotId` の `eligibleIds`・`PHASE_TIMING_KEY`）・`usage-consumption.mjs`（`isConsumptionDepleted`）・`scripts/combat/tnx-combat.mjs`（`_eligibleSpotIds`/`_actorHasProcessAction`・3フロー配線） / **ステータス**: 実装済・実機確認済（2026-07-23 ユーザー）

#### メカニクス監査

- **サブターンのスポットは「そのプロセスの用途を持つ者」だけに止まる**（2026-07-23 ユーザー確定・→ Combat_Flow「トラッカー上の表現」）: セットアップ／イニシアチブ／クリンナップのスポット走査を、そのプロセスのタイミング（`process:setup`/`initiative`/`clean-up`）の用途を使える形で持つ参加者に絞る。持たない者は「次へ」で自動的に飛ばす。
- **「使える」＝消費が枯渇していない**。消費設定が無い用途は常に使える（**設定なしを枯渇に含めない**＝ユーザー厳命）。`inert`（使用回数制限なし）・設定不備（対象不明）・消費量0/負は枯渇に含めない。戦闘タブ非表示（hideInCombatTab）の用途は数えない。
- **プロセスは自動通過しない**（2026-07-23 ユーザー是正）: 手番が回る者が居なくてもプロセスは入って留まり、「次へ」を押して初めて既定処理（CSカレント確定／次のメイン確認／AR 回復・次カット）が走り次へ進む。当初 Code は空プロセスを自動 advancePhase していたが「自動通過はしない」で撤去。メインは対象外（AR=0 で自前スキップ）。

#### 設計レビュー

- 純ロジック（TDD）: `firstSpotId`/`nextSpotId` に任意の `eligibleIds`（Set）を追加し、渡せばその集合に絞る（null=従来どおり全員）。`isConsumptionDepleted(rows)`＝限度資源で正の消費量に残量が足りない行が一つでもあれば true（`inert`/`problem`/kind なし/amount≤0 は除外・空は false）。テスト RED→GREEN。
- Foundry グルー: `_eligibleSpotIds(phase)` が `PHASE_TIMING_KEY` でフェーズ→timing.processName を引き、各 combatant のアクターが `_actorHasProcessAction`（timing 一致＋`resolveConsumeRowsForActor`→`isConsumptionDepleted` で枯渇でない）を満たすかで集合を組む。startCombat / advanceCut（`nextSpotId`）/ advancePhase（`firstSpotId`）の3点に配線。循環 import なし（usage-consumption は静的 import ゼロ）。
- 経緯: ユーザー当初「戦闘タブでタイミングをスキップ」と表現したが実体はトラッカーのサブターン進行（Code が戦闘タブ表示と取り違え→是正）。さらに「プロセス丸ごとスキップ」でなく「既定処理だけ走る」＝手番の配り方だけ変える、と是正。最終的に自動前進も撤去（「次へ」を押すまで進まない）。
- 待機への影響: イニシアチブ用途が無ければ毎カット、イニシアチブは既定処理だけで通過し待機宣言の機会が出ない（ユーザー了承済み。要れば「候補は常にイニシアチブで手番が回る」に変更可）。
- テスト 1758 green・eslint 0・stylelint clean。

#### 課題・申し送り

- 待機を残す要件が出たら、イニシアチブの eligible に候補を常に含める（＝待機のために止まる）を接続。
- AR 消費用途は AR 不足で枯渇＝手番なしになる。setup/cleanup タイミングで AR を消費する用途は稀だが、派生 `actionRank.inCombat` の陳腐化で誤判定しうる点は 13 系の inCombat 依存の残課題（表示側は combat.started へ移行済み・13-5）。

### 2026-07-22 フェーズ13-5 メインプロセス割り込み（挿入メイン・実機確認済）

> **〔2026-08-05 全面改訂〕本エントリの AR 挙動・戻り先・終了ボタンは下記「AR・メジャー・割り込み・支援の再設計」で置換。** 「メイン中の割り込み＝終了・戻らない」→**サスペンド／レジューム（元メインへ戻る）**・「2ボタン（AR-1して終了/終了）」→**「割り込みを終了」1ボタン＋consumesAr で AR 自動判定**・`planInterruptStart`/`planInterruptEnd`/`buildInterruptEndUpdate` 廃止→`pushInterruptFrame`/`popInterruptFrame`＋`interruptStack`。起点（用途トグルで対象に `canInterrupt`）・入口ゲート設計は有効。

**2026-07-22** / **関連実装** `scripts/module/combat-progression.mjs`（`planInterruptStart`/`planInterruptEnd`/`buildInterruptEndUpdate`）・`scripts/combat/tnx-combat.mjs`（`startInterrupt`/`endInterrupt`/getter/ガード）・`combat-tracker-view.mjs`（footer/row）・`interrupt-grant.mjs`（新規）・`tnx-socket-handler.mjs`（3ソケット）・`usage.mjs`（`grantsInterrupt`）・`usage-check-context.mjs`・`tnx-character-sheet-base.mjs`・`tnx-usage-sheet.mjs`・`tnx-combat-tracker.mjs`・`templates/{app/usage-sheet,combat/tracker}.hbs`・`css/tnx2.css` / **ステータス**: 実装済・**AR/戻り/終了は 2026-08-05 に再設計で置換**（起点設計は継続）

#### メカニクス監査

- **割り込み（挿入メイン）**（正本 Combat_Flow「割り込み（挿入メイン）」）＝順番外のメインプロセスを差し込む。二つの在ゲーム機能を一つのプリミティブに還元: イニシアチブでの割り込み（自分のメインを順番外で）・追加行動技能（AR 無消費で別キャラにメインを与える＝Aのメジャーを犠牲に）。
- **起点＝用途の「割り込みを許可」トグル**（宣言/判定）: 使用で対象（対象なし＝自分）の combatant に `canInterrupt`。トラッカー入口は**対象の操作者/RL のみ**・フェーズ非依存・行使でワンショット消費。フラグを対象に立てる設計で自己割り込みと他者付与を一本化（ユーザー考案）。
- **メイン中の割り込み＝そのメインを終了**（2026-07-22 ユーザー確定）: 通常/挿入いずれのメイン中でも、割り込み時点でそのメインを終了（AR−1・CS0）し**戻らない**。戻り先は常にサブターン（通常メイン中→イニシアチブ）。追加行動技能の「Aのメジャーを犠牲に」と一致。
- **挿入メイン終了＝2ボタン**（「終了」＝AR据え置き／「AR を−1して終了」＝AR−1）・CS は両方据え置き（通常メインの CS→0 は適用しない＝2026-07-22 ユーザー確定＝指定は AR のみ）。
- **完全自動化はしない**（ユーザー確定）: 技能組み合わせ・AR 消費の自動化を持たない。割り込みの実行はトラッカー入口の手動操作。

#### 設計レビュー

- **経緯**: 当初の 13-5「リアクション宣言タイミング統合」は 12＋確定方針で充足済みと判断しクローズ→ユーザーが「割り込みでメインを奪う」機能を要望→設計対話（起点＝トラッカー宣言 vs アイテムロール→ユーザー考案の「用途トグルで対象にフラグ→入口ゲート」に確定）→実装。付与を「結果カードの手動ボタン」と一旦説明したが、実装は**用途使用時の即時付与**を採用（カードへ新フラグを通す配線が判定/攻撃/宣言の各カード生成に広く及び risky・かつ実際の割り込みはトラッカー入口＝対象操作者/RL ゲートを押して初めて起こるため付与を早めても制御は失われない）。ユーザーはコミットで承認（明示裁定ではない＝将来手動ボタン化の余地あり）。
- **純ロジック（TDD）**: `planInterruptStart`（メイン中は endMainId を返す・戻り先を計画）・`planInterruptEnd`（退避サブターンへ・spot 再算出フラグ）・`buildInterruptEndUpdate`（AR のみ・CS 不干渉）。テストは RED→GREEN。スタック（元メインへ戻す）方式は「メイン中の割り込み＝そのメイン終了」裁定で撤去し、戻り先1つ（`interruptReturn`）へ単純化。
- **Foundry グルー**: `startInterrupt`（メイン終了記帳＋挿入フラグ＋許可消費）・`endInterrupt`（AR記帳＋復帰＋spot再算出）・advanceCut ガード（挿入メイン中は core キーバインドでも通常前進しない）。非GMは3ソケット（`interruptStart`/`interruptEnd`/`interruptGrant`）で GM へ委譲・要求者の所有権を検証。付与は `interrupt-grant.mjs`（`currentTargetActors`→combatant フラグ・GM委譲）を buildUsageCheckContext と _useDeclarationUsage の2チョークに配線（check/declaration 両対応）。循環 import 回避（socket→interrupt-grant は dynamic）。
- **表示バグ是正（同日）**: トラッカーの AR・CSカレント表示がアクター派生 `combatSpeed.inCombat` 依存で、reset 時点に依存して陳腐化し行ごとに AR が出たり消えたりしていた。描画中の `combat.started` 直接判定へ変更して解消（`getCombatantByActor` は v12 deprecated だが機能はする＝原因ではない）。
- テスト 1748 green・eslint 0・stylelint clean。

#### 課題・申し送り

- 付与タイミングは即時採用。RL/対象が「使用のたびでなく結果カードのボタンで許可を届けたい」場合は手動ボタン方式へ切替可能（カード flag の配線が必要）。
- `canInterrupt` はカット境界での自動クリアを持たない（戦闘終了・行使で消える）。カットをまたいで許可を持ち越したくない要件が出たら 13-6 のカット境界イベントで掃く。
- スポットのユーザー順タイブレーク未写像（13-4 申し送り継続）。

### 2026-07-22 フェーズ13-4 カット進行トラッカー（サブターンモデル・実機確認済）

**2026-07-22** / **関連実装** `scripts/module/tnx-combat-tracker.mjs`・`combat-tracker-view.mjs`・`combat-turn-order.mjs`・`combat-progression.mjs`・`scripts/combat/tnx-combat.mjs`・`tnx-socket-handler.mjs`・`tnx.mjs`・`conditions.mjs`・`damage-chart.mjs`・`condition-resolution.mjs`・`templates/combat/*`・`css/tnx2.css` / **ステータス**: 実装済・実機確認済（2026-07-22 ユーザー）

#### メカニクス監査

- **サブターンモデル**（Combat_Flow「トラッカー上の表現」＝正本）: メインプロセス＝キャラのターン、セットアップ/イニシアチブ/クリンナップ＝特定キャラに紐づかない全体の場（サブターン・同一カットに複数回）。イニシアチブは次のメイン行動者を**状態から確認する場**（イニシアチブで行動するキャラ＝次のメイン行動者とは限らない）。1カット＝1ラウンド。
- **スポット走査**＝プロセスごとの行動権を CS順に巡回。「次へ」で渡す（宣言者はスポットの操作者か RL）。待機は候補本人の操作者の宣言（CSカレント→1）。
- **メジャー未実行のメイン＝「メジャーで何もしなかった」扱い**（2026-07-22 裁定）＝メイン終了は常に AR−1・CSカレント0（ムーブ/マイナー無限反復の遮断）。「メジャーなしで終了」分岐は全廃。
- **行動不能＝読み取り**（Combat_Flow §4 追記）: 戦闘不能タグ（blocksMainProcess＝気絶/失神/仮死/昏睡＋terminal の完全死亡/精神崩壊）または脱落マーク。手動ボタンは脱落切替と機能が被るため棄却。確認時に上位の行動不能者へ AR−1 自動記帳。**抹殺は対象外**（applyAtActEnd＝アクト終了時適用・アクト中は行動可＝ユーザー訂正。Code は terminal 一律ブロックと誤推定→Damage_Rules 既記載の読み落とし）。
- **脱落マークの自動連動**: blocksMainProcess の付与で自動オン・除去で（該当が残らなければ）自動オフ。「dead」だけ core 特別ステータスで自動脱落になる非対称を解消。
- **動転（ment-9）の制御判定要求を自動化**: 状態直下 `controlNegate`（感情）を新設し既存機構に配線。成功＝ダメージ消滅（2026-07-09 裁定の一般則）・物を落とす処理は卓運用。

#### 設計レビュー

- **経緯（重要）**: 当初 Code は FVTT の turn/round モデル（turns・nextTurn・combatant・ライフサイクル）を読まずに離散プロセスボタン・assignMain・行動起動ボタンで自作し、ユーザーの連続指摘（プロセスのつながり無視・行動権の受け渡し不在・行動のボタン制御・待機の宣言者）を受けて全面棄却→**実ソース精読**（templates/sidebar/tabs/combat/*・combat-tracker.mjs・combat.mjs）→サブターンモデルで再設計。フェーズ設計時の「上書き可能か」への安請け合いが根本原因（監査記録として保持）。
- 実装: `TnxCombatTracker`（PARTS 3枚差し替え・既定機能温存・「カット進行」統一）／`nextTurn` 1本の前進（`advanceCut`＝スポット送り→尽きたら `advancePhase`）／core `turn` を「今の番」へ常時同期／`startCombat` override＝**シード→単一 update**（core は「フック→update」順のためフック内 update は上書き・競合＝実機で turn 固定・AR 未付与が発覚。**セットアップ入り時点で AR 付与済み**の要件もこれで満たす）／round 変更の `updateCombat` フックで全クライアント reset（core `updateCombatantActors` は render のみ）／プレイヤーの「次へ/手番終了」は `cutAdvance` ソケットで GM 委譲（要求者の所有権検証）。
- 純ロジック（TDD）: `planAdvance`・スポット走査（`firstSpotId`/`nextSpotId`・cantAct 除外）・`confirmMain`（ペナルティ算出）・`footerPlan`/`rowActions`・`blocksMainProcess`/`actorCannotMainProcess`。v13 の単一ルート要素制約・`_onRender` の `.combat-tracker` 前提も実機エラーから是正。
- 付随（同日）: 状態・負傷アイコンの整理（ユーザー主導・blood=肉体負傷専属/髑髏=完全死亡等）・damage-chart の notes 全廃（UI 説明文の規約違反＝未自動化6件はコードコメントへ）・スタイル既定アイコン door-closed（ユーザー）。
- テスト 1730 green・eslint 0 error・stylelint clean。

#### 課題・申し送り

- 13-5: リアクションの宣言タイミング・回数のトラッカー統合。13-6: 境界イベント（カット進行終了時のシーン終了確認ダイアログ）。13-7: FS 実接続。
- スポットのユーザー順タイブレーク（participantOf.userOrder）は未写像（id 順で決定的）。必要になったら接続。
- 驚愕（AR−1）・士気喪失（AR 0）の付与時発火は未接続（damage-chart のコードコメント参照）＝13 以降の接続候補。

### 2026-07-23 オプションのホスト指定統合（別大分類ホスト対応）＋準備ゲート

**2026-07-23** / **関連実装** `scripts/data/item/common/extensible.mjs`（`SLOT_KINDS.outfitName`）・`common/outfit-base.mjs`（`hostName`→`hostKey`）・`part-helpers.mjs`（`matchesHostDescriptor`／`computeHostOccupancy`／`resolvePartRowsForDisplay`）・`module/outfit-dictionary.mjs`（新規）・`item/tnx-outfit-sheet.mjs`・`actor/tnx-character-sheet-base.mjs`・`data/actor/common/character-base.mjs`・`templates/item/outfit-sheet.hbs`・`templates/actor/parts/tab-outfits.hbs`・`css/tnx2.css` / **ステータス**: 実装済・**実機未確認**（ユーザーの Foundry 確認待ち）

#### メカニクス監査

- **別大分類ホスト**（搭載兵器＝武器→ヴィークル 等）を装備先に指定可能へ。旧「装備先候補＝自身の大分類で絞る」は誤りで、正しくは**オプションが宣言したホスト記述子**（大分類/小分類/除外/特徴 or 識別キー）で絞る。IANUSオプション（general／ホスト ianus・同大分類 cyberware）とは別軸＝IANUS は Item type の相違、搭載兵器は**大分類**の相違（Outfits.md 統合節）。
- **アイテム名指定（スロットなしホストへの装備）＝便宜スロット「アイテム名」（容量1）**。「対象1つにつき1個だけ準備可能」は容量1の占有超過で自動的に効く（別の準備数制御を作らない・ユーザー確定）。名前照合の別系統を占有の一種へ統合。
- **スロット名の見せ方（2026-07-23 ユーザー指摘で是正・二段）**: 当初は種別ラベル「アイテム名」を直出し→ホスト名へ→**「スロット名という概念自体が破綻」**（分類名「地上車両」も不自然・入れ子で所属は自明）。最終決定＝**入れ子行のスロット名タグとゲージのツールチップを廃止**。親アイテム行の**容量ゲージは残す**（残容量・超過。名前不要）。占有パネルのホストスロット占有チップは従来どおり（便宜スロットはホスト名）。
- **入れ子行の列は移動先の表に合わせる**（2026-07-23 ユーザー指摘で是正）: 搭載兵器のように準備で別大分類ホスト配下へ移る行は、元カテゴリの列（攻/受/射）をそのまま持ち越さず**装備先グループの列**で計算する。
- **オプションの準備ゲート**: オプションは装備先（親）が準備済みでなければ準備できない（`isOption` 規則の具体化）。旧来は部位設定時点で準備扱いになっていた。実効フラグ `isPreparedEffective`＝保存 `isPrepared` かつ親準備済み。トグルのゲート＋連動解除（ホスト準備解除で配下オプションも解除）。
- **アイテム名は辞典参照**（自由記入廃止・ユーザー提案）: `packs/outfits`＋`works-outfits` から上位分類で絞ったプルダウン。保存＝識別キー、表示＝逆引き現在名。**旧データは考慮しない**（ユーザー確定）。

#### 設計レビュー

- **二本立て統合**: ホスト指定が①部位行記述子（ラベル＋アイテム名絞り・大分類跨ぎ可）②`parentItemId`（占有/入れ子/AE転送/`@item.parent`・自身の大分類で絞り）に分裂し二重入力だった。実バインドを `parentItemId` 一本へ寄せ、記述子は「定義上のターゲット指定」に純化。装備先候補とアイテム名辞典の絞りを共有純関数 **`matchesHostDescriptor`** に一本化（[[feedback_extract_reusable_mechanisms]]）。下流（占有・入れ子・AE転送・`@item.parent`）は `parentItemId` 参照のため無改修で整合。
- **辞典ローダ** `outfit-dictionary.mjs` は `getIndex(fields)` 方式（KI-026 孤児化回避）＋パック単位キャッシュ（skill-dictionary 踏襲）。ラベルは所持品の識別キー逆引き優先・辞典名フォールバック（`resolveItemNameByKey`。保持名キャッシュなし＝[[feedback_no_stale_caches]]）。
- **`isPreparedEffective`** はアクター `prepareDerivedData`（`_prepareOptionPreparedGate`・items 準備後）で派生＝非トグル経路でも古くならない安全網。占有計算は元々ホスト準備でゲート済みのため無改修。
- **便宜スロット `outfitName`** は純関数 `computeHostOccupancy` に容量1で内包（実スロットなしホストも登録・占有発生時のみ 1/1 表示）。`SLOT_KINDS` へ追加（ホスト slots 設定 UI は lookup 専用なので漏れ出ない）。CSS 色（琥珀）追加。
- 検算: part-helpers に `matchesHostDescriptor`・outfitName 占有・hostKey ラベル注入のテスト追加。全 1773 test green・eslint 0 error・stylelint clean。**UI レンダリングは実機未確認**（辞典プルダウン・占有ゲージ・準備トグル連動）。

#### 課題・申し送り

- 実機確認（ユーザー）で見る点: アイテム名辞典プルダウンが上位分類で絞れるか／搭載兵器をヴィークルに装備でき、ヴィークル表の列（攻等）で表示されるか／スロットなしホストの便宜スロットが**ホスト名**でゲージ・入れ子表示されるか／準備トグルの無効化・連動解除。
- コンバイン合成の部位ラベルは hostKey 解決子を渡していない（コンバイン元がオプションのケースは稀・種別ラベルへ縮退）。必要になれば接続。

### 2026-07-24 RL 任意ダメージのダメージ算出前フェーズ＋固定/カード算出の選択

**2026-07-24** / **関連実装** `scripts/module/rl-grant-logic.mjs`（`buildRlDamageStagingFlag`／`RL_DAMAGE_MODES`）・`module/rl-grant.mjs`・`module/damage-flow.mjs`（`openRlFixedDamage`／`openDamageRollDialog` 固定分岐）・`module/attack-flow.mjs`（`renderAttackCard` の RL 分岐）・`module/request-presets.mjs`（プリセット mode）・`journal/tnx-scenario-sheet.mjs`・`templates/chat/rl-damage-card.hbs`（新規）・`templates/app/rl-grant-damage.hbs`・`templates/journal/scenario-sheet.hbs` / **ステータス**: 実装済・**実機確認済**（隔離 Foundry＋Playwright）

#### メカニクス監査

- RL 任意ダメージが「ダメージ算出の直前」効果（カバー等）を挟めなかったのを是正。当初（2026-07-20）はダメージカードを直接投稿していたため、カバーの起動口（攻撃カード限定＝命中確定後・ダメージカードを出す前）に相当する中間の器が無かった。**命中確定・ダメージ算出前の中間カード**を一度挟む形へ（確定で命中はするがダメージは未算出）。
- **決め方は2択**（`rlGrant.mode`）: 固定ダメージ（指定値を直接算出）／カードを出す通常算出（攻撃力＝RL 設定の基準値＋カードの数字）。基準値を持つのは物理のみ（精神・社会は攻撃力の概念が無くカードのみ）。X 種別・軽減可否 RL 判断・軽減分岐なしの 2026-07-20 原則は不変。
- カバー成立（`coveredBy`）後の被弾付け替えは両モードとも既存の `buildDamageTargets` で展開＝カバーした側が元々対象なら2つのダメージが別々に入る挙動（Damage_Rules）も踏襲。
- プリセットは決め方（mode）も保存・再現（カード系ギミックを丸ごと保存できるように。当初 mode を外したのは誤り＝カード系が保存不能になる取りこぼしだった。既存プリセットは無印＝固定で後方互換）。

#### 設計レビュー

- **攻撃カードの器を流用**（形でなく機能での流用・[[feedback_function_over_form_reuse]]）: カバーの起動（対象行クリック→カバー技能判定→`coveredBy`）・被弾付け替え・「ダメージカードを出す」導線は攻撃カード（`attackCheck`）上に実装済み。RL 中間カードは `attackCheck` フラグ＋`rlGrant` マーカーで、判定表示を持たない専用テンプレート（`rl-damage-card.hbs`・`.tnx-attack-status` のみ）に `renderAttackCard` のライブ描画を委ねる。カバー用の別カード型を新設して再実装する案は退けた。
- 対象は全員 `state:"hit"`・`confrontation:[]`＝リアクション導線なし。非カバーの対象行クリックは `handleTargetRowReactionClick` が `confrontation` 空で即 return＝無操作（安全）。`renderAttackCard` の RL 分岐は最小（`isRl` で命中行の制御値表記を抑止・固定モードのボタン文言を「ダメージを算出」に）。
- カードモードは `openDamageRollDialog` にそのまま合流（攻撃者=RL・攻撃力=基準値・命中確定済みの通常攻撃と同型）。固定モードだけ冒頭で `openRlFixedDamage` へ分岐し、`buildRlDamageRollFlag`＋`buildDamageTargets` で固定値のダメージカードを直接生成する。
- 値ラベルの寸法是正（[[feedback_self_review_ui_output]]）: 付与ダイアログ/プリセット編集の `.rl-form-group` は 90px 固定ラベル列で「基準値（攻撃力相当）」が溢れる（実機で scrollWidth 92px＞90px を検出）。共有列を広げると他ダイアログ/行がずれるため、入力面は「基準値」に短縮（列内に収まる・mode プルダウンが文脈を与える）。中間カードの内訳（roomy な `.cr-calc-label`）は「基準値（攻撃力相当）」のまま明示。
- 検算: 純ロジック（`buildRlDamageStagingFlag`／プリセット mode）を TDD で追加。全 1787 test green・eslint 0 error・stylelint clean。**隔離 Foundry＋Playwright で実機確認済**——ダイアログ同期（モード/ラベル/可視）、中間カード（中立表記・命中行に制御値なし・カバー可）、固定算出の台帳、カードモードのロールダイアログ合流（攻撃力＝基準値）、`coveredBy` の被弾付け替え（2エントリ）、プリセット編集のモード欄。

#### 課題・申し送り

- カバーの**判定そのもの**（技能ロール→カード→スート判定→`coveredBy`）は攻撃カードと共有の未改変機構で、中間カードは前提（`attackCheck`＋`state:"hit"`＋`!damageRolled`＋対象行の起動口）を全て満たす。実機は起動口の存在（coverable クラス／クリック配線）と `coveredBy` の付け替えまでを確認し、カード操作を伴う判定本体は未改変につき机上で整合確認（テスト世界に山札/カバー技能の整備が無いため）。
- FS判定の敗北時処理からの RL ダメージ発火は将来（`buildRl*` の呼び出し元は現状シーンコントロールの「ダメージ付与」のみ）。

### 2026-08-05 AR・メジャー・割り込み・支援の再設計（フェーズ13・実機確認待ち）

**2026-08-05** / **関連実装** `combat-progression.mjs`（`buildArDecrementUpdate`/`pushInterruptFrame`/`popInterruptFrame`・旧 `buildEndMainUpdate`/`buildCantActUpdate`/`planInterruptStart`/`planInterruptEnd`/`buildInterruptEndUpdate` 廃止）・`combat-turn-order.mjs`（`isMajorActionTiming`）・`tnx-combat.mjs`（`markMajorAction`/`_addMajorActed`/`majorActed`/`interruptStack`/`interruptConsumesAr`/`startInterrupt`/`endInterrupt`/`advancePhase`）・`tnx-check-flow.mjs`（`_execute` メジャー記帳＋`focusSupportTargetId`）・`usage-check-context.mjs`（`usageTiming`）・`usage.mjs`（`interruptConsumesAr`）・`tnx-usage-sheet.mjs`＋`usage-sheet.hbs`（トグル）・`interrupt-grant.mjs`（consumesAr 伝搬）・`tnx-socket-handler.mjs`（`markMajor`・`_onInterruptEnd`）・`combat-tracker-view.mjs`／`tnx-combat-tracker.mjs`（終了1ボタン）・`focus-system-result.mjs`（`autoApplyFocusSupport`＝支援 AE 付与／`applyFocusProgress`＝支援 AE 合算・消費）・`tnx-rl-request-app.mjs`（`onDoCheck`＝支援対象をレティクル／無指定は対象選択ダイアログ）・`attributes.mjs`（`focus.progressBonus` 実フィールド） / **ステータス**: 実装済・**実機確認済（2026-08-06・v0.13.0）**

#### ルール解釈（正本 Combat_Flow「AR・メジャーアクション・プロセス所有の一般則」2026-07-26／Focus_System「支援判定の細則」）

- **AR はトップレベルのプロセス所有者が、プロセス終了時に、メジャーを行っていれば −1（同時に CSカレント→0・カット終了まで）。** 空メジャー廃止＝実際にメジャーを行った者だけ（2026-07-22「常に AR−1」撤回）。AR 減少 ⟺ CS0（行動不能の AR−1 にも適用・待機は別）。AR≥2 は CS0 の手番で再行動（confirmMain 既定）。
- **サブターン（setup/initiative/cleanup）は無所有**。そこで行ったメジャー（例=支援判定）は本人がそのサブターン終了時に払う。
- **割り込み＝サスペンド／レジューム**（旧「メイン中の割り込み＝終了・戻らない」撤回）。元プロセス（メイン/サブターン）は終了せず退避し、挿入メイン終了で厳密に復元。**consumesAr**（用途宣言・既定=真）で挿入メインが AR を消費するか決める：自己割り込み=有償・追加行動技能=無償（付与者が自分のメジャーとして肩代わり）。所有モデルは設計思想の正本として残し、実装は用途側 consumesAr で表現（自己/他者を自動導出しない＝将来の神業例外も用途で明示）。
- **支援判定**＝ムーブ／マイナーを伴わないメジャー・指定技能はアクション技能。**支援者が判定時に普通にターゲット（レティクル）で対象1体を指定**（2026-08-05 ユーザー確定）。成功で**対象へ支援 ActiveEffect（進行 +1・duration＝カット終了）を付与**し、対象の次の進行判定で合算・消費（AE 除去）。AR は一般則でイニシアチブ終了時に引く。専用 `pendingSupport` カウンタは廃止（基本機能＝AE で表現）。AE は対象に乗る＝FS 非スコープ。

#### 監査結果（✅）

- 一般則の三本柱（メジャー記帳／AR−1∧CS0 一本化／サスペンド・レジューム＋consumesAr）を正本の文言どおり符号化。メジャー判定は用途タイミング（`action`×`major`／`initiativeMajor`）。支援はタイミングに依らずメジャーなので FS 側で記帳（`markMajorAction` は冪等＝`_execute` の記帳と二重にならない）。
- 純ロジックは TDD（`isMajorActionTiming` 5＋`buildArDecrementUpdate` 統合＋`pushInterruptFrame`/`popInterruptFrame` 8）。テスト 1795 green・eslint 0 error。

#### 課題・申し送り（実機確認の要点）

- **実機未確認**（A+B/C/D 一括）: ①メジャー用途のカードプレイ→プロセス終了で AR−1∧CS0（空メジャーは無charge）②割り込み開始→退避→挿入メインでメジャー→「割り込みを終了」で consumesAr に応じ AR 処理→元位置へ復帰（メイン再開/サブターン再開）③追加行動（consumesAr=偽）＝無償で付与者が肩代わり④入れ子割り込み⑤支援＝対象をターゲット→成功で対象へ支援 AE 付与→その対象の進行判定で AE 合算・消費。
- 支援は普通にターゲット（レティクル）で対象1体を指定（0/複数は中止）。支援ボーナスは対象キャラに乗る AE で、**中身のある本物の change**＝`system.focus.progressBonus += 1`（実フィールド `AttributesTemplate.focus.progressBonus` を追加・素値0・ネイティブ適用が着地）。空の見えない効果（フラグだけ）は禁止（2026-08-05 ユーザー指摘で是正）。カスタム適用パスはアクター上の素の `system.*` を self 分岐で空返し＝二重加算なし。duration＝カット終了で**自動失効はフェーズ15**（今は進行判定での消費＝AE 除去が主経路）。アイコンは未使用の `icons/svg/regen.svg`。自己割り込みの起点（用途トグル）は 13-5 のまま。
- 神業の割り込み特別措置は将来（consumesAr の用途宣言で例外を明示できる素地は用意済み）。
### 2026-08-08 フェーズ14-2 セッション進行の実行状態・境界イベント・台本スキーマ拡張（実機確認済 2026-08-27）

**2026-08-08** / **関連実装** `scripts/module/session-logic.mjs`（新規・純ロジック）・`session-state.mjs`（新規・ワールド設定 `sessionState`）・`appearance-state.mjs`（新規・登場フラグ）・`user-flag-schema.mjs`（`isScenePlayer`）・`combat-events.mjs`（TNX_HOOKS 追加）・`tnx-combat.mjs`（案1委譲）・`tnx-action-handler.mjs`（切り札→シーンカード）・`tnx-socket-handler.mjs`（`sessionSceneCard`）・`tnx-scenario-sheet.mjs`＋`scenario-sheet.hbs`＋`tnx2.css`（台本の行編集）・`tnx-character-sheet-base.mjs`（CS初期化の共用化）・`tnx.mjs`（設定登録）／本体 `53cacd8`（CLAUDE.md 改訂＝ユーザー編集を同梱） / **ステータス**: 実装済・**実機確認済（2026-08-27 の一括確認）**。当初は 14-3（シーン進行パネル）合流後の予定（起動形が確定してから一括＝2026-07-25 方針。台本の行編集 UI のみ 14-2 分として確認可能）

#### メカニクス監査

- **アクト中に「シーン外」は存在しない**（2026-08-08 ユーザー確定）: シーンが終了したら（ミドルフェイズ中は舞台裏を挟んで）すぐ次のシーンが始まる。切替＝現行終了→次開始の一括（`switchScene`）。アクト開始（`startAct`）も先頭シーン（または指定行）の開始まで一括で行う。
- **アクト開始を踏んだ瞬間の自動設定**（2026-08-07 裁定）: 報酬点＝`bountyBase`←外界点実効値・`bounty`←0（ポストアクトの清算を兼ねる）、CS＝シートの「プレアクト初期化」と同一計算。**対象はハンドアウトの `actorId` に設定されたキャストのみ**（2026-08-08 裁定）。未設定ハンドアウトは件数を通知してスキップ。プレアクトのその他の処理は自動化しない。
- **登場状態＝Actor のフラグ・シーンプレイヤーか否か＝User flag**（2026-08-07 裁定）: 登場するのはキャラクター（ゲスト等も同じ器・RL は判定なしで直接切替）。シーン開始でシーンプレイヤーのキャラクターが判定なしで登場・シーンの終了処理で全員退場＋シーンプレイヤー解除（計画の確認ポイント1・2＝承認済み）。対象化ゲート（登場中のみ対象化可）の実装は 14-5。
- **切り札のシーン消費化＝そのシーンのシーンカードになる**: カードの移動挙動は不変で、`sessionState.sceneCardId` に「現在のシーンカード」参照を記録するだけ（2026-08-07 裁定＝特殊効果システムは作らない・コード参照点のみ）。**追補（2026-08-08 ユーザー指摘・`6d41d63`）**: 当初は切り札経路のみ配線し、既存の提示経路 `drawNeuroCard`（HUD のシーンカードエリア→ニューロデッキのドロー）を「14-3 で新設するパネル操作」と誤認して先送りしていた＝シーンカードが生まれる2経路のうち片方だけ記録される不整合。記録関数を `recordCurrentSceneCard` に汎用化し**ドロー提示・切り札消費の両経路**から記録するよう是正。
- **13 案1「シーンも終了」＝終了境界の先行発火**: シーンは現行のまま `sceneEnded` を立て、次の切替で `tnxSceneEnd` を二重発火しない（「シーン中」効果の二重失効防止）。

#### 設計レビュー

- 実行状態の正本＝ワールド設定 `sessionState` 一箇所（FS判定の `activeFocusSystems` と同方式・onChange で 14-3 のパネルを再描画）。書き込みは GM ゲート。プレイヤークライアントで起きる切り札経路のみ `sessionSceneCard` ソケットで activeGM に委譲。
- 純ロジック TDD（RED→GREEN・37件）: `normalizeSceneRow`/`normalizeHandoutRow`（**読み出し時正規化・一括書き換えなし**）・`parseStageRef`（`scene:`/`subScene:` 複合値＝`usage.effects` の複合セレクト前例踏襲）・`findSceneRow`/`firstSceneRow`・`buildPreActInit`/`buildCombatSpeedInit`（シートボタンと共用化＝計算の一本化）・`planSceneSwitchEvents`/`planActEndEvents`・チーム操作（1アクター1チーム・非破壊）。
- `TNX_HOOKS` に `tnxActStart`/`tnxActEnd`/`tnxSceneStart` を追加・`tnxSceneEnd` の payload に `sceneId`。**シーン/アクト系フックは data 単独引数**（combat 引数なし）＝案1も `endSceneFromCombat` 委譲で同形（アクト外＝シーン外戦闘は `sceneId:""` の素発火にフォールバック）。購読者（15）不在のため現状の挙動に影響なし。
- 台本編集（アクトシート＝編集専用の領分）: シーン行にエリア・舞台・シーンプレイヤー（User セレクト）、ハンドアウトにキャストセレクト。旧 `player`（キャスト名文字列）は表示フォールバックへ降格（stale 名を正としない）。シーン表グリッドを8列化（`tnx2.css`・スコープ内変更）。ウィザードは正規化で吸収（未改修・廃止は 14-7）。
- 検証: テスト 1827 green（+64）・eslint 0 error（警告15件は既存）・stylelint clean。

#### 課題・申し送り

- 実機確認は 14-3 合流後に一括（シーン進行の起動形＝パネルがまだ無い）。
- 舞台リンクの適用（背景差し替え／通常 Scene アクティブ化）とサブシーン実体＝14-4。対象化ゲート・チーム免除 UI・登場判定＝14-5。切替メッセージ送信・シーンカードドローのパネル操作＝14-3。
- `sessionState.teams` の宣言操作はプレイヤー起点の導線が要るなら 14-5 でソケット委譲を検討（現状 GM ゲートのみ）。

### 2026-08-08 フェーズ14-3 シナリオコントロールパネル（実機確認済 2026-08-27）

**2026-08-08** / **関連実装** `scripts/module/tnx-scenario-panel.mjs`＋`templates/app/scenario-panel.hbs`（新規）・`session-state.mjs`（loadAct/actStarted・startAct 再編・listActJournals）・`session-logic.mjs`（チャットビルダー4種）・`tnx.mjs`（全員可視ボタン・updateJournalEntry フック）・`tnx-scenario-sheet.mjs`＋`scenario-sheet.hbs`＋`tnx2.css`（旧「切替」オミット・送信系のビルダー共用化・パネル CSS）／本体 `f525f35` / **ステータス**: 実装済・**実機確認済（2026-08-27 の一括確認）**

#### メカニクス監査

- **名称＝シナリオコントロールパネル**（2026-08-08 ユーザー裁定）: シーンだけでなくシナリオ（アクト）全体の操作盤。トレーラー・ハンドアウトの配布もここから行う。
- **アクトの読み込みと開始の分離**（2026-08-08 ユーザー裁定「アクトシートからの読み込みはアクト開始とは関係なく行える」）: 読み込み＝参照セットのみ（自動設定・イベント・シーン開始なし）。プレアクトの配布（トレーラー/ハンドアウト/テキスト送信・情報公開）は読み込み状態で使用可。開始＝自動設定（報酬点/CS・ハンドアウト actorId のキャストのみ）→ `tnxActStart` → 先頭シーンへ（シーン外なしの不変条件は開始後に適用）。
- **シーン切替演出**: 見出しチャット（旧「切替」ボタンの書式踏襲・シーンプレイヤーはライブ解決）→ **ニューロデッキの自動ドロー**（2026-08-07 裁定「シーンカードは自動でニューロデッキのドローを起動するだけ」＝毎切替1枚・マスターシーン含む）。「現在のシーンカード」はドロー側（`recordCurrentSceneCard`）が記録。
- **旧「切替」ボタンのオミット**（2026-08-08 承認）: journal `currentState` フラグを更新する旧経路と sessionState の**状態二重化を解消**。他の送信系ボタンは無害な重複として 14-7 まで残置。
- 情報公開（isPublic）・内容の開示（isDisclosed）のトグルはパネルに配置（正本は従来どおりアクトシートの infoItems フラグ・シートの編集欄は据え置き）。

#### 設計レビュー

- パネル＝ApplicationV2（id `tnx-scenario-panel`・幅420・全員が開ける・操作は RL のみ描画）。再描画は sessionState onChange＋`updateJournalEntry`（アクティブなアクトのみ）。シーンコントロールバーのボタンは FS判定と並ぶ全員可視ツールとして一括登録に整理。
- アクトシートの列挙＝`flags.core.sheetClass === "tokyo-nova.TnxScenarioSheet"`（作成ボタンが焼き込む値と一致）。
- チャット内容の組み立てを純ロジック `buildSceneSwitchMessage`/`buildTrailerMessage`/`buildHandoutMessage`/`buildInfoMessage` に切り出し（TDD 10件・既存書式を等値で移植）、アクトシートの送信系ハンドラも同ビルダーへ差し替え＝一本化（シート側ボタンの撤去は 14-7）。
- `startAct` は読み込み済みアクト前提の無引数形へ再編（14-2 の journal 引数形を置換・呼び元はパネルのみ）。`endAct` 後は読み込み状態（未開始）へ戻る。
- 検証: テスト 1837 green（+10）・eslint 0 error（警告15は既存）・stylelint clean。

#### 課題・申し送り

- **実機確認（14-2/14-3 一括・ユーザー）**: 台本の行編集（エリア/舞台/シーンプレイヤー/ハンドアウトのキャスト）→ パネルで読み込み→配布→開始（自動設定・先頭シーン・切替チャット・シーンカードドロー）→ 切替 → 案1（戦闘終了「シーンも終了」→次の切替で二重発火しない）→ アクト終了。
- 舞台リンクの適用（背景差し替え/Scene アクティブ化）＝14-4 で `switchScene` 後段に接続。登場判定ボタン・チーム UI＝14-5、舞台裏スポット＝14-6 で本パネルに載る。
- パネルの情報トグルとシート情報タブの編集欄の重複整理・送信系ボタンの撤去＝14-7。

### 2026-08-08 フェーズ14-4 サブシーン（ワールド独立の盤面状態・実機確認済 2026-08-27）

**2026-08-08** / **関連実装** `scripts/module/subscenes.mjs`＋`tnx-subscene-panel.mjs`＋`templates/app/subscene-panel.hbs`（新規）・`list-order.mjs`（新規・汎用並び替え純ロジック）・`tnx-scenario-panel.mjs`（切替演出へ `applyStageRef` 接続）・`tnx-scenario-sheet.mjs`＋`scenario-sheet.hbs`（舞台セレクトにサブシーン optgroup）・`tnx.mjs`（設定登録・RL ボタン・updateScene フック）・`tnx2.css`／本体 `ed7b755` / **ステータス**: 実装済・**実機確認は後続サブフェーズ合流後に一括**（2026-08-08 ユーザー方針「実際にアクトシートが読み込める段階になるまでは正確な実機確認ができません」）

#### メカニクス監査

- **サブシーン＝Scene に属さないワールド独立のリスト**（2026-08-08 ユーザー裁定「シーンに属さない運用が可能なら独立させてほしい」）: 中身は名前＋背景画像のみで Scene 依存の情報がなく、適用先を「その時アクティブな盤面 Scene」とすることで所有関係を不要にした。当初案（盤面 Scene のフラグ保存）は撤回。正本＝ワールド設定 `subScenes` 一箇所（GM のみ書き込み）。
- **適用＝アクティブ Scene の背景（`background.src`）差し替え**。トークンは常駐のまま（確定方針）。「現在どのサブシーンか」は背景一致で導出＝状態を持たない（正本の二重化回避）。**→是正（同日・`1dc02d1`・ユーザー指摘「サブシーン切替時もシーンの読み込みが走ってしまう」）**: `background.src` の update はキャンバス完全再描画＝シーン読み込みを誘発しサブシーンの意味を失うため、**ドキュメントは書き換えない方式**へ変更——適用＝アクティブ Scene のフラグ `subSceneOverride` への記録のみ（フラグ更新は再描画を誘発しない）・表示＝各クライアントが背景メッシュのテクスチャを直接差し替え（`foundry.canvas.loadTexture`→`canvas.primary.background.texture`・canvasReady/updateScene/設定 onChange で再適用＝永続・全員同期）・「適用中」＝フラグ判定・「元の背景に戻す」ボタン追加。運用前提＝盤面 Scene に初期背景を1枚設定（背景なしは背景メッシュ自体が無い・GM に警告）。動画背景は対象外（静止画想定）。
- **舞台リンクの適用（設計判断8の実装）**: TNX シーン切替と同時に `applyStageRef` が走る——`subScene:<id>`→背景差し替え／`scene:<id>`→通常 Scene のアクティブ化／参照なし→現状維持。**通常 Scene 切替時のトークン自動同期はしない**（2026-08-08 Code 推奨をユーザー黙示了承＝登場状態の権威は Actor フラグでトークンはその反映・自動生成は配置が決められず過剰・RL のトークン管理に委ねる）。
- 並び替え＝**グリップ DnD＋上下ボタンの併設**（2026-08-08 ユーザー指示「ドラッグでも可能に」・TNX 標準の適用・順序はワールド設定の配列順で永続化）。

#### 設計レビュー

- 汎用並び替え純ロジック `list-order.mjs`（`moveItemBy`/`moveItemTo`・TDD 7件・非破壊・クランプ）＝リスト系 UI の共通部品として新設（DnD 挿入意味論は FS判定エディタと同一）。
- パネル＝ApplicationV2（id `tnx-subscene-panel`・RL 専用ボタンから起動）。グリップのみ draggable（行全体にすると名前入力の選択を奪う＝FS エディタの知見踏襲）。背景変更は `foundry.applications.apps.FilePicker.implementation`（v13 API リファレンスでクラスパス確認済み）。
- 再描画: 設定 onChange＋`updateScene` フック（適用中ハイライトが背景・アクティブ切替に追随）。
- 検証: テスト 1844 green（+7）・eslint 0 error（警告15は既存）・stylelint clean。舞台セレクトの selected 判定は Handlebars ブロックパラメータの字句参照（`scene.stage`）に統一（`#if` を挟む深さでの `../` 解決の不安定を回避）。

#### 課題・申し送り

- 実機確認（14-2〜14-4 一括・後続合流後）: サブシーンの保存/適用/並び替え・舞台リンク（サブシーン/通常 Scene）の切替追随・FilePicker。
- 登場/退場の盤面反映（アクティブ盤面上のトークン表示操作）＝14-5。

### 2026-08-08 フェーズ14-5 登場判定＋チーム（対象化ゲートは撤回・実機確認済 2026-08-27）

**2026-08-08** / **関連実装** `scripts/module/appearance-logic.mjs`＋`appearance-check.mjs`（新規）・`appearance-state.mjs`（トークン双方向同期）・`session-logic.mjs`（`teamHasAppearing`）・`session-state.mjs`（チーム加入時自動登場・`appearTeam`/`exitTeam`）・`tnx-socket-handler.mjs`（`sessionTeam`）・`tnx-check-flow.mjs`（完了継続 `appearance`）・`tnx-character-sheet-base.mjs`（`extraCheckBonuses`）・`tnx-formula.mjs`（`label` 行）・`tnx-scenario-panel.mjs`＋テンプレート＋CSS／本体 `a3b8f29` / **ステータス**: 実装済・**実機確認済（2026-08-27 の一括確認）**

#### メカニクス監査

- **登場判定**（正本 Appearance_Check）: パネルのボタン→技能選択（既定候補=社会/コネ分類。**他の技能も選択可＝可否の裁定は卓・システムは制限しない**）＋ゴースト登場チェック（2026-08-07 裁定）→判定。TN＝エリア→固定値の自動供給（レッド8/イエロー10/グリーン10/ホワイト12/サンクチュアリ12・手入力なし）。**危険値は達成値に加算**（2026-08-07 裁定・グリーン×1/ホワイト×2・台帳に「危険値（エリア名）」行）。**サンクチュアリの登場不可は合計でなく個別装備**（危険値が0未満の装備の携帯＝＋2と−1の合計+1でも不可）。成功→登場状態付与（ゴースト時 isGhost も）。失敗→登場しないだけ（再試行自由＝手札入れ替えとしての登場判定・回数制限なし）。エリア未設定シーンは目標値なしで判定可（成否が出ないため自動登場せず卓裁定）。
- **対象化ゲートは作らない**（2026-08-08 ユーザー裁定「対象可ゲートは『制限』です。作らないでください。非登場のキャラクターを対象可能な特技が存在します」）: 14-1 記録の設計判断2「ゲートは target-resolution へ一点実装」は**撤回**。「登場しているキャラクターしか対象にできない」はルールの説明であって実装指示ではなく、**システムは可否を制限しない**原則（技能組み合わせと同じ）が対象選択にも適用される。登場状態は表示・参照のための状態。
- **チーム**: 宣言はいつでも可＝結成は誰でも・参加/離脱は自分のキャラクター（PL は `sessionTeam` ソケットで activeGM 委譲・所有権検証）。チーム免除＝「チームで登場」（登場中メンバーがいるときのみ・残りを判定なしで一括登場）「チームで退場」（退場も同時）。**登場中メンバーのいるチームへ後から加入したキャラクターは自動で登場する**（2026-08-08 ユーザー指示・シーン進行中のみ）。
- **盤面反映＝トークン表示との双方向同期**: 登場状態（Actor フラグ・権威）⇄ アクティブ盤面のリンクトークン hidden。トークンの表示切替が RL の「直接切替」導線そのもの。非リンク（分身コピー等）は対象外・Scene 跨ぎの自動生成なし（確定方針）。

#### 設計レビュー

- 純ロジック TDD: `appearanceCheckParams`（エリア表・−0 正規化）/`hasNegativeDangerOutfit`（携帯条件は appearanceModifier 集計と同じ isCarrying）/`isAppearanceSkillKey`（society/contact プレフィックス）＋`teamHasAppearing`＋`evaluateBonusRows` の `label` 行（識別キー逆引きを通さない固定ラベル＝従来 source は逆引き専用で "用途" に落ちるため追加）。
- 起動は唯一の起動関数 `_activateItemCheck` へ集約（新しい起動関数なし）。**`extraCheckBonuses`**＝入口が供給する判定ボーナス行を共通前段の checkBonuses へ**追記**する汎用口（上書きでない）。完了継続 `appearance` は CONTINUATIONS 登録＋`_execute` 分岐（rerunOnSuccessOnly・setAppearing は冪等＝再判定の再実行に安全）。
- 同期は activeGM 代行（hidden は GM 専権）・同値短絡でループ防止。パネル再描画は appearing フラグの updateActor 購読。
- 検証: テスト 1854 green（+10）・eslint 0 error（警告15は既存）・stylelint clean。

#### 課題・申し送り

- 実機確認（14-2〜14-5 一括・後続合流後）: 登場判定（TN/危険値/サンクチュアリ/ゴースト）・チーム（PL 委譲・自動登場・一括登場/退場）・トークン双方向同期・シーン終了の全員退場。
- 手札上限のゲスト合算の「登場中のみ」絞り込み（Card_Operations・任意項目）はフェーズ14 では見送り（YAGNI・登場状態 API は揃ったため必要になれば差し替え可能）。
- 舞台裏スポット巡回＝14-6（非登場の列挙は appearance-state からライブ導出）。

#### 是正（2026-08-08・`b904123`・ユーザー指摘4件）

- **チーム編成の欠落（重大）**: 当初実装は PL の自己参加のみを設計し、**RL（担当キャラなし）の到達経路を検証していなかった**——RL 視点では「結成→空のチーム→解散」しかできない「作っただけ」の機能だった（「意図のない実装をしないでください」）。RL の編成操作＝メンバー追加（チーム行のセレクト＋追加。**候補はキャスト＋ゲスト**＝2026-08-08 裁定「チームにはゲストを入れることもできます」・`19a5ef2`）・除去（メンバーチップの×）・チーム名インライン編集（`renameTeam` 新設）を追加。教訓＝**新機能は主要な操作者ごと（RL/PL）に到達経路を先頭から歩いて検証してから完了と言う**（reachability 規範の再違反）。
- 登場判定ダイアログのゴーストチェックボックスの崩れ: label 内包の旧構造→ v13 標準の form-group/form-fields 構造へ。
- パネルの見切れ（スクロールバーなし）: シナリオコントロール/サブシーン両パネルに max-height（100vh−140px）＋ window-content の overflow-y を追加。
- 現在のシーンカード名に生 HTML: ニューロカード名はワールドデータ由来で HTML を含みうる——タグを剥がした素の文字列だけを表示。**→再是正（`3d1cdd5`・ユーザー指摘「タグを無視して表示しているだけ」）**: 表示すべきはカード名だけでなく**キーワード（faces[0].text）・暗示（description）のリッチテキスト**であり、チャットカードと同じく `enrichHTML` して `{{{ }}}` で **HTML として描画**する（剥がすのでなく適用する）。
- パネル最大高が長すぎ（`3d1cdd5`）: `100vh−140px`→`min(560px, 100vh−200px)`。
- **「次のシーンへ」（`3d1cdd5`・ユーザー指示「基本的には『次のシーンへ』で送れたほうが良い」）**: 台本順（フェイズ順→行順・空フェイズは飛ばす）の次の行へ送る基本操作を追加（純ロジック `nextSceneRow`・TDD 5件）。シーン一覧の各行ボタンは直接ジャンプ用として併存。

### 2026-08-08 フェーズ14-6 舞台裏スポット巡回（シーン送りの順序制約・実機確認済 2026-08-27）

**2026-08-08** / **関連実装** `scripts/module/session-logic.mjs`（`hasBackstage`/`backstageQueue`/`nextBackstageSpot`/`isBackstageFinished`）・`session-state.mjs`（backstage 状態・操作・`canAdvanceScene`）・`tnx-scenario-panel.mjs`＋`scenario-panel.hbs`＋`tnx2.css`／本体 `93d0506` / **ステータス**: 実装済・**実機確認済（2026-08-27 の一括確認）**

#### メカニクス監査

- **要点はシーン送りの順序制約**（2026-08-08 ユーザー指示「次のシーンへ進めるボタンを舞台裏を回しきるまで表示させるな」）: リサーチシーンは「シーンを閉じる」→ 舞台裏 → 回しきって初めて「次のシーンへ」が出る。**舞台裏があるのはリサーチシーンのみ**（同日ユーザー確定）。他フェイズは制約なし。
- **舞台裏でやることは「回す」だけ**（同日ユーザー指示）: 舞台裏で何もしないことはできるので、システムは起動の機会を順に提示するのみ。判定するか否かに介入しない・回数制限も自動化しない（Time_Management 確定と一致）。
- 列＝非登場者のライブ導出（登場すれば列から消える）＋RL の手動追加（登場中・ゲストも可＝「登場しつつ舞台裏でも判定する」スタイル技能の表現）。回す相手が誰もいなければ、閉じた時点で回しきり扱い（進行が詰まらない）。
- 舞台裏の状態はシーン単位（`_applySceneEntry` でリセット）。

#### 設計レビュー

- 純ロジック TDD（15件）: `hasBackstage`（フェイズ判定）・`backstageQueue`（非登場＋手動追加・名前順・重複しない）・`nextBackstageSpot`（先頭から開始／末尾で null＝回しきり／列から外れたスポットは先頭へ戻す頑健性）・`isBackstageFinished`（未オープン=false／巡回中=false／`started` かつスポット解除=true／空列=true）。
- `sessionState.backstage{open,started,spotActorId,extraActorIds}`。`canAdvanceScene()` がパネルの「次のシーンへ」表示を一点で制御（舞台裏を持たないシーンは常に true）。
- **13 のスポット機構は共有しない**: `nextSpotId` は CS 順ソート（`spotOrder`/`compareTurnOrder`）・AR・行動不能の絞り込みと不可分で、舞台裏と共通するのは「配列の次の要素」1行のみ。抽象化すれば形だけの共有になり、13 側に舞台裏の都合を持ち込むリスクがある（§4.3）。
- 検証: テスト 1874 green（+15）・eslint 0 error（警告15は既存）・stylelint clean・テンプレートのブロック対応を機械検証。

#### 課題・申し送り

- 実機確認（14-2〜14-6 一括）: リサーチシーンで「シーンを閉じる」→ 回す → 回しきり → 「次のシーンへ」の順序。他フェイズで制約が出ないこと。
- 情報収集判定（16）・購入判定（17）はこの舞台裏の巡回に刺さるインスタンスとして実装する。

#### 経緯（Code の逸脱・2026-08-08）

ユーザーの指示は一貫して**シーン送りの順序制約**だったが、Code は「舞台裏を回すだけでよい」を自分の仕様案（済み記帳・自動記帳・手動トグル）への差分と読み、**言及されていない機構を勝手に取り下げる**宣言をした。さらにユーザーの明示的な停止指示（「答えるまで絶対に進めないでください」）の後、自分の説明で許可が下りたと判断して実装を再開し、作業ツリーの巻き戻しに至った。教訓は既存メモリ（指示は自分の設計への最小差分として読まない・解釈がユーザーの述べた理由を解消するか検算する／停止指示は明示の再開指示まで有効）の再確認。

### 2026-08-08 フェーズ14-7 アクトシートの実行系全削除＋パネル側の追加（実機確認済 2026-08-27）

**2026-08-08** / **関連実装** 削除段階=`b3ce42a`（アクトシートの設定タブ・ウィザード・チャット/アクター/シーンへの働きかけを機能ごと削除・596行削減）／追加段階=`7a483d6`（`general-skill.mjs`・`tnx-item-sheet.mjs`・`session-state.mjs`・`session-logic.mjs`・`exp-award-logic.mjs`＋`tnx-exp-award-app.mjs`（新規）・`tnx-access-cards.mjs`・`tnx-hud.mjs`＋`hud.hbs`・`tnx-scenario-panel.mjs`・`tnx.mjs`・CSS） / **ステータス**: 実装済・**実機確認済（2026-08-27 の一括確認）**。**残り＝アクトシートの編集系改修**（名前欄・長文の ProseMirror 化・アクトコネクションのハンドアウト D&D 登録と開始時配布）

#### メカニクス監査

- **削除段階**（2026-08-08 ユーザー指示「まずは機能の削除だけ」）: 設定タブ（カード山6スロット・管理アクション5種）・セットアップウィザード・チャット送信系（トレーラー/ハンドアウト/テキスト/情報）・情報技能行の判定起動を機能ごと削除。アクトシートは台本の編集専用になった。**管理アクション5種は復活させない**（コンテキストメニュー等で代替＝ユーザー裁定・確定）。
- **アクト限定フラグ `isActLimited`**（一般技能・2026-08-08 裁定＝アクトコネクション専用名を避けた汎用名）: 用途は**アクト終了時の自動削除の目印のみ**（経験点まわりには触れない）。アクト終了で全キャストから自動削除・件数通知。
- **キー被りチェック＝アクト開始をブロック**（2026-08-08 裁定＝キーはプレアクトに相談してずらすもの）: 参加キャスト（ハンドアウト actorId）のキースタイル識別キーを照合し、重複があればエラー表示して開始中止。
- **切り札の自動配布**（アクト開始時・2026-08-08 ユーザー要望）: **キースタイルの識別キー⇔ニューロカードの画像ファイル名（拡張子除く）は完全同一**（2026-08-08 ユーザー確定）——この同一性で対応カードを特定する。画像パスはシステム同梱アセット参照で**ローカライズ不変**・既存デッキにも存在（Code 当初案のカード名パースはローカライズで壊れるため却下）。キースタイル未設定/複数・担当ユーザー不在・カード不在・置き場使用中は警告スキップ。既存の手動配布（DealTrumpDialog）は残置。キー被りブロックが切り札の一意性を保証する（同じカードを2人に配る事態が起きない）。
- **HUD ステータスカードエリア**（2026-08-08 ユーザー裁定）: 提示式カードでなく**状態からの自動表示**——シーンプレイヤー（本人のみ・User flag）／~~舞台裏（全員・開始〜終了）~~（**2026-08-22 訂正**: 舞台裏カード＝文字通り「舞台裏にいる＝登場していない」の表示。→ 同日「HUD 参加者パネル」エントリの是正2）／ゴースト（本人・isGhost）／抹殺（本人・conditionKind=erased）。これに伴い**アクセスカード新規作成から該当4枚を除外**（残り＝切り札・YES・NO・SEALED。既存ワールドの作成済みカードは不変）。他ユーザーのシーンプレイヤー確認はシナリオコントロールパネルの表示で行う。
- **経験点配布＝半自動**（2026-08-08 裁定・パネルの機能）: アクト終了確定後に配布アプリを開く。チェック8種（依頼1/会場1/参加5/RP5/PS5/SPS5/助力5/進行5）＋神業×1・登場シーン×1（上限5・**手入力**＝自動追跡は作らない）→自動合計→確定で**各ユーザーの履歴（User flag）に行を自動追加**（既存の履歴集計=exp.total に乗る）。RL 分＝~~会場手配＋min(PL合計÷3, PL人数)~~（**2026-08-30 是正**: 正しくは会場手配＋PL合計÷min(3, PL人数)[切り捨て]。→ 2026-08-30 エントリ）。取得条件の判定自体は主観を含むため自動化しない（§4.2）。

#### 設計レビュー

- 純ロジック TDD（13件）: `findDuplicateKeys`（同一キャスト内重複・空キーは数えない）・`matchTrumpCard`（basename 完全一致＝kabuto が kabuto-wari に誤爆しない）・`EXP_AWARD_CHECKS`/`calcPlayerExpTotal`（登場5点キャップ・負数0）/`calcRlExpTotal`（floor・min。式は **2026-08-30 是正**＝除数側が min）。
- 配布アプリ＝ApplicationV2（id `tnx-exp-award`・幅760・行状態は Map 保持・変更で全再描画・数値は number-input-spinner 規約）。履歴記帳は既存 `historyAdd`+`saveUserFlagHistory`（exp.total 再計算込み）。
- HUD 追随フック: sessionState onChange（舞台裏）・updateUser（自分の flag＝シーンプレイヤー）・updateActor（担当キャラの isGhost）・ActiveEffect create/delete/update（担当キャラ＝抹殺）。
- 検証: テスト 1887 green（+13）・eslint 0 error（警告15は既存）・stylelint clean。

#### 課題・申し送り

- ~~アクトシートの編集系改修~~ → 完了（下記追補 `4bde02e`）。
- 実機確認（14-2〜14-7 一括・後続合流後）。

### 2026-08-08 フェーズ14-7 追補 アクトシート編集系改修（ルーラーシーン・登場設定・アクトコネD&D・ProseMirror）

**2026-08-08** / **関連実装** `4bde02e`（`tnx-scenario-sheet.mjs`・`scenario-sheet.hbs`・`session-logic.mjs`・`session-state.mjs`・`appearance-logic.mjs`・`appearance-check.mjs`・`skill-dictionary.mjs`・`tnx-scenario-panel.mjs`＋`scenario-panel.hbs`・CSS） / **ステータス**: 実装済・**実機確認済（2026-08-27 の一括確認）**

#### メカニクス監査

- **ルーラーシーン**（2026-08-08 ユーザー指摘＝用語は「マスターシーン」でなく**ルーラーシーン**・MS チェックボックスは役割が分かりにくい）: シーンプレイヤーセレクトに GM ユーザーを「（RL）」付きで含め、**GM 選択＝ルーラーシーン**とする。**ルーラーはプレイヤーではないため、ルーラーシーンにシーンプレイヤーはいない**（2026-08-08 裁定）——isScenePlayer は誰にも立てず、パネル・切替メッセージの表示は**「ルーラーシーン」とだけ**（「シーンプレイヤー：なし/ルーラーシーン」という表現にしない）。旧データの `isMasterScene` は読み出し時の読み替えで吸収（一括書き換えなし・§既存規範）。
- **シーン行の登場設定**（2026-08-08 ユーザー要望）: モード3値＝エリア準拠（既定・従来挙動）/数値指定（TN 直接入力）/**登場：不可**（シーンプレイヤー以外登場できない）。不可は登場判定の起動時に警告して中止。**指定技能**（複数可・辞典プルダウンから識別キーで登録）は登場判定ダイアログの先頭に「シーン指定」群として提示——**候補の提示であって制限ではない**（他の技能も選べる・システムは可否を制限しない原則）。数値指定は **TN の差し替えのみ**でエリアの性質は変えない——危険値係数はエリアに従って乗り、サンクチュアリの装備ブロックも fixed のまま生きる。
- **情報項目の指定技能**（2026-08-08 ユーザー要望＝辞典プルダウン＋その他自由記述）: 技能行に識別キーの select（「その他（自由記述）」＝キー空で従来の name 入力を表示）。表示は**辞典逆引きの現在名を〈〉囲い**で解決し生キーを出さない（識別キー規約）。辞典から消えたキーは残存 name をフォールバック。→ **2026-08-09 裁定で自由記述は廃止**（`5c0e782`→`6f9dc21`・「アクトコネも辞典に追加して使用する運用になったため、自由記述はもはや必要ない」）: 使用技能は**辞典のタグ入力**＝**1 行＝技能の集合＋共通の目標値**で、選ぶと行の中に識別キーが**配列として積み上がる**（シーンの指定技能と同じ `.tnx-tag-field`）。**目標値の異なる技能は使用技能の行そのものを足す**運用のため行は残す（ユーザー明示「行の追加が別目標値用なので残す」）。**Code は初版で「同じ形式」を1行1技能のプルダウンと誤読**し、ユーザーが行を残す理由をわざわざ説明した意図を取り違えた（`5c0e782` を `6f9dc21` で是正）。
    - データ: 技能行に `identificationKeys`（配列）。旧形式（`identificationKey` 単体）は読み出し時に 1 件として読む純関数 `infoSkillKeys` で正規化し**一括書き換えはしない**（タグを足した行から順に配列で保存される）。行の作成は `{id, identificationKeys: [], tn}`。
    - 表示解決は `resolveInfoSkillName`（単数）→ **`resolveInfoSkillNames`（複数・`names` 配列）**へ。辞典から消えたキーは表示から落とし、**解決できるキーが一つも無いときだけ**旧い自由記述の `name` をフォールバックにする（従来の単数版の意味を保存）。パネル一覧ラベルと送信メッセージは行内技能を「/」で連結（送信は従来どおり同じ目標値でまとめる書式）。
    - UI: タグ数で高さが変わるため **FS 判定のイベント行に倣って行をカード**にし切れ目を見せる。目標値・削除は一つのグループにまとめ、**カードの一番下＝追加プルダウンと同じ行**に揃える（`4a196a6`・当初はタグ領域の 1 行目に揃えたが「＋ 技能を追加」と段が合わず浮いた＝ユーザー指摘。下端一致で実測）。セクションの＋（行追加＝目標値の異なる技能）と行内の「＋ 技能を追加」は説明を書き分ける。
- **アクトコネクションの D&D 登録**（2026-08-08 裁定＝アクト限定コネは辞典に載らない→プルダウン不可・ハンドアウト行へのドラッグ＆ドロップで設定）: UUID 参照で保存・チップ表示は fromUuid ライブ解決（名前キャッシュなし）。一般技能のみ受け付け。**アクト開始時に actorId のキャストへコピー付与**（`isActLimited` を立てる＝アクト終了時に自動削除）。同じ識別キー（または同名）の技能を既に持つ場合はスキップ（再開始の重複付与を防ぐ）。参照切れは警告してスキップ。

#### 設計レビュー

- 純ロジック TDD（+12件・計1895 green）: `appearanceCheckParams` の mode 分岐（fixed の TN・none の blocked・fixed でも sanctuary 装備ブロック）・`normalizeSceneRow`（appearanceMode/Value/Skills）・`normalizeHandoutRow`（actConnections）・`buildSceneSwitchMessage`（rulerScene/旧 isMasterScene →「ルーラーシーン」表示）・`resolveInfoSkillName`/`withResolvedInfoSkillNames`（〈〉囲い・†省去・フォールバック・元データ非破壊）。
- 長文5箇所（トレーラー/ハンドアウト内容/テキスト/情報本文/切替メッセージ）を `<prose-mirror name=… toggled>` 化。**保存経路は既存のフラグ書き込みのまま**（change＋save の両イベントを購読・二重発火は同値保存で冪等）。指定技能の追加用 select は `data-no-save` で保存系リスナーから除外。
- 識別キー→現在名の逆引きは `loadGeneralSkillNameByKey()`（skill-dictionary・キャッシュ済み entries から Map 生成）に一本化し、シートのチップ・パネルの一覧ラベル・送信メッセージが共用。アクトコネチップは `itemDisplayName`（〈〉囲い）。
- アクト名編集はシートの入力欄→`document.update({name})`（空文字は無視）。MS 列の削除でシーン行グリッドは 7→6 列。
- 検証: テスト 1895 green・eslint 0 error（警告15は既存）・stylelint clean。

#### 課題・申し送り

- 実機確認（14-2〜14-7 一括・後続合流後）: prose-mirror の保存タイミング・D&D 登録・登場判定ダイアログの「シーン指定」群・ルーラーシーン表示。

### 2026-08-08 フェーズ14-7 是正 アクトシートの致命的不具合（エディタ本文消失・レイアウト崩れ・セレクト表示）

**2026-08-08** / **関連実装** `00fec96`・`de12c9e`・`31c3ed2`・`2b3cb03`・`1f446ac`（`scenario-sheet.hbs`・`tnx-scenario-sheet.mjs`・`tnx-item-sheet.mjs`・`editor-sections.mjs` 新設・`tnx2.css`） / **ステータス**: 是正済・隔離実機で検証済

ユーザー実機指摘（レイアウトぐちゃぐちゃ・ハンドアウト内容のエディタを開く手段がない）を受け、隔離 Foundry（develop 複製）＋Playwright で全数検証して是正。メカニクス変更はなし（全て表示・保存経路の技術是正）。

#### 設計レビュー

- **prose-mirror の本文消失**（`00fec96`・最重大）: Foundry の `HTMLProseMirrorElement` は生値を **`value` 属性**から初期化し innerHTML は表示専用（本体ソースで確認）。14-7 の `4bde02e` は innerHTML にしか渡しておらず、**編集を開くと空・保存で既存本文が `""` 上書き**（隔離実機で再現）。5箇所全てに `value="{{…}}"` を付与。あわせて編集導線を item シートで確立済みの **tnx-editor-section**（枠付き＋トグルボタンをヘッダーへ移設＝常時視認）へ統一し、移設ロジックを `editor-sections.mjs` に一本化して item シートの既存インライン実装も同関数へ統合。旧 textarea 用 CSS を撤去。
- **セレクトが保存値を選択表示しない**（`de12c9e`）: Handlebars の**ブロックパラメータは `../` 付きパスでは解決されない**（scratchpad で単体再現）。`../handout.actorId`・`../scene.playerUserId`・`../scene.area` の3箇所が常に未選択表示（保存は成功・14-2 由来の型を 14-7 も踏襲していた）。`../` を外して直参照に。
- **レイアウト崩れ**（`31c3ed2`）: コアの `select { width: 100% }`／`h5 { font-size: 20px }` の上書き漏れが根因——登場判定行のセレクトが行を占拠しラベルが1文字ずつ縦折り返し・「使用技能」見出しが巨大化。ほか「シーンプレイヤー」見出しの2行折り返し（nowrap＋省略）・情報項目の技能セレクト幅欠落による「目標値:」折り返し・**情報の内容ブロックの入れ子欠落**（使用技能・本文がブロック外の兄弟だった従来からの構造）を是正。目標値を number-input-spinner 化（数値入力の規約準拠）し、シート内3種のスピナーアクションを `spinUp`/`spinDown` に一本化。シートを resizable 化。
- **詳細設定の開閉消失**（`2b3cb03`）: 全再描画方式のため `<details>` が入力のたびに閉じる（登場判定モード変更はスピナー表示に再描画が要るため必ず閉じていた）。open 状態をシーン行 id 単位で `_preRender`→`_onRender` 保持（スクロール保持と同方式）。
- **既定幅 830**（`1f446ac`）: 800 では「シーンプレイヤー」見出しが実測 5px 不足で省略されるため。
- 検証: テスト 1895 green・eslint 0 error・stylelint clean。隔離実機で「本文あり→編集を開くと本文表示→保存で本文保持」「エリア red 保存→表示レッド」「details 維持＋スピナー出現」「情報ブロック入れ子・h5 12.6px・スピナー保存」をスクリーンショット付きで確認。

#### 課題・申し送り（ユーザー言及の「機能面で細かい修正点」候補・未着手）

- シーン行・ハンドアウト・テキスト・情報項目の並び替え（表・リスト系の TNX 標準。RL プリセットのみ上下ボタンあり＝不整合のまま）
- ~~指定技能セレクトの初期値が辞典先頭の技能のまま「＋」を押せる（プレースホルダ未設置）~~（是正2 で解消）・キャストセレクト固定 140px の窮屈さ
- prose-mirror の表示は生 HTML のまま（@UUID リンクの enrich 未実施）

### 2026-08-08 フェーズ14-7 是正2 ハンドアウトのコネ統合・情報項目の実行系トグル廃止

**2026-08-08** / **関連実装** `e6901a7`（`scenario-sheet.hbs`・`tnx-scenario-sheet.mjs`・`session-logic.mjs`・`session-state.mjs`・`tnx-scenario-panel.mjs`・`tnx2.css`・tests） / **ステータス**: 是正済・隔離実機で検証済

ユーザー指摘（コネ欄とアクトコネが別々・ドロップエリアが他シートと別方式・「情報の内容」ヘッダーが意味不明・実行系トグルの削除漏れ・自由記述欄が狭い）＋裁定変更（アクトコネの登録方式）への対応。

#### メカニクス監査

- **ハンドアウトの構造**（ユーザー指摘で確定）: コネ欄＝アクトコネそのもの（別欄にしない・ラベル「コネ」で足りる）。**推奨スートはコネに付随する情報**（コネ行に対で置く。当初 Code はスタイルと同じ行に括り誤読→是正）。**指定スタイルはコネと無関係の独立項目**で、「推奨」でなく**必ず取得するもの**（ラベルは「スタイル」で足りる・データフィールド名 `recommendedStyle` は据え置き＝表示のみの是正）。**入力はスタイル辞典のプルダウン**（`805ba72`・「必ず取得するもの」の指定＝辞典の有限選択肢からの指定であり、テキスト入力は不適切＝スートと同種。当初 Code はラベルのみの指摘と狭く読み→ユーザー指摘で是正。キー保存・表示/送信は辞典逆引き・旧自由テキストは「旧: …」フォールバック）。
- **アクトコネは必ず一つ**（2026-08-09 ユーザー裁定・`9f0e79c`）: 複数登録（チップ＋追加）を廃し単一セレクトへ。保存は単一キー `actConnection`（旧配列 `actConnections` は読み出し時に先頭の文字列キーへ読み替え・一括書き換えなし）。付与・送信も単一キー前提。
- **2026-08-09 追補（`9d469f1`・同日の追加指摘一式）**:
  - シーン表: 「登場：不可」→**「不可」**・「舞台」→**「使用シーン」**に改称。**エリアは詳細設定へ移動**（メイン行の詰め込みで使用シーンセレクトが見切れていた）。詳細設定はエリア→登場判定→指定技能→切り替えメッセージの順。列構成 6→5（36px 3fr 2fr 3fr 32px）。
  - 情報項目: 「その他（自由記述）」の記述欄を**1行内に戻す**（2行目全幅はかえって分かりにくいとの指摘・スペース的に1行に収まる）。
  - **ハンドアウトはユーザーに付与される**（ユーザー裁定）: 参照を `actorId`（キャスト）→ **`userId`（対象ユーザー）**へ変更。自動設定・コネ付与のキャストは user.character から解決（`_resolveHandoutAssignments`・旧 actorId はキャスト直接参照として読み替え＋担当ユーザー逆引き・書き換えない）。対象ユーザー欄はハンドアウト自体の内容ではないため最下部。**PC名欄（pcName）は廃止**（割当欄と重複）。
  - **レイアウトの全面改修**（`4b6cc68`・ユーザー指示「全面的に改修・再び表示崩れが起きたら意味が無い」）: 場当たりの個別配置を廃し、item シート群の確立意匠＝**tnx-fieldset＋ラベル列グリッド（96px+1fr・`tnx-form-field > .form-group`）**へ統一。シーン詳細設定は「登場設定」fieldset（エリア・登場判定・指定技能）＋切替メッセージエディタ、ハンドアウトはコネ（付随スート）/内容/PS/対象ユーザーを同グリッド、RL プリセットも同グリッド適用（split-fields は既存意匠のまま）。初版はユーザー却下（「やらないほうがマシ」）→是正 `f41c24c`: **(1) fieldset+legend は撤回**——item シートの意匠であってこのシート（セクションヘッダー/カード/折りたたみ）の整理語彙に無い異物。「確立意匠の流用」は**対象シート自身の語彙**に揃えるのが正（他シート族の意匠を持ち込むのは統一でなく異物混入）。「詳細設定」の折りたたみ自体をグループとする。**(2) 固定 96px ラベル列は撤回**——短いラベルでコントロールとの距離が開く。グループ共有グリッド（`max-content` 列＋ラベル右揃え＝display:contents で form-group を透過）でラベルは常に 10px 密着・コントロール左端はグループ内一致。**(3) コントロール幅の規律**——セレクトは内容幅（180〜380px）・スピナー/チェック行は左置き・全幅はテキスト入力と長文エディタのみ（幅=期待される入力量の合図）。検証は密着 10px とコントロール左端一致の実測＋全タブのスクリーンショット組版検査。反省の記録: 「壊れた状態との差分」を合格基準にした症状単位のパッチ（詰問1回目）→「整列」だけ達成して幅・グループ語彙を設計しない改修（却下2回目）。**基準は対象シート自身の意匠語彙とあるべき姿・組版として見る・整列と幅の両方を設計する**。→ さらに是正 `5b0c5ec`（指摘3回目「無意味な隙間が多すぎる・グリッドレイアウトそのものをやめてほしい」「要素を縦積みするだけのレイアウトをやめてほしい・右側が空きすぎ」）: **(4) ラベル列グリッドはハンドアウト／シーン詳細設定では不採用**——共有ラベル列は**グループ内の最長ラベル（対象ユーザー＝84px）に合わせて伸びる**ため、コネ・PS のような短いラベルの左に無用の隙間が空く（グループ共有＝密着は「最長ラベルの行だけ」成り立つ性質だった）。ラベルをコントロールに直付けした横並びの行（`.handout-row--meta/--ps/--user`・`.scene-detail-row`）に置き換え。**(5) 1欄1行の縦積みをやめる**——エリアと登場判定を同じ行に並べ、**残り幅を等分**（`flex:1 1 0`・上限 380px）して右の空きを消す。指定技能はチップ→追加プルダウンを同じ行に。ラベル列グリッド（`.tnx-form-field`）は **RL プリセット専用**として残す（ラベル長が揃い、行が単一コントロール）。**幅の規律は「内容幅」から「行の残り幅を分け合う（上限 380px・全幅はテキスト入力と長文エディタ）」へ更新**。反省の記録: **「隙間なく整列している」ことと「幅が使えている」ことは別**で、共有ラベル列は前者を満たしても後者を壊す。**適用範囲を全域に広げる前に、その方式が破綻する条件（ラベル長のばらつき・1行1欄）を先に挙げること**。
  - **指定技能はタグ入力＝共通部品 `.tnx-tag-field` に一本化**（`40c4d4d`・ユーザー指摘「追加時のレイアウトが崩れる・**モデルケースは FS 判定シートに既にあるはず**」）: タグを行に直接並べると、追加のたびに折り返し位置が変わり追加プルダウンが右や下へ流れ、ラベルの下にまで回り込む。**同じ問題は FS 判定エディタで解決済み**（2026-07-21「タグ領域を先に確保し、追加プルダウンはその下に固定する」）だったため、`.fs-editor` スコープの `.fs-skill-*` を汎用の `.tnx-tag-field` / `-list` / `.tnx-tag` / `-remove` / `-add` / `-empty` へ**移設して両方で共用**（FS 側は 28px 高さの維持のみ残す。JS 参照 2 箇所も改名）。シーン側は追加プルダウンの「＋ 技能を追加」を選ぶこと自体が追加操作（＋ボタンと `addAppearanceSkill` アクションは廃止）・未設定は「（未設定）」。行内では `flex: 1 1 0`——**flex の折り返し判定は縮む前の内容幅で行われる**ため、基準を内容幅にするとタグが増えた分だけタグ領域ごとラベルの下へ落ちる。**教訓: 「複数選択の登録＋追加」という同じ UI 問題は、既存の解を先に探して踏襲する**（今回は自前でゼロから組み、既に解決済みの失敗を再発させた）。**部品化の切り分け**（`dfd058c`・ユーザー指摘「大きさが他の箇所と合っていない・**まったく同じで良いわけはない**」）: 共通部品が持つのは**構造と部品固有の意匠**（タグ領域の先行確保・プルダウンの位置と最大幅・タグの丸み）まで。**高さと文字サイズは置いたシート自身のスケール**に従う——FS エディタは他の入力が 30px の縮小スケールなので 28px/0.85em、アクトシートは他のセレクトと同じ 32px/14px。当初は FS の寸法ごと共通部品に入れ、アクトシートで一つだけ小さく浮いた。**「確立意匠の流用＝対象シート自身の語彙に揃える」（`f41c24c` の教訓）は寸法にも及ぶ。**
  - **登場判定「不可」のシーンでは指定技能欄を出さない**（同 `5b0c5ec`・ユーザー指摘「どう考えてもおかしい」）: `appearanceCheckParams` は `mode==="none"` で即 `blocked`＝エリアも指定技能も参照されない（登場判定そのものが起きない）ため、設定欄が残るのは論理の破綻。テンプレートで `{{#unless (eq scene.appearanceMode "none")}}` によりゲート。**既存の `appearanceSkills` は消さずに保持**し、モードを戻せば再表示される（表示条件の変更をデータの書き換えに波及させない）。
  - **ハンドアウト名＝「①〈スタイル名〉用ハンドアウト」形式の自動生成**（ユーザー裁定・**番号は前置**＝当初 Code は後置で実装し即指摘で是正）: スタイルセレクトが見出しなしで1行目に置かれ名前の構成要素を兼ねる。番号はスタイル指定行（共通・自由記述以外）の通し番号（丸数字①〜⑳・純関数 `circledNumber`/`handoutTitleSuffix`/`handoutDisplayTitle`/`handoutNumberOf`・TDD）。「共通」（`@common`）＝共通ハンドアウト（番号なし）・「自由記述」（`@free`）＝title 自由入力（記入欄を自動表示エリアまで伸長）。パネル一覧・チャット送信の見出しも同形式（`buildHandoutMessage` は解決済み見出しを受け取る純関数に）。
- **アクトコネの登録＝辞典コネ技能のプルダウン**（ユーザー裁定・D&D は撤回）: 一般技能辞典のうち識別キー contact プレフィックスの技能から選ぶ。「必然、指定のコネは必ず辞典に格納しなければなりません」＝**辞典格納が運用前提**。保存は識別キー配列（`actConnections`・旧 `{uuid}` 形式は実機確認前のため読み捨て）・表示は辞典逆引きチップ（〈〉囲い・生キー非表示）・アクト開始時の付与も辞典解決（getIndex→getDocument・KI-026 の孤児化を起こさない）。
- **情報項目の公開/非公開・未調査/調査済みトグルはシートから削除**（ユーザー指摘＝実行系全削除(b3ce42a)の削除漏れ。操作はシナリオコントロールパネルの領分・14-3 で移行済み）。「情報の内容」ヘッダー行も廃止し、内容の削除は枠内右上のアイコンに。

#### 設計レビュー

- 旧 `connections` 自由テキスト・キー以外の旧スート文字列は、チップ/セレクトの空値ラベルに「旧: …」で表示フォールバック（データは書き換えない＝旧 player と同方式）。推奨スートはキー保存のセレクト（`HANDOUT_SUIT_OPTIONS`）でチャット送信もラベル解決（`handoutSuitLabel`）。
- コネ登録 UI はシーン指定技能と同型（プルダウン＋チップ・`data-no-save`・未選択プレースホルダ）。同型化に合わせ指定技能セレクトにも「- 技能を選択 -」を追加（先頭技能のままの誤登録防止）。
- 「その他（自由記述）」の記述欄は 2 行目全幅に配置（`:has(.skill-name)` で wrap・実測 534px）→ 1 行内へ戻した後、**2026-08-09 に自由記述ごと廃止**（`5c0e782`・`.skill-key--free` / `.skill-name` の CSS も削除。`legacyName` は `_prepareContext` で「識別キーが無く name がある行」にだけ立てる＝辞典行に残る旧 name はラベルに出さない）。
- 検証: テスト 1899 green（+4）・eslint/stylelint clean。隔離実機で「コネ登録→チップ〈コネ：検証用〉→識別キー保存」「スート選択→保存→表示」「トグル/ヘッダー不在・削除アイコン枠内」をスクリーンショット付きで確認。ハンドアウト送信のコネ表示名解決はコード検証のみ（アクト開始付与も同・実機確認は 14-8 一括に含める）。

### 2026-08-09 使用回数の最大値を式で入力可能にする（設計・実装未着手）

**2026-08-09** / **関連実装** `scripts/data/item/uses.mjs`(新設)・`helpers.mjs`・`common/outfit-base.mjs`・`style-skill.mjs`・`miracle.mjs`・`actor/common/character-base.mjs`・`usage-consumption.mjs`・`tnx-character-sheet-base.mjs`・`tnx.mjs`・`tnx-item-sheet.mjs`・`tnx-outfit-sheet.mjs`・`tnx-style-skill-sheet.mjs`・3シートの hbs・`tnx2.css`＋tests / **ステータス**: 実装済（実機確認は 14-8 一括）

ユーザー要望「最大『レベル』回のような表現が現状できない。最大値欄に式で値を入力可能にしてほしい」に対する設計。フェーズ14 のロードマップ項目ではなく、合間の機能追加（PATCH 相当）。

#### メカニクス監査

- **「レベル回」の意味**（2026-08-09 ユーザー裁定）: 使用回数の最大値がレベルを参照する場合、**AE でレベルが上がっても使用回数は増えない**。Code は当初これを「実装順序ゆえの制約」として提示したが、ユーザーはこれを**ルールとしての正解**と裁定した。したがって式の評価は AE 適用**前**の値を読むのが正しく、後述の順序境界は妥協ではなく意図した挙動である。
- **神業の母数バフは AE で増える**（既存記述の追認）: [Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md) の「万能神業（ファイト！等）による母数増加は AE で `uses.max` を増やす」は正しい仕様。ただし着地点が無く**実際には効いていなかった**（[KI-038](../01_Wiki/Issues/Known_Issues_Detail.md)）。本改修で新設する実効値 `uses.maxTotal` を AE 着地点として登録し、記述どおりに動くようにする（2026-08-09 ユーザー裁定「登録する」）。[Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md) §2.2 の「`uses` には効かない」は訂正対象。
- **対象タイプ**（2026-08-09 ユーザー裁定）: アウトフィット共通・スタイル技能・神業の**3タイプとも同じ表示**にする。ただし**実質的に式を使うのはアウトフィットとスタイル技能のみ**。神業の最大値（母数）はスタイルレベル連動フックが機械維持するため、式を入れても次のレベル変更で数値に上書きされる——これは仕様として許容し、専用の分岐は設けない（ユーザー「神業についてはそれで結構です」）。
- **スコープ外**: 残弾（`ammo.max`）・常備化個数（`quantity.max`）。同じ形なので必要になれば同じ手順で足せる。

#### 設計（データモデル）

- `uses.max` を `NumberField` → **`StringField`**（3モデル: `outfit-base.mjs` / `style-skill.mjs` / `miracle.mjs`）。**数値も式も同じ1欄で受ける**——用途の `checkBonusSelf` / `damageBonusSelf` と同じ既存流儀（`parsePlainNumber` が純数値を Roll を介さず即返す）。数値欄と式欄に割らないのは真実の所在を1つに保つため。initial はアウトフィット/スタイル技能 `""`・神業 `"1"`。
- migrateData: 数値→文字列。共通関数 `migrateUsesMaxToString(source)` を `helpers.mjs` に既存の `migrateUsesValueToSpent` と並べる。**呼び出し順は「spent 移行 → max 文字列化」**（前者が `typeof uses.max === "number"` を読むため。神業の `usageCount` 移行も同じ理由で先）。
- `uses.spent` は数値のまま（消費実績であって設計値ではない）。
- 派生（スキーマ外）で **`uses.maxTotal`（number）**。`levelTotal` / `actionRank.maxTotal` と同じ位置づけ。

#### 設計（派生パイプライン）

既存の「base → total を置いてから AE が total を改変する」原則のまま 2 段。

1. **アイテム単体の段**（各 DataModel の `prepareDerivedData`）: `uses.maxTotal = 評価(uses.max)`。式には `@item.self` だけを供給する（アクター無所属＝辞典・ワールド直下・コンペンディウムでもシートが値を出せるようにするため）。アウトフィットは `computeItemEffectiveValues` に相乗り、スタイル技能・神業は各 `prepareDerivedData` から共通関数を呼ぶ。
2. **アクターの段**（`character-base.mjs` の `_applyEffectBuffs()` の**直前**）: 所有アイテムの `uses.maxTotal` を `buildFormulaData(actor, null, item)` のフル文脈（`@system.*` / `@item.<識別キー>.*` / `@item.self`）で再評価する。
3. **AE の着地**: `resolveItemTotalPath("uses.max")` を `uses.maxTotal` へ写像（`uses.maxTotal` 綴りも同じ所へ正規化）。AE 適用後の 0clamp を `actionRank.maxTotal` の隣に置く。

- **順序境界**: 式が参照する値は **AE 適用前**の実効値。これは上記メカニクス監査の裁定（レベルが AE で上がっても使用回数は増えない）と一致する。技術的にも、②を AE の前に置くことで AE（`system.uses.max`）を `uses.maxTotal` へ素直に着地させ、**追加／上書き／乗算のモード意味論をそのまま活かせる**（後段に置くと AE 分を消すため差分アキュムレータが必要になり、上書き・乗算が壊れる）。`actionRank.maxTotal`（派生で base を置き、その上に AE が乗る）と同型。
- 評価不能（自由文・ダイス入り・構文エラー）は `0`。式どうしの相互参照は前提としない（既存の式機構と同じ）。

#### 設計（読み手・書き手の全数掃引）

実効フィールド新設のため `uses.max` を触る全箇所を洗う（grep 済み・下記で全数）。

読み手 → `maxTotal` へ:
1. `usage-consumption.mjs` `ITEM_RESOURCES.uses` の `remaining` / `max` / `update` クランプ（消費経路の単一チョークポイント。ここ1箇所で消費ダイアログの「残り X/Y」も不足判定も正しくなる）
2. `tnx-character-sheet-base.mjs` 神業スロットの並び生成
3. `tnx-character-sheet-base.mjs` 神業使用時の残り判定と spent クランプ

書き手（機械維持の経路・数値を書き続ける）:
4. `tnx.mjs` 神業のスタイルレベル連動、`tnx-character-sheet-base.mjs` 多重取得の +1 / −1 → `String(newMax)` を書く。「現在の母数 +1」の土台は素値が数値としてパースできればそれ、できなければ（＝式が入っている）`maxTotal` を土台にする。
5. `tnx-outfit-sheet.mjs` / `tnx-style-skill-sheet.mjs` の制限オフ時リセット値 `0` → `""`

#### 設計（UI）

- 3シートの「最大値」欄（`outfit-sheet.hbs` / `style-skill-sheet.hbs` / `miracle-sheet.hbs`）を `type="number"` → `type="text"`。placeholder は既存の式欄の流儀を踏襲（`式（例: @item.self.system.levelTotal、3）`）。
- 入力欄の直後に**実効値バッジ**（`＝ 3`）。**素値と一致するとき（純数値）は出さない**＝式のときだけ。評価不能は `＝ ?`。式を表示するなら実効値も見せる（説明文言ではなく動的データなので UI 文言規約に抵触しない）。
- 「消費済み」欄は数値入力のまま（使用回数はスピナー不使用の既存例外を維持）。

#### 設計（テスト）

- 純粋関数 `computeUsesMaxTotal`（純数値／空／評価不能→0／負→0clamp）
- `resolveItemTotalPath("uses.max") === "uses.maxTotal"`（両綴りの正規化）
- migrateData の数値→文字列（3モデル）。既存の「`uses.max` は NumberField で initial 0」テスト2件（`style-skill.test.mjs` / `outfit-base.test.mjs`）は StringField 版へ差し替え
- `ITEM_RESOURCES.uses` が `maxTotal` を読むこと（`usage-consumption.test.mjs`）

#### 課題・申し送り

- 実装時に更新する記録: Active_Effects.md §2.2（`system.uses.max → uses.maxTotal` を着地表へ・「uses には効かない」を訂正）／Formula_Reference.md（`@item.<key>.system.uses.maxTotal` の追加と順序境界の明記）／Usage_System.md（神業母数バフが実際に効くようになった旨）／Item_Types.md・DataModel.md（`uses.max` の型）／KI-038 を Resolved へ。
- 実機確認は 14-8 の一括確認に含める（起動形は変わらないが、使用回数は消費フロー全体に効くため単独では閉じない）。

#### 実装結果（2026-08-09・上記設計どおり）

- **新設 `scripts/data/item/uses.mjs`**（`part-helpers.mjs` と同じ「item 直下のユーティリティ」・`tests/template-integrity.test.mjs` の非 DataModel 登録簿に宣言）: `resolveUsesMax`（数値も式も受け、0clamp・切り捨て・評価不能は 0）・`usesMaxTotalOf`（読み手の実効値リーダー）・`usesMaxBaseOf`（母数を機械維持する書き手の土台）・`computeUsesMaxTotal`／`ForActor`・`clampUsesMaxTotalForActor`・`migrateUsesMaxToString`。
- **StyleSkill の評価順**: `prepareDerivedData` を `_applyLevelRef()` → `computeUsesMaxTotal(this)` に分解した（旧実装は levelRef の早期 return が末尾を巻き込むため）。「1シーンにレベル回」がレベル自動参照（10-3）の結果を読むのに順序が要る。
- **端数は切り捨て**（Code 判断・TNX の除算慣例に合わせた。裁定余地あり）。
- 検証: テスト **1946 green（+28）**・eslint 0 error（警告15は既存）・stylelint clean。
- **UI（2026-08-09 ユーザー是正後の確定形）**: 最大値欄は `type="text"`＋プレースホルダ「数値 または 式」のみ。**消費済み欄と同じ 200px**（`.usage-limit-group:has(.usage-field--formula) .usage-field input`）で、式は左揃え・消費済みは数値なので中央揃えのまま。
  - **却下された初版（ユーザー指摘「=n はいりません。余計なことをしないでください」）**: 実効値バッジ `＝ 3`（評価不能は `＝ ?`）を入力の右に出し、最大値欄だけ行の残り幅いっぱい（308〜338px）に広げていた。バッジと、それを支えるためだけに置いた派生 `uses.maxResolved`・`usesMaxIsFormula`・`resolveUsesMaxDetail` は**すべて撤去**（機能が消えたら支える仕組みも残さない）。
  - **教訓**: 「式を表示するなら実効値も見せる」という一般則（`feedback_formula_needs_input` 由来）を、**求められていない表示の追加**の根拠にしていた。要求は「式を入力できるようにする」であって「計算結果を見せる」ではない。欄の幅も同様で、**入りうる最長ではなく実際に入る式の長さ**（`@item.self.system.levelTotal`＝実測 173px）で決めるのが正しく、行いっぱいに広げたのは過剰だった。
  - 幅の実測（隔離ブラウザ・実 CSS＋実マークアップ）: 200px で `@item.self.system.levelTotal`（173px）も `… - 1`（192px）もプレースホルダも収まり、両欄の幅が一致、4状態すべて行高 52px（「/」の位置が崩れない）、横スクロールなし。150px を先に試したが式が収まらず却下。

#### 課題・申し送り（更新）

- **Foundry 実機での確認は未実施**（14-8 の一括に含める）: ①「レベル回」の式を入れた技能の残量が消費ダイアログで正しいこと ②`system.uses.max` の AE（神業母数バフ）が実際に効くこと＝KI-038 の本丸 ③旧データ（数値 max）の移行が保存時に文字列で着地すること。
- 残弾（`ammo.max`）・常備化個数（`quantity.max`）は式非対応のまま。これで `resolveItemTotalPath` が死んだ着地先（`ammo.max.total` 等）を返す唯一の残りになった——必要になったら uses と同じ手順で足せる。

### 2026-08-09 フェーズ14-8 RL による登場（登場判定なし）・名前の非公開・シーン行の登場キャラクター事前設定

**2026-08-09** / **関連実装** `appearance-logic.mjs`・`appearance-state.mjs`・`session-state.mjs`・`session-logic.mjs`・`tnx-scenario-panel.mjs`＋`scenario-panel.hbs`・`tnx-scenario-sheet.mjs`＋`scenario-sheet.hbs`・`data/helpers.mjs` / **ステータス**: ✅ 実機確認済（隔離 Foundry・GM/PL 2クライアント）

#### メカニクス監査

- **RL は登場判定を経ずに任意のキャラクターを登場させられる**（ユーザー裁定）。当初の指示は「ゲスト」だったが、同日「当然、トループやエキストラも任意に登場させることができます」「RL はキャストを登場判定なしで登場させることも当然できます」と訂正され、**対象はキャラクター4種すべて**に確定。→ 候補は `CHARACTER_ACTOR_TYPES`（cast/guest/troop/extra）の非登場者。危険値・サンクチュアリ・エリア TN のどのゲートも通さない（判定を行わないため）。
- **「登場：不可」の意味＝登場判定が不可**（ユーザー訂正）。ただし**チームであっても登場できない**（同日裁定）→ 判定経路だけでなく**チーム免除も塞ぐ**。したがって既存の表示「登場：不可」と判定時の警告「シーンプレイヤー以外登場できません」は**そのままが正確**（一度「登場判定：不可」への言い換えを提案したが、チーム免除も塞ぐ裁定を受けて撤回）。RL の登場・事前設定はこのゲートの外側（RL の指定は判定でもチーム免除でもないため）。
- **名前の非公開**（ユーザー指示）: 「ルーラーシーンの場合は特に、登場したキャラクターの名前が情報として出ていないために非公開にしたい場合がある」→ **RL が登場させるとき**と**台本の事前設定**で指定でき、表示は「？？？」。ルール上の状態ではなく**卓への開示の制御**なので、登場状態と対の Actor フラグ（`appearingHidden`）に置き、退場で一緒に落とす。
- **誤りの訂正（Code）**: 「ルーラーシーンでは登場0人があり得る」と述べたが誤り。シーンは登場しているキャラクターが演じる場であり、ルーラーシーンはむしろ RL のゲストが出るシーン。観測していたのは「RL が入れる手段が無いのでフラグが立たない」という**本作業が埋めた穴**であって、シーンの状態ではない。0 人前提の分岐は作らず、RL 操作列は `actStarted` の間つねに描画する扱いにとどめた。
- **シーンプレイヤーと事前設定が重複した場合は名前を伏せない**（Code 判断・裁定余地あり）: シーンプレイヤーは卓に開示された主役のため。

#### 監査結果

✅ 上記はいずれもユーザー明示裁定に基づく。Code 判断は2点（①重複時にシーンプレイヤー側を採る ②トークンのネームプレートとチャット名は非公開の対象外＝従来どおり RL 管理）で、いずれも未裁定のため申し送り。

#### 課題・申し送り

- **トークン名・チャット名は「？？？」の対象外**（Code の線引き・ユーザー未裁定）。盤面に実名のネームプレートが出るため、卓での運用次第では要拡張。
- 「登場：不可」のシーンで PL がチームに参加したとき、登場しない理由の通知は GM クライアントにしか出ない（既存のチーム操作と同じ制約＝ソケットで GM が実行するため）。

### 2026-08-09 フェーズ14-8 巡回シーンとイベントシーン（固定でないリサーチシーンの表現）

**2026-08-09** / **関連実装** `session-logic.mjs`・`appearance-logic.mjs`・`session-state.mjs`・`scene-entry-dialog.mjs`（新設）・`residence-area.mjs`（新設）・`appearance-check.mjs`・`tnx-scenario-panel.mjs`＋`scenario-panel.hbs`・`tnx-scenario-sheet.mjs`＋`scenario-sheet.hbs` / **ステータス**: ✅ 実機確認済（隔離 Foundry・GM クライアント）

#### ルール解釈（ユーザー言語化）

- **シナリオの形式は一様ではない**（ユーザー）。すべてのシーンを厳密に決めた台本もあれば、**特定のイベント以外は固定の登場判定目標値もシーンプレイヤーも持たないリサーチシーンが続く**台本もある。後者は「ハンドアウトの1番目から順にシーンプレイヤーを回し、条件を踏んだところでイベントシーンが起動する」形式。
- **巡回の正本は「位置」ではなく「誰がまだシーンプレイヤーをしていないか」**（ユーザー）。イベントシーンのシーンプレイヤーを務めた者はその分が消化されるので、**順番は飛ぶ**。「なるべくシーンプレイヤーをしていないプレイヤーに回してあげる必要がある」ため、既定は未消化の先頭。
- **巡回順はハンドアウトの並び順**（ユーザー承認）。共通ハンドアウトと対象ユーザー未設定の行は順から外す（Code 提案・承認済み）。
- **巡回シーンのエリア・登場判定目標値は台本で決めない**（ユーザー）。シーン切替時のダイアログで決める。巡回シーンでは「エリア」「登場判定」の欄そのものを出さず、常に登場判定「未設定」扱い。
- **舞台＝住宅施設**（ユーザー指定）。候補は**シーンプレイヤーのキャストと登場キャラクター、またはそれらとチームを組んでいるキャラクターが所持している住宅施設**（当初「常備化している」と述べられたのち同日「所持している」に訂正）。住宅施設は登場判定目標値とエリアをそもそも持っているので、舞台にしたシーンはその2つを自動で読み取って適用する。
- **目標値の初期値は 10**（ユーザー指定）。舞台にもエリアにも依らず手で指定する場合の既定値。
- **登場判定「未設定」は巡回シーン以外でも選べる**（ユーザー）。選んだシーンはシーン開始時に同じダイアログが出る。**「未設定」でも指定技能は台本で設定でき**、ダイアログからも追加・削除できる。
- **イベントシーンは「起動条件」を持つ**（ユーザー）。条件は**自動判定しない**——RL が読んで判断し、任意のタイミングで巡回をやめてイベントへつなぐ。巡回シーンの後ろに複数のイベントが並んでいる場合は、**次の巡回シーン（またはクライマックス）との間のイベントから1つを選んで起動**する（想定と違う順番でイベントを巡るシナリオがあり得るため）。
- **イベントを終えて巡回に戻りたい場合は、台本に巡回シーンをもう一度置く**（ユーザー）。したがって台本は `[巡回][イベント…][巡回][イベント…][クライマックス]` の形になる。
- **「次のシーンへ」の出し分け**（ユーザー）: イベントシーンの次がイベントシーンなら出さず「イベントシーンを起動する」だけにする。次が巡回シーンなら両方出す。**リサーチ先頭のイベントは、オープニングからの「次のシーンへ」で普通に起動する**（順送りで入る経路を塞がない）。
- **イベントシーンは実行済みフラグを持つ**（ユーザー提案）。**開始時**に立て、**リサーチのイベントが全て実行済みになったら「クライマックスへ」を出す**。踏まれなかったイベントは選択ダイアログの「**起動せず実行済みにする**」で消化できる（Code 提案・ユーザー承認）。

#### 実装

- シーン行に `kind`（normal/rotation/event）と `eventCondition` を追加。`appearanceMode` に `unset` を追加。**巡回行は読み出し時に `area=""`・`appearanceMode="unset"` へ伏せる**（生データは書き換えないので、種別を戻せば元の指定が生きる）。
- 実行状態に `scenePlayerDone`（今巡の消化）・`scenePlayerUserId`（シーンプレイヤーの選択そのもの）・`doneEventSceneIds`・`sceneOverride`（その場で決めた値）を追加。消化はフェイズが変わったときと全員が務め終えたときにクリア。
- 住宅施設を舞台にした場合の適用は**新しい計算経路を作らず**、既存の `appearanceCheckParams({area, mode:"fixed", fixedValue})` に流す（14-7 の「数値指定＝目標値の差し替えのみ・エリアの性質は生きる」に乗る）＝危険値係数とサンクチュアリの装備ゲートが住宅エリアのランクからそのまま効く。

#### 監査結果

✅ 上記はいずれもユーザーの明示裁定・承認に基づく。Code 判断は3点。

1. **巡回順から外す行の条件**（共通ハンドアウト・対象ユーザー未設定）— 提示して承認済み。
2. **消化記録のリセット契機**（全員消化・フェイズ変更の2つ）— 提示して承認済み。
3. **住宅施設の実効目標値の読み方**＝`(appearanceTargetTotal ?? appearanceTarget) + 住宅エリアの appearanceTargetMod`（AE 込みの実効値）。ユーザー未裁定だが、「表示は全箇所実効値」の既存規範に沿う。

#### 課題・申し送り

- **イベント行の block を消化しきった状態でイベント行にいると、前へ進むボタンが1つも出ない**（`次のシーンへ` は「次がイベント」で塞がれ、`イベントシーンを起動する` は候補が空）。シーン一覧からのジャンプが逃げ道になるが一手多い。ユーザー指示どおりの帰結なので実装は素直に従い、扱いはユーザー裁定に委ねる。
- **巡回シーンの切替チャットは行の番号・名前をそのまま繰り返す**（`SCENE 2 : リサーチ` が毎回出る）。シーンプレイヤー行で区別が付くため通し番号は作っていない。
- **キャストシートの「登場」列が住宅施設の実効値を読んでいない**（`tnx-character-sheet-base.mjs` の `effectiveValues` が素値 `appearanceTarget` を使用。`tnx-outfit-sheet.mjs` は `appearanceTargetTotal` 優先）。本作業では触っていない（無関係な不具合を混ぜない）。**別途の修正対象**。

### 2026-08-12 フェーズ14-8 ハンドアウトのコネにワールド直下を含める

**2026-08-12** / **関連実装** `d11f629`（`skill-dictionary.mjs`・`tnx-scenario-sheet.mjs`・`tnx-scenario-panel.mjs`・`session-state.mjs`・tests） / **ステータス**: 実装済・隔離 Foundry で実機確認済

ユーザー要望「ハンドアウトのコネの検索対象にワールド直下も含めたい」への対応。

#### メカニクス監査

- **拡張はコネだけ**（2026-08-12 ユーザー裁定）。理由はコネ固有の運用にある——アクトコネクションは**そのアクト限りの技能**なので、恒久的な一般技能辞典に置きたくない。他の技能欄（シーンの指定技能・情報項目の使用技能・登場判定ダイアログの候補）が参照するのは既存の技能で、辞典にあるのが自然なため**辞典のみのまま据え置く**。逆引き（`loadGeneralSkillNameByKey`）も共有のまま触っていない＝コネ専用の索引を新設して差し替えた（当初 Code は共有逆引きを広げる案を出したが、範囲がコネに確定した時点で取り下げ）。
- **2026-08-08 の「辞典格納が運用前提」は撤回ではなく緩和**。保存形式（識別キー1つ・`actConnection`）・単一セレクト（2026-08-09 裁定）はそのままで、キーの解決先が「辞典」から「辞典＋ワールド直下」に広がっただけ。旧配列の読み替え・「旧: …」フォールバックも無改修。
- **識別キーは必須**（構造的必然でありユーザーの選択ではない）: コネかどうかの判定が識別キーの `contact` プレフィックスに依存しているため、識別キーの無いワールドアイテムは**コネと判定できない**＝候補に出ない。ユーザーは「プレフィックス統一は前提」と明言。絞り込み条件は辞典と**同一**（一般技能かつ固有名詞技能かつ `contact_`）。
- **同じ識別キーがある場合は辞典を優先**（Code 提案・ユーザー承認）。識別キーは一意な参照なので衝突は重複であって上書きの意図ではなく、辞典は全クライアントに見えるため解決が揃う。
- **プルダウンで出所（辞典／ワールド）を分けない**（2026-08-12 ユーザー指摘で Code 案を撤回）。Code は当初「同名が並んだとき区別できない」を理由に optgroup 分割を提案したが、**辞典優先で同一キーが1件に畳まれる以上、残るのは「同名・別キー」だけ**で、それは辞典の中だけでも起こる。この変更で新しく生じる問題ではないので分割は理由を持たない。並びは名前順で辞典とワールドが混ざる。

#### 設計レビュー

- **新設 `loadContactSkillIndex()`**（識別キー → `{name, uuid}` の Map）にコネの3経路を載せた: アクトシートの選択肢・ハンドアウト送信の表示名・アクト開始のコピー付与。**3箇所そろえないと部分実装**になる（選択肢だけ広げると、選べるのにパネル表示が「（参照切れ）」・アクト開始で警告されて配布もされない）。
- 組み立ては純関数 `mergeContactEntries(packEntries, worldEntries)`（絞り込み・辞典優先・名前順）に切り出して TDD **+6＝2043 green**。Foundry 依存は収集だけ。
- ワールド分は `game.items` の**都度読み**（`loadSkillEntries` のパック単位キャッシュに載せない）＝アイテムを作った直後に候補へ出る。**辞典側は従来どおりキャッシュ**なので、辞典に追加したコネはリロードまで出ない（本変更以前からの既存挙動・全ての辞典プルダウンで同じ・触っていない）。
- アクト開始のコピー付与の解決を `pack.getIndex`→`pack.getDocument` から**索引の uuid → `fromUuid`** へ。辞典側は内部で `getDocument`（キャッシュ優先）に落ちるので **KI-026 の孤児化は起きない**。uuid は文字列組み立てでなく `pack.getUuid()` で作る（書式の権威は Foundry 側）。これに伴い `loadSkillEntries` の返り値に `id` を追加（加算のみ・他の利用者に影響なし）。
- 参照先が広がったため警告文言を「コネ技能が**辞典に**見つかりません」→「コネ技能が見つかりません」へ（機能を広げたら、それを説明する文言も直す）。
- **実機確認（隔離 Foundry・実データ複製）**: ワールドのコネがプルダウンに出て選択が保存される／同キーは辞典側の名前で1件に畳まれる（`contact_kaburi` がワールド名でなく「コネ：辞典カブリ」）／optgroup が無い／識別キー無し・小分類違いは候補に出ない／送信チャットが〈コネ：ワールド太郎〉と現在名で出る／**アクト開始でキャストへ `isActLimited` 付きでコピーされ「コネを 1 件配布しました」**／参照切れキーで新しい警告文言が出る。ワールドアイテムの `toObject()` に乗る `folder`（null）・`ownership` は埋め込みで問題を起こさないことも確認（懸念していたが実測で無害）。
- 検証: テスト 2043 green・eslint 0 error（警告15は既存）。CSS 未変更。

#### 課題・申し送り

- 辞典に追加したコネがリロードまで候補に出ない件は**既存挙動**（全辞典プルダウン共通）。コネだけ都度読みにする非対称は、ワールド直下が「今このアクトのために作る」置き場である以上むしろ妥当と判断した。辞典側のキャッシュ無効化は必要になってから（§4.3 YAGNI）。

### 2026-08-12 フェーズ14-8 情報項目の段（追加で判明する内容）

**2026-08-12** / **関連実装**（`session-logic.mjs`・`tnx-scenario-sheet.mjs`・`tnx-scenario-panel.mjs`・`scenario-sheet.hbs`・`scenario-panel.hbs`・`tnx2.css`・tests） / **ステータス**: 実装済・隔離 Foundry で実機確認済（チャットカードは対象外＝下記）

ユーザー指摘「同じ項目でも要求技能が違う場合と目標値で内容が増える場合の2種類があるが、正確に表現できているのは要求技能が違う場合のみ」への対応。ルール正本は [Scenario_Progress](../01_Wiki/Game_Rules/Scenario_Progress.md)「情報項目の構造」。

#### メカニクス監査

- **内容ブロック＝調査の枝**という既存構造の読みを確定させた。枝は自分の使用技能（技能の集合＋共通の目標値の行）を持ち、要求技能が違う情報はここで分かれる。これは 2026-08-09 の「1行＝技能の集合＋共通の目標値」と矛盾しない——枝の中の**行**は同じ内容への複数の入口、枝の**間**は別の内容。
- **段（`content.tiers`）＝同じ枝のまま目標値が上がると積み増される内容**。従来はこれを「同じ技能を書いた別の枝」で代用するしかなく、(1) 並列（枝）と累積（段）が同じ見た目になる、(2) 技能集合を段の数だけ書き直す、(3) 順序と累積が機械から読めない、という3点で不正確だった。
- **段ごとに要求技能は変わらない**（2026-08-12 ユーザー指摘）。Code は設計確認の際に「段ごとに技能が変わり得るか」を問うたが、**枝の同一性は「その技能で調べていること」それ自体が与える**ので、技能が変われば定義上それは別の枝＝別行であり、問い自体が案の前提を崩していた。**撤回**し、段は目標値と本文だけを持つ形に確定。
- **段の目標値は達成値の絶対値と比べる**（Code 提案・ユーザー承認）。枝が「〈情報：ストリート〉8／〈情報：ネットワーク〉10」のように入口ごとに違う目標値を持つとき、段の「12」が誰に対する12かが問題になる。入口ごとの難易度差を段へ伝播させる案（8+4／10+4）は原作の書き方に無い発明なので採らず、どの入口から入ったかを問わない絶対値とした。
- **達成値から段を自動開示する解決はフェーズ16（情報収集判定）の領分**。本作業は台本表現・開示トグル・送信への反映まで。

#### 設計レビュー

- 純ロジックを3本に切り出して TDD **+14＝2057 green**:
  - `infoTiers(content)` — 目標値の昇順で読む。**並べ替えは読み出し時だけ**で保存順は書き換えない（トグルや目標値変更のたびに保存データの並びが動くのを避ける）。目標値未入力の段は末尾（入力するまで足した位置に留まる）。同値は保存順を保つ（安定ソート）。
  - `infoTierLabel(tier)` — パネル行とチャット見出しで同じ言い回し「さらに目標値 N」を使う。
  - `toggleInfoDisclosure(content, tierId)` — **開示の累積を構造で保つ**: 段を開けば入口本文と下位段も開き、段を閉じれば上位段も閉じ、入口を閉じれば全段が閉じる。複製を返し保存順は保つ。
- **手動並び替えを付けない**（表・リスト系の TNX 標準の例外）。段の順序は目標値が決めるもので、手で入れ替えられると意味が壊れるため。
- シートは既存の意匠に載せた: 段のヘッダー＝`tnx-editor-section__header`（ラベル「さらに目標値」＋スピナー＋削除を**行の幅で分け合い**、編集トグルだけ右端へ `margin-left: auto`）。段の本文は `prose-mirror` の高さ 150px（**Foundry コアの `min-height: 150px` が下限**で 120px は効かないことを実測して確認）。段の一覧は左罫線＋字下げで「別の枝ではなく本文の続き」を示す。
- `_onInfoItemChange` は `tierId` 分岐を**技能行より先**に置く（段と技能行が同じ `name="tn"` を使うため）。
- パネルは内容行の下に段の行を入れ子（`scp-info-tier`・字下げ 36px）で並べ、段ごとに開示トグル。ラベルは~250px 幅で省略されないことを実測。
- **既存データの移行はしない**。すでに「同じ技能で目標値違いの内容ブロック」で書かれた台本は並列のまま残り、段へ移すかはユーザーの手作業（正本データを黙って書き換えない）。
- **実機確認（隔離 Foundry）**: 保存順 16→12 の段がシートで 12→16 と昇順表示され保存順は不変／「さらに判明する内容を追加」で段が増える／スピナーで目標値が保存される／段の本文が段へ保存される（`tierId` 分岐）／パネルで段12を開くと入口も開き段16は閉じたまま・段12を閉じると段16も閉じ・入口を閉じると全段が閉じる。
- 検証: テスト 2057 green・eslint 0 error（警告15は既存）・stylelint clean。

#### 課題・申し送り

- **チャットカードには手を入れていない**（2026-08-12 ユーザー指示「現状では全く話にならないので後でまとめて整理します。中途半端に触らないでください」）。段は既存書式に沿って積むだけ。作業中に見つけた**本文の二重 `<p>`**（テキストエディタの HTML を `<p>` で包み直すため `<p><p>…</p></p>` になり、カードに空段落が挟まる。段だけでなく入口本文も同じ）は、一度直したが上記指示を受けて**差し戻した**。まとめて整理する際の対象。

---

### 2026-08-12 スタイル選択プルダウンの並びを正規順に固定

**2026-08-12** / **関連実装**（`identification.mjs`＝`STYLE_SORT_KEYS`/`styleSortPosition`・`skill-dictionary.mjs`＝`loadSkillEntries` の辞典別ソート／カスケードのスタイル段・tests） / **ステータス**: 実装済・実機確認は後続と一括

ユーザー指示（識別キーで33スタイルの並びを提示）。辞典のスタイルを選ぶプルダウンは、システム内のどこでもこの順で表示する。

#### メカニクス監査

- 並びの正本はユーザー提示の識別キー列（カブキ→…→ニューロ→コモン→ヒルコ→…→ウツワ）。原作の掲載順であり、Code が名前や五十音から導けるものではないので**リストとして固定**する（`kabuto-wari` のハイフン表記もそのまま＝別表記は一致させない）。
- 2026-07-19 の「技能を選択するプルダウンすべてに正規ソート順」と同じ考え方をスタイルへ広げたもの。技能の正規順（`GENERAL_SKILL_SORT_PREFIXES`＝プレフィックス一致）とは**別の軸**なので、関数も定数も分けた。

#### 設計レビュー

- **中央適用**: `loadSkillEntries` の基底ソートを辞典別にし、スタイル辞典のときだけ `styleSortPosition` を使う。これで `loadSkillChoices([STYLE_PACK])` を通す3経路——スタイル技能シートの「スタイル」欄・アウトフィットの「専用」・アクトシートのハンドアウト「指定スタイル」——が一括で正規順になる（各シートに並べ替えを書かない）。
- **カスケードのスタイル段だけは別経路**（`buildSkillCascadeSteps`）。ここはスタイル**技能**の `style` フィールドから拾うため辞典の並びに乗らない。同じ位置関数で明示的にソートした。
- リストに無いキーは `Infinity`＝末尾へ名前(ja)順（自作スタイル等が混じっても落ちない）。
- 検証: テスト **2063 green**（+6: `styleSortPosition` の位置/ハイフン/未知キー/ソート検算・カスケードのスタイル段の並び）・eslint 0 error（警告15は既存）。CSS 変更なし。

### 2026-08-12 フェーズ14-8 技能の源を辞典＋ワールド直下へ（収集層の一般化・uuid のバグ修正）

**2026-08-12** / **関連実装** `96a4be2`（`skill-dictionary.mjs`・tests） / **ステータス**: 実装済・隔離 Foundry で実機確認済

前項（コネのみワールド直下・`d11f629`）の切り分けが誤りだったことをユーザー指摘で確認し、技能全体へ一般化した。

#### メカニクス監査

- **原則＝アイテムの置き場所で選べる技能の集合が変わらない**（2026-08-12 ユーザー指示）。Code は前項で「アクト限定だから辞典に置きたくない」をコネ固有の理由として扱い、それを根拠に他を**除外**した。これは「コネでは特に必要」の理由であって「他では不要」の理由ではない。**同じ原則は 2026-07-18 に既に確定していた**——「辞典＝同パック限定／ワールド直下＝`game.items`」の区別を撤去したときの裁定（`tnx-usage-sheet.mjs` の注記「プルダウンが辞典を参照する以上、ワールド直下からも辞典を参照できてしかるべき」）。Code はその逆向きを同じ原則の話と気づかず個別案件として処理した。**教訓: 既存裁定の原則が向きを変えて再来していないかを、個別対応に入る前に確認する。**
- **範囲は技能だけ**（2026-08-12 ユーザー裁定）。「実際にワールド直下に置く可能性があるのは恐らくコネだけだが、万が一コネ以外の技能を置いたときのために」＝**実運用の見込みではなく破綻しない形を採る**という判断。スタイル辞典・組織辞典・アウトフィット辞典など**技能でない辞典には源を置かない**。
- **FS判定・情報項目・登場判定の候補は一般技能のまま**（2026-08-12 ユーザー裁定・ルール上の理由）: それらは「情報収集ができる」「登場判定ができる」「特定の FS 判定が使える」というスタイル技能側の効果を**代用判定で受ける仕組み**であり、シナリオ側がスタイル技能を要求技能に指定することはない。今後スタイル技能を使う特例機能を設ける可能性はあるが、その場合も要求技能にはならない。
- **同じ識別キーは辞典優先**（前項と同じ規則を全技能へ）。

#### 設計レビュー

- **バグ修正が主眼**（ユーザー指示「明確にバグだと感じるので直してください。例え、実際にはワールド直下に技能アイテムを作ることがないとしても」）: `loadContactSkillIndex` が**収集層の返り値を全て辞典項目と見なして** `pack.getUuid(e.id)` で uuid を組み立てていた。収集層がワールド項目を返すようになると、ワールド項目に**存在しない辞典 uuid** を作る。組み立てを収集層へ移し、エントリが自分の uuid を持つ形にした（辞典＝`pack.getUuid`・ワールド＝`item.uuid`）。**発現しない潜在的欠陥でも構造として直す**というのがユーザーの判断。
- **収集層は2つ**（`loadSkillEntries` と `loadDictionarySkillItems`）で、両方に源を足せば上に乗る全て——選択肢（`loadGroupedGeneralSkillChoices`・`loadSkillChoices`・`loadOnomasticChoices`）・逆引き（`loadGeneralSkillNameByKey`）・カスケード（`loadCascadeData`）・用途タイプ索引・用途のベース技能候補——が**呼び出し側を変えずに**広がる。呼び出し側を個別に直す案は採らない。
- 振り分け: 一般＝`type==="generalSkill"`／スタイル＝`type==="styleSkill"` かつ `special.works.value !== true`／ワークス＝同 `=== true`（スタイル技能とワークス専用技能は同じ型で、`works.value` で分かれる）。
- **キャッシュの分離**: `_cache` は**辞典分だけ**を保持し（`loadPackSkillEntries`）、ワールド分は都度読みして合成する。ワールドはライブなので、作った直後に候補へ出る。辞典に追加した技能がリロードまで出ないのは従来どおり（既存挙動・触っていない）。
- 合成は純関数 `mergeSkillEntries(packEntries, worldEntries, position)`（辞典優先・正規位置→名前順・識別キー無しは除外）。正規位置関数を引数に取るのは、スタイル辞典だけ `styleSortPosition` を使うため。テスト +6＝**2069 green**。
- `loadContactSkillIndex` は「一般技能から contact を絞るだけ」に縮小（コネの挙動・`mergeContactEntries` とそのテストは不変）。**コネの UI・仕様には触れていない**（2026-08-12 ユーザー指示＝アクトコネは HO からの自動生成を検討中のため保留）。
- **実機確認（隔離 Foundry）**: 3辞典それぞれにワールド項目が出る（一般＝社会/コネ群・スタイル＝カブキ群・ワークス＝組織群）／同キー `medicine` は辞典の「医療」が勝つ／識別キー無しは除外／**uuid が出所どおりで `fromUuid` が実体を返す**（ワールド＝`Item.…`・辞典＝`Compendium.…`）／スタイル辞典（技能でない）は全て辞典のまま／グループ化選択肢・逆引き・カスケードの群・`loadDictionarySkillItems` まで広がり `medicine` の重複が1件に畳まれる。
- eslint 0 error（警告15は既存）。CSS 未変更。

#### 課題・申し送り

- **ハンドアウトのコネ表示が「コネ: 〈コネ：名前〉」と二重になる**（2026-08-12 ユーザー指摘）。`buildHandoutMessage` の見出し「コネ:」と辞典名の接頭「コネ：」が重なる。アクトシートの欄も同じ（ラベル「コネ」＋選択肢「コネ：〜」）。**Code は前項の実機確認でこの文字列を出力していながら見落とした**（「現在名で出る」とだけ報告）。→ アクトコネは**ハンドアウトからの自動生成**を検討中のため、表示形式ごと改めて指示を受ける（今回は触らない）。
- ワークス専用技能の組織キーが組織辞典に無い場合、カスケードの群ラベルが生キーになる（`data.orgNames?.[oKey] ?? oKey`）。**従来からの挙動**で辞典項目でも同じだが、ワールド項目のほうが起こりやすい。無関係な修正を混ぜないため本作業では触っていない。
- `loadWorldSkillEntries` / `loadDictionarySkillItems` はワールドの技能アイテムに `toObject()` を呼ぶ（型で絞った後）。ワールド直下の技能は少数の想定なので都度読みで問題ないが、多数置かれる運用になったら計測する。

### 2026-08-12 フェーズ14-8 判定要求の複数指定＋スタイル/ワークス技能対応

**2026-08-12** / **関連実装** `720d971`（`tnx-rl-request-app.mjs`＋`rl-request-app.hbs`・`request-presets.mjs`・`identification.mjs`・`tnx-scenario-sheet.mjs`＋`scenario-sheet.hbs`・`tnx2.css`・tests） / **ステータス**: 実装済・隔離 Foundry で実機確認済

#### メカニクス監査

- **判定要求の指定技能は複数可**（2026-08-12 ユーザー要望）。意味は既存カードのまま「**このうちいずれかの技能で応じてよい**」——新しい意味づけは作らない（FS判定の支援判定要求が 2026-07-21 から同じ形で動いている）。
- **スタイル技能・ワークス専用技能も指定できる**（同日要望）。**ただし FS判定・情報項目・登場判定の候補は一般技能のまま**（同日裁定・ルール上の理由）: それらは「情報収集ができる」「登場判定ができる」「特定の FS 判定が使える」というスタイル技能側の効果を**代用判定で受ける仕組み**であり、シナリオが要求技能にスタイル技能を指定することはない。
- **土台の半分は既に出来ていた**（Code 調査 2026-08-12）: カードは `identificationKeys`（配列）を保持し、PL 側も 2 件以上なら `_promptDesignatedSkill` で応じる技能を先に選ばせ、以降は単数と同じ経路を通る。新規に作ったのは UI と、応答側に残っていた一般技能前提の解消。

#### 設計レビュー

- **指定方法＝プルダウン＋ドロップの二本立て**（2026-08-12 ユーザー案・Code 案を却下して採用）。一般技能は既存のグループ化プルダウン、スタイル技能・ワークス専用技能は**ドロップゾーン**（一般技能も落とせる）。**どちらから足しても同じチップ列に積まれる**ので、辞典をまたいだ指定が自然にできる。
    - **却下: 技能カスケード**（Code 初案）。カスケードは「今どの辞典を見ているか」という**モード**を持つ道具で、1 件追加するたびに段を辿り直す。既存のタグ入力の追加口が全て単一プルダウンなのも同じ理由で、カスケードは「1 つの欄を確定する」場所（用途シート・スタイル技能シート）にしか使われていない。**Code は「多段絞り込みの確立形式」だけを見て、「繰り返し追加する場所に置ける形式か」を検算していなかった。**
    - **却下: 検索コンボボックス**（Code 第2案・`combobox.mjs` の流用）。`<datalist>` は `<optgroup>` を持てないため、グループ表示には部品側に仕様外の細工が要る。**ユーザー裁定「無理やりオプショングループを再現してバグる方が怖い」** ——共有部品を壊す危険が、検索で得られる利便を上回る。一般技能プルダウンは従来のまま（検索なし）。
    - 33 スタイル分の optgroup を組む機構が丸ごと不要になる点も D&D の利点（`STYLE_SORT_KEYS` は 33 件）。コストは「持ち出し元を開いておく必要がある」こと——判定要求でスタイル技能を指定する頻度は高くない想定で釣り合うと判断。頻度が高いと分かれば後から検索プルダウンを足せる。
- **識別キーを持たないアイテムは指定できない**（コネと同じ構造的制約）。落とされたら名前つきで理由を出して弾く。技能でないアイテムも同様。
- **指定技能が空＝「その他」の要求**（自由入力の技能名＋スート選択）。旧セレクトの空選択肢と同じ意味を、チップ 0 件で表す（状態が 1 つ減った）。
- **応答側の一般技能前提を 3 箇所とも解消**: 逆引き（`requestSkillLabel`）を全技能辞典へ——スタイル技能のキーが**生キーで表示**されていた。ついでに逆引き不能を生キーでなく「（参照切れ）」に（識別キーは保存する参照であって表示するものではない）。所持技能の照合（`findItemByIdentificationKey`）を `generalSkill` 固定から技能 2 種へ——**持っているスタイル技能を見つけられず代用判定に落ちていた**。`type` が配列を取れるよう加算的に拡張（既存呼び出しは不変）。
- **プリセットの複数化**: `identificationKeys` を新設し、旧単数 `identificationKey` は 1 件として読む純関数 `presetSkillKeys`（**一括書き換えはしない**——技能を足し引きした行から順に配列へ移り、その時点で旧単数を落とす）。アクトシートの行も同じプルダウン＋ドロップに。
- **CSS は共通部品に 1 つ追加**: `.tnx-tag-add-row`（追加の手立てが 2 つ並ぶとき同じ行で幅を分け合う）。判定要求ダイアログとアクトシートのプリセット行の両方で使うため、場当たりのスコープ CSS にせず共通部品側へ置いた。
- **実装中に見つけて直した自分の不具合**: ドロップハンドラで `event.currentTarget` を `await` の後に読んでいた（ディスパッチ終了で null になる）ため、プリセット行へのドロップが黙って何もしなかった。**隔離 Foundry の実操作で発覚**（純ロジックのテストでは出ない）。`_onDropAcquireZone` は先に読んでおり、既存の正しい順序に倣っていなかった。
- 検証: テスト **+4＝2073 green**・eslint 0 error（警告15は既存）・stylelint clean。**隔離 Foundry**: 一般2件＋スタイル＋ワークスの混在チップ／識別キー無し・技能でないドロップの拒否と文言／空でその他セクションが戻る／送信カードのラベル「〈白兵〉・〈元力：炎〉」とキー配列／複数要求で「指定技能を選択」が出て**所持スタイル技能が代用判定に落ちない**こと／プリセットの旧単数読み替え・追加・削除・読み込み元での復元。レイアウトはスクリーンショットで確認（チップの折り返し・追加行の幅配分）。

#### 課題・申し送り

- 一般技能プルダウンに検索は付けていない（上記の裁定）。固有名詞技能が埋まって長くなり実際に困ったら、そのとき改めて手段を選ぶ。

### 2026-08-12 フェーズ14-8 是正 判定要求の技能名自由入力を廃止（ドロップが自由入力に相当）

**2026-08-12** / **関連実装** `fcc5b74`（`tnx-rl-request-app.mjs`＋`rl-request-app.hbs`・`request-presets.mjs`・`tnx2.css`・tests） / **ステータス**: 是正済・隔離 Foundry で実機確認済

#### メカニクス監査

- **技能名の自由入力欄は不要**（2026-08-12 ユーザー裁定）。**ドラッグ＆ドロップ自体が自由入力に相当する**——辞典に無い技能を要求したければ、その場でアイテムを作って落とせばよい。自由入力欄が存在した理由は「プルダウンに無い技能を指せない」ことだったので、ドロップを入れた時点で存在理由が消えていた。
    - **Code の理由付けが誤っていた**（ユーザー指摘「論理的思考ができていませんよ」）: Code は「応答側が技能アイテムを解決できないので**能力値判定と役割が重なる**」を廃止理由に挙げた。能力値判定は別のメカニクスであり、技能判定の代わりにはならない。**結論が同じでも根拠が違えば別の判断**であり、誤った等価で正当化した。正しい等価は「自由入力 ⇔ ドロップ」。
- 技能判定は**指定技能が必須**になった（空なら警告して送信しない）。スートは常に PL 側の技能アイテムから決まる（RL が選ぶ経路が消えた）。
- **応答側の「識別キーの無い要求」は残す**: FS判定は技能未設定の行からキー無しの要求を出しうる（`buildSupportRequest` が空配列を返す経路）ほか、過去のチャットカードも動く必要がある。廃止したのは**RL が新しく自由入力の要求を作る口**だけ。

#### 設計レビュー

- 削除物: `.other-skill-section`（技能名入力＋スート4つ）・`SUIT_OPTIONS`・`customSkillName`・skillCheck の validSuits 分岐・対応 CSS 2 ルール。プリセットからも `customSkillName`/`validSuits` を落とした（プリセット編集 UI にはもともと欄が無く、読む先が消えたため）。
- **`closeOnSubmit: true` → `false` に是正し、送信できたときだけ明示的に閉じる**。検証で弾いたときにダイアログごと閉じ、**入力が全部消えていた**。指定技能が必須になって普通に踏む経路になったため表面化した（対象未設定の警告も同じ問題を抱えていたが、旧実装では技能セレクトの既定値が先頭の一般技能だったので踏みにくかった）。
- 検証: テスト **+1＝2074 green**・eslint 0 error（警告15は既存）・stylelint clean。**隔離 Foundry**: 欄とスートが消えたこと（ラベルが 判定種別/技能/能力値/目標値/説明 の5つ）／指定なしの送信が「指定技能を1つ以上追加してください。」で止まり**ダイアログと入力（説明文）が残る**こと／対象未設定でもチップごと残ること／両方揃えば送れて今度は閉じること／能力値判定へ切り替えても壊れないこと。

### 2026-08-12 フェーズ14-8 アクトコネをHO送信カードのボタンで受け取る（コネ欄は自由入力へ）

**2026-08-12** / **関連実装** `bb7bca4`（`handout-contact.mjs`＋`handout-card.hbs`（新規）・`session-logic.mjs`・`session-state.mjs`・`tnx-scenario-panel.mjs`・`tnx-scenario-sheet.mjs`＋`scenario-sheet.hbs`・`tnx.mjs`・`tnx2.css`・tests） / **ステータス**: 実装済・隔離 Foundry で実機確認済

#### メカニクス監査

- **コネ欄＝相手の名前の自由入力**（2026-08-12 ユーザー裁定で辞典参照から差し戻し）。**相手は NPC とは限らない**——正本 [Scenario_Progress](../01_Wiki/Game_Rules/Scenario_Progress.md)「プレアクト」に「ハンドアウト指定 NPC とのコネと、キャスト間コネクションの2種」とある（2026-08-13 ユーザー指摘で入力欄のプレースホルダーを「NPC 名」→「コネの相手」に是正。**Code は同セッションでこの行を読んでいながら NPC 前提の文言を書いた**）。入力するのは素の名前で「コネ：」は含まない——生成される技能の名前が「コネ：<入力値>」になる。これにより旧「コネ: 〈コネ：名前〉」の二重表記も解消した。
- **受け取りは HO 送信カード最下部のボタン**（同日指示）。押すと **HO の対象ユーザーのプレイヤーキャラクター**（`user.character`）にコネ技能を生成する。**アクト開始時の自動配布は廃止**（自由入力では識別キー参照が成立しないため。受け取りをボタン一本に集約）。
- **プレイヤーキャラクター未設定なら、HO の指定スタイルを持つキャストを新規作成し、プレイヤーキャラクター登録まで自動で行う**。**自動作成の目的は「アクトコネがインポートされたアクターを用意すること」**（ユーザー明言）——だから指定スタイルが無い（共通／自由記述）ときは**スタイルを付けずに作る**（作成そのものが目的なので中止しない）。**キースタイルは自動で立てない**（同日裁定）。
- **生成する技能**（ユーザー指定）: 固有名詞技能（`generalSkillCategory="onomasticSkill"`）／アクション技能（`isAction`）／報酬点使用可能（`usesBounty`）／**初期取得の社会・コネ**（`onomasticSkill.isInitial`）／**アクト限定**（`isActLimited`）。アクト限定なので既存の `endAct` の掃除（`_deleteActLimitedSkills`）でそのまま消える。
- **識別キーは名前と無関係な一意 ID**（`contact_` ＋ `randomID()`・ユーザー指示「コネ名に関連した識別キーを生成するとバグる」）。`contact_` の接頭だけ付けるのは**コネ小分類の判定がこのプレフィックスに依存している**ため。アクトを越えて残したいときはユーザーがキーを付け直す運用。
- **押せる人は絞らない**（同日裁定）。他ユーザーが押すのは正規の手順ではないだけで、**わざわざブロックする必要はない**——アクター作成はワールド権限、他人のキャストへの書き込みは所有権で Foundry 側が弾くので、失敗の理由を警告に出すだけにした。Code は当初ソケット委譲か RL 限定かを設計判断として持ち込もうとしたが、不要だった。
- **既存データは考えない**（同日裁定「既存のコネを設定済みの行は削除して作り直します」）＝旧配列 `actConnections` の読み替えと旧 `connections` のフォールバックを撤去した。

#### 設計レビュー

- 二重押しはカードのフラグ（`handoutContact.granted`）で止め、「コネを取得済み（対象名）」に差し替える（報酬点カードの `received` と同方式・`applyMessagePatch` で非作者は GM へ委譲）。加えて**同名のコネを既に持っていたら作らない**ので、カードを跨いだ重複も起きない。
- カードのフラグには**送った時点の値を写す**（`contactName`/`userId`/`styleKey`）。台本を後で編集してもカードは送った時点の記録として成立する。
- チャットカードは専用テンプレート（`handout-card.hbs`）へ。**意匠は判定要求・報酬点カードと同じ `.tnx-check-request`（`cr-req-header` ＋ `cr-req-body`）を踏襲**し、差分だけ `.tnx-handout-card` にスコープした CSS で足した（判定要求はタイトルが固定ラベルなので小さいが、ハンドアウトはタイトルがそのハンドアウトの名前なので主役として出す）。担当・スタイル・コネ・推奨スートはラベル幅を揃えた行、本文と PS は罫線で区切る。
- 純関数 `buildHandoutCardData`（素の HTML を組む `buildHandoutMessage` を置き換え）と `buildContactSkillData` / `isContactGranted` を切り出して TDD **+5＝2078 green**。
- **ボタンの文言から相手の名前を外した**（実機のスクリーンショットで判明）: 「コネ：<長い名前> を受け取る」は 272px の実チャット幅で2行に折り返した。名前はすぐ上のコネ行に出ているので「コネを受け取る」に。
- **実機確認（隔離 Foundry）**: 欄が input になり自由入力が保存される／カードの組版（実幅 272px でボタンが1行）／PC ありは既存キャストへ生成／PC 無しはキャスト自動作成＋スタイル付与（`isKey=false`・level 1）＋OWNER 付与＋`user.character` 登録まで通る／二重押しで増えず「取得済み」表示になる／生成技能の5フラグと一意キー。**キャスト新規作成時に既存の初期技能 14 件が自動インポートされる**（既存の作成フックがそのまま効く＝自動作成したキャストがすぐ使える）。
- 検証: テスト 2078 green・eslint 0 error（警告15は既存）・stylelint clean。

### 2026-08-13 フェーズ14-8 是正 生成したアクトコネが一般技能リストの並び順を無視する

**2026-08-13** / **関連実装** `caee24e`（`identification.mjs`・`handout-contact.mjs`・`tnx-character-sheet-base.mjs`・tests） / **ステータス**: 是正済・隔離 Foundry で実機確認済

#### 原因

キャストシートの一般技能リストは `item.sort` で並ぶ（`_prepareSkillsData`）。前項（`bb7bca4`）のコネ生成は `sort` を振らずに `createEmbeddedDocuments` していたため、Foundry の既定（末尾に付ける）が効いて正規ソート順（`GENERAL_SKILL_SORT_PREFIXES`）を無視した。

**同じ不具合は 2026-07-10 に一度直っている**——辞典ドロップが辞典側の sort を持ち込んで並びを崩していた件で、＋ボタンからの作成と同じ `_calcInsertSortValue` に揃えた。**Code は3つ目の生成経路を足すときにその前例を確認せず、同じ穴に落ちた。教訓: アイテムを作る経路を新設するときは、既存の生成経路が作成データに何を足しているかを先に読む**（`sort` のように「振らないと既定が効く」項目は、抜けても例外にならず表示だけが崩れる）。

#### 設計判断

- **挿入位置の計算を `identification.mjs` へ `calcSkillInsertSort` として切り出した**。シートの静的メソッド `_calcInsertSortValue` はその委譲になる（挙動は不変・既存2経路は無改修）。生成経路が3つになった時点で、シートクラスの静的メソッドはシート外から呼ぶ置き場として不適切になったため（§4.3 の「理解と合意なしに書き換えない」に触れないよう、**実体の移動と委譲だけ**にして計算そのものは1行も変えていない）。
- 純関数なのでテストを付けた（+5＝**2083 green**）: 後ろの技能があれば手前に挿す／無ければ末尾（最大 sort＋100000）／識別キー無しは末尾／空配列・null／挿入値で並べ替えると正規順になる。

#### 検証

- **隔離 Foundry**: 既存コネ（`contact_seed`）と識別キー無しの自作技能がある状態で新しいコネを受け取り、**「コネ群の末尾かつ自作技能の手前」**に入ることを確認（従来は最後尾）。コネしか無いケースでは社会の直後＝末尾に入る（`contact` が正規順の最後のため正しい）。シートの静的メソッド経由でも同じ値が出ることも確認。
- テスト 2083 green・eslint 0 error（警告15は既存）。

### 2026-08-13 フェーズ14-8 アクトコネの3モード指定＋アクト終了の後始末を確認制へ

**2026-08-13** / **関連実装** `97d4225`（`session-logic.mjs`・`handout-contact.mjs`・`session-state.mjs`・`tnx-scenario-sheet.mjs`＋`scenario-sheet.hbs`・`tnx-scenario-panel.mjs`・`skill-dictionary.mjs`・`tnx2.css`・tests） / **ステータス**: 実装済・隔離 Foundry で実機確認済

#### メカニクス監査

- **コネの指定方法を3つにする**（2026-08-13 ユーザー指示）: **PC**＝ハンドアウトから選ぶ／**NPC**＝辞典のコネ技能をドロップして**そのままキャストシートへ複製**／**自由記述**＝名前を打つ。既存 NPC を指せる口が無かったのが動機。**PC と自由記述はプロセスが共通**（名前の出所が違うだけで技能を組み立てる）で、**NPC だけが既存アイテムの複製**。
- **ハンドアウト由来のアクトコネは、ルール上の「キャスト間コネクション」とは別物**（2026-08-13 ユーザー指摘）。PC モードは「HO の指定する相手がたまたま PC である」だけで、相互に張る仕組みではない。正本 Scenario_Progress の「ハンドアウト指定 NPC とのコネと、キャスト間コネクションの2種」のうち、**実装しているのは前者だけ**——後者は別の仕組みとして未実装。**Code は前回この2つを混同し、PC モード＝キャスト間コネクションだと仮定して「相互にしない」という但し書きを書いた**（前提そのものが誤り）。
- **3モードとも `isActLimited` を立てる**（ユーザー指示「これで追加されるコネには全て『アクト限定』を立ててください」）。NPC の「そのまま複製」はデータの出所の話であって、アクト限定は複製後に別途立てる。他のフラグ（報酬点使用可能・初期取得等）は**辞典アイテムに書かれたまま**にする。
- **アクト終了の後始末を確認制に**（同日指示）: 無言の全削除をやめ、全キャストのアクト限定技能を一覧するダイアログを出す。**既定は全部オフ＝削除**（アクト限定はそのアクト限りが原則）。**チェックしたものだけ残し、残したものは `isActLimited` を落とす**——落とさないとアクトのたびに同じものを聞かれ続けるし、「わざわざ編集しなくても維持できる」という要件がそこまで含む（Code の読みをユーザーに提示して確認済み）。「後で」（キャンセル/閉じる）は**全部維持**——誤操作で消えるより残るほうが安全。

#### 設計レビュー

- **モードごとに値の置き場を分けた**（`actConnection` / `actConnectionHandoutId` / `actConnectionUuid`）。1つのフィールドを使い回すと、モードを切り替えたときに前のモードの入力が壊れる。PC・NPC は**参照で持ち名前は都度解決**（キャスト名・辞典アイテム名の変更に追随・名前をキャッシュしない既存規約）。
- **PC の候補は `_buildScenePlayerHandouts` をそのまま流用**（共通ハンドアウトを除外し「①スタイル名（キャスト名）」のラベルを作る処理が既にあった）。自分自身の行だけ追加で除外する。
- **NPC の表示名は「コネ：」の接頭を落とす**——辞典のコネ技能名は「コネ：<相手>」なので、カードの「コネ」見出しと合わさると「コネ: コネ：キース」になる（8/12 に一度出した二重表記と同じ穴）。`formatGroupedSkillNames` が使っていた `stripSkillCategory` を export して共用。**技能そのものの名前は変えない**（複製なので）。
- 純関数を切り出して TDD **+11＝2094 green**: `contactType`/`CONTACT_TYPES`（既定は自由記述・不正値は落とす）・`normalizeHandoutRow` のモード別フィールド・`resolveHandoutContact`（3モードの解決・PC でキャスト未設定なら空・NPC の接頭落とし・参照切れ）・`planActLimitedCleanup`（削除/維持の振り分け・アクターごとに束ねる）。
- **実機確認（隔離 Foundry）**: 3モードの欄が**指定方法・入力欄・推奨スートで1行に収まる**こと（ユーザー指摘「入力欄がかなり長い」→ 幅を詰めた）／PC 候補が自分の行を除くこと／NPC のドロップ設定と「名前＋✕」表示／送信カードのフラグ（type/contactName/itemUuid）／受け取りで PC=相手のキャスト名・NPC=辞典コピー（キーは辞典のまま）・自由記述=入力値の技能ができ、**いずれもアクト限定が立つ**こと／後始末ダイアログの一覧・既定オフ・維持したものだけ残り `isActLimited` が落ちること。ダイアログのキャスト名列が省略されていたので幅を 8em→10em に広げ、全文は `title` で出すようにした（実測して是正）。
- 検証: テスト 2094 green・eslint 0 error（警告15は既存）・stylelint clean。

#### 課題・申し送り

- **キャスト間コネクション（ルール上のもう1種）は未実装**。プレアクトでキャスト同士が張るコネであり、ハンドアウトの器とは別の導線が要る。必要になった時点で設計する。

### 2026-08-14 フェーズ14-8 是正 後始末を経験点配布の後へ・維持時に初期取得フラグも落とす

**2026-08-14** / **関連実装** `da998d4`（`session-state.mjs`・`tnx-exp-award-app.mjs`・`tnx-scenario-panel.mjs`） / **ステータス**: 是正済・隔離 Foundry で実機確認済

#### メカニクス監査

- **後始末は経験点配布の後**（2026-08-14 ユーザー指示）。**維持したコネはそのアクトの経験点で買う扱いになる**ため、配布前に聞くと払う原資が無い（経験点は所持量を超えて消費できない＝KI-040）。処理の順序としても配布が先が正しい、というのがユーザーの判断。
- **維持したものは `onomasticSkill.isInitial` も落とす**（同日指摘）。**初期取得の社会・コネが立ったままだと経験点を消費せず、無料でコネが手に入ってしまう**。前回 Code は `isActLimited` だけを落としていた——「アクトのたびに聞かれ続けない」という運用面だけを見て、**フラグがルール上持っている意味（経験点の消費免除）を見ていなかった**。
- **経験点の最大値超過をブロックしている件は KI-040 として起票**（ユーザー「ブロックすべきではない」「一気に直す必要はない」）。今回の順序変更は回避であって、制限そのものは残っている。

#### 設計レビュー

- `endAct` から後始末を外し、**ポストアクトの流れを持つパネル側**（`_onEndAct`）が配布アプリの後に呼ぶ形へ（`_deleteActLimitedSkills` → export した `promptActLimitedCleanup`）。配布アプリは `_onClose` で `onFinish` を**一度だけ**呼ぶので、確定・キャンセル・✕のどれで閉じても後始末に着地する（順序の保証を1箇所に置き、経路ごとの配線を増やさない）。
- **「後で」で全件削除されていたのを是正**（実機で発覚）。`DialogV2` は**コールバックが `null` を返すとボタンの `action` 文字列で解決する**ため、`keepKeys === null` では「後で」を拾えない。さらに**文字列はイテラブル**なので `new Set("cancel")` が作られ、維持キーが1つも一致せず全件削除に落ちていた。`Array.isArray(keepKeys)` で「確定が返した配列のときだけ処理する」に変更。
    - **教訓**: 「キャンセルは null」という前提はフレームワーク側の仕様に依存する。既存コード（`_promptDesignatedSkill`）が `res || null` と書いていたのは同じ罠を踏んだ跡で、**その書き方の理由を読み取れていれば避けられた**。返り値の型で分岐する（真偽やフォールバックに頼らない）。
- **実機確認（隔離 Foundry）**: アクト終了直後は配布アプリだけが出て後始末は出ない／配布の確定で閉じた後に後始末が出る／キャンセル・✕でも出る／4経路（後で・✕・確定(未チェック)・確定(チェック)）の結果が意図どおり（維持は `actLimited=false` `isInitial=false`・未チェックは削除・後で/✕は全部維持）。
- 検証: テスト 2094 green（純ロジックの変更なし）・eslint 0 error（警告15は既存）。

### 2026-08-14 経験点の超過ブロック撤廃（KI-040・フェーズ14 の枠外）

**2026-08-14** / **関連実装** `tnx-cast-sheet.mjs`・`tnx-character-sheet-base.mjs` / **ステータス**: 解消済・隔離 Foundry で実機確認済

#### メカニクス監査

- **経験点は所持量を超えて消費できる**（2026-08-14 ユーザー裁定）。卓の運用では前借り・後払いが起こりうるため、システムが止める筋合いではない（可否は卓が決める、という本システムの原則に合う）。超過は禁止事項ではなく**通常起こりうる状態**なので、警告は「気づかせる」ためだけのもの。ユーザー指示は「超えたときに一度だけ警告を出し、それ以降は特に何かする必要はありません」＝**繰り返さない・他に何も足さない**。
- 経験点は 2026-07-20 裁定で「没収・減算が存在しない累積の記録」であり、**残高ではない**。その性格からしても「残高が足りないから買えない」というモデル自体が原作に無かった。今回の撤廃はその整合を取るもの。
- 「超えた」の定義は `exp.value < 0`＝**消費経験点 > 総経験点**（`value = (total + 170) - realSpent`・`spent = realSpent - 170` なので `value < 0 ⇔ spent > total`）。シートのヘッダが出しているのも消費と総なので、警告の文言もその語彙で書いた（`exp.value` は非表示）。

#### 設計レビュー

- **ブロックは `_applySkillLevelExp` の1箇所だけだった**（起票時「未特定」）。能力値成長・アイテム取得・コネ維持は元からブロックしておらず、**同じ「超過」が経路によって止まったり止まらなかったりする**状態だった。
- 同メソッドは撤廃対象のゲートを支えるためだけに存在していた: (a) 持っていたコスト表 `_getSkillExpCost` は権威側 `_calcSingleItemCost` と値が食い違う（一般技能 10 対 実際 5／演出 2 対 実際 10 ほか）、(b) 直後の `system.exp.value` 直接減算は `updateItem` → `_recalcCastExp` の全量再計算に必ず上書きされる過渡値で、`exp.value` は画面にも出ない。よって**ゲートを外すとメソッドごと残骸**になる——オーバーライドを削除し、基底のフック（常に `true` を返すだけになる）と `if (!ok) return;` も併せて削除した。死んだ抽象を残さない。
- **警告は権威点（`_recalcCastExp`）に置いた**。ここが消費の確定する唯一の場所なので、全経路が同じ判定を通り、経路ごとに警告を配線する必要が無い。転移の判定は「書き込み前の `exp.value >= 0` かつ新しい値 `< 0`」——`_recalcCastExp` は毎回**全量再計算**なので、直前の値との比較だけで「超えた瞬間」が取れる。
    - 前述 (b) の直接減算を残したままだとこの判定は壊れる（先に負の値が書かれてしまい、再計算が「既に超過」と読む）。撤廃が判定の前提でもある。
    - `updateCastExp` の合流機構（KI-008）で 2 周目が走っても、1 周目で `exp.value` が負に落ちているため 2 周目は転移と見なさない＝**警告は1回**。
- **通知の宛先**は `actor.isOwner` のクライアント。再計算は全クライアントで走るため、無関係なプレイヤーに出さない絞り。Foundry はドキュメント更新を**ローカル適用後**にフックを呼ぶので、**操作した本人のクライアントは必ず「旧 `exp.value` ＋ 新レベル」を見る**＝転移を確実に検出する。GM クライアントは他クライアントの書き込みとの競合次第で出たり出なかったりする（情報通知のため実害なしと判断。決定的にするには操作者 `userId` を再計算まで通す配線が要り、「それ以降は特に何かする必要はありません」の指示に照らして見送った）。
- **ユニットテストは追加していない**。変更箇所はシートの静的メソッド内で、`ApplicationV2` 継承チェーン全体のモックが要り現行の `tests/setup.mjs`（DataModel フィールドのモックのみ）では読み込めない。判定自体は 2 項の比較のため、純関数化して 1 行を検証する価値も薄いと判断した。代わりに**実機で 4 段階の状態遷移を確認**（下記）。
- **実機確認（隔離 Foundry）**: 残 141 のキャストで医療 Lv1→16（コスト 150）。①レベルが実際に 16 になり（従来は巻き戻し）、`exp.value` = -9、ヘッダが 消費 50 / 総 41、警告はちょうど 1 回「キャスト: 消費経験点が総経験点を 9 点超えました。」②超過状態のまま射撃 Lv1→3 → 値は -29 に動くが警告は増えない。③医療を Lv1 に戻す（予算内へ復帰）→ 警告なし。④再び Lv16 → 新たな超過として再度警告（29 点）。警告トーストは 1 行に収まる。
- 検証: テスト 2094 green（純ロジックの変更なし）・eslint 0 error（警告15は既存）。

#### 課題・申し送り

- `_recalcCastExp` は全クライアントで走り、末尾で `actor.update` を呼ぶ（権限の無いクライアントでは失敗しうる）。今回の変更以前からの構造で、本件では通知の絞りに `actor.isOwner` を使うことで触れずに済ませた。整理するなら EXP 同期機構ごとの設計判断が要る。

### 2026-08-15 キャストシートの履歴をそのキャストのセッションだけにする

**2026-08-15** / **関連実装** `tnx-history-mixin.mjs`・`tnx-cast-sheet.mjs`・`tnx-exp-award-app.mjs`・`tnx-record-sheet.mjs`・`user-flag-schema.mjs`＋tests / **ステータス**: 実装済・隔離 Foundry で実機確認済

#### メカニクス監査

- **レコードシート＝プレイヤーの記録全体／キャストシート＝そのキャストを連れて行ったセッション**、という分担にする（2026-08-15 ユーザー指示）。全件表示は、独自アプリが無く「同じプレイヤーのキャスト同士で履歴を共有していた」頃の名残（ユーザー説明）。
- **総経験点は据え置き＝プレイヤーの全セッション分**。経験点はプレイヤーに付与されキャストは消費の単位（本ファイル冒頭の前提・`_recalcCastExp` は同期中なら User flag の total を読む）。あるキャストが別のキャストで稼いだ分を使えるのは正しいので、**表示だけを分ける**。結果としてキャストシートには「合計が総経験点に一致しない履歴」が並ぶ——仕様として承知のうえ（Code から提示し、ユーザーは表示の話として指示を継続）。
- **紐づけ無しの行はキャストシートに出さない**（ユーザー「当然キャストシートには出しません」）。既存データの一括推測による割り当ては行わない（根拠のない書き換えになるため）。

#### 設計レビュー

- **`origin` を流用しない**。既存の `origin` は「誰がこの行を作ったか」で、書き手ごとに値の種類が違った（キャストシート＝cast UUID／レコードシート＝**User の id**／**配布アプリ＝設定なし**）。しかも同期 OFF 時の由来分離（`separateHistoryByOrigin`）が使っている。**今の本流である配布アプリの行が全部落ちる**うえ、2つの意味が混ざるため、`castUuid`（そのセッションに連れて行ったキャスト）を別に持たせた。
- **情報は記帳の時点で既に手元にあった**——配布アプリは `user.character` を castName として表示済み（2026-08-09 裁定＝キャスト=ユーザーの割当キャラクター・cast 型のみ）。焼き込むだけで済み、セッション状態を辿る配線は要らなかった。
- **絞り込みは表示層だけ**（`_prepareHistoryForDisplay` の第2引数）。cast が合併を持つ配り方・User flag 権威・`exp.total` の集計はどれも不変＝EXP 同期機構に触れていない。
- **同期していないキャストには絞り込みをかけない**。`performUnsyncSeparation` が他キャスト由来を除去済みで `system.history` が元々そのキャスト固有のため、絞ると**既存の自分の行まで消える**。`ownerUserId && syncWithOwner` のときだけ絞る。
- **レコードシートにキャスト指定欄は足さなかった**。当初「手入力した行がどのキャストシートにも出ない穴が残る」として提案したが、**キャストシート自身の「行を追加」が既にそのキャストで焼く**（`_performHistoryUpdate` が User flag にも書くので両方に並ぶ）。口は既にあり、足すのは先回りの拡張だった。
- TDD **+9＝2103 green**: `_prepareHistoryForDisplay` の絞り込み（該当のみ・紐づけ無しは除外・絞り込み後も日付昇順・空/未定義）と `_calculateTotalExp` が絞り込みと無関係に全件を合計すること。
- **実機確認（隔離 Foundry）**: 同期キャストに User flag の8行が配られている状態で、シートには**そのキャストの1行だけ**が出る（別キャストの行・紐づけ無しの seeded 行・実データの既存5行はいずれも出ない）／レコードシートは8行すべて出る／**同期していないキャスト**は紐づけ無しの行がそのまま出る／配布アプリの確定で書かれた行が `castUuid` を持ち、そのキャストのシートに出る／キャストシートの「行を追加」を実際に押すと行が増え、`castUuid` が焼かれ、User flag 側にも同じ値で伝わる。
- 検証: テスト 2103 green・eslint 0 error（警告15は既存）。

#### 課題・申し送り

- **配布アプリの日付が UTC**（`new Date().toISOString().slice(0, 10)`）。JST の午前中に確定すると前日の日付で記帳される（実機確認中に 8/15 07:33 JST の記帳が `2026-08-14` になった）。本件とは別の既存不具合のため触っていない。

### 2026-08-15 経験点配布の記帳日付をローカル日付にする（KI-041）

**2026-08-15** / **関連実装** `exp-award-logic.mjs`・`tnx-exp-award-app.mjs`＋tests / **ステータス**: 解消済・隔離 Foundry で実機確認済

#### メカニクス監査

- **記帳は基本的に日付が変わる前に行われる**（2026-08-15 ユーザー言明）ため、通常はその日のローカル日付が正しい。**深夜は前日として扱う**——その時刻の記帳は前日から続いているアクトのものだから。
- 「深夜」の境界は 5 時とした（労働基準法の深夜業 22 時〜翌 5 時と同じ区切り）。**ルールでなく運用の区切り**のため Code が選び、定数 1 箇所（`LATE_NIGHT_CUTOFF_HOUR`）に置いて後から変えられるようにした。

#### 設計レビュー

- **アクト開始時刻を持ち回る案は採らなかった**。意味としては「アクトが行われた日」が最も正確だが、(a) セッション状態に開始時刻が無く、(b) `endAct` が配布アプリを開く前に状態を `DEFAULTS` へ戻すため、設定フィールドの追加＋2 層の配線が要る。さらに複数日にまたがるアクトでは開始日が実態とずれる。ユーザーの言明自体が「記帳の時刻」を基準にした規則だったため、時刻から決める純関数に寄せた。
- 純関数として `exp-award-logic.mjs` に置いた（配布アプリの純ロジックの既存の置き場・テスト済みモジュール）。`Date` の**ローカル getter** で組み立てる（`toISOString` は UTC を返すのが原因そのもの）。`setDate(getDate() - 1)` は月・年またぎを自前で計算しない。
- TDD **+9＝2112 green**: 日中／7 時台（不具合の再現ケース）／23:59／0 時台／4:59／5:00／月またぎ／年またぎ／0 埋め。
- **実機確認（隔離 Foundry）**: ローカル 2026-08-15 08:01 に配布を確定 → 旧実装が入れていた UTC 日付は `2026-08-14`、記帳されたのは `2026-08-15`。同じ関数で深夜（8/16 02:00）が `2026-08-15` を返すことも併せて確認。
- 検証: テスト 2112 green・eslint 0 error（警告15は既存）。

### 2026-08-15 フェーズ14-8 シナリオコントロールの送信カードを 4 種ともカード意匠に揃える

**2026-08-15** / **関連実装** `session-logic.mjs`・`tnx-scenario-panel.mjs`・`templates/chat/{scene-switch,text,info}-card.hbs`・`handout-card.hbs`・`tnx2.css` / **ステータス**: 実装済・隔離 Foundry で実機確認済

#### ルール解釈

体裁の是正であり、**卓に出す情報の中身は 1 つも足していない/落としていない**。ルール由来の表示規則は次のとおり据え置いた。

- **ルーラーシーンは「ルーラーシーン」とだけ出す**（2026-08-08 裁定）。「シーンプレイヤー: なし」等にしない＝ルーラーはプレイヤーではないため。旧データの `isMasterScene` の読み替えも維持。
- **上演中のシーン番号は実行状態のカウンタ**（巡回シーンの再入場も 1 シーン・14-8）。台本の行番号ではない。
- **情報の段は目標値の昇順**・開示の累積（入口と下位段）はトグル側の規則のまま。**段の目標値は達成値の絶対値と比べる**（2026-08-12 裁定）ので、カードでも入口の技能行とは独立した「さらに目標値 n」として積む。
- **技能名は識別キーの逆引き＋〈〉囲い**（`withResolvedInfoSkillNames` → `formatSkillName`）を通した表示のまま。カードは受け取った表示名を出すだけで、解決には関与しない。

#### 監査結果 ✅

- 4 種（シーン切替・トレーラー・シナリオテキスト・情報）とも、送信の可否判定・モード判定（`disclosed` / `targets` / 送れない）は旧実装と同じ条件で動く。テストで固定済み。
- 情報カードの本文は**リッチテキストをそのまま置く**形に直した。旧実装は ProseMirror の `<p>` をさらに `<p>` で囲っていたため空段落が挟まっていた（14-8 の申し送り「体裁の是正は行っていない」に含まれていた件）。

#### 課題・申し送り

- 下記 2 件はいずれも同日中にユーザー指摘で解決（→ 次エントリ）。

### 2026-08-15 フェーズ14-8 情報の「目標値の送信」が本文まで送っていた（是正）／パネルのセクションの出し分け

**2026-08-15** / **関連実装** `session-logic.mjs`（`buildInfoCardData`）・`templates/app/scenario-panel.hbs`＋tests / **ステータス**: 解消済・隔離 Foundry で実機確認済

#### ルール解釈

- **「目標値の送信」＝情報項目名・使用技能・目標値だけを卓に出すもの**（2026-08-15 ユーザー言明「元々は情報項目名と技能、目標値のみで本文は送っていなかった」）。RL が「何がどの目標値で判るか」を示す操作であり、**未開示の内容そのものは出さない**。内容が卓に出るのは開示済みの送信だけ。
- **段（追加で判明する内容）は目標値だけを出す**（Code 判断・段は 2026-08-12 の新設で当時の初版には無かった）。段の目標値は「同じ入口のまま達成値が伸びると更に判る」ことを示す**目標値の情報**なので目標値の送信に含め、段の本文は入口本文と同じく伏せる。目標値を持たない段は出すものが無いので落とす。
- **チーム・情報はアクト上演中の操作、配布・送信は開始前の操作**（2026-08-15 ユーザー指示）。チーム宣言はいつでも可（14-5）という以前の整理から、**表示の出し分け**を上演中に寄せた（宣言できる時期の変更ではなく、開始前のパネルに出す必要がないという整理）。

#### 監査結果 ✅

- **原因の特定**: 初版（`04800da`）の情報送信は、開示済み分岐でのみ `content.text` を出し、目標値分岐は技能行だけを出していた。**2026-06-07 `a5a51a2`（フェーズ3-5 の ApplicationV2 移行）で 2 つの分岐を共通ヘルパー `buildSkillHtml` に畳んだ際に、目標値分岐が開示済み分岐の本文出力を引き継いだ**。以後 14-3 の `buildInfoMessage`、2026-08-15 のカード化まで持ち越されていた（カード化で入った不具合ではない）。
- 是正後、目標値の送信は技能行と段の目標値だけを出す。技能も目標値も持たず本文だけの内容は送信対象に入らない（初版と同じ＝出すものが無い）。開示済みの送信は無改修。

#### 課題・申し送り

- なし（実機で開始前＝アクト/配布・送信のみ、開始後＝チーム/シーン/情報/アクトになることを確認済み）。

### 2026-08-15 フェーズ14-8 情報項目の目標値は等価／指定技能の表記を一本化

**2026-08-15** / **関連実装** `session-logic.mjs`・`skill-dictionary.mjs`・`tnx-scenario-panel.mjs`・`tnx-rl-request-app.mjs`・`focus-system-request.mjs`・`info-card.hbs`・`scenario-panel.hbs`・`scenario-sheet.hbs`＋tests / **ステータス**: 是正済・隔離 Foundry で実機確認済

#### ルール解釈（ユーザー言明・正本は [Scenario_Progress](../../01_Wiki/Game_Rules/Scenario_Progress.md)）

- **1 つの枝が持つ複数の目標値は等価**。「低い＝主／高い＝追加」という関係は**無い**。むしろ**一番高い目標値が通常の目標値**で、低い目標値は達成値がどうしても出ないときの受け皿。ほとんどのシナリオは一番上まで抜けることを想定して書かれている。
- ただし**高い目標値を主役に格上げするのも誤り**（ユーザーが先回りで釘を刺した）。**表示上は全て等価**。
- **指定技能の書き方は登場判定と同じ**（同じ小分類の固有名詞技能は一つに束ねる）。

#### 監査結果 ❌→✅（Code の誤り 2 件）

1. **「さらに目標値 n」という表記**（2026-08-12 に Code が導入）。(a) 日本語として係り先の無い副詞が宙に浮いた文の断片で、ラベル（名詞句）ではない。(b) 「段は入口に積み増される追加情報だ」という**仕組みの説明を UI に書き込んでいた**——しかもその関係は**事実として存在しない**。「UI はメモ欄ではない」規則への違反として指摘を受けた。→ 表示は値そのもの（`目標値 12`）に。
2. **パネルの入れ子表示**（入口＝親・段＝字下げ）が、無い上下関係を構造で主張していた。→ **一番上は指定技能だけ**・目標値は全て同じ深さに並べる。

#### 実装（表示の規則）

- パネル: `指定技能` を見出し、その下に `目標値 n` の開示トグルを同列に並べる（本文プレビューは**出さない**＝2026-08-15 ユーザー「邪魔」）。
- チャット（目標値の送信）: `〈社会：ストリート、警察〉目標値: 8, 12, 15`（技能行ごとに横並び）。
- チャット（開示済みの送信）: `目標値｜本文` の 2 列（罫線の無い表）。
- アクトシートの編集欄も「さらに目標値」「さらに判明する内容を追加」→「目標値」「目標値を追加」。
- **指定技能の表示規則を 1 つに集約**（`formatDesignatedSkills`）。従来は登場判定だけが束ね表記で、情報項目・判定要求・FS 判定は技能ごとに `〈〉` を作って「・」で連ねる別実装だった。**操作単位のチップは対象外**（1 チップ＝1 技能で ✕ を持つため束ねられない）。

#### 課題・申し送り

- **アクトシートの編集欄は構造としては入口＋追加行のまま**（本文は枝の `text`、以降は `tiers[]`）。表示の語は等価に直したが、保存形式が「入口 1 つ＋段の配列」なので編集欄の並びも入口が先頭に固定される。完全に等価な「目標値の集合」へ寄せるには保存形式の変更（＋移行）が要るため、必要になったら設計フェーズで扱う。

### 2026-08-15 フェーズ14-8 判定はブロックしない（登場判定の強制失敗）

**2026-08-15** / **関連実装** `appearance-logic.mjs`・`appearance-check.mjs`・`tnx-check-flow.mjs`・`check-result.hbs`＋tests / **ステータス**: 実装済・隔離 Foundry で実機確認済

#### ルール解釈（ユーザー確定・正本は [Check_Rules](../01_Wiki/Game_Rules/Check_Rules.md)「自動失敗の条件」と [Appearance_Check](../01_Wiki/Game_Rules/Appearance_Check.md)）

- **「このゲームで『判定そのものがブロックされる場面』はほぼ（ひょっとしたら全く）存在しません。どの参加者にも平等に『判定を失敗する権利』があります。そのことによって手札を入れ替えていくからです」**（verbatim）。
- 適用: 「登場：不可」のシーン・サンクチュアリの危険値装備携帯の**両方**（Code の確認質問「サンクチュアリも同じ扱いか」に対し「そうです。判定ブロックはしません」）。判定は通常どおり行え、**結果が達成値にかかわらず必ず失敗**する。
- 変更しない点: チーム免除が「登場：不可」で塞がれる挙動（2026-08-09 裁定）・RL による登場（判定を経ない）——判定でないものはこの原則の対象外。

#### 監査結果 ❌→✅（Code の符号化誤り）

従来実装は 2026-08-09 の「登場判定が不可」を**起動のブロック**（警告トーストで判定フロー自体を開始しない）と符号化していた。これは登場できないシーンで PL から**手札入れ替えの手段まで奪う**——「失敗してもよい判定で手札を回す」（Appearance_Check「手札入れ替えとしての登場判定」）は TNX の基本手段であり、ブロックはルールの意図に反していた。

#### 実装

- `appearanceCheckParams`: `blocked` を廃し `forcedFailure: null|"none"|"sanctuary"`（理由コード）を返す。TN・危険値修正の算出は不変（サンクチュアリは TN 12 のまま強制失敗）。
- 判定フロー: `ctx.forcedFailure`（表示ラベル）を汎用機構として追加——結果計算後に `success=false`・`diff=null` に固定し `result.forcedFailure` へ理由を載せる。達成値・内訳は通常どおり。目標値なし（登場：不可）でも成否バナーを出す。
- **再判定・事後修正でも覆らない**: `_buildRecheckContext`/`startRecheck` が forcedFailure を引き継ぎ、`_applyCheckModify` は成否の再計算をしない（達成値の修正自体は通る・成否は失敗のまま）。
- 完了継続 `appearance` は success===true でのみ登場させる既存ゲートのため無改修。

#### 課題・申し送り

- テスト 2122 green（appearanceCheckParams の意味変更を既存テストの書き換えで固定）。**隔離 Foundry E2E**: 登場：不可で PL の判定フローが走り「失敗｜登場：不可」（スート一致・達成値10 のケースを含む）／GM 手動修正 +99 でも成否は失敗のまま／サンクチュアリ＋危険値装備で「失敗｜サンクチュアリ・危険値装備 目標値 12」。登場フラグが立たないことも確認。
- 成否バナーの長い注記で「失敗」が文字割れ → `.cr-result` を flex-wrap にして注記ごと次行へ落とす是正を同時に実施（全結果カード共通・従来より劣化する経路なし）。
- 強制失敗が要る場面が今後他にも出たら `ctx.forcedFailure` に理由ラベルを渡すだけで乗る（登場判定専用にしていない）。

### 2026-08-16 フェーズ14-8 登場判定の専用チャットカード（用途は新設しない）

**2026-08-16** / **関連実装** `appearance-logic.mjs`（`appearanceCardInfo`）・`appearance-check.mjs`・`tnx-check-flow.mjs`・`check-result.hbs`＋tests / **ステータス**: 実装済・隔離 Foundry で実機確認済

#### ルール解釈（ユーザー指示・2026-08-16）

- **登場判定を用途（usage）としてモデル化しない**: 登場判定は毎回指定技能が変わり、シナリオによっては通常指定されないタイプの技能が指定されることもある。どの技能から起動されるか事前に確定できないため、アイテム（技能）側に用途を配る設計は成り立たない。パネルのボタン→任意の技能選択→`_activateItemCheck` という既存の起動形を維持する。
- **専用カードが必要な理由**: 汎用の技能判定カードでは、卓から「これが登場判定であること」と帰結が読めない。登場はシーンの状態が変わる卓全体の関心事なのに、帰結（シーンに登場した）は判定者ローカルのトースト通知のみでチャットに記録が残らなかった。
- **エリアは設定されているときだけ出す**（ユーザー指示＝「未定」を出す見栄えの悪さを避ける）。

#### 実装

- 専用化は**判定文脈（`ctx.appearance`）をキーにした結果カード描画の分岐**（攻撃カード・移動カードと同型）。基底部品（check-card-head／check-calc-rows）はそのまま、種別タグを「登場判定」に差し替え、エリア行（設定時のみ）・ゴースト宣言行・成功時の帰結行「シーンに登場した（ゴースト）」を足す。
- 表示情報の組み立ては純関数 `appearanceCardInfo(cc, result)`。帰結表示は `success===true` のときだけ＝完了継続（自動登場）の適用条件と同一（目標値なし＝success null は自動登場しない従来裁定のまま出さない）。
- `ctx.appearance` に `areaLabel` を追加（継続文脈として再判定スナップショットにも乗る＝再判定の置き換え描画でも専用カードが保たれる）。

#### 課題・申し送り

- テスト **+4＝2126 green**。**隔離 Foundry E2E**: ①登場：不可（エリア未設定）＝タグ「登場判定」・エリア行なし・失敗＋理由・帰結なし ②サンクチュアリ＝エリア行「サンクチュアリ」・目標値12・強制失敗 ③数値指定 TN1＋ゴースト宣言＝成功で帰結行「シーンに登場した（ゴースト）」・登場フラグ／isGhost とも立つ。3枚とも目視精査。
- 帰結行は投稿時点の成否の静的描画。事後修正で成否が変わった場合は「修正後の成否」行が生きた真実（既存カードと同じ規約・帰結の自動適用は継続再実行が担う）。

### 2026-08-16 フェーズ14-9 情報収集判定（旧フェーズ16 の統合）

**2026-08-16** / **関連実装** `info-gathering.mjs`（新設）・`session-logic.mjs`・`tnx-hud.mjs`・`tnx-rl-request-app.mjs`・`tnx-check-flow.mjs`・`tnx-socket-handler.mjs`・`check-result.hbs`・`hud.hbs`＋tests / **ステータス**: 実装済・隔離 Foundry で実機確認済（GM/PL 2クライアント）

#### ルール解釈（ユーザー裁定・正本は [Scenario_Progress](../01_Wiki/Game_Rules/Scenario_Progress.md)「情報収集判定の裁定」）

- **公開状態3段階**: 非公開＝存在のみ（「非公開の情報」・技能/目標値/内容とも伏せる。「『非公開』は本来技能や目標値も非公開の段階」）／公開＝項目名・技能・目標値（判定を試みられる）／開示＝内容。
- **判定成功時は自動で開示**（RL 承認なし）・**達成値が抜いた目標値まで一括開示**・**回数制限は設けない**（「そういうことは今後も他の機能でもしません」・卓運用の「基本1シーン1回」は卓が守る）。
- **HUD に非公開の項目も存在表示**・**ツールチップは常設**（開示済み=内容／未開示でも「万が一チャットが流れていった場合に備えて」指定技能と目標値）・**起動ボタンは項目に1つ**（技能行が複数ならダイアログで選択）——後者2つは Code の設計提示への訂正として裁定（ダイアログ閲覧案・技能行ごとのクリック案は却下）。

#### 実装

- 開示は純関数 `discloseInfoByAchievement`＝達成値以下の目標値を持つ入口・段を一括で開く（入口の閾値＝挑んだ技能行の tn・段は絶対値比較＝2026-08-12 裁定のまま・開くだけで閉じない・「段が開けば入口も開く」累積規約を維持）。
- 起動＝HUD の項目ボタン→技能行選択→**判定要求の応答機構と共用**（`resolveDesignatedSkillResponse` 抽出＝複数キー選択・所持技能の実解決・用途/コンボ候補・代用判定）→唯一の起動関数 `_activateItemCheck`。判定の成否基準＝挑んだ技能行の目標値。
- 完了継続 `infoGathering`（CONTINUATIONS・再判定/事後修正の rerun も単調適用）。開示の書き込みは `infoDisclose` ソケットで activeGM に委譲（sessionTeam と同型）。
- 結果カード＝文脈分岐（登場判定と同型・用途なし）: 種別タグ「情報収集判定」＋項目名行＋成功時の帰結行「情報を開示した」。

#### 監査結果 ✅

- E2E で裁定を逐条確認: マスク表示／公開の見え方／ツールチップ2モード／達成値10で entry(5)+段10 が開き段12 は閉のまま＝**一括開示の境界どおり**／開示は PL の判定から RL 承認なしで反映（ソケット委譲）／回数制限なし（何度でも挑める）。

#### 課題・申し送り

- テスト +11＝2137 green。実機スクリーンショット6枚を目視精査。
- 観測: 辞典から**生インポート**した一般技能（テスト用 createEmbeddedDocuments）はスート指定が全 false で「用途に不備（共通スートなし）」警告になる。実キャストの初期インポート技能は正常。テストデータ由来の観測＝14-9 スコープ外。辞典データのスート整備 or 取得経路の正規化の要否は必要になったら扱う。

### 2026-08-22 フェーズ14-8 チーム自動登場のオミットとゴーストトグル

**2026-08-22** / **関連実装** `session-state.mjs`・`appearance-state.mjs`・`appearance-check.mjs`・`tnx-scenario-panel.mjs`・`tnx-socket-handler.mjs`・`session-logic.mjs`・`scenario-panel.hbs`・`tnx.mjs` / **ステータス**: 実装済・隔離 Foundry で実機確認済

#### ルール解釈（ユーザー裁定・正本は [Appearance_Check](../01_Wiki/Game_Rules/Appearance_Check.md)「回数」「チームと登場の運用」）

- **登場判定はシーンプレイヤー以外なら誰でも1シーンにつき1回**。シーンプレイヤーは判定なしで自動登場。システムは回数を制限しない（卓運用・情報収集判定とは「そもそも性格が違う」が回数カウンタを持たない点は同じ）。**登場中は振れない**ゲートのみ機械的に維持（ユーザー確認）。
- **チームの登場資格**: 「一人が登場判定に成功するか、シーンプレイヤーなら、自動的に登場しますが、厳密には登場時に登場判定を行い、成否にかかわらず登場可能という処理になります」（verbatim）。
- **運用の再設計**（ユーザー提案・Code 同意）: **自動登場は RL の操作に任せる**——「チームで登場」機能はオミットし、「RL が毎回、登場判定をするかしないか（しないことも別に選択できる）、プレイヤーに確認を取り、全員がやりたい処理を済ませたと判断した段階で登場させる」（verbatim）。**成功時の自動登場は維持**（「シーンに登場したという結果は変わらないので」）。
- **ゴーストは名前の表示非表示と同様のトグル**を、キャストタブの登場操作とヘッダーの登場キャラクターチップの両方に設ける（verbatim 相当）。

#### 実装

- 削除: `appearTeam`/`exitTeam`（session-state・sessionTeam ソケット op・パネルのボタンと委譲）・joinTeam の後乗り自動登場・「登場：不可」のチーム免除ゲート・純関数 `teamHasAppearing`。チームの結成・参加・離脱・編成（舞台候補が使用）は維持。
- 追加: `setGhost`（appearance-state・isGhost を持たない種別は無視）＝登場判定のゴースト宣言・手動登場行トグル・チップトグル（`setActorGhost`）の3経路を一本化。**退場（setAppearing(false)）で名前非公開と同様にゴーストも落とす**。isGhost 変更で パネルを再描画するフックを追加。

#### 監査結果 ✅（是正1件込み）

- E2E: チーム編成 UI 健在＋登場系ボタン消滅／手動登場行のゴースト登場（appearing+isGhost）／チップトグルの実フリップ／退場で両状態解除／登場判定成功のゴースト宣言経路（継続関数の直接検証）。
- 初回実装は **isGhost 変更でパネルが再描画されず**チップのトグル表示が古いままだった（実機検証で検出）→ updateActor フックに isGhost を追加して是正。

#### 課題・申し送り

- テスト 2139 green（teamHasAppearing のテスト削除）。回帰スクリプト（登場判定カード）の成功経路はカード運でスート不一致が続き未踏——継続関数の直接検証で代替した（フロー本体は無改修）。
- 旧「チームで登場」を前提にした記録（Scenario_Progress の同時登場行・Appearance_Check のチーム免除節・申し送り表）は正本を更新済み。

### 2026-08-22 フェーズ14 寄り道: HUD 参加者パネル+ステータスパネル移設+カード D&D 移動

**2026-08-22** / **関連実装** `tnx-hud.mjs`・`hud.hbs`・`tnx-action-handler.mjs`・`tnx-socket-handler.mjs`・`tnx.mjs`・`tnx2.css` / **ステータス**: 実装済・隔離 Foundry で実機確認済（GM/PL 2クライアント）

#### ルール解釈

- ルール解釈の変更なし（物理的なカード移動と HUD 表示の改修）。表示仕様はユーザー指定: ステータスパネルは手札の左・手札/切り札と同サイズ／プレイヤーの手札パネルは**参加者パネル**に改称し開示設定によらず常時表示・RL の行も表示・RL のステータス表示は常に「RL」／`revealPlayerHands` オフは裏面のみ・**RL の手札は常に裏向き**・裏面カードの内容がツールチップ等で漏れないこと／5枚目以降の折返しは手札列の中で行う（ステータスを含む行全体では折り返さない）／受け渡しは「指定枚数を渡す」+行への D&D に一本化／捨て札→山札・捨て札→手札の D&D を追加・種類の異なるカードは移動不可。

#### 実装

- 参加者 = 自分以外の接続中ユーザー（手札所持）。並びは受け渡しダイアログと同じ規則（PL 名前順→GM 名前順）。ステータスの組み立ては `TnxHud._buildUserStatusCards(user)` に共通化（自分のステータスパネルと共用。シーンプレイヤー/ゴースト/抹殺=本人・舞台裏=全員）。裏面は `card.back.img || トランプ裏面アセット`・title/alt にカード名なし。
- D&D は既存の `data-drag-type`/`data-drop-zone` 機構に `discard-card→deck/hand`・`hand-card→participant` を追加。捨て札→山札は RL のみ（右クリックメニュー「1枚山札に戻す」と同権限）。
- 受け渡しの実体は `_passHandCards` に一本化。相手の手札を所有しない場合（PL→他PL/RL）は `passHandCard` ソケットで activeGM が代行（手札の ownership は本人+GM のみ OWNER のため。sessionTeam 等と同型・要求者の手札=移動元の検証あり）。
- 再描画の追随を拡張: updateUser（全ユーザーの自スコープフラグ）・userConnected（入退室）・他ユーザーの担当キャラクターの AE/isGhost 変化でも HUD を再描画（従来は自分の分のみ）。

#### 監査結果 ✅

- 隔離 Foundry E2E（GM+Player1 の2クライアント）で検証項目すべて green（初回 run の 2 fail はテスト側の状態汚染と山札カウント計測誤り＝`availableCards` で再検証して green）: パネル常時表示（両視点・RL 行と「RL」プレート）／reveal ON=PL 表向き+title・RL 裏面／reveal OFF=全員裏面・title なし／自ステータス 100x150・手札の左・下端揃え／7枚=4+3 の列内折返し・はみ出しなし／D&D 5経路（GM→PL 直接 pass・PL→RL 委譲 pass・捨て札→手札・捨て札→山札・PL の捨て札→山札は拒否）／種類違い（切り札→捨て札）は移動しない／右クリックメニュー=「指定枚数を渡す」「捨てる」のみ／アコーディオン開閉と設定永続化／下バー収納で参加者/ステータスも退避。スクリーンショット8枚を目視精査。

#### 課題・申し送り

- **手札→山札の D&D は見送り**（Code 判断）: 対応する既存操作がどこにも無く、ゲーム操作の新設＝裁定を要するため。「エリア間はすべて移動」へ一般化するなら裁定を仰いで追加する。
- 旧 `passSingleCard`（1枚+相手選択ダイアログ）・`selectAndPassToUser`（行クリック起動）は支える仕組みごと撤去。
- PL→他PL/RL の受け渡しは従来 `Cards#pass` 直呼びで、手札 ownership（本人+GM のみ）により権限不足のはずの経路だった（実機未検証のまま存在）。今回の activeGM 委譲で実働化した。
- テストは既存 2139 green 維持（HUD 表示層のみの改修で新規ユニットテスト対象の純関数なし。検証は E2E で実施）。

#### 是正（2026-08-22 同日・ユーザー指摘2件）

- **幅の余分と手札の見切れ**: 手札列を常に4枚分の固定幅(193px)・ステータス列を常に46pxで**予約**する実装だった（「4枚で折返し」を固定幅確保で実装した Code の判断＝指示にない幅の予約）。さらに参加者+アクセスの合計幅が置き場所を超えると flex 自動縮小でパネルが**内容より狭く**潰れ、リストが横スクロール化して手札4枚目が見切れた。→ 予約の全廃（手札列は `max-width`＝折返し上限のみ・空のステータス列は非描画・行は `align-self: flex-start`）+ `.hud-bottom-panels > * { flex-shrink: 0 }`（縮小せず右へ伸びる）+ `overflow-x: hidden`。
- **ステータスの整列と強調**: 幅46pxで揃えていたためアクセスカード素材（179x249≠2:3）が手札(69px)より低く表示・自分用パネルは100x150の寸法強制で歪んでいた。→ **高さ基準**（参加者行69px・自分用150px・RLプレート50x69）＋幅は素材比のまま＋アクセントの縁+グローで強調。
- 隔離 Foundry で9項目 green（高さ一致・素材比・強調・2枚行の縮み・空ステータス行の詰め・RLプレート寸法・自パネル150px・幅1000pxウィンドウで見切れ/横スクロールなし）・スクリーンショット4枚を目視精査。テスト2139 green 維持。

#### 是正2（2026-08-22 同日・ユーザー裁定=ステータスの意味論）

- **ルール解釈の訂正（verbatim）**: 「舞台裏カードは舞台裏判定の手番を表わすものではありません。文字通り舞台裏にいる＝登場していないことを表わすカードです」。旧実装（14-7）の「舞台裏枠の開始〜終了の間は全員に表示」は誤解釈につき撤回（2026-08-08 エントリに訂正注記済み）。
- **表示指示（verbatim 相当）**: 「空のステータスはそうとわかるようパネルを置いてください。勝手に消さないでください。ただし空の状態というのは基本的に無いと思います。登場していて、かつシーンプレイヤーでない場合はそうとわかる表示を行ってください」。
- **実装**: プレイヤーの登場状態を三態・排他で表示——シーンプレイヤー（scene_player）／**登場中**（`isAppearing(user.character)`・専用カード画像が無いため文字プレート。参加者行 50x69・自分用パネル 108x150）／**舞台裏**（behind_the_scene＝登場していない）。RL には三態を適用しない（参加者行は「RL」プレート・GM 自身のパネルは従来どおり）。ゴースト／抹殺は追加表示のまま。ステータス列は常設し、万一空のときは破線の器で空を明示。`getBackstage()` 依存を撤去し、登場フラグ（appearing）の変化でも HUD を再描画。
- **検証**: 隔離 Foundry 9項目 green（非登場=舞台裏カード〔行/自パネル〕・登場=「登場中」プレート〔行/自パネル108x150〕・登場+ゴースト併記・シーンPL排他・RL行不変・GM自パネルに三態が出ない・空の器）＋スクリーンショット5枚目視精査。テスト 2139 green 維持。
- **意匠差し替え（2026-08-22 同日・ユーザー指摘「ダサい」）**: 素の文字プレートをやめ、アクセスカードの構成（英題+下端の和文リボン）に合わせた CSS カードに刷新——斜めスキャンライン+奥行きグラデの地に "ON STAGE"（2行）、下端に「登場中」リボン。実機で両サイズの描画を目視確認。
- **RL プレートも同意匠に統一（2026-08-22 同日・ユーザー提案）**: "RULER"+「RL」リボンの CSS カードに刷新（旧・素の「RL」文字プレートは撤去。状態が常に RL である点は不変）。

### 2026-08-22 HUD 全体リデザイン(意匠統一)

**2026-08-22** / **関連実装** `tnx2.css` のみ(構造・挙動は不変) / **ステータス**: 実装済・隔離 Foundry で実機確認済

#### 設計レビュー(意匠のみ・ルール解釈なし)

- ユーザー指示: 「直近追加のステータス表示と情報項目はとても出来が良いのでこちらに寄せます。HUD全体をこちらのふたつに寄せてリデザインしてください」。
- 参照2箇所からデザイン語彙を抽出——**角落としプレート3層構造**(枠レイヤー+左アクセントバー+切り欠き追従グロー/内側の地/テクスチャ)・**アクセントのヘアライン区切り**・**六角ネオンチップ**・**斜めテクスチャ**——し、パネル基底(`.hud-panel`/`.main-area`/`.neuro-area`)・見出しチップ・残枚数チップ・参加者行(ミニプレート)・ヘッダー・空スロット(破線の器)・トグルボタン・判定待機表現(破線 outline → 手札パネルの枠と地のゴールド切替・切り札モード=オレンジ)へ適用。
- **技術ノート(将来の改修者向け)**: ①パネルのホスト要素に `clip-path`/`filter` を掛けない——パネル内に生成されるコンテキストメニューの fixed 配置が壊れ描画も切り取られるため、枠・地とも擬似要素側に持たせる。②地は**不透明トークン**で敷く——半透明にすると下の枠レイヤー全面が透けてパネル全体が枠色に染まる(初回実装で発生し是正)。③色味の掛かりは**固定サイズの radial-gradient**——%指定の線形グラデは大きいパネルで色掛かりが広がりすぎる(同上)。
- 検証: 隔離 Foundry で GM/PL 全景・右カラム・下バー・折り畳み・判定待機・コンテキストメニュー(表示/位置/前面/項目)を確認、スクリーンショット目視精査。テスト 2139 green・stylelint クリーン。
- **チップ意匠のブラッシュアップ(2026-08-22 同日・ユーザー指摘)**: 見出し/残枚数チップの面の縦グラデ陰影は「3DCGで安易に影やハイライトをグラデで入れると安っぽくなる現象」(verbatim 相当)——発光は面でなく**リム(管)と文字**が持つべきもの。面=ダークガラス平塗り・リム=clip 形状追従の二段 drop-shadow(芯+ブルーム)・文字=近接+広域の二層グローに変更。2倍ズームのスクリーンショットで精査。
- **チップ発光文字化(2026-08-22 同日・ユーザー指摘)**: 「チップの色が背景の色と近すぎ」「白いボールドのゴシック体というのはまさしく『ダサさ』の原因。文字自体を発光色に」——文字=アクセントシアン(ウェイト500)+二層グロー・面=背景より一段暗い黒ガラス(rgba(1,6,9,0.94))で分離。2倍ズームで精査済み。

### 2026-08-23 フェーズ14-8 登場表現の改修（トークンの存在＝登場・退場確認ダイアログ）

**2026-08-23** / **関連実装** `appearance-state.mjs`・`appearance-logic.mjs`・`appearance-check.mjs`・`session-state.mjs`・`tnx-scenario-panel.mjs`・`tnx.mjs` / **ステータス**: 実装済・隔離 Foundry で実機確認済（GM/PL 2クライアント）

#### メカニクス監査（ユーザー指示・正本は [Appearance_Check](../01_Wiki/Game_Rules/Appearance_Check.md)「盤面表現」）

- **「トークンを盤面に出す＝登場」への転換**（2026-08-23 ユーザー指示）: 未登場＝トークン無し・登場＝トークン有り・**ゴースト登場＝トークン不可視**・**退場＝トークン削除**。旧「未登場＝トークン不可視」を置換。理由＝ココフォリアのように駒を「登場エリア」へ移動させる形は FVTT では分かりづらい。
- **退場確認ダイアログ**（ユーザー指示）: 退場＝トークン削除になった副作用として、×ボタン以外でも（権限を与えられていれば）トークン削除で退場になるため、**両経路で「本当に退場するか」を確認**してから適用する。
- **チーム連動の退場カスケードは実装しない**: 依頼文の「チームを組んでいると自動でトークンまで消えてしまう」は、現行コードにチーム退場の自動化が存在しない（2026-08-22 オミット）ため、「退場の適用がトークン削除まで及ぶ」ことへの懸念として全退場経路の確認ダイアログで対応した。チーム退場の自動化を復活させる趣旨であれば別途裁定が要る（Code 解釈・要確認事項として報告）。
- Code 設計判断（裁定余地）: ①**最後の1体規約**＝同一アクターの複数トークン（トループの分身）は残りがある限り削除しても退場でない ②シーン切替・アクト終了の一括退場は進行フローの一部なので確認ダイアログを出さない ③トークン hidden→isGhost の逆同期はしない（isGhost は CS 修正の機構的意味を持つため盤面の表示操作から黙って変えない）。

#### 実装

- 同期の書き換え（appearance-state・registerAppearanceTokenSync）: 登場フラグ→トークン作成（プロトタイプから・盤面中央スナップ・位置衝突はグリッド単位で右＝純関数 `pickTokenDropPosition`・activeGM 代行）／退場→全削除／isGhost→hidden（一方向）／createToken→登場・preDeleteToken→確認→削除＋退場・deleteToken 後段＝一括削除の帳尻。同期・確認済み削除は操作オプション `tnxAppearanceSync` でバイパス（確認ダイアログの連鎖・多重を防ぐ。preDeleteToken は発行クライアントでのみ走るため、ダイアログは操作した本人にだけ出る）。
- `confirmExitDialog`（appearance-state）: ×ボタン（パネル `_onExitActor`）とトークン削除で文言を分けた共通ダイアログ（DialogV2.confirm・displayActorName＝名前非公開を漏らさない）。
- ゴースト順序の是正: `appearActor`・登場判定の完了継続とも **setGhost → setAppearing** の順に変更＝トークンが最初から不可視で作られる（旧順序は一瞬可視で作られてから隠れる）。

#### 監査結果 ✅

- 純ロジック TDD +8＝**2147 green**・eslint 0 error（警告15は既存）・stylelint クリーン。
- **隔離 Foundry E2E（GM 18項目＋PL 5項目 all green）**: 削除→ダイアログ→キャンセル（残存・登場のまま）／OK（削除＋退場）・未登場の残置トークンは素通し・手動登場＝中央に可視作成・チップのゴーストトグルで hidden フリップ（往復）・ゴースト登場＝hidden で作成・×ボタン両分岐・トークン配置＝登場・分身2体＝1体目素通し/最後の1体でダイアログ・**PL の削除は PL 側にだけダイアログ**（GM 側に出ない）・PL 権限でフラグも落ちる・継続経由ゴースト＝activeGM 代行で hidden 作成・退場の同期削除に他クライアントでダイアログが出ない。スクリーンショット4枚（両ダイアログ文言・チップ・PL 視点）を目視精査。

#### 課題・申し送り

- ダイアログの Yes/No 表記はコア既定（既存の確認ダイアログ群と同じ）。
- Scene 跨ぎの自動生成はしない（従来方針のまま）。アクティブ盤面が無ければ同期しない。アクティブ盤面 Scene 自体の削除はトークン単位のフックが走らないため追随しない（RL が上演中に盤面を消す運用は想定外・必要になったら扱う）。

### 2026-08-23 フェーズ14-8 チームの退場連動（ゲーム設定）＋確認ダイアログはチーム退場時のみ

**2026-08-23** / **関連実装** `session-logic.mjs`・`appearance-state.mjs`・`session-state.mjs`・`tnx-scenario-panel.mjs`・`tnx-socket-handler.mjs`・`tnx.mjs` / **ステータス**: 実装済・隔離 Foundry で実機確認済（GM/PL 2クライアント）

#### メカニクス監査（ユーザー裁定・正本は [Appearance_Check](../01_Wiki/Game_Rules/Appearance_Check.md)「盤面表現」）

- **チームの退場連動＝ゲーム設定**（ユーザー提案「チーム結成時、一人の退場で自動退場する」を設定化＝落としどころ・Code 同意・既定オフは Code 推奨をユーザー黙認）: 入場は「判定をするかしないか等、判断する部分が多いので自動では瑕疵が多い」が、「退場に関してはルールの厳密さを優先するなら一度にトークンが削除された方が面倒は無くて良い」（verbatim 相当）。
- **確認ダイアログはチーム退場時のみ**（ユーザー裁定「確認ダイアログはチーム退場時の挙動を想定して提案したものなので今後はチーム退場時のみとします」）: 前日実装の全経路ダイアログを置換。機能的読みとして**連動が自分以外の登場中メンバーに及ぶときだけ**確認を出す（巻き込みの無い退場に確認は不要＝Code 設計判断・巻き込まれるメンバー名とチーム名を明示）。
- **逮捕令状はチームから抜けてから退場**（ユーザー裁定「まず最初に退場前に抜ける形に。ルールに明記はされていませんが、便宜的にもそれが正しいはず」）: 強制退場は退場連動に乗せない。本エントリではルール記録のみ・適用は社会ダメージ系自動化（フェーズ14 内・別途設計提示）で実装する。
- Code 設計判断（裁定余地）: ①連動は手動の退場操作（×・トークン削除）のみ ②確認を経ない削除（一括削除等）は連動しない＝単独の帳尻のみ（連動は確認とセット） ③未登場のメンバーは巻き込まない。

#### 実装

- 純関数 `teamLinkedExitTargets(teams, actorId, {linked, isAppearing})`（session-logic）＝退場対象と巻き込み一覧・チーム名の解決。
- 退場の入口を appearance-state に一本化: `manualExitTargets`（設定＋sessionState 直読み※循環回避の注記付き）→ `confirmTeamExitDialog`（others があるときだけ）→ `applyManualExit`（GM=直接・PL=`teamExit` ソケットで activeGM 代行＝要求者の所有権検証・連動対象は GM 側で再解決）。session-state の旧 `exitActor` は撤去（×ボタンの唯一の呼び元が新経路に移行）。
- preDeleteToken: 連動が他メンバーに及ぶときだけ veto→確認→適用（トークンの削除は退場同期に任せる）。それ以外は素通し＝deleteToken 後段（activeGM）が最後の1体の削除を退場に落とす（単独退場の本経路 兼 一括削除の帳尻）。
- ゲーム設定 `teamLinkedExit`（world・config・Boolean・既定 false・名称「チームの退場連動」）。

#### 監査結果 ✅

- 純ロジック TDD +5＝**2152 green**・eslint 0 error（警告15は既存）・stylelint クリーン。
- **隔離 Foundry E2E 12項目 all green**（GM/PL 2クライアント）: 既定オフ／オフ=×・トークン削除とも確認なしで即退場＋トークン削除／チーム所属でもオフは単独退場（相方残存）／オン=×起点・削除起点の両文言でチーム名・相方名を明示→キャンセルで両者残存→OK で全員退場＋トークン全削除／**PL のトークン削除は PL 側にだけダイアログ→ソケット委譲で全員退場**／チーム未所属は設定オンでも確認なし。スクリーンショット3枚（×起点・削除起点・PL 視点）を目視精査。

#### 課題・申し送り

- 社会ダメージ系の自動化（逮捕令状の強制退場＝チーム離脱→退場・次シーン登場不可の符号化・sceneDeferred 発火機構）→ **フェーズ15 へ**（2026-08-23 同日裁定: 当初「登場判定を扱うフェーズはここなので自動化タイミングもここ」→「時間に関わる部分なので次フェーズ」に訂正・設計は [Phase_14_Tasks_Detail](../01_Wiki/Phases/Phase_14_Tasks_Detail.md)「申し送り詳細」に保存＝未裁定2点込み）。

### 2026-08-23 フェーズ14-6/14-8 是正 舞台裏の手番表示の文言＋シーン一覧フェイズ見出しの視認性

**2026-08-23** / **関連実装** `templates/app/scenario-panel.hbs`・`css/tnx2.css` / **ステータス**: 是正済・隔離 Foundry で実機確認済

#### 設計レビュー（是正・ユーザー指摘）

- **「舞台裏を回す」「回しきりました」＝「UI はメモ欄ではない」規則への違反**（ユーザー「これではメモです。重大な規約違反をまたも起こしています。二度としないでください」）: ルール手順の記述をボタンに、報告文（丁寧語の文）を状態表示に書いていた。2026-08-15「さらに目標値 n」と同型の違反（文・仕組みの記述であって UI の表記でない）。是正＝スポット送りの操作は**「次へ」**（カット進行トラッカー footerPlan の確立済み表記・同一操作＝スポットを次の人へ渡す）、回しきりは状態表示**「済」**（巡回セクションの済/次と同じ語彙）。アイコンも fa-rotate → fa-forward-step（機構の説明でなく前進の表象）。
- **シーン一覧のフェイズ見出しが行に紛れて見難い**（ユーザー指摘・方法は Code 判断で可の指示）: 見出し文字を発光色（`--tnx2-text-accent`）にし、フェイズグループ間の間隔を拡大。セレクタは `.scp-phase-group > .scp-phase-label` にスコープ＝舞台裏/巡回/シーン一覧の節見出しは従来のまま（共通 CSS はスコープを絞る規範に準拠）。
- 併せて自己精査で是正: 舞台裏・巡回の名前が ellipsis で見切れていた（「名前・ラベルは折り返す」2026-08-15 規則との不整合）→ `overflow-wrap: anywhere` に統一。

#### 監査結果 ✅

- 本体 3 コミット（`06b9949` 文言是正 / `02ad87c` 見出し視認性 / `762167a` 折り返し）・テスト **2152 green**・stylelint クリーン。
- 隔離 Foundry で実機確認: 巡回中＝「次へ」ボタン＋スポット行ハイライト・回しきり＝「済」・フェイズ見出しの発光色（computed rgb(120,200,216)）と間隔・長い名前の折り返し（2行・はみ出し実測ゼロ）をスクリーンショット4枚で目視精査。

### 2026-08-23 フェーズ14-8 是正 進行ボタンの固定高で長いシーン名が溢れる

**2026-08-23** / **関連実装** `css/tnx2.css` / **ステータス**: 是正済・隔離 Foundry で実機確認済

#### 設計レビュー（是正・ユーザー指摘）

- 「次のシーンへ（シーン名）」がボタンの固定高（`.tokyo-nova .tnx-btn` の `height: 36px`）から溢れていた。指示＝「文字列の長さによってきちんと高さも変わるように」。
- 是正＝`.tnx-scenario-panel .tnx-btn:not(.tnx-btn--sm)` に絞って `height: auto`＋`min-height: 36px`＋`overflow-wrap: anywhere`（共通 CSS はスコープを絞る規範）。1行のボタンは従来と同じ 36px。
- 初版で `--sm`（次へ・読み込む等）に `min-height` が漏れて 26px→36px になる退行を実測で検出（`--sm` は `height` しか持たず min-height に負ける）→ `:not(.tnx-btn--sm)` で除外して解消。

#### 監査結果 ✅

- 本体 `642b225`・stylelint クリーン。隔離 Foundry 実測: 長名（全角約80字）＝高さ94px・はみ出しゼロ／静的ラベル＝36px 維持／`--sm`＝26px 維持。スクリーンショット目視（4行折り返し・アイコン左・全文可読）。

### 2026-08-25 フェーズ14-9 是正 KI-042 開示消失＋開示内容の自動公開＋未開示の併記

**2026-08-25** / **関連実装** `scripts/module/info-gathering.mjs`・`session-logic.mjs`・`tnx-check-flow.mjs`・`templates/chat/check-result.hbs`・`templates/chat/parts/info-disclose-outcome.hbs`・`scripts/tnx.mjs` / **ステータス**: 実装済・隔離 Foundry E2E 実機確認済

#### ルール解釈

- 発端はユーザー報告「判定はできるが判定後の情報公開が未完成」「そもそも HUD にもツールチップにも開示の結果が出ない」。隔離実機（develop 複製）で判定フローを実操作で追い、**GM 接続下では既存実装が健全に動くこと**と、**activeGM 不在時に開示が黙って消え、結果カードだけ「情報を開示した」と出る**こと（KI-042）を切り分けた。ユーザーが後者を原因と確認。
- 是正の裁定（2026-08-25 ユーザー承認）: ①帰結行は開示の**実適用後**に刻む（実適用と表示を一致させる） ②activeGM 不在は判定をブロックせず警告のみ（「判定そのものはブロックしない」の系統） ③判定成功で**今回新たに判明した分**の公開カードを自動送信（新規分のみ＝Code 提案を承認。全開示済みの再掲はツールチップと手動送信が担う） ④**「未開示の情報がある場合は、公開されている情報の下に表示してください。形式は完全に未開示の時と同様です」**（verbatim・開示済み表示の全面＝ツールチップ/手動送信/自動公開カードに適用）。正本は Scenario_Progress に記録済み。

#### 実装

- 純関数（session-logic）: `newlyDisclosedInfo`（開示前後差分）・`buildInfoDiscloseCardData`（新規分＋未開示併記のカードデータ・新規なしは null）・`buildInfoCardData` 開示モードへ未開示併記を追加（内部を `infoCardBlocks(mode, filter)` に統一）。
- 配線: 結果カードの帰結行を初回描画から撤去（`infoCheck.disclosed` は `checkResult.infoDisclosed` フラグ駆動）。GM 側 `applyInfoDisclosure` が差分ゲートを通過したときだけ正本を書き、`applyMessagePatch` で帰結行を挿入（`cr-info-outcome` マーカーで冪等）し、公開カード（info-card.hbs）を送信。PL 側は activeGM 不在で警告し emit しない。messageId は `_postResultChat` の戻り値→完了継続、および `_rerunContinuation` 両呼び元→CONTINUATIONS rerun に通した。

#### 監査結果 ✅

- 「開示された」という帰結の表示が実適用の裏付けなしに出る経路が構造的に消えた。差分ゲートは開示の単調性と整合し、再判定・事後修正の rerun で重複送信しない。
- テスト +7＝2159 green・eslint 新規指摘なし。隔離 Foundry E2E（GM+PL）: 成功＝帰結行・公開カード・チップ点灯・ツールチップ併記／rerun 達成値12＝追加分のみのカード・帰結行1個のまま／同値再実行＝カード増なし／PL 単独＝警告表示・帰結行なし・フラグ不変。スクリーンショット目視精査（チャット幅で併記の折り返し・はみ出しなし）。

#### 課題・申し送り

- なし（PL 単独でも判定自体は完走することを E2E で確認済み。調査中に一度「GM 不在で手札プレイが無反応」と誤認したのは、テストスクリプトが A のスート宣言ダイアログを見落としたため——実装の不具合ではない）。

### 2026-08-25 フェーズ14-9 是正2 起動ダイアログの統合（技能×目標値の平坦化）＋文言・見切れ

**2026-08-25** / **関連実装** `scripts/module/session-logic.mjs`・`info-gathering.mjs`・`tnx-hud.mjs` / **ステータス**: 是正済・隔離 Foundry で実機確認済

#### 設計レビュー（是正・ユーザー指摘3点）

- **セレクトの見切れ**: 行選択の選択肢が技能行の束ね表記（〈社会：ストリート、企業、真教〉（目標値 5））で、ダイアログ幅を超えて切れていた。
- **「挑む技能」＝メモ的文言**（「またもメモじみた UI」）: 既存語彙へ統一——判定要求の「指定技能を選択」ダイアログと同じ**「判定に使う技能」**に。
- **多段ダイアログ＝設計ミス**（「三回も『この技能で判定』を選ばされる」）: 行選択→（行内の）指定技能選択→代用判定と、同名ボタンを3連打させていた。**技能×目標値を平坦化した選択肢リスト**（`infoCheckOptions`）に統合し、ダイアログ1回で選び切る。確定した単一キーを `resolveDesignatedSkillResponse` へ渡すため共有機構の技能チューザーは発火しない（keys.length===1 の既存ショートカット）。代用判定は「未所持だから別技能で代用する」という**別の決定**なので独立ダイアログのまま（最大2回・所持していれば1回）。選択肢が1つならダイアログ自体を出さない。平坦化で選択肢の表記が単一技能（〈社会：ストリート〉（目標値 5））になり、見切れも構造的に解消。
- 撤去: `infoCheckRows`（行列挙）とその専用ダイアログ `promptInfoCheckRow`。HUD の canCheck ゲートも `infoCheckOptions` へ（機能撤去時は支える仕組みも同時削除）。

#### 監査結果 ✅

- 2026-08-16 裁定「起動ボタンは項目に1つ」「ダイアログで選ばせる」は不変（ダイアログの中身が行→技能×目標値になっただけ・裁定更新は Scenario_Progress に記録）。
- テスト 2160 green（infoCheckRows の3件を infoCheckOptions の4件へ置換・辞典から消えたキーの脱落を追加）・eslint 新規指摘なし。
- 隔離 Foundry E2E: ダイアログ1回（title 情報収集判定・ラベル 判定に使う技能・選択肢4件が全文表示・`scrollWidth ≤ clientWidth` 実測で truncated=false）→〈社会：ストリート〉確定→**代用判定へ直行**（指定技能選択は出ない）→用途→カードプレイ→開示・帰結行・公開カードまで無退行。スクリーンショット目視精査。

### 2026-08-26 フェーズ14-9 是正3 指定技能への応答の統合（4系統ダイアログ・指定充足宣言・社会下位区分）

**2026-08-26** / **関連実装** `designation-response(-logic).mjs`(新)・`skill-dictionary.mjs`・`session-logic.mjs`・`info-gathering.mjs`・`tnx-rl-request-app.mjs`・`appearance-check.mjs`・`tnx-hud.mjs`・`scripts/data/item/{general,style}-skill.mjs`・両技能シート・`css/tnx2.css` / **ステータス**: 実装済・隔離 Foundry E2E 実機確認済

#### ルール解釈

- **代用判定は指定技能の代用**（2026-08-25 ユーザー是正 verbatim）であり、選んだ技能の未所持フォールバックではない。応答の決定は「この指定にどう応じるか」の一段＝縦積みボタン1回・4系統（指定所持/代用技能/指定充足/代用判定常設）。未所持指定はグレーアウトで存在を見せる。
- **指定充足宣言は代用技能と別系統**（2026-08-26 裁定）: 「あらゆる業界社会による情報収集判定として扱える」等は、判定種別限定・逐次裁定不要の**指定の充足資格**であり、同一性（組み合わせ資格）を与えない。一本化案は「ルール上存在しない権利をデータモデルが語る」ためユーザーが却下——妥当。
- **社会下位区分をデータに持つ**（2026-08-26 裁定）: 「列挙で代替」の旧決定は団体社会の無限性で前提が崩れた。当時の見送り理由（AE フェーズでの技能再改修への拒否反応）も記録した。既存データは区分フィールド優先＋プレフィックス導出で無移行。
- **登場判定は統合ダイアログに載せない**: 「指定は候補の提示であって制限ではない」（2026-08-07 裁定）と衝突するため、候補グループへの合流の形で指定充足・代用技能を効かせた（設計からの読み替え＝要ユーザー確認事項として明示）。
- 正本の更新: Check_System「指定技能への応答」新設・Character_Creation の下位区分決定・Scenario_Progress のダイアログ形式・2026-08-12「代用判定で受ける」注記。

#### 実装

- 純ロジック: `buildDesignationOptions`（4系統・行ごと・重複排除=指定>代用>充足・checkKind ゲート）・`standInMatchesKey`（あらゆる社会=プレフィックス可・下位区分=辞典の societyClass 必須）・`infoDesignationRows`・`onomasticTypeOf`/`composeOnomasticName`/`SOCIETY_CLASSES`・辞典索引に区分2フィールド追加＋`loadSkillClassByKey`。
- ダイアログ: DialogV2 縦積みボタン（disabled ネイティブ+`tnx-trigger-disabled` 共通見た目）・「（目標値 n）」は nowrap スパン化で括弧を割らない折り返し。判定要求の旧 `_promptDesignatedSkill`・`_promptSkillUse`（モードダイアログ+自動代用遷移）は撤去（撤去規範）。
- シート: generalSkill=区分/下位区分セレクト（固有名詞時のみ・社会時のみ）＋名前プレフィックス表示（入力は固有名詞のみ・保存時合成・旧名は編集時のみ正規化）＋識別キープレフィル。styleSkill=「指定技能として扱う」行 UI（代用対象欄と同じ行0追加ボタン+フィラー行構造）。

#### 監査結果 ✅

- テスト +21＝2174 green・eslint/stylelint 新規指摘なし。
- 隔離 Foundry E2E: 情報収集=4系統ボタン（社会3件グレーアウト+充足技能+電脳+行ごと代用判定）→充足技能で完走（達成値10・開示・帰結行・公開カード無退行）／代用判定タイトル「代用判定: 〈社会：ストリート、企業、真教〉」=指定の束／判定要求=充足なし・目標値なし・グレーアウト／登場判定=充足技能が「社会・コネ」群へ合流／シート=区分選択→キープレフィル・下位区分出現・「社会：千早重工」合成保存・指定充足行の表示。スクリーンショット目視精査（レジェンド折れ・括弧割れの2件を検出し是正）。

#### 是正（2026-08-26・ユーザー指摘）

- **名前プレフィックスの編集モード表示が閲覧モードと不一致**（「謎に大きく発光色」「閲覧モード時との差が考慮されていない」）: Code が根拠なく 20px＋アクセント色を与えていた。閲覧モード（charname-view）と同じ 14px／700／text-primary に統一し、入力欄の左パディングも詰めて「社会：千早重工」が両モードで同一に読める形へ。隔離実機で編集/閲覧の両モードを撮り比べて確認。
- **プレフィックスと入力文字のインライン高さの微差**（同日追指摘・是正2回）: 1回目の是正はプレフィックスの**箱**を入力欄と同一寸法にし、検証もボックス矩形の一致で済ませたため**文字は揃っていなかった**（input 内部のテキスト配置は行高の中央計算と一致しない＝外形を揃えても文字は揃わない。ユーザー「依然として同程度ズレたまま」）。2回目で flex を `align-items: baseline` にして**実テキストのベースラインを直接一致**させ、検証も 3倍スケールのスクリーンショットで文字ピクセルを見る方式に改めて解消を確認。教訓: テキスト整列の検証は箱の座標でなく文字の描画で行う。
- **閲覧/編集モード間のアイテム名の縦ズレ**（同日追指摘・全アイテム共通の根本是正）: 同根の症状が共有ルール `.charname-view`（全アイテムシートの閲覧時名前）と `.charname input`（編集時）の間にもあった。3倍スケールのピクセル解析で**閲覧が編集より 1px 上**と実測し、`.charname-view` のパディングを 4px/4px→5px/3px に配分し直して一致させた（再実測で両モードの文字行が (34,72) に完全一致）。検証はスクリーンショットの輝度行解析（Pillow）＝目視より厳密な方式を導入。
- **是正4＝同一部品化（2026-08-26・ユーザー再々指摘＝最終是正）**: 是正3(input 側 1px 補償)は Code の全計測環境(隔離・ライブサーバー双方・複数スケール)で一致したが、**ユーザーの実クライアントでは 1〜2px のズレが残存**(スクリーンショット提示・固有名詞技能シート)。input の文字配置定数はエンジンのビルドに依存しうるため、**補正の方針自体を放棄**し、閲覧モードの名前を編集と同じ input 部品(readonly+pointer-events none・`.charname-view-input`)で描画する**構造的同一化**へ変更——同一部品+同一ルールのため、どのエンジンでもモード間の文字位置が構造的に一致する。名前表示の全数調査: モード分岐を持つのは 4 シート(general-skill/housing-area/outfit/style-skill)のみで全て置換・残り 4 シート(life-path/miracle/organization/style)は常時 input でズレが存在しない=**全アイテムを網羅**。旧 `.charname-view` ルールと 1px 補償(3px/5px)は撤去(パディング 4px/4px へ復帰・行高 22px は維持)。反省: ①検証対象の取り違え(Code は普通の技能で計測し、ユーザーが見ていたのは固有名詞技能シート——ただし後の計測では固有名詞側も手元では一致しており、真因はエンジン差) ②「直った」の宣言をユーザーの見ている実物で確かめずに繰り返した ③診断スニペットのバグ(v13 の adopted stylesheets を見落とし「未適用」と誤診)。教訓: 環境依存の描画差は補正でなく構造で殺す・視覚検証はユーザーと同じ対象/条件で行う。
- **是正3（同日・ユーザー再指摘「依然として編集モードの方が下がる」）**: 決定実験として**完全同一の幾何**（境界1px＋パディング4px＋行高22px=内容領域と一致）で両者を比較し、**Chromium の input はスケールに依らず span の行ボックスより常に 1px（CSS）下へ文字を描く**定数オフセットを特定（DSF 1/1.1/1.25/1.5/1.75/2/3 の全てで正確に1px）。是正＝①行高を内容領域の高さ 22px に一致させ縦中央寄せの丸めを排除（スケール/ズーム非依存化） ②異常の発生源である **input 側**をパディング上3px/下5px で 1px 補償（共有ルールのため全アイテムの名前入力を一括是正・閲覧側は素の 4px/4px に戻す）。全7スケールで文字行の完全一致を実測。固有名詞プレフィックス行は input へのベースライン追従のため自動的に整合（3倍で確認）。

#### 課題・申し送り

- カスケードの小分類・ソート等の既存プレフィックス依存箇所は今回触れていない（区分ヘルパーへ寄せる余地あり・必要時に）。
- FS 判定種別の指定充足は未配線（enum 拡張のみで受けられる形）。

### 2026-08-26 判定バフの社会下位区分キー（check.society.<区分>）

**2026-08-26** / **関連実装** `scripts/data/item/helpers.mjs`・`tnx-check-flow.mjs`・`scripts/tnx.mjs` / **ステータス**: 実装済・隔離 Foundry で実機確認済

#### ルール解釈

- ユーザー指示「今回の変更(社会下位区分のデータ化)に合わせて、AE キーの判定ボーナスも特定のカテゴリの社会を指定可能に」。旧 Character_Creation 決定で「対象が有限なため列挙で代替」とされていた**区分単位バフ**の正式実装であり、下位区分データの第2の消費者。
- 文法は既存グループ参照(`check.style.*`/`check.works.*`)と同型の `check.society.<下位区分キー>`。「あらゆる社会」は既存のプレフィックス形 `check.society_*` が既に担うため重複させない。未分類(societyClass 空)の社会技能は下位区分キーに合致しない。

#### 実装

- `parseEffectTargetKey` のグループ分岐に society を追加・`checkChangeMatches` は criteria.skills の `societyClass` で照合。
- 照合値は**コピーの値が優先・空なら辞典(+ワールド直下)の値**。`_computeCheckBonus` が同期のため、辞典解決は `TnxCheckFlow.open`(全判定の唯一の入口)で起動時に `ctx.societyClassByKey` へ載せる。再判定スナップショットにも ctx ごと保存される。
- AE 設定シートの候補全列挙へ4実キーを追加(有限値の全列挙規約)。

#### 監査結果 ✅

- テスト +1＝2175 green(照合4系・未分類の不一致・種別ゲート)・eslint 新規指摘なし。
- 隔離 Foundry 実機: ①コピーに industry を持つ〈社会：N◎VA〉の判定に「区分バフ検証 +3」が**1回だけ**載り達成値へ加算 ②コピー未分類+ワールド直下辞典で industry 分類の新技能でもフォールバックで +3 ③非社会技能(医療)の成功判定には載らない。
- 検証中の観測(既存仕様・本件のスコープ外): 独立に作成した同名 AE は identity(effectId|id)が異なるため加算される。重複排除の対象は付与・転送で複製された同一効果(effectId 伝搬)であり、手動の同名別効果は別インスタンス扱い——2026-07-17 裁定の「同名効果」の読みと照らした適否は必要になったら確認。

### 2026-08-27 フェーズ14-8 一括実機確認（アクトのライフサイクル通し）

**2026-08-27** / **関連実装** `session-state.mjs`・`tnx-scenario-panel.mjs`・`subscenes.mjs`・`appearance-check.mjs`・`css/tnx2.css` / **ステータス**: 確認完了（フェーズ14 の完了条件を満たす）

14-2〜14-7 は「実機確認は後続サブフェーズ合流後に一括」（2026-08-08 ユーザー方針）として持ち越されていた。実セッション相当の流れが通る段階に達したため、隔離 Foundry（develop 複製・HEAD `b6dc55b`・Playwright 実操作）で通しの確認を行った。

#### 確認結果 ✅

- **アクト終了**（14-7）: `tnxSceneEnd`→`tnxActEnd` の順で発火。`sessionState` が既定へリセットされ（`actId` のみ保持）、パネルが「開始前」構成（アクト＝読み込み/開始・配布・送信）へ戻る。経験点配布アプリが起動し、その `onFinish` にアクト限定技能の後始末が乗っている（2026-08-13 の順序裁定どおり）。
- **アクト開始**（14-7）: 報酬点（`bounty`←0・`bountyBase`←外界点実効値）と CS が**撹乱した値を上書きして**全キャストに再設定される（`buildPreActInit`）。切り札はキースタイルの識別キー⇔カード画像名で3人分とも正しく配布（`kabuto-wari`/`tatara`/`chakra`）。`tnxActStart`→`tnxSceneStart` が発火し、先頭シーンの入場（シーンカードのドロー・シーンプレイヤーの自動登場・切替チャットとニューロカードの送信）まで完走。
- **シーン進行**（14-2/14-3）: 「次のシーンへ」を3回。毎回 `tnxSceneEnd`→`tnxSceneStart` が正しい順で発火し、`sceneNumber` が 1→4 へ。シーンプレイヤーはハンドアウト参照から解決され（ルーラーシーンは `@ruler`）、前シーンの登場者が退場し新シーンプレイヤーが自動登場する。パネルヘッダー（フェイズ・SCENE n・シーンプレイヤー・登場行・シーンカード・登場中チップ）は表示情報の出し分けを含め正常。
- **サブシーン**（14-4）: 作成→適用（`isCurrentSubScene` が当該行のみ true）→リネーム→並べ替え→上書き解除→削除の一巡が API・パネルとも正常。パネルは横はみ出し 0。
- **舞台裏スポット巡回**（14-6）: リサーチのシーンで「シーンを閉じる」→ 舞台裏が開き、キューが**未登場キャストのみ**で構築される。「次へ」で順に巡回し、回しきると状態表示が**「済」**になりボタンが消える（2026-08-23 の文言是正どおり）。
- **登場判定ダイアログの CSS**（未コミット分の検証）: チェックボックス単独行のラベルを内容幅で確保する規則が効いており、「ゴーストとして登場する」が1行に収まる（checkbox 行のみ `flex: 0 0 auto`／セレクト行は `1 1 0%`）。チェックボックスは `.form-fields` の右端のままで配置は不変。
  - 検証手法の注意: `document.styleSheets` の走査では当該規則が見つからず「未適用」と誤診しかけた（v13 は adopted stylesheets を使う）。**算出スタイル（`getComputedStyle`）を正とする**——2026-08-26 の同種の誤診と同じ罠。

#### 途中で誤って「欠陥」と判断した件（同日撤回）

検証の初回試行では、切り札の自動配布が `neuroDeck.pass()` の「already been drawn」で例外を投げ、`startAct` が部分適用のまま無表示で停止した。Code はこれを KI-043 として起票し、「アクト終了がデッキを回収しないため2アクト目の開始でほぼ必ず踏む」と判断してフェーズの締めを止めた。

**これは誤り**。同日のユーザー是正——「**そもそも、ニューロカードはリセットされている状態しか想定されていません**」「テスト環境ではいちいちリセットしていないだけです」（verbatim）——により、**アクト開始時点でニューロデッキがリセット済みであることは卓の運用前提**であり、部分的にドローされた状態は想定外だと確定した。リセットは RL が HUD のニューロデッキ右クリック「リセットする」（`recall()`）で行う既存操作。KI-043 は撤回し、前提は [Card_Operations](../01_Wiki/Game_Mechanics/Card_Operations.md)「デッキの前提状態」に記録した。

**反省**: 実機で観測した事象そのものは正しかったが、**その事象が起こる前提が卓で成立するかを確かめないまま深刻度を決めた**。検証環境（リセットせず使い回していた develop 複製）の状態を通常運用と同一視したのが原因。観測→severity の間に「この状態は運用上あり得るか」を挟むこと。同型の失敗として、この確認では adopted stylesheets の走査でも「未適用」と誤診しかけている（どちらも**手元の観測をそのまま結論にした**もの）。


### 2026-08-29 フェーズ15-1 時間境界の購読＝適用本体の基盤と効果の持続時間の器

**2026-08-29** / **関連実装** `scripts/module/time-boundary-logic.mjs`（新規・純ロジック）・`scripts/module/time-boundary.mjs`（新規・グルー）・`combat-events.mjs`（`actorExit` 追加）・`appearance-state.mjs`（退場でフック発火）・`tnx.mjs`（購読登録・持続時間タブ注入・`tnxDurationLabel` ヘルパ）・`effects-sheet-mixin.mjs`・`templates/parts/active-effects-list.hbs` / **ステータス**: 実装済・隔離実機確認済（本体 `79dfeeb`）

#### メカニクス監査

- **持続は入れ子の単位であり、上位の境界は下位の単位も畳む**（Time_Management の階層＝アクト > シーン > カット > メインプロセス）。個別の対応表ではなく単位の順序で判定する。カット終了では「メインプロセス中」の効果も失効する。
- **カット進行の終了はシーンを終わらせない**（Combat_Flow「開始はシーンと連動・終了は非連動」）。したがってカット進行終了はカットまでしか畳まず、「シーン中」の効果は残る。RL がシーン内で治療まで認める運用のための非連動をここで壊さない。
- **退場＝そのキャラクターにとってのシーンの終わり**（2026-08-29 ユーザー裁定）。一度退場したら再登場はできない以上、退場後にシーン持続の効果を保つ意味がない。
- **「治療まで」はアクト終了まで残る**（Bad_Status「ダメージが治療されるかアクト終了まで回復しない」）。境界では畳まれない。
- **無期限（持続の指定なし）はどの境界でも失効しない。** 読めない値も無期限として扱う——読めない値で勝手に効果を消さないため。

#### 設計レビュー

- **購読口を 1 本に集約**（`time-boundary.mjs`）。境界ごとの手続きを各機構に散らすと、追加のたびに購読が増え適用順序が決まらなくなる。「何が・いつ・どう畳まれるか」は宣言側（効果の持続欄・`uses.type`・`CONDITION_KINDS`）に置き、ディスパッチャは宣言を読んで適用するだけにする。
- **純ロジックとグルーの分離**（`time-boundary-logic.mjs` は Foundry 非依存・TDD +20 テスト）。境界と持続の対応は `DURATION_RANK` / `BOUNDARY_RANK` の順序比較 1 本で、境界が増えても分岐が増えない。
- **シーン境界を `tnxSceneEnd` ではなく退場で駆動する。** `endAct` もシーン切替も `_applySceneExit()`（全員退場）を**先に**走らせてから境界イベントを発火するため、`tnxSceneEnd` の時点では登場者が 0 人で、購読しても対象が空になる。退場を購読すれば全員に届き、途中退場も同じ経路で処理できる（設計判断としての一本化であって回避策ではない）。
- **失効の対象はアクターに乗った効果だけ。**〈**2026-09-02 訂正**〉この項のうち「アイテムに乗った**供給元の定義**を消してはならない（消すとアイテムの設定そのものが失われる）」は正しい。しかし続けて書いた「用途で対象へ付与されたコピーがアクター側の実体で、持続はそのコピーに引き継がれて失効する——器を定義側に置き、失効を実体側で起こす形が自然に成立している」は**誤りだった**。付与には**アイテム着地**があり（2026-07-13 の v3 再設計＝Code 自身の設計）、その付与コピーはアイテムの上に置かれる実体である。二つある着地の片方だけを見て全体に一般化した記述で、**当時ユーザーへ選択として提示していない**（この項はユーザー裁定ではなく Code 判断）。是正は KI-048（2026-09-02）。
- **適用は activeGM のみ**（他人のアクターを更新する権限が要る。`appearance-state` / `info-gathering` と同じ作法）。
- **持続時間タブはネイティブ欄を隠して置き換える。** Foundry 標準のラウンド/秒は本システムでは動かない（TNX は持続を実時間で測らず、本システムは `worldTime` に一切触れないため実時間で減る経路が無い）。動かない欄を残す意味が無い。保存はフォーム送信（即時 `setFlag` は未保存のフォーム状態を巻き戻す＝2026-07-13 の是正と同じ理由）。

#### 実機確認（隔離 Foundry v13.351・develop 複製・Playwright）

- 持続時間タブに「持続」プルダウン（なし/メインプロセス中/カット中/シーン中/アクト中/治療まで）が出て、選択が送信でフラグに載る（`tnxDuration: "scene"`）。効果一覧の「効果時間」列に「シーン中」が出る。
- `tnxCutEnd` → メイン中・カット中が失効しシーン中/アクト中/無期限は残る。`setAppearing(actor,false)`（実経路）→ シーン中まで失効しアクト中/無期限は残る。`tnxActEnd` → 登場していないアクターでも無期限以外すべて失効。アイテムに乗った「カット中」は残る。
- **実機で見つけて直した点**: ネイティブの持続欄は `<fieldset>` で囲まれており、中の `.form-group` だけを隠すと**空の枠が 2 つ残る**。タブの直接の子ごと隠すよう是正した（DOM を実測して特定）。

### 2026-08-29 フェーズ15-2/15-3 使用回数のリセットと持続の実インスタンス接続

**2026-08-29** / **関連実装** `time-boundary-logic.mjs`（`planItemBoundaryUpdates`）・`time-boundary.mjs`（`resetItemsOn`）・`focus-system-result.mjs`（支援 AE の持続）・`tests/module/time-boundary-uses.test.mjs`（新規）・`tests/module/usage-effects.test.mjs` / **ステータス**: 実装済・隔離実機確認済（本体 `ca39c1c`・`78bf773`）

#### メカニクス監査

- **使用回数の期間は持続と同じ入れ子の単位**（Time_Management の階層）。「カット中 N 回」はカット終了で、「シーン中 N 回」はシーン終了（＝退場）で、「アクト中 N 回」はアクト終了で戻る。上位の境界は下位の単位も戻す。
- **神業は期間の指定を持たないがアクト単位**（Time_Management「神業: アクト単位」）。`uses.type` が空でも type=miracle ならアクトで戻す。神業以外で期間の指定が無いものは境界で戻さない（回数制限の期間が決まっていない＝境界の対象ではない）。
- **消費アイテムの個数はアクト単位**（Time_Management「消費アイテム: 個数はアクト単位」）。アクト終了で常備化個数（`quantity.max`）まで戻る。破壊等で失われた分は別（ここでは扱わない）。
- **FS の支援 AE はカット終了で失効する**（Focus_System「AE の基本 duration はカット終了」）。使われなかった支援がカットを越えて残らない。使われた分は進行判定での消費（AE 除去）が先に働く。

#### 設計レビュー

- **使用回数のリセットは持続と同じ物差しで測る**（`DURATION_RANK` を共用）。別表を作ると単位の対応が二重管理になる。
- **変化しないものは patch に含めない。** 消費していない技能・個数が満ちているアイテムを毎境界で書き戻さない——ユーザーの正本データへの無用な書き込みを避けるため。
- **付与コピーが持続を引き継ぐ分担**: 持続の器はアイテム側の定義に置き、失効はアクターに乗ったコピー側で起こる。`buildGrantedEffectDataFrom` が flags を深いコピーで引き継ぐため追加実装は不要で、この分担を回帰テストで固定した（新規実装ではなく既存挙動の固定）。〈**2026-09-02 訂正**〉この回帰テストが固定していたのは**持続フラグがコピーへ引き継がれること**だけで、**掃引が届く場所にコピーが置かれるか**は一切見ていなかった。アイテム着地の付与コピーはアイテム上に置かれるため掃引に乗らず、持続を持ちながら失効しない（KI-048）。
- **FS 支援 AE の `duration: {rounds:1}` を撤去**。本システムでは動かない指定であり、残しておくと「持続が設定されている」ように見えて実際には効かない。TNX の持続へ一本化した。
- **`check.suitChange` の「1回の判定」は持続の選択肢に加えなかった**。Time_Management の時間単位に無い粒度で、加えるならルール裁定が要る（Active_Effects の申し送りとして残す）。

#### 実機確認（隔離 Foundry v13.351・Playwright）

- カット終了＝カット技能のみ spent 0／退場＝シーン技能も 0／アクト終了＝アクト技能と神業が 0 かつ消費アイテムの個数が 0/3 → 3/3。
- アイテムに乗った効果（持続=カット中）から作った付与コピーはカット終了で消え、**供給元は残る**。〈**2026-09-02 注**〉この確認は**アクター着地**の付与コピーだけを通っており、アイテム着地の側は確かめていない（動く方だけを確認していた＝KI-048 の見落としが実機確認も素通りした理由）。
- 支援 AE と同じ形の効果を付与 → 進行ボーナス 1 → カット終了後 0（効果ごと除去）。

### 2026-08-29 持続時間の選択肢から「治療まで」を削除（15-3 是正）

**2026-08-29** / **関連実装** `time-boundary-logic.mjs`（`TNX_DURATIONS`/`DURATION_RANK`/`BOUNDARY_RANK`）・`tests/module/time-boundary-logic.test.mjs` / **ステータス**: 是正済（本体 `6e2f241`）

ユーザー指摘「持続時間『治療まで』の存在意義が理解できません」。**存在意義は無かった**。

#### 何が間違っていたか

- **「アクト中」と挙動が完全に同一だった。** 持続の判定は単位の深さの比較 1 本で、`untilTreated`(5) と `act`(4) はアクト終了(5)の下で同じ答えを返す——定義されているどの境界でも区別が付かない。名前だけが違う選択肢を 2 つ並べていた。
- **書いていたテストも何も固定していなかった。** 「治療まではアクト終了までどの境界でも失効しない」は `act` に差し替えてもそのまま通る。自分のモデルから書いたテストで自分を検算していた典型（CLAUDE.md §5.1）。
- **繋がる先が無かった。** 治療の対象は負傷であり、RL が手で組んだ効果に「治療する」操作は存在しない。本来の「治療するまで回復しない」BS は**元の負傷 AE の生存から導出**する設計（Phase_15 設計判断7）で、この欄を一切使わない。選んでも「治療で消える」と誤解させるだけだった。

#### 原因

Time_Management の時間単位（メインプロセス中／カット中／シーン中／アクト中）に無いものを、ダメージチャートの `inflicts` が持つ `duration: "治療まで"` という**別系統の表記**につられて選択肢に混ぜた。同じ「持続」という語が指す対象が違う（片方は時間単位、片方は回復条件）ことを見落とした。

#### 是正

選択肢を Time_Management の時間単位＋無期限の 5 つに戻し、アクト終了の深さを 4 に詰めた。BS 側の「治療まで」は設計判断7 の導出（負傷 AE の生存）のまま変更なし。

### 2026-08-29 フェーズ15-4 BS の回復（境界駆動＋行動を支払う回復）

**2026-08-29** / **関連実装** `conditions.mjs`（`recovery`/`downgradeTo`/`payment`）・`time-boundary-logic.mjs`（`planConditionRecovery`/`planActionRecoveryRows`）・`time-boundary.mjs`（`recoverConditionsOn`・プロセス系の購読）・`tnx-character-sheet-base.mjs`・`templates/actor/parts/tab-combat.hbs`・`css/tnx2.css` / **ステータス**: 実装済・隔離実機確認済（本体 `0588ce5`・`db91b40`）

#### メカニクス監査

- **BS ごとの回復条件を宣言として符号化**（Bad_Status の「BS 一覧」がそのまま `CONDITION_KINDS` の `recovery` になる）。恐慌＝本人のメインプロセスの直前／萎縮・憎悪＝**本人**のメインプロセス終了（2026-08-29 裁定・「メインプロセス中」効果の "誰のかを問わない" とは主語が違う）／酩酊(小)・電子妨害＝クリンナップ／重圧・捕縛・邪毒・狼狽＝行動を支払う。
- **酩酊(大)はクリンナップで(小)に変換**。(小)を同時に受けていても (小) は同じ境界で回復するため、結果は (小) が 1 つになる（重複しない）。当初 Code は「(小) が既にあるなら変換しない」と書いたが、(小) も同時に消える以上その心配は起きない——**同じ境界で起こることを同時に見る**必要があった。
- **BS の全解除は背板**（Bad_Status 共通ルール「カット進行が終了するか、シーンが変われば全て解除」）。個別の回復タイミングはそれより早く効く前倒し。**カット終了（次カットへ続く）では落とさない**——落ちるのはカット進行の終了とシーンの変わり目（＝退場）。
- **「治療するまで回復しない」BS のゲートは元の負傷 AE の生存から導出**（設計判断7）。フラグを立て落としせず、負傷が残っている間だけ通常の回復を飛ばす。アクト終了は「ダメージが治療されるかアクト終了まで」の後半なのでゲートを越えて落ちる。
- **放棄は行動を行ったものとみなす**（2026-08-29 裁定）＝メジャーの放棄は AR の消費。したがって通常のメジャー行使と同じ記帳に積み、プロセス終了時に AR−1＋CS カレント 0 が一般則で適用される。
- **回復は「全て」**（2026-08-29 裁定）。捕縛を武器ごとに複数受けていても 1 回のメジャー放棄で全部戻る。したがって **1 BS 種別 = 1 ボタン**。
- **邪毒の宣言は一度**（2026-08-29 裁定）。マイナーの放棄時点でメジャーの放棄まで含む。二段にすると「マイナーだけ放棄してプロセスが終わる」行き止まりが作れてしまう。

#### 設計レビュー

- **UI の置き場は戦闘タブ**（ユーザー裁定）。状態を編集する操作ではなく**アクションを支払う操作**なので、CS・AR・タイミング別の起動チップと同じ面に属する。状態バッジのメニューは効果の編集・治療の入口。
- **押せるのはカット進行中だけ**（Code 判断）。カット進行外は AR という原資自体が存在しない（付与型・戦闘タブも「—」表示）ため支払いが成立しない。行動回数の制限ではなく原資の有無による区別で、「行動回数は自動管理しない」方針と衝突しない。無効化は既存の `applyTriggerDisable` を再利用。
- ~~狼狽は `payment` を持たせず行を出さない~~ → **2026-08-29 に是正**（ユーザー指摘「あなたが記録していないという怠慢を私に押し付けるのをやめてください」）。当初 Code は「Bad_Status に回復条件の記載が無いので UI を出さない」と判断し、回復条件をユーザーに聞き直そうとした。**これは誤り**——回復条件は `conditions.mjs` のコード内コメント（「ムーブ不可＋メジャー達成値-10(回復=マイナー)」）に残っており、**過去にユーザーから伝えられた内容を Code が正本でなくコメントに書いて済ませていた**のが実態だった。記載の空白ではなく記録漏れ。Bad_Status に「狼狽」の項を補填し、重圧と同じ `payment: "minorUse"` で行を出すようにした。
  - **教訓**: 正本に無いルールを見つけたら、まず**自分の記録漏れを疑う**（コード内コメント・実装の既定値に痕跡が残っていないか探す）。聞き直しは痕跡が無いことを確かめてから。
- **`planActionRecoveryRows` の並びはレジストリ順**。受けた順に並べると行が入れ替わって押し間違える。

#### 実機確認（隔離 Foundry v13.351・Playwright）

- クリンナップ: 酩酊(大)→(小)・電子妨害消滅・恐慌と重圧は残る。恐慌は**本人**のメイン開始でのみ回復（他人のメインでは残る＝コンバッタント解決を含めて確認）。カット進行終了と退場で全解除。
- 「治療まで」: 負傷が残る間は全解除でも消えず、負傷を消すと次の境界で消える。
- UI: カット進行外は 3 行とも無効（「カット進行中のみ」）・カット進行中は有効・捕縛の放棄で捕縛 2 つが同時に消えメジャー記帳が積まれる。
- **実機で見つけて直した点**: 共通の `.tnx-btn` が `width: 100%` のため、最初の実装ではボタンが行幅を全部取り**名前が 1 文字ずつ縦に折り返していた**。共通側は触らず（CSS のスコープ規範）、隣接する `combat-ref-line` と同じ体系（固定の名前列 84px ＋行幅 420px ＋内容幅ボタン）に合わせた。

### 2026-08-29 フェーズ15-5 戦闘不能の解決とアクト終了の後始末

**2026-08-29** / **関連実装** `conditions.mjs`（気絶/失神の `resolveAt`・仮死/昏睡の `becomes`・新状態 `incapable`）・`time-boundary-logic.mjs`（`planActEndDamageCleanup`/`buildIncapableEffectData`/`planIncapableExpiry`/`hasIncapable`/`collectLostCharacters`）・`time-boundary.mjs`・`appearance-logic.mjs`・`session-logic.mjs`（`backstageQueue`）・`treatment-flow.mjs`／`recovery-flow.mjs`・`tnx-scenario-panel.mjs` / **ステータス**: 実装済・隔離実機確認済（本体 `7dbf5fb`・`090a3ef`・`345f31f`）

#### メカニクス監査

- **気絶/失神はカット進行終了で自動回復・仮死/昏睡はシーン終了で完全死亡**（Damage_Rules）。前者は BS の全解除とは別経路＝自分の解決条件による（全解除の対象はバッドステータスだけ）。昏睡も表記どおり**完全死亡**（精神崩壊ではない）。
- **「解決」は回復とは限らない。** 酩酊(大)→(小) の緩和と 仮死→完全死亡 の悪化は同じ機構（`resolveAt` ＋ `becomes`）で表せる。当初 `recovery`/`downgradeTo` と名付けていたが、死亡遷移を載せた時点で名前が実態からずれたため改名した。
- **残存ダメージ消去は負傷と非終端の状態だけ**（Scenario_Progress「ポストアクト」）。**終端状態は残す**——キャラロストは後始末で無かったことにするものではない。
- **「行動」はカット進行内の行動に限らない**（2026-08-29 ユーザー指摘）。アクトの大半はシーン進行で、そこでの判定・行為も行動。Code は当初「メインプロセス不可／リアクション不可」というカット進行内の語だけで「2シーン後から行動可」を捉えており、**盤面に居るが動けない**という誤った像で設計しかけた。正しくは**何もできない**——ユーザー明示の帰結は「**登場判定が自動失敗**」「**舞台裏の手番が回らない**」。
- **抹殺の「アクト終了時の適用」＝ロスト確認に載せること**（2026-08-29 ユーザー裁定）。抹殺はもともとマーカーで、`blocksMainProcess` の対象外。適用されるアクト終了の後には塞ぐべき行動が無いため、提示以上の機構を持たせない。

#### 設計レビュー

- **`system.physicalDamage` 等の数値欄には消費者が一つも無い**（全数 grep で確認）。ダメージは負傷 AE で表現されているため、残存ダメージ消去は AE の除去で行い、使われていない数値欄は触らない。
- **行動不可は状態（可視）として持つ。** フラグだけの見えない効果にしない（2026-08-05 の是正と同じ理由）。期限はシーン番号で数え、効果側の `conditions.incapable.actableFromScene` に持つ。
- **付与は治療の除去経路2つ（所有者が直接消す／GM へ委譲）で共用する `buildPostTreatmentRest`。** 境界での死亡遷移やアクト終了の後始末は別経路（time-boundary）を通るため、そちらでは乗らない＝**治療のときだけ乗る**という切り分けが構造で保証される。`deleteActiveEffect` フックで一律に拾う案は、死亡遷移（仮死の効果が消える）でも発火してしまうため採らなかった。
- **登場判定の強制失敗はシーン側の設定より優先。** 行動不可はキャラクター側の絶対条件のため。判定自体はブロックしない（2026-08-15 裁定）。
- **舞台裏の除外は RL の手動追加より優先**（何もできない、が絶対条件のため）。

#### 実機確認（隔離 Foundry v13.351・Playwright）

- カット進行終了で気絶が消え仮死は残る／退場で仮死が完全死亡へ変わり気絶は残る。
- アクト終了で負傷・気絶・仮死が消え抹殺は残り、故障/破壊が解除される。
- 治療で行動不可が乗り期限＝現在シーン+2／登場判定が `incapable` で強制失敗／舞台裏の列から外れる／期限で外れる。
- ロスト確認は終端状態を持つ 2 名だけを挙げ、気絶のキャラクターは挙げない。

### 2026-08-29 フェーズ15-6/15-7 邪毒の継続ダメージと社会ダメージ系の自動化

**2026-08-29** / **関連実装** `time-boundary-logic.mjs`（`planPoisonTicks`/`planSceneDeferredFiring`/`planSceneDeadlineExpiry`/`appearanceBlockOf`/`buildForcedExitFlags`）・`condition-resolution.mjs`（`applyPoisonTick`）・`time-boundary.mjs`（`registerForcedExitWounds`）・`damage-chart.mjs`（soc-17）・`appearance-logic.mjs`・`appearance-check.mjs` / **ステータス**: 実装済・隔離実機確認済（本体 `36b846d`・`f18af4e`）

#### メカニクス監査

- **邪毒はクリンナッププロセスのたびに「山札から1枚引いて出た数字 ＋ n」点の肉体ダメージ**（Bad_Status）。Bad_Status の申し送りは「フェーズ13」を指していたが**未実装のまま残っていた**もの。ジョーカーは既存の扱い（引き直し or 値の指定）を踏襲し、邪毒は「出た数字」だけを使うためスートは聞かない。
- **カードを引くのは人**（2026-08-29 ユーザー指示で是正）。当初 Code は境界でカードまで自動で引いていたが、「**自動でカードを引かれると訳が分からないことになる**」。想定は受けたキャラクターを操作しているプレイヤーか RL。システムはクリンナップに受付カードを出すだけにし、衰弱/重圧と同じ「効果決定」カードの「山札を引く」ボタンで引かせる（カードを2枚に分けない既存の作法もそのまま）。
  - **教訓**: 「自動適用」と「自動でカードを引く」は別物。卓がカードを引く行為は結果の帰属が卓にあり、システムが代わりに引くと何が起きたのか追えなくなる。適用の自動化は、**その手前にある人の操作まで奪う理由にはならない**。
- **逮捕令状はチームから抜けてから退場**（2026-08-23 裁定＝強制退場は退場連動に乗せない）。以後この負傷が生きている間は登場判定が自動失敗になり、負傷自体は「次のシーン」が終われば消える（付与シーン+2 から自由＝2026-08-29 裁定）。
- **「次のシーン」効果（社会6/7/8 信用失墜・スキャンダル・信頼喪失）は、発火機構が無いあいだ永遠に休眠したままだった。** 付与時は `pendingScene` で休眠し、消費側（技能ブロック・報酬点ブロック）は既に `pendingScene` を見ていたのに、`sceneFired` を立てる側が存在しなかった。シーン開始で立てることで初めて効くようになった。
- **逮捕令状は登場を塞ぐだけで、舞台裏の可否は対象にしない。** ルールに記載が無いため（行動不可は「何もできない」と明示裁定があるので舞台裏も塞ぐ）。**記載の無いところを埋めない。**

#### 設計レビュー

- **邪毒は「誰にいつ」と「ドローと適用」を分けた。** 対象と強度の決定は純ロジック（`planPoisonTicks`・乱数を持ち込まない）、山札のドローとチャート適用は既にそれを持つ `condition-resolution` に置き、`time-boundary` は境界の判断だけを担う。
- **邪毒のダメージは黙って適用しない。** 2026-08-29 の「自動適用・報告なし」は回復・リセット・失効の話で、キャラクターが受けるダメージは他のダメージと同じ見え方をすべきもの。既存の状態解決カード（`condition-outcome`）の意匠で、カード＋強度の内訳を全行表示する。
- **期限と登場不可の機構を一般化した。** 行動不可（治療後2シーン）と逮捕令状（次のシーンまで）はどちらも「このシーン番号から自由」という同じ形なので、`freeFromScene` ＋ `planSceneDeadlineExpiry` に統合。`appearanceCheckParams` も真偽値ではなく**理由コード**（`appearanceBlock`）を受け取る形にして、両方が同じ口に乗るようにした。
- **逮捕令状の適用は activeGM のみ。** チーム（ワールド設定）と他人のアクターを更新する権限が要るため、`createActiveEffect` を GM 側で1回だけ購読する（ダメージを適用したのが誰であっても同じ結果になる）。

#### 実機確認（隔離 Foundry v13.351・Playwright）

- 邪毒: 山札 108→107・カード 4 ＋ 強度 3 ＝ 7 点の肉体ダメージが適用され `phys-7` の負傷が付く・チャットカードに内訳が出る。
- 逮捕令状: 付与でチーム 1→0・登場 true→false・期限＝シーン3+2=5・登場ブロック `arrested`。シーン4 では消えずシーン5 で消える。
- 「次のシーン」効果: 休眠 `true` → 発火後 `false`。

#### 申し送り（コードにだけ書かれていた未自動化の効果）

15-8 の掃き出しで見つかったもの。**フェーズ15（時間境界）の範囲外**（どちらも付与時の効果）なので実装していない。置き場所はユーザー裁定を待つ。

- **驚愕**（精神チャート）: アクションランク −1。
- **士気喪失**（精神チャート）: アクションランク 0・可能なら戦闘中止。

※チャート各項目の効果文そのものは §4.1 により wiki に網羅しない方針のため、ここでは「機械的な帰結が未接続である」という事実だけを記録する。

### 2026-08-30 RL 経験点の計算式是正と配布ダイアログのメモ表記廃止

**2026-08-30** / **関連実装** `exp-award-logic.mjs`（`calcRlExpBreakdown` 新設・`calcRlExpTotal` 是正）・`tnx-exp-award-app.mjs`・`templates/app/exp-award-app.hbs`・`css/tnx2.css`＋tests / **ステータス**: 是正済・隔離実機確認済

#### メカニクス監査

- **RL の経験点＝会場の手配（1点）＋『プレイヤーの取得経験点の合計』÷『3かプレイヤー人数の小さいほう』（切り捨て）**（2026-08-30 ユーザー是正）。除数が min(3, 人数) であり、商と人数を比べるのではない。
- 旧記録（Scenario_Progress「`÷3` もしくは `プレイヤー人数` の少ない方」）がこの誤読で書かれており、14-7 の実装 `min(floor(PL合計÷3), PL人数)` はそれに追随していた。正本の記録も同時に是正した。
- 誤式は RL の取得を PL 人数（数点）で頭打ちにしており、正式（およそ PL 1人分の取得に相当）と大きく乖離していた。

#### 設計レビュー

- 内訳（PL合計・除数・配分・合計）を `calcRlExpBreakdown` に一本化し、`calcRlExpTotal` はその total を返すだけにした（ダイアログの表示と記帳が同じ式を読む）。除数 0（PL 0人）は配分 0＝0除算しない。
- **RL 行の擬似コードメモ「会場の手配＋min(プレイヤー合計÷3, プレイヤー人数)」を廃止**（UI 文言規約＝静的な解説文は置かない。再三の是正対象がここに残っていた）。置き換えは実数の内訳表示「プレイヤーの取得経験点の合計 17 ÷ 3 ＝ 5」＝全て動的データ（除数 0 のときは非表示）。会場チェックボックスは「会場」列に整列させた（従来は「依頼」列の下に置かれ、メモセルに隠れてずれが見えなかった）。
- CSS: `.exa-rl-note` → `.exa-rl-share`。旧規則の `text-align: left` は `.exa-table td`（詳細度で優越）に負けて元々死んでいたため引き継がず、表の他セルと同じ中央寄せに従う。
- 検証: テスト 2287 green（本件 +3）・eslint 0 error（警告15は既存）・stylelint clean。

#### 実機確認（隔離 Foundry v13.351・Playwright）

- PL 3人に 17 点（5+5+神業2／5）を実クリックで入力し会場チェック → RL 行「プレイヤーの取得経験点の合計 17 ÷ 3 ＝ 5」・合計 6。
- RL 行のセル数はヘッダー12列と一致（1+1+1+colspan8+1）・横はみ出し 0・内訳は1行に収まる。

### 2026-08-30 経験点配布「全て自動で入力する」ボタンと登場シーン数の記帳

**2026-08-30** / **関連実装** `exp-award-logic.mjs`（`sumMiracleSpent`/`buildAutoFilledRow`）・`session-logic.mjs`（`recordSceneAppearance`）・`session-state.mjs`（`appearanceCounts`/`appearedThisScene`・`registerAppearanceExpTracking`）・`tnx-exp-award-app.mjs`（`collectExpAutoFill`・autoFillAll）・`tnx-scenario-panel.mjs`・`tnx.mjs`・`exp-award-app.hbs`・`tnx2.css`＋tests / **ステータス**: 実装済・隔離実機確認済

#### メカニクス監査

- **チェック8種は全て ON で埋める**（2026-08-30 ユーザー言明・verbatim）:「基本的によっぽど厳密に裁定しない限り、経験点の取得条件は全て満たされたものとして配布することがほとんどです」。Code 当初案の「会場だけ自動 ON から除く」等の気の利かせは「余計なことをしないでください」で却下——ボタンの語義どおり全部 ON、修正は RL の個別操作。
- **神業欄＝そのアクトで消費した回数**（`uses.spent` の合算・ユーザー承認）。配点条件の「上手く使った」の判断は自動化しない——自動値は下書きで、上手くなかった分は RL が減らす（14-7 の半自動裁定・§4.2 の枠内）。アイテムロールを通さず口頭処理された神業は数字に乗らない。
- **登場欄＝アクト中に登場したシーン数の実数**（ユーザー承認）。1シーン1点なので同一シーン内の再登場（退場→再登場）は数えず、途中退場してもそのシーンの1は残る。巡回シーンの再入場は別シーン（シーン番号の意味論と同じ）。上限5は従来どおり合計側（`calcPlayerExpTotal`）が掛け、入力欄は実数のまま。
- RL 行（会場チェック）はボタンの対象外——実世界の事実であり、PL 側の集計から導けるものではない。

#### 設計レビュー

- **元データの採取はアクト終了処理より前**が本質: 神業の `uses.spent` は `tnxActEnd` 境界のリセットで 0 に戻り、sessionState も既定化されるため、配布アプリが開いた時点では消えている。パネル `_onEndAct` が `endAct()` の**前**に `collectExpAutoFill()` で採取し、`awardContext.autoFill` としてダイアログへ渡す。
- **登場シーン数の記帳は activeGM の updateActor 購読一点**: 登場フラグ（`appearing`）の書き込み元（登場判定・RL の登場操作・台本の事前設定・トークン配置）がどのクライアントでも false→true 遷移を一点で拾え、ワールド設定への書き込み権限も GM 側で完結する。連続登場（事前設定の一括登場等）で増分が失われないよう、書き込みは1本のキューに直列化。数えるのはキャストのみ（配布が読むのは割当キャスト＝cast 型のみ・2026-08-09 裁定）。
- sessionState の追加欄は `appearanceCounts`（累積・Actor id→数）と `appearedThisScene`（シーン内重複防止）。アクトの読み込みで両方リセット（DEFAULTS 経由）・シーン入場で後者のみクリア。クリアの setState は自動登場ループより**前**にあるため、新シーンの自動登場が正しく数えられる。
- ボタンは押すたびに自動値で上書き（冪等）。記帳は従来どおり「確定」まで行わない。純ロジック（`recordSceneAppearance`/`sumMiracleSpent`/`buildAutoFilledRow`）を分離して TDD。
- ヘッダ配置の注意: `.tokyo-nova .tnx-btn` は**全幅既定**（D&D 流の縦積みボタン）のため、ヘッダ内では `width: auto` に戻す——初回描画でボタンが全幅を取り、アクト名が縦書きに潰れた（実機確認で発見・是正）。
- 検証: テスト 2297 green（+10）・eslint 0 error（警告15は既存）・stylelint clean。

#### 実機確認（隔離 Foundry v13.351・Playwright）

- 記帳: 登場で 1・同一シーンの退場→再登場は増えず・別キャストは独立・シーン替え後の再登場で累積 2。
- 採取→ボタン実クリック: autoFill が {神業2, 登場2}/{0, 1}/{0, 0} を返し、全行チェック8個 ON・カウント充填・合計 36/33/32。RL 行は非対象（会場チェック不変）・内訳「プレイヤーの取得経験点の合計 101 ÷ 3 ＝ 33」。
- レイアウト: ボタンは内容幅で右寄せ・アクト名と同一行・横はみ出し 0。RL 会場チェックの「会場」列整列も DOM 実測（cx 455/454）で確認。

### 2026-08-31 フェーズ16-1 分類基盤（副分類・isCyber 完全統合・製作技能の対応分類）

**2026-08-31** / **関連実装** `483e7f4`＝`outfit-categories.mjs`（`outfitClassifications`/`hasClassification`/`majorOfMinor`/`buildCategoryKeyGroups`）・`outfit-base.mjs`（`additionalCategories` 新設・isCyber 廃止＋migrateData 移行）・`helpers.mjs`・`part-helpers.mjs`・`conditions.mjs`・`repair-flow.mjs`・`outfit-dictionary.mjs`・`tnx.mjs`・シート2種＋テンプレート＋`tnx2.css`＋tests / **ステータス**: 実装済・隔離実機確認済（ユーザー実機確認は 16-5 一括）

#### メカニクス監査

- **「複数の分類を持つアウトフィットは両方（すべて）の分類として扱う」**（2026-08-30 ユーザー言語化・正本 Purchase_and_Modification）を分類集合（主分類＋副分類）の一本化ヘルパーで符号化した。分類を読むルール挙動——部位ホスト照合・「※」表示・電子妨害カウント・修理対応分類・サービス免疫（故障/破壊されない・バックグラウンドの準備/携帯強制）・AE 分類狙い——は全てヘルパー経由に統一。
- **挙動変化（原則の帰結・設計時に明示済み）**: 電子妨害は従来 isCyber 横断を数えていなかったため、体内武器等（副分類サイバーウェア）が新たにカウント対象になる。AE の分類狙い（`system.category.<キー>`）も副分類に当たるようになる。
- **isCyber は副分類に完全統合**（2026-08-30 ユーザー裁定 verbatim）: フィールド廃止・シートの特徴フラグ撤去・「主分類サイバーウェア→自動セット」撤去。「※」表示は「主分類がサイバーウェア以外で分類集合にサイバーウェアを含む」に読み替え（表示規則の意味は不変）。
- **製作技能の対応分類**: 区分=製作のときのみ表示・サービスとその配下は選択肢に存在しない・小分類も選べる（2026-08-30 ユーザー裁定どおり）。読み手は 16-2（辞典フィルタ）・16-4（改造判定）で接続する。

#### 設計レビュー

- **データ移行は migrateData（遅延・非破壊）**: isCyber=true かつ主分類≠サイバーウェアの既存データは、ドキュメント構築時に副分類サイバーウェアへ読み替えられる（DB の書き換えは次回保存時）。**一括書き換えは行っていない**。
- **辞典 index の生データ互換**: `pack.getIndex` は migrateData を通らないため、`outfitClassifications` が旧 isCyber=true を副分類サイバーウェアとして包摂する（読み替えは一点・恒久の常設移行表ではなく分類集合の定義の一部）。部位行の保存済み記述子 `hostFeature: "isCyber"` も同様に読み替えで互換維持。
- **AE 着地点 isCyber の消滅**: AE_FLAG_PARAMS から除去＝AE でサイバーウェア分類を付与する経路は無くなった（配列フィールドは AE change の素直な対象にならないため。必要になれば別設計）。
- 副分類 UI はユーザー指定どおり（1行目右端チェック＝ツールチップ・2行目＋・3行目以降ゴミ箱）。チェック状態は副分類の有無から導出（状態レス）。行の選択肢は全分類の樹（主分類の型絞りを適用しない＝型跨ぎの横断がまさに用途）。
- 選択肢ビルダー `buildCategoryKeyGroups` を新設し、修理対応分類（用途シート）の同型実装を統合（横断機構の単体関数化）。
- `modifiableParams` 撤去は読者ゼロ（定義と空配列初期化のみ）を grep 全数で確認してから実施。
- 検証: テスト 2325 green（+28）・eslint 0 error（警告は既存分）・stylelint clean。

#### 実機確認（隔離 Foundry v13.351・Playwright）

- migrateData: isCyber=true の武器構築→ `additionalCategories=[{cyberware,""}]`・isCyber 消滅・装飾名「…※」。主分類サイバーウェアは移行なし・※なし。
- 副分類 UI 実クリック: チェック展開→アイテム/生体装備→＋→サイバーウェア（大分類のみ）→保存配列一致→ゴミ箱削除→閲覧表示「武器／白兵武器 ＋ アイテム／生体装備」→チェック解除で全消去。右端列（チェック/＋/ゴミ箱）の縦整列・はみ出し 0。
- 製作技能: 区分=製作で「対応分類」出現（optgroup にサービス無し）・白兵武器を保存・区分を社会へ変えるとクリア。
- キャストシートのアウトフィットタブ描画正常（既存どおり行名に※は付かない＝従来仕様・退行ではない）。

#### 是正（2026-08-31・`7a6ddc1`）: 固定分類の disabled プルダウン化と分類行の縦位置統一

- ユーザー指摘（実 UI スクリーンショット）: ①大分類固定型（タップ等）で小分類プルダウンがラベル残り全幅に伸びて長すぎる ②1行目と副分類行の縦位置が揃っていない。
- 対応＝**disabled プルダウン方式を採用**（提示二択のうち Code 推奨）: 大分類固定・両分類固定の文字列ラベル表示を廃止し、disabled の select で表示。1行目が副分類行と同じ select＋select＋右端固定幅列の構造になり、位置合わせ CSS なしで整列する。固定であることは disabled の標準表現（既存規約 opacity 0.5・not-allowed に従う）。
- 右端列（チェック/追加/削除）の基準幅は 20px 指定がチェックボックスの min-content（26px）に押し広げられて select 幅に 3px 差を生んでいたため、実寸 26px に統一。
- 隔離実機で 3 型（大分類固定 tap／両分類固定 cyborg／非固定 weapon）の select 座標を実測し、全行 x=419/660・幅 233 の完全一致を確認。

### 2026-08-31 フェーズ16-2 辞典ブラウザ基幹とアイテムカード・ツールチップ

**2026-08-31** / **関連実装** `8fd44d4`＝`tnx-dictionary-browser.mjs`(アプリ)・`dictionary-browser-data.mjs`(タブ定義/読み込み/絞り込み純ロジック)・`dictionary-cards.mjs`(共用カードビルダー)・`outfit-view.mjs`(略号行の切り出し)・`item-card-tooltips.mjs`(シート側配線)・`identification.mjs`(`decoratedItemName` 関数化)・テンプレート2種＋`tnx2.css`＋system.json(サンプルPC辞典)＋tests / **ステータス**: 実装済・隔離実機確認済(ユーザー実機確認は 16-5 一括)

#### メカニクス監査

- タブは 2026-08-31 ユーザー明示の8種をこの順で実装(スタイル技能/アウトフィット/一般技能/スタイル/神業/オーガニゼーション/ライフパス/NPC)。**ドロー表は入れない**(裁定=埋め込み参照・メカニクス参照の性格)・**サンプルPC辞典もブラウザ非掲載**(裁定どおり compendium 一覧にのみ出る)。
- 表示形式は 2026-08-30 裁定どおり: スタイル技能=スタイル別見出し＋7項目パラメータ行カード/アウトフィット=大分類・小分類見出し＋略号行カード/一般技能=リスト＋D&D状態異常風カード/スタイル=スタイルデータ骨格(キャッチコピーなし・神業を `miracle.id` ライブ解決で合成・能力値/制御値4枠)/神業=名前(ふりがな)・使用回数・条件・解説(種別フラグ非表示)/ライフパス=種別ごと表形式/オーガニゼーション・NPC=リスト。
- 絞り込みは分類集合(`outfitClassifications`)経由=**副分類でも大分類/小分類フィルタに合致**する(生体武器を「武器」で発見できる)。一覧の**束ねは主分類**(置き場所の権威・副分類は絞りにのみ効く=Code 設計判断)。
- ツールチップの適用範囲=シート上で内容が見えないアイテム行(一般技能・スタイル技能・アウトフィット・神業・スタイル)。所持アイテムは**実効値**・辞典は素値(ビルダーが total 優先で読む)。

#### 設計レビュー

- カードは**単一ビルダー＋単一テンプレート**(dictionary-card.hbs・kind 出し分け)にし、ブラウザとシートツールチップが同じ部品を使う(2026-08-30 ユーザー指定の「同じ形のカード型」を構造で保証)。
- アウトフィット略号行はシートの `_prepareView` 内ローカル実装を `outfit-view.mjs` へ移設し、シート・ブラウザ・ツールチップの3読者で共用(`formatWeaponRangeLabel` も同居へ移設・importer 2箇所付け替え)。`tnxDecoratedName` ヘルパーの実体も `decoratedItemName` として関数化。
- 読み込みは全タブ getIndex(fields:["system"]・NPC はメタのみ)=KI-026 準拠。index は migrateData を通らないため分類照合は分類集合が旧 isCyber を包摂。
- ツールチップ配線は HUD シーンカード(14-8)と同じ data-tooltip-html 方式。シートはレンダー後に非同期で属性を流し込む(ホバー時には出来上がっている)。
- 神業・スタイルの「条件」は StringField だがリッチテキスト(HTML)が入っている実データを実機で発見→専用枠で HTML 描画に是正(パラメータ行に入れると生タグが見えた)。
- description はエンリッチ(＠リンク解決)未適用=リファレンス埋め込みサブフェーズで統合予定。D&D の「高度」トグルは持たない(フィルタ数が少ないため・Code 判断)。
- 検証: テスト 2340 green(+15=絞り込み/整列/タブ定義の純ロジック)・eslint 0 error・stylelint clean。

#### 実機確認(隔離 Foundry v13.351・Playwright・実パック+播種テストデータ)

- 起動ボタンが辞典タブ上部に表示・クリックでブラウザ起動。8タブすべて描画し件数がパック実サイズと一致(styleSkill 1/outfit 4/generalSkill 28/style 31/miracle 31/organization 1/lifePath 3/npc 3)。
- 大分類=武器フィルタ→3件(主分類武器2＋**副分類武器の生体剣**=束ねはアイテム側のまま)・検索・購入値レンジ動作。
- ブラウザのリスト行ホバーとキャストシートのアウトフィット行ホバーの両方で共用カードのツールチップ表示(「体内ブレード※ 武器／白兵武器 ＋ サイバーウェア」=※装飾と副分類表示も確認)。
- スクリーンショット目視: アウトフィット/スタイル技能カードのルルブ形式・スタイルカードの神業合成・一般技能リスト・ライフパス表(アイコン折返しを是正)・条件のリッチテキスト描画。

#### 是正（2026-08-31・`f5e85e1`）: 辞典ブラウザへの一斉指摘対応

ユーザーから「全然ダメ」として7項目の指摘。全て是正し、誤読・検証不備の原因も記録する。

- **一般技能の誤読（最重要）**: 「リスト形式を意識しつつカードに」を Code は「リスト＋ホバーでカードのツールチップ」と読み違えて実装した。正しくは**ブラウザにカード（横幅広め・縦積み）を直接表示**し、アクターシート側のツールチップがその表示を踏襲する、という関係。ブラウザは是正済み（正規ソート順も適用）。
- **ツールチップの右寄せ**: 実機スクリーンショットで**全要素が右寄せ**になっていた。Code は当初「コア `#tooltip` の寄せ既定の継承を打ち消していなかった」と機序で説明したが、**ユーザーに「言い訳として成立していない」と却下された——日本語は基本右寄せで書かれない。右寄せの日本語画面は一目で異常であり、見ていれば必ず気づいた**。つまり本質的な原因は継承仕様ではなく、**自分の出力を見なかったこと・「正しく書かれた日本語に見えるか」という最低限の基準を当てなかったこと**にある（CSS の継承は「どう壊れたか」の機序にすぎず「なぜ納品されたか」を説明しない）。是正＝text-align の明示リセット＋左寄せの実測確認。規律＝**撮った画像は全部目視し、日本語として正しい組版かを最初に見る**。
- カードの寸法: 固定幅 252px・種別ごと固定高（styleSkill 330 / outfit 285・**tall(ヴィークル/全身義体/式神装備)380=4/3** / style・organization 430 / miracle 370=手持ち最長が最小フォントで収まる実測値）。**入りきらない解説・条件は文字サイズを段階縮小して収める**（`fitDictionaryCards`・あふれは desc 内側と card 突き抜けの2経路を測る——当初 desc しか測らず神業の条件が下端で切れたのを実機で発見して是正）。全カードタブで overflow=0 を実測。
- ワークス絞り込み: スタイル技能タブへワークス専用技能を移設し「ワークス技能」チェック＋所属組織（`special.works`）・組織別グループ掲載。アウトフィットに「ワークス専用装備」チェック＋所属組織（専用 `exclusive` の組織参照）。一般技能タブは単一辞典となり参照元グループを撤去。
- スタイルカードから神業を撤去（神業タブと重複＝Code 合成案の撤回）・style/miracle は**登録順（document sort）**でソート。
- ふりがな: ルビ化（`<ruby><rt>`・名前と同色・名前幅に均等割り）。アイテムシートの `furigana-input`/`furigana-view` も text-muted → text-primary へ是正（もともとの表示も同罪との指摘）。
- オーガニゼーション＝スタイルと同様のカードグリッドへ。NPC タブから cast 型を除外（キャストは辞典に載せない）。
- 検証: テスト 2347 green（+7）・eslint 0 error・stylelint clean。隔離実機で全タブ再撮影＋実測（幅252/高さ285・380、縮小フォント 7.6〜10.1px で全文収容、ツールチップ text-align:left、NPC にキャスト不在、ワークスフィルタ→組織グループ「千早グループ」表示）。

#### 追補（2026-08-31・`68eb073`）: 寸法裁定の実装とコア UI の N◎VA テーマ化

理解確認を経てユーザー承認済みの5項目＋「UI改修は前フェーズまでの負債」としてのコア UI 全面テーマ化。

- **カード寸法の最終形**: スタイル技能/アウトフィット=**222px 固定**（デフォルトウィンドウ幅 980 で結果エリア実効幅 684（スクロールバー溝 10px 確保後）に余白込みちょうど3枚。ウィンドウを広げても大きさ不変・枚数が増える=実測 980 で3枚/1300 で4枚）。スタイル/神業/オーガニゼーション=**2列固定・幅はウィンドウ追従**（980 で 338px/1300 で 498px を実測）。一般技能=全幅追従。リサイズ時は ResizeObserver で縮小フィットを再計算。`scrollbar-gutter: stable` でスクロールバー出没による幅揺れを防止（226px 案は溝の 10px を見落とし2枚落ちした——実測で発見し 222px へ確定）。
- **起動ボタン**: 辞典タブ**ヘッダーのボタンエリア**（Create Compendium 等の action-buttons 行）へ移動。資料タブの前例（renderJournalDirectory の「アクトシートを作成」）と同方式・flex-wrap 折返し・**独自スタイルなし=標準ボタンの見た目**（スタイル二択は「まず標準に合わせ、UI 側の全面 N◎VA ナイズも実施」の両取りで決着）。
- **サンプルPC辞典**: system.json `packFolders` のシステムフォルダへ追加（入れ忘れの是正）。
- **コア UI テーマ**: コアのテーマ変数を `body.system-tokyo-nova-axleration`（theme-* と同特異度・読み込み順で優越）で上書き——warm 系(橙)→シアン・cool 系(紫黒)→青黒・light/dark スケール→青系・text/scrollbar/checkbox。**チャットはコアが変数を `.chat-message` 要素自身に定義するため、継承では届かず同要素スコープで上書き**（v13 はチャットログを themed theme-light 領域として扱う設計。当初 body 上書きで通知ポップアップが羊皮紙のまま＝実機で発見し是正）。whisper/blind/emote も系統色化・bright ボタンの橙固定ホバー(#fda948)をシアン化・ドロー表罫線の固定色も是正。
- 検証: テスト 2347 green・eslint 0 error・stylelint clean。隔離実機で暗色盤面の全景・チャット展開（通常/ウィスパー/エモート）・通知ポップアップ・アクター/辞典ディレクトリ・ボタン配置・寸法(DOM実測)をスクリーンショット全数目視。

#### 追補2（2026-08-31・`db40d21`）: ドラッグ有効化・ツールチップ固定サイズ化・輪郭是正

- **シートのアイテム行ドラッグ**: 配線(DragDrop・_onDragStart・_onSortItem)は存在したが `dragstart` 権限が**編集モード限定**で、閲覧モードのアウトフィットタブが「ドラッグできない」状態だった。ゲートを isEditable のみに緩和(2026-08-31 ユーザー指示=並び替え・ワールド/辞典への持ち出し対応)。E2E で閲覧モードの並び替え([A,B,C]→A を C へドラッグ→[B,C,A])を実測確認。持ち出しは標準ペイロード(item.toDragData)のためディレクトリ・未ロック辞典への D&D がコア処理で通る。
- **ツールチップの固定サイズ化**(2026-08-31 ユーザー指示=辞典ブラウザと同じ大きさのカード状): スタイル技能 222×330・アウトフィット 222×285(tall=ヴィークル/全身義体/式神装備 380)。ツールチップは表示時に JS を実行できないため、**オフスクリーンの固定サイズスロットで文字縮小フィットを事前確定**した HTML を data-tooltip-html へ注入する(buildSizedTooltipHtml)。寸法定数はブラウザ CSS と対で保守(TOOLTIP_CARD_SIZES)。実測 222×285/222×330・一般技能等は従来の可変(幅上限420)のまま。
- **ツールチップの輪郭**(2026-08-31 指摘=アウトラインが暗く下地と同色で見分け不能): カード枠をアクセント色(#00c8e0)+グローに。実測 borderColor rgb(0,200,224)。
- 検証: テスト 2347 green・eslint/stylelint clean・隔離実機 E2E(ドラッグ実操作・ホバー実測・スクリーンショット目視)。

#### 追補3（2026-08-31・`0847e3e`）: 長名ツールチップ崩れの是正

- ユーザー指摘「著しく崩れたツールチップ＝長いアイテム名の未考慮」を隔離実機で再現: 縮まない分類ラベル(flex 0 0 auto)がヘッダの名前列を幅ゼロまで圧縮し、**名前が1文字ずつの縦積み**になってヘッダ 571px・横はみ出し 124px・固定カードを 376px 突き抜けていた。
- 是正(実装中のユーザー追指示で方針確定): **名前と分類の横並びをやめ縦積み**(分類は名前の直下・折り返し可)。**一般技能カードのパラメータ行(初期スート・特性等)も2列をやめ1行全幅の縦積み**(右に空きがあるのに半分幅で改行される指摘)。保険として本文最小でも収まらない極端長名はヘッダ文字も段階縮小(fitDictionaryCard 第二段・通常は発動しない)。
- 実測: 長名33字+副分類2件でヘッダ 571→123px・横はみ出し0・固定カード内に全収容(名前3行折返し)。長ふりがなのルビ折返し・一般技能の全幅パラメータ行・ブラウザ側カード(短名)の退行なしをスクリーンショット全数目視。
- 教訓(自己記録): 固定幅カードの新設時に**長い動的文字列の見本での検証を最初から行う**(feedback_self_review_ui_output の既存規律の適用漏れ)。

#### 追補4（2026-08-31・本体1コミット）: アイコンの名前隣接固定

- 長名時に**アイコンだけ別行へ折れる**のは一貫性に欠けるとの指摘。アイコン+名前(+英名・分類)を不可分ブロック `.tnx-dict-card__identity`(flex basis 0・min-width 0)に束ね、名前がどれだけ長くてもアイコンが名前のすぐ左に付く構造へ。長名は identity 内側の titles で折り返す。実測=アイコン右端と名前左端が8px間隔で隣接・同一行帯・カードはみ出し0。短名カードは見た目不変。

#### 追補5（2026-08-31・本体1コミット）: 略号グリッドのセル幅をルルブ配置に統一

- 住宅のセ(電／ア)等で長ラベルが数値を途中分割する不具合の是正。ユーザー提供のルルブ実紙面10ページを正本に、型ごとの固定セル幅配置（1/2・1/3・全幅）を `buildOutfitSummaryRows` の width 指定として符号化（詳細表=Game_Rules/Outfits.md の「略号グリッドのセル幅配置」節）。当初 Code は「部位=長くなるデータ持ちの型は全幅」という型別例外を推定したが、実紙面照合で否定（武器・義体とも電制|部位ペア）。タップのソ|ハ|CS 3連・住宅の登単独行・式神のタイプ/部位単独行・最小単位の購|隠|電制 3連（p91）も実紙面から確定。
- 値は nowrap＋実測昇格（1/2→全幅・1/3は3連ごと1/2へ格下げ）。列間ギャップ10→6px（1/3セルに「30／2」級の値が収まる幅の確保）。
- 適用3読者=辞典カード・ツールチップ・シート説明タブ（シートは `style-skill-display-grid--outfit` 修飾で同配置）。隔離実機で住宅/タップ/最小単位/防具の4型を実測（全行1行表示・分割ゼロ・3連/単独行の座標一致）＋武器長名ツールチップ・スタイル技能/一般技能カードの回帰確認済み。テスト2347件グリーン。

#### 追補6（2026-08-31・本体1コミット）: 溢れの逃がしを文字幅圧縮へ改定＋データ文字縮小

- 追補5の実測昇格（1/2→全幅・1/3は3連ごと1/2格下げ）は「項目が下の行に移る」としてユーザー指示で撤回。溢れた**値のみ** scaleX で文字幅圧縮して収める方式へ（ラベル非圧縮・シートの squeeze-text と同手法・transform-origin 左）。型別の静的配置（全幅/1/3幅）は不変。
- 略号グリッドの文字サイズを 0.9em→0.8em（枠に対して大きいとの指示）。
- 実測: 購「300／12」が1/3セルのまま scaleX(0.85) で収まり行送りゼロ・クリップなし。4型ツールチップ＋ブラウザカードの回帰確認済み。テスト2347件グリーン。

### 2026-08-31: 16-3 購入判定

**メカニクス監査**（ルール解釈の正しさ）:
- 正本=Purchase_and_Modification.md「購入判定」。経路3分岐を purchase-logic.decidePurchasePath に符号化: ①購入値「ー」「解説参照」=購入手続き自体が存在しない（判定ブロックとは別の整理・ボタン不能化）②購入値≤外界実効値=常時入手（境界=同値も入手・判定なし）③超過=判定（TN=購入値）。
- カードなし特例（Check_Rules の信用特例）: 達成値=外界実効値＋消費報酬点。汎用判定フローに分岐を作らず購入フロー内で処理（実装方針どおり）。報酬点は消費時点一元ゲート（hasBountyBlock）＋system.bounty 実減算（判定の報酬点と同じ着地）。差分値は成功時のみ表示（既存エンジンの規約に一致・隔離検証で発見し是正）。
- 使用技能=用途タイプ「購入」（2026-08-31 裁定・詳細は Purchase_and_Modification.md）。購入用途を持たないアクターはカード判定の選択肢をグレーアウト（常時入手・カードなし特例は用途に依存しないため可能）。
- 成功の帰結=辞典原本（fromUuid live 解決）の複製付与・1判定=1個。再判定は rerunOnSuccessOnly（失敗→成功の遷移でのみ付与）。

**設計レビュー**（技術的妥当性）:
- 実装形は登場判定と同型: 起動入口（ブラウザの購入ボタン）→ 唯一の起動関数 `_activateItemCheck`（usageId 直接指定＋TN＋継続文脈 ctx.purchase 注入）→ CONTINUATIONS.purchase＋_execute 分岐。純ロジック（purchase-logic）と Foundry 依存（purchase-flow）を appearance-check/logic と同じ分割にし vitest 6件を併設。
- 方式選択=縦積みボタンダイアログ（designation-response 前例・disabled でグレーアウト提示）。報酬点入力=_promptBountyUsage 再利用。ボタン不能化=applyTriggerDisable 共通機構。
- 結果カード: カード判定=check-result 基底に購入行＋帰結行（cr-flow-outcome=帰結行クラスの汎用名を追加）。常時入手/カードなし=purchase-result.hbs（tnx-check-result 意匠共用・内訳全行）。
- 「改造して入手」（変則効果）: grantPurchasedItem(actor, uuid, modSpec) の modSpec が席。16-4 で改造専用修正フィールドと同時に有効化。
- 隔離実機確認済（全経路: ボタン表示/不能化・常時入手（確認→付与→カード）・カードなし成功（報酬点10消費→付与・実減算10→0）/失敗（非付与・差分値なし）・カード判定（TN注入ダイアログ→山札プレイ→失敗カード・購入行表示・非付与）・完了継続（失敗=非付与/成功=付与）・実行アクター解決（選択トークン優先→担当キャラ）・購入用途なしアクターのグレーアウト）。テスト2359件グリーン。ユーザー実機確認は 16-5 一括。

#### 追補（2026-08-31・本体1コミット）: 16-3 購入用途のアイテムロール起動

- 購入用途を技能(アイテムロール)から起動した場合、素の判定でなく**アウトフィットのみの辞典ブラウザ(選択モード)**を開き、対象の購入ボタンで起動元の用途の判定へ直接合流する(2026-08-31 ユーザー指示=D&D 5e のドロップエリア起動の絞り込みブラウザと同型・理解確認を経て承認)。
- 方式選択(用途選択＋カードなし特例)はスキップ=技能から起動した時点で「カードで判定する」意図が確定しているという読み(理解確認済み)。常時入手(購入値≤外界)と不能化は通常経路と同じに生きる。カードなしで買う場合は従来どおりブラウザ起点。
- 実装: `_activateItemCheck` に購入用途の分岐(カバーと同じ「起動→選択→文脈つき再入」型・新しい起動関数なし)。選択モード=TnxDictionaryBrowser.openOutfitPicker(別 id で通常ブラウザと共存・タブレール非表示・`tnx-dict--picker` の2列グリッド)。確認キャンセル・不能警告時は選択ブラウザを閉じない(alwaysAcquire が入手成否を返す)。
- 隔離実機確認済(選択モードの表示/不能化・購入品→判定ダイアログ合流+ピッカー閉鎖・常時品→確認 No=開いたまま/Yes=付与+閉鎖・通常ブラウザ経路の回帰=方式ダイアログ・タブレール無傷)。初回実装でタブレール除去により2列ずれるレイアウト崩れを自分で発見し是正(ピッカー用グリッド定義)。テスト2359件グリーン。

#### 追補2（2026-08-31・本体1コミット）: 16-3 プレアクト購入＋入手フラグ・経験点計上の是正

- アクト未開始時の通常ブラウザ購入＝**プレアクト購入**（既存 `isPre-play` フラグの活用・裁定詳細は Purchase_and_Modification.md）。判定フローを走らせない即時取得（プレアクトは手札未配布・報酬点未設定のため操作的にも判定は成立しない）。チャットカードは出さない（卓の進行外・通知＋シート反映のみ）。技能起点はアクト状態にかかわらず通常の購入判定。
- 不能化はレンダー後の共通グレーアウトのまま判定関数を切替（`decidePreActPurchase`=購入値＋常備化経験点・純関数でテスト4件追加=2363件）。
- 是正: 判定入手の isCheckAcquired 未設定＋経験点計上の isPre-play 未読（**KI-044**・Resolved 起票）。
- 経緯の反省: 当初 Code は「プレアクト購入=常備化（経験点計上）」と真逆の理解を提示した。原因は機能の実名（プレアクト）で正本を検索せず、自分の仮説語（常備化経験点・消費経験点）だけで探索して最初に見つかった isCheckAcquired で打ち切ったこと。正本（Outfits.md の状態フラグ表）には定義が明記されていた。
- 隔離実機確認済（プレアクト時のグレーアウト切替=常備化経験点「ー」不能・購入→確認→isPre-play付与・経験点0／プレアクト中の技能起点=通常判定合流／アクト中=通常3分岐の回帰／判定付与=isCheckAcquired・経験点0）。

#### 追補3（2026-08-31・本体1コミット）: プレアクト購入の成立条件の是正

- 追補2のプレアクト購入は**条件を確認せず無条件付与＋常備化経験点「ー」不可**で実装しており誤り（ユーザー指摘「ただ入手されるだけになっています。プレアクト購入の条件を確認していませんね？」）。正本=同日ユーザー verbatim（Purchase_and_Modification.md に記録）: 成立条件＝**購入値 ≤ そのキャストの外界（実効値）**・**常備化経験点による取得は入手であっても購入ではない**（誤ゲート撤去=常備化経験点「ー」でも購入値が外界以下なら可）。
- 外界条件はアクター依存（クリック時に選択トークン/担当キャラで解決）のため、ボタンのグレーアウトは購入値基準のまま・フロー側で判定して警告。確認ダイアログに条件（購入値 X は外界（Y）以下）を明示。
- 経緯の反省（2連続）: フラグ設計時（2026-06-13）に条件を聴取せず、今回も条件の存在を確認しないまま実装した。ルール提供を受けた時点での即 verbatim 記録を徹底する。
- 隔離実機確認済（外界5のキャストで: 購入値15=警告・非付与／購入値3=確認→付与 isPre-play／常備化経験点「ー」購入値4=付与可・経験点0）。テスト2365件グリーン（条件テスト更新+2）。

#### 追補4（2026-08-31・本体1コミット）: プレアクト購入条件の確定(常備化経験点ゲート復元)

- 追補3で撤去した「常備化経験点「ー」不可」はユーザー verbatim「常備化経験点が「-」のアイテムはプレアクト購入できません。正しいです」で復元。条件の確定形=①購入値が値を持つ ②常備化経験点が値を持つ ③購入値≤外界実効値（正本 Purchase_and_Modification.md に verbatim 2件と経緯を記録）。
- Code の誤読の記録: 「常備化経験点による取得は入手であっても購入ではない」を「常備化経験点欄は購入条件に無関係」と読み替えて②を撤去した。正しくは常備化と購入の**概念の区別**（常備化は購入の手続きではない）であり、欄の条件性とは別問題。
- ②はデータ固有＝プレアクトモードのグレーアウトで提示（理由ツールチップ）・③はアクター依存＝クリック時警告。隔離実機確認済（常備化「ー」購4=不能表示／購3・常備化1=付与 isPre-play／購15=外界超過警告）。テスト2365件グリーン。

#### 追補5（2026-08-31・本体1コミット）: プレアクトの外界不足ボタン不能化＋常備化「ー」のインポート弾き

- プレアクト時、購入値>外界のアウトフィットは購入ボタン自体を不能化(2026-08-31 指示「押せない状態にした方が良い」)。外界は描画時点の実行アクター(選択トークン→担当キャラ)で解決・選択変更に再レンダーは追従しないため、クリック時のフロー側ゲートを最終防衛として維持。
- 常備化経験点「ー」のアウトフィットは**ドラッグ&ドロップによるインポートのみ**を弾く(2026-08-31 指示・中間確認で「あくまで弾くのはドラッグアンドドロップによるインポート」と限定)。実装=_onDropItem の早期ガード(住宅エリア弾きと同型)。対象アクター=キャストと**所有アクターがキャストのトループ**(2026-08-31 ユーザー指示で2段階に確定: 「所有アクターが設定されている場合のみ」→「所有アクターがキャストの時のみ」。所有者はドロップ時に live 解決して型判定)・所有者なし/ゲスト所有のトループ・ゲスト/エキストラは RL の自由装備のまま。購入フローの付与(isCheckAcquired/isPre-play つき複製)とシート上の新規作成は対象外。
- 隔離実機確認済(プレアクト・外界5: 購8/購15=外界超過で不能・常備化「ー」=不能・購3=可/キャストへの常備化「ー」ドロップ=非付与・常備化ありドロップ=付与・ゲストへの同ドロップ=付与)。テスト2365件グリーン。

### 2026-08-31: 16-4 改造判定

**メカニクス監査**（ルール解釈の正しさ）:
- 正本=Purchase_and_Modification.md「改造判定」＋分類別テーブル(2026-08-30)＋重複規約(2026-08-31 verbatim)。テーブルは modification-params.mjs に符号化し、行ごとにテストで正本照合(19件): 小分類特則(IANUS/全身義体/タップ/住宅施設/ドラッグ)が大分類行に優越・複数分類は和集合(「両方の分類として扱う」)・該当なしは「その他」(隠・電制)。
- 重複規約: 1つのアウトフィットの1つの項目への改造は技能を問わず1回(項目選択で改造済み不能表示＋適用時二重ガード)・別項目は同一技能で重複可。
- 項目選択は判定前(2026-08-31 裁定)。＋[レベル]=当該改造技能のレベル実効値。ヴィークル分類の制御値修正は合算後に最大0クランプ。住宅の登場判定目標値は±二択・防/セは一括加算。「ー」「解説参照」は項目単位で不可、目標値式が購入値参照(@outfit.buy)の技能では購入値「ー」の対象を選択不可。
- ドラッグ特殊(2026-08-31 ユーザー提案の実装形): 所持ドラッグを[レベル]個まで選択し、各ドラッグの用途タイミングを実効解決で一時的にマイナーアクションへ上書き(設定欄の素値は不変・改造行の削除で解除)。
- 組み合わせ技能の適否(対応分類の一致)はシステムで制限しない(一般則)。

**設計レビュー**（技術的妥当性）:
- 表現形=専用修正フィールド `modifications`(2026-08-30 裁定): 素値不変・prepareDerivedData の computeItemEffectiveValues 直後(アクター段 AE の前)に合流=AE は改造込み total の上に重なる。シート説明タブに改造セクション(行+出所スナップショット+削除)・略号行/辞典カード/ツールチップは実効値を自動反映。
- フローは修理と同型(useModification→完了継続 ctx.modification・rerunOnSuccessOnly・GM委譲=modificationApply ソケット・帰結カード=postConditionOutcome)。起動は唯一の起動関数の type 分岐。TN 注入=targetValueExtra.outfit(治療の condition と同型)。
- 「改造して入手」有効化: 購入用途の acquireModified(用途シート発動タブ・購入タイプ時のみ表示)。判定前に項目選択→purchase 継続の modSpec→grantPurchasedItem が複製の改造記録へ書き込み。結果カードに改造行(purchaseCardInfo.modText)。
- タイミング読者の切替は単一ヘルパー effectiveUsageTiming に集約(戦闘タブ分類・メジャー記帳の2読者)。
- 隔離実機確認済(対象選択→項目選択(武器の4項目)→判定 TN=@outfit.buy=8・成功適用=攻S+4→S+6・失敗/二重適用は非適用・改造済み項目の不能表示・ヴィークル制-1+2→0クランプ・住宅登場10→8(素値不変)・ドラッグ複数選択UI(改造済み不能・最大[レベル]個)+実効タイミング action/minor・シート改造セクション+削除で実効値復帰・用途シートのトグル・改造して入手=判定前項目選択→複製に改造行+isCheckAcquired・帰結カード4種目視)。テスト2385件グリーン(項目レジストリ19件+購入カード2件追加)。ユーザー実機確認は 16-5 一括。

#### 追補6（2026-08-31・本体1コミット）: スロットゲージの二段表示

- KI-046 是正の続報: 改造・AE でスロット容量が増えるとピップ列が名前列で見切れる(ユーザー報告)。解決形=**ゲージがある行は「アイテム名(1段目)＋ゲージ行(2段目)」の二段表示**(ユーザー提案・Code の閾値切替案[ピップ→固定幅バー]より採用)。ゲージ行は折り返し可(flex-wrap)＝容量がどれだけ増えても構造的に収まる(見切れ禁止の一般則・Code 補強)。ゲージなしの行は従来の単段のまま。
- 隔離実機確認(タップにソ+6/ハ+4の検証行→0/8・0/5 の二段表示・はみ出し実測ゼロ・単段行の回帰なし)。テスト2387件グリーン・stylelint クリーン。

### 2026-08-31: 16-x リファレンス・リンク埋め込み

**メカニクス監査**: ルール解釈なし(UI/参照基盤)。ドロー表の裁定(ブラウザに載せない・埋め込み参照)の受け皿として、コアの @UUID コンテンツリンクで「埋め込み参照→クリックで表を開く」が全表示面で成立する。リンクからの自動ドローは未実装(必要になったら追加=YAGNI・設計提示済み)。

**設計レビュー**:
- 方針(2026-08-31 ユーザー承認): ゼロから参照機能を作らず、コアの @UUID リンク(エディタへのドキュメントドラッグで挿入)を土台に「読む全表示面での解決」と「辞典アイテムリンクのカード型ツールチップ」を仕上げる。
- エンリッチ全面化の掃引結果と着地: ①辞典カード=buildDictionaryCard の中央シームで一括(16-2 申し送りの統合) ②技能解説の展開パネル=enrichedSkillDescriptions マップ(lookup 参照) ③アウトフィット行展開 ④ライフパス表 ⑤情報カード=enrichInfoCardData(純関数 buildInfoCardData を汚さない描画シーム・チャット/HUDツールチップ/開示カードの3読者) ⑥アクトコントロール送信カード4種=renderChatCard 単一シーム(text/handout/scene-switch/info・フィールド名は session-logic のビルダーと静的照合) ⑦シナリオシート閲覧ビュー6面=prose-mirror(toggled) の表示側にエンリッチ済み HTML・**value プロパティは素データ保持を実機確認**(編集・保存の安全性=データ破壊なし)。
- リンクのカード型ツールチップ: item-card-tooltips の機構を一般化(buildCardTooltipHtml を行ツールチップと共用・全カード種別の固定サイズ LINK_TOOLTIP_SIZES)。適用面=チャット(renderChatMessageHTML)・アクター/アイテム/シナリオシート・ジャーナルページ・辞典ブラウザカード内。
- リンク意匠は既存テーマ上書きで N◎VA に馴染むことを実機確認(シアンのピル)・追加 CSS なし。
- 隔離実機確認済(ブラウザカード内リンク解決+ホバーで辞典カード・チャット解説カードのリンク+ツールチップ・スタイル技能パネルのリンク+ツールチップ・一般技能はカードツールチップ側で解決・シナリオシート閲覧ビューのリンク解決+value素データ不変+フラグ無傷)。テスト2387件グリーン。ユーザー実機確認は 16-5 一括。

### 2026-09-01: ウェット/対象条件によるダメージ・効果の無効化

**メカニクス監査**: ユーザー言語化(2026-09-01)=「スタイル技能の中には『ウェットの対象には効果がない』ことが明記されているものがいくつか存在する」「特定のダメージ増強のみ効果がない場合もある。カード分のダメージや、ウェットに効果がないという記述がないダメージ効果のスタイル技能と合わせて、ダメージをプラスするがウェットには効果がないスタイル技能を組み合わせた場合、そのスタイル技能の効果だけが適用されなくなる」「ウェットだけでなく、特定スタイルへのダメージ増加も同じような方法で設定できる必要があるかもしれない」。

- **粒度が2つある**という読みを採った: ①用途**全体**が効かない(技能そのものの記述) ②**1つのダメージ増強だけ**が効かない(組み合わせの中の1行)。②が要望の核で、「行ごとに条件を持てる」ことが必須。
- **ウェットの判定条件**は新設せず、電子妨害のウェット分岐と**同一定義**を関数化した(`isWetActor`＝準備中/準備不要のアウトフィットに識別キー `background_wet`)。ルール上の「ウェット＝サイバーウェアを持たない」の表現はアウトフィット1件で表され、辞典データ側の入力がその実体(2026-09-01 ユーザー裁定でキー確定)。**判定の根拠を二重に持たない**ことが正しさの条件。
- **判定はブロックしない**: ウェットが対象でも判定・対決は行える(既存の一般規範)。無効化されるのはダメージと適用効果のみ。判定ボーナスに対象条件を持たせないのは、判定の評価時点で対象が確定していないため(仕組み上の理由であってルールの制限ではない)。
- **無効化の可視化**: 無効化された行は内訳から落とさず「値 0 ＋理由」で残す。ルールの効果が「働かなかった」ことも卓が読める情報である、という扱い。

**設計レビュー**:
- 4 つの表現に整理(用途が細かい順): 行の対象条件 / 自身の修正値の対象条件 / 式の `@target.isWet` / 用途トグル `noEffectVsWet`。加えてキャラ恒常のフラットな対象依存として AE `damage.vsWet[.系統]` / `damage.vsNotWet[.系統]`(既存 `vsStyle`/`vsWorks` の対称を完成させる位置)。
- 対象条件は `{kind: none|wet|style|works, mode: exclude|only, key}` の1形。**ウェット専用の作りにせず**、ユーザー要望どおりスタイル/ワークスも同じ機構に統合した(専用フラグを増やさない)。照合は純粋関数 `target-condition.mjs`(Foundry 非依存・テスト可能)。
- **対象未解決ではゲートしない**(条件を検査できない状況で無効化しない)を全経路の規約に。ダメージ台帳・効果トレイ・AE 照合(`isWet` 未供給は不一致)で一貫。
- 表示は帰属名と理由を分離(`sources[].note`)。表示側が「ダメージ修正（名前・理由）」と組むため、括弧の二重化と見切れを回避(可変長ラベル行に `cr-calc-row--wrap` を新設して折り返し)。
- 隔離実機確認済: `isWetActor`(準備/準備不要/未準備/別キー)・行と自身の条件のゲートと注記・`@target.isWet`・AE の vsWet/vsNotWet・用途シートの条件セレクト(種類変更でキー/極性の出し入れ・none 復帰でリセット)・**攻撃判定→攻撃カード→ダメージカードの統合経路でフラグが運ばれること**・ダメージ台帳の 0 行と理由・効果トレイのウェット行グレーアウトと付与スキップ(ウェットに効果が乗らず非ウェットには乗る)。テスト2409件グリーン。
- **未解消の制約(KI-047 起票)**: 複数対象攻撃では、対象依存の加算が先頭の命中対象で評価される既存の近似のため、対象ごとに違う結果にならない。単体対象では正しく、用途トグルは対象ごとに評価するので影響を受けない。修正はダメージカードのフラグ構造の変更を要するため、§4.3 に従い設計合意を待つ。

### 2026-09-01: ダメージ評価の単位を対象行へ（KI-047 解消）

**メカニクス監査**: ユーザー確定=「先日対象に関するチャットカード上の表示を D&D を踏襲したものにしました。これを受けて、複数対象への攻撃自体をそもそも対象ごとに評価すべきだと思います。判定はもちろん基本一括ですが、一行ずつ評価するイメージです」。

- ルールの読み: 攻撃は 1 回の判定で複数の対象を巻き込むが、**ダメージの成立は対象ごと**(防御力・受け値・軽減 AE・対象条件つきの増強がそれぞれ違う)。一括なのは「判定＝カードプレイと達成値」「ダメージカードの値・攻撃力・手動修正」まで。この線引きはルール側の要請であり、実装の都合(共有台帳)で対象依存の値まで共有していた旧実装が誤りだった。
- **表示と計算の単位を一致させる**という原則が判断の軸。D&D 踏襲で対象行が表示の単位になった以上、計算も対象行で回る。近似(先頭対象で代表)は、表示が対象ごとである限りユーザーから見て不整合になる。

**設計レビュー**:
- ダメージカード生成時に対象ごとへダメージ修正を評価し(`evaluateDamageBonusesPerTarget`)、各対象行が `bonusRows` を持つ。対象非依存の AE(`damage.dealt`)は 1 回だけ集計して全対象へ同じ行として乗せる(共有行として畳まれる)。
- 表示の畳み込みは純関数 `splitSharedBonusRows`(damage-logic.mjs)。行の同一性は「帰属名・注記・値」の組で、AE の対象条件つき行は対象により個数が変わるため**多重集合**で突き合わせる(位置ベースでは対応が崩れる)。全対象に共通する個数分だけ共有へ。
- 合計の入口を `damageRollTotals(f, { bonusRows })` に一本化。台帳の「攻撃側合計」は**共有行だけの数字**(対象に依らない)で、対象固有の差は各対象行の内訳に出る。`modifyDamage` の上書き基準・GM の手動修正の基準も同じ数字に揃えた(表示と操作の基準がずれない)。
- 旧カード(`bonusRows` なし)は `f.damageBonuses` へフォールバック＝過去のチャットカードも従来どおり描画・適用できる。
- 対象の解決に失敗した行があっても内訳の対応がずれないよう、適用ダイアログは元インデックス(`srcIndex`)で対象ごとの行を引く。
- **カード追加時の再評価は不要**と確認した: 式の `@card` は**命中判定のカード値**(`f.cardValue`)であり、ダメージカードの追加では変わらない。設計提示時に「追加のたびに再評価が要る」と述べたのは誤りで、実装では行っていない。
- 隔離実機確認済: ウェット1体＋非ウェット1体への同時攻撃で、共有台帳=共通行のみ(攻撃側合計17)・ウェット対象17(無効化2行を理由つき0で表示)・非ウェット対象27(+7・+3 が効く)・適用ダイアログのプレビューと適用済み表示も対象ごと・チャート結果も別(腰部損傷/頭部損傷)。テスト2415件グリーン。

### 2026-09-02: アイテムに着地した付与コピーの失効（KI-048 是正）＋前提の記録の訂正

**2026-09-02** / **関連実装** `scripts/module/time-boundary-logic.mjs`（`planItemGrantExpiry` 新設・`planEffectExpiry` の doc 訂正）・`scripts/module/time-boundary.mjs`（`expireEffectsOn`）・`tests/module/time-boundary-logic.test.mjs`（+6） / **ステータス**: 是正済・隔離実機確認済

**メカニクス監査**:

- **持続は「効果の実体」に属する**。用途で付与された効果は、着地がアクターであれアイテムであれ**一回性のインスタンス**であり、宣言された持続で失効しなければならない。着地先で持続の有無が変わるのはルールの側に根拠が無い。
- **アイテムに乗る効果は三種で、資格が違う**——供給元の定義（アイテムの設定そのもの。境界で消せば設定が失われる）／転送コピー（供給元が正の従属物。除去は供給元の側の事象で起こる）／付与コピー（一回性の実体）。境界で消してよいのは三つ目だけ。
- ユーザー指摘（2026-09-02・verbatim）「アイテムへの付与が持続時間を無視しているというのは初耳です。これは信じがたい裏切りと言わざるを得ません。なぜ勝手にそのようなことをしたのですか」。**この挙動はユーザーの裁定ではなく Code の判断**で、選択として提示されていなかった。

**設計レビュー**:

- 判別を**純関数**（`planItemGrantExpiry`）へ置き、グルーは「計画を受け取って消す」だけにした（掃引の他の段と同じ形）。三種の区別が一箇所に書かれ、テストで固定できる。
- 判別の印は `grantedFrom`。アイテム着地の付与は用途の経路（`grantUsageEffect`）だけが作り、そこは常に供給元 uuid を持つ実効果からの複製なので必ず印が付く。RL 任意付与はアクターにしか着地しないため、印を持たない付与コピー（プリセット・その場で組んだ効果）はアイテム上に現れない。
- **TDD**: 6 件を先に書いて赤を確認（`planItemGrantExpiry is not a function`）してから実装。付与コピーは消える／供給元の定義は残る／転送コピーは残る／畳まれない持続は残る、を個別に固定した。
- **記録の訂正**: 15-1・15-2/15-3 の該当項目に訂正注記を入れた。誤りは「掃引の範囲」そのものより、**抜けを「自然に成立している」と正当化して記録に焼き込み、ユーザーの判断機会を消した**ことにある（§6 の運用＝記録の書き手は Code・承認はユーザー、に反する）。

**実機確認（隔離 Foundry v13.351・develop 複製・Playwright）**:

- 武器（識別キー狙い・攻撃力 +3・持続「シーン中」）を、チャットカードの「効果を適用」（実 DOM のボタン押下）で付与 → 攻撃力 5 → 8・付与コピーが武器上に載る（`grantedFrom` あり・持続 scene）。
- 退場＝シーン境界を適用 → **付与コピーが消え攻撃力 5 に戻る**。同じ武器の転送コピー（`transferredFrom`）と供給元アイテム上の定義は**いずれも残る**。

**この検証で見つけた別件（KI-049 / KI-050 起票・別コミットで是正済み＝次エントリ）**:

- **KI-049**: 複数アイテムの一括追加で転送コピーが個数分だけ多重に作られる。`materializeItemTransfers` の「探す→無ければ作る」が非同期フックの同時多発に対して原子的でない。実操作相当の再現＝4 アイテム同時ドロップで転送コピー 4 個・攻撃力 5 → 13（正しくは 7）。
- **KI-050**: 新規に追加したアイテム**自身**が持つアイテム狙い AE が転送されない。`createItem` フックが追加されたアイテム自身を走査から外しており（新しいアイテムを「狙われる側」としてしか見ていない）、Foundry はアイテム内包で作成された効果に `createActiveEffect` を発火しないため他の経路でも拾えない。

### 2026-09-02: アイテム狙い AE の転送の実体化を是正（KI-049 多重転送 / KI-050 未転送）

**2026-09-02** / **関連実装** `scripts/module/serial-queue.mjs`（新規・`runSerial`）・`scripts/data/item/helpers.mjs`（`planTransferCopySync`）・`scripts/tnx.mjs`（`materializeItemTransfers`・`createItem` フック）・`scripts/module/tnx-usage-sheet.mjs`（`updateUsageActions` を共通機構へ）・テスト +9 / **ステータス**: 是正済・隔離実機確認済（本体 `8b3955d`・`b5f1da2`）

**メカニクス監査**:

- ルール上の意味は変えていない。**「供給元が正の片方向同期」（2026-07-12 確定）の不変条件を、実装が守れていなかった**のを守らせただけである。守るべき不変条件を明文化した——**供給元 × アイテムごとに転送コピーは 1 つ**。
- 卓から見た症状は「強化が余計に乗る」（KI-049）と「強化が無言で効かない」（KI-050）で、どちらもルールの表現ではなく実体化の取りこぼし。

**設計レビュー**:

- **競合の正体は「読む → 判断する → 書く」の非原子性**。`createItem` はアイテムごとに発火して全供給元を走査するため、一括追加では同じ (供給元, 対象) に対する判断が並走し、全部が「コピーが無い」と結論する。**アクター単位の直列化**（`runSerial`）でこの窓を閉じた。
- **直列化は共通機構として切り出した**。2026-07-17 に用途配列（`system.actions`）へ入れた直列書き込みキューと**同じ機構**であり、二つ目を書き足すと同型の実装が分岐する。`serial-queue.mjs` に一本化し、既存の `updateUsageActions` もこれを通す（挙動は不変・孤児回避の正準解決はそのまま）。
- **収束する形にした**（`planTransferCopySync`）。「1 つを残して他を除去」を計画として返すため、**過去に多重作成されたデータもその供給元の次の変更で畳まれる**。読み込み時の一括修復は**行わない**——ユーザーの正本データを常設経路で書き換えないため（同値の更新では Foundry の差分が空でフックが発火しないので、収束は実際の変更で起こる）。
- **KI-050 はフックの走査対象の誤り**。「新しいアイテムは狙われる側」としか見ていなかった。追加されたアイテムは**供給元側にもなりうる**——この非対称に気づけなかったのは、転送の設計を「既存の供給元 → 新しい対象」の向きだけで考えていたため。
- **KI-050 に純ロジックの追加は無い**（フックの走査対象の一箇所）。抽出できる判断が「アクター自身＋全所持アイテム」以上の内容を持たず、テストが何も固定しないため。隔離実機での確認をもって検証とした。

**実機確認（隔離 Foundry v13.351・develop 複製・Playwright）**:

- KI-049: 対象武器を含む 4 アイテムの同時ドロップ → 転送コピー 1・攻撃力 7（修正前は 4 個・13）。手で 3 重にしたデータは供給元の実変更で 1 つへ収束（攻撃力 8）。狙いを外した供給元のコピー除去も従来どおり。
- KI-050: 対象武器を置いたアクターへ「その武器を狙う効果」を内包したアイテムを単独でドロップ → 即座に転送コピー 1・攻撃力 5 → 9。続けて無関係なアイテムを 2 つ足しても多重にならない。

**教訓**: KI-048 の検証で出力に並んだ「転送コピー 14 個」は、当該検証の合否には無関係な数字だった。**見ている対象の合否だけを読んで、視界に入った異常を流さない**——ここで流していれば、値が黙って多重に乗る不具合が卓に残った。

### 2026-09-02: 技能・神業を狙う AE をキャラクター付与へ（乗り先の種別で経路を分ける）

**2026-09-02** / **関連実装** `scripts/data/item/helpers.mjs`（`isOutfitItem`・`analyzeGrantLanding(changes, items)`・`planCapabilityTransferCleanup`）・`scripts/data/actor/common/character-base.mjs`（`_resolveBuffApplications` の skill/category を遠隔適用へ）・`scripts/tnx.mjs`（`materializeItemTransfers` の転送先をモノに限定・`cleanupCapabilityTransferCopies`＝ready の一回限り掃除・設定 `capabilityTransferCleanupScheme`）・`scripts/module/usage-effects.mjs`（付与先の所持アイテムで着地を決める）・テスト +18 / **ステータス**: 実装済・隔離実機確認済

**メカニクス監査**: ユーザー言語化（2026-09-02・verbatim）「アイテムのパラメータ変更などは、以前にも話した通り、D&D のエンチャント用途に近い挙動をすべきだと思ったのでアイテムに直接転送ですが、技能のレベルに関してはキャラクターが持つ技能のレベルとして扱うべきで、キャラクター付与の方が良いと思います」。分かれ目を「**アウトフィット＝モノ／それ以外＝キャラクターの一部**」で引き、神業（`uses.max`＝母数）も技能と同じ側に置くことを提示し「基本的にはそれであっています」で確定。

- **技能のレベルはキャラクターの能力の値**であり、モノの性能ではない。効果の実体はキャラクター側に住み、そこから当該技能の実効値へ届くのが正しい。着地点（`levelTotal`）は変わらず、**効果の実体がどこに住むか／どう届くか**だけが変わる。
- **モノ（アウトフィット）は従来どおりエンチャント**＝対象アイテムへ物理転送する（2026-07-13 の裁定はモノ側でそのまま維持。当時の争点「転送された効果が対象アイテムに無条件で効くこと・対象アイテム上に見えること」はモノにだけ当てはまる）。
- 卓から見える差: 技能を狙う一時付与が**持続どおり失効する**（アクター着地＝掃引に乗る）・**付与先の選択ダイアログが出ない**・分類狙い「全一般技能+1」が**1つだけに乗る誤り**が消える・技能アイテムの中に転送コピーが隠れない。

**設計レビュー**:

- **キー文法は変えない**。`item.<識別キー>.system.level` の綴りのまま、経路の分岐条件だけを「乗り先の種別」に置いた（`isOutfitItem`＝`OUTFIT_TYPES` に属するか）。式参照（`@item.<識別キー>.system.levelTotal`）も無変更。
- 遠隔適用は `_resolveBuffApplications` の skill/category スコープで、**モノを除いた**アイテムへ `itemChangeTargets` で照合する。転送側（`materializeItemTransfers`）は `targets` に `isOutfitItem` を掛けてモノ以外を狙わない。この二つは補集合なので**同じ効果が転送コピーと遠隔適用の両方で効くことはない**。
- 使用時付与の着地判定（`analyzeGrantLanding`）に**付与先の所持アイテム**を渡す。疑似分類は静的に非モノ。識別キーは解決して判断し、**1件も解決されないときはアイテム着地扱いのまま**（「付与先になれるアイテムがない」の既存警告に載せる＝黙ってアクターに乗せて何も起きない、を避ける）。所持アイテムを渡さない旧呼び出しは従来の判定を保つ。
- **既存データの扱い**: 旧経路で技能アイテムの上に作られた転送コピーは、残すと遠隔適用と二重に乗る。**GM の初回ロードで一回だけ除去**する（部位キー移行と同じ版番号ゲート・常設経路には配線しない）。触るのは転送コピー（`transferredFrom`）だけで、付与コピー・供給元の定義・モノ上のコピーには手を出さない。**ユーザーの正本データへの副作用**なので着手前に選択肢を提示したが、ユーザーは問いを取り下げて「続けてください」と指示したため推奨案（一回限り掃除）で実装し、本エントリと CHANGELOG に明記した。
- **TDD**: 遠隔適用 6・種別 3・着地 5・掃除 4 の計 18 件を先に書いて赤を確認してから実装。全 2448 件グリーン・eslint 0 error。

**実機確認（隔離 Foundry v13.351・develop 複製・Playwright）**:

- 恒常: 技能（識別キー狙い +1）→ `levelTotal` 2→3・**技能上に転送コピー 0**。同じ供給元の武器狙い +2 → 攻撃力 7・**武器上に転送コピー 1**（エンチャント経路は不変）。
- 旧データの掃除: 技能に旧経路の転送コピーを置くと 4（二重）→ 版番号を戻して再ロード → コンソールに「1 件除去」・`levelTotal` 3・コピー 0・版番号 1。develop 複製の初回ロードでは除去 0 件（ユーザーの実データ複製に残骸は無かった）。
- 使用時付与: 技能 +2（シーン中）をチャットカードの「効果を適用」で付与 → **ダイアログなし**・付与コピーは**アクター上**（技能上には無い）・`levelTotal` 5 → 退場（シーン境界）で失効し 3。

### 2026-09-03: 17-1 神業の起動一本化と神業由来の印

**2026-09-03** / **関連実装** `scripts/module/miracle-logic.mjs`（新設・純ロジック）・`scripts/module/miracle-flow.mjs`（新設・投稿と既定使用）・`templates/chat/miracle-card.hbs`（新設）・`scripts/actor/tnx-character-sheet-base.mjs`（旧 `_onUseMiracle` 撤去・`_activateItemCheck` の神業分岐・`_useDeclarationUsage` の既定消費と神業カード）・`scripts/module/usage-check-context.mjs`（既定消費・ctx.miracle）・`scripts/module/tnx-check-flow.mjs`（結果カードの `miracle` フラグ）・`scripts/module/usage-effects.mjs`（付与コピーの `fromMiracle`）・`scripts/data/item/miracle.mjs`（`isKill`/`isDefence`/`isAll` 撤去）・`templates/actor/parts/header.hbs`・`tests/module/miracle-logic.test.mjs`（+17）・`tests/data/item/miracle.test.mjs` / **ステータス**: 実装済・隔離実機確認済（本体 `8976a08`）

**メカニクス監査**:

- ルールの位置づけ（ユーザー言語化 2026-09-03）: 神業はゴールデンルール「RL の絶対権限」のひとつ下に位置する強制力の強いルール。実装で採る形は「挙動は既存の用途の器で表現してよい。必要なのは**その用途が神業のものであるという印**」（ユーザー確認）。17-1 はこの印の出どころと伝播だけを作り、効き先（軽減・リアクション・治療・打ち消しのゲート）は 17-2/17-3 に置く。
- **印の出どころはアイテムの型**。専用フィールドは持たない（判定条件を二重に持たない）。用途が0件でも印は立つ。
- **用途は前提条件にしない**（Code 決定・ユーザーは一任）。用途0件の神業がロールで使用回数を消費して神業カードを出すのは、「使用＝回数消費」が神業の定義に含まれるため。宣言だけの神業（《天罰》《脱出》《真実》等）はこれで完成する。
- **消費先が空の神業用途に自身の使用回数×1を既定消費する**のは、「消費は消費先の設定からのみ」の一般原則（コンボ参加技能の遠隔消費を廃した文脈）の適用範囲外と判断した。神業の使用が回数消費を伴うことはルールの定義であり、用途を作った途端に減らなくなるほうが誤り。補完は実行時のみで保存しない。
- **形骸化フラグの撤去**（ユーザー裁定 verbatim「殺し神業、防御神業のような区分も無くしましょう。完全に形骸化しているフラグです」）。辞典31件はすべて false・読み手ゼロ・設定 UI 無し。DataModel から落とすと既存データの当該キーは次回保存時に捨てられる（副作用として明記済み・実害なし）。
- 撤去した旧経路（`_onUseMiracle`）は用途システム・チャットカード統一・アイテムロール規範のいずれよりも前の実装で、素の HTML カードを出し、消費を自前で行い、万能神業をワールド直下のアイテムから選ばせていた。派生神業の選択は 17-5 で作り直す（選び方4つ・実行経路1つ）。

**設計レビュー**:

- 純ロジック（`miracleUseGate` / `miracleConsumeUpdate` / `withDefaultMiracleConsumption` / `miracleOriginOf` / `isMiracleOrigin` / `buildMiracleCardData`）を Foundry 非依存モジュールに分離し、TDD で先に赤を確認（`Failed to load url` → `is not a function`）してから実装。消費は既存の実効値読み（`usesMaxTotalOf`）に乗せ、尽きたときの `isUsed=true` は旧経路の挙動を維持（手動リセットの起点）。
- 起動は唯一の起動関数に集約。神業ボタンは `startSkillCheck` へ付け替えるだけで、ハンドラは既存の `_onStartSkillCheck`。用途0件の分岐は「用途が無ければ解説カード」の手前に置いた。
- 神業カードは送信カードの骨格（`.tnx-check-request` ＋ `cr-req-header/body`）を踏襲した専用テンプレート。ふりがなはルビで名前と同色（辞典カードの規約）。条件が空なら行ごと畳む。用途つきのときは解説カードと同じ器（`tnx-usage-use-card` ＋ 効果エリア）で包み、効果トレイの差し込みは既存フックのまま。
- 印の伝播: 神業カードのフラグ `miracle`（`postMiracleCard`）。`resolveUsageEffectData` は用途の親が神業なら付与コピーに `fromMiracle` を刻む（供給元が組み合わせ側でも「神業の用途から生じた効果」であることに変わりない）。ダメージカードへの刻印はダメージ付与そのものを作る 17-3 で行う。
- **是正（同日・本体 `3c31361`）**: 当初、判定経路（`buildUsageCheckContext` の `ctx.miracle`・結果カードのフラグ）にも印と既定消費を配線し「判定タイプの神業は未検証」と報告した。**神業は判定を行わない**——効果文31件のいずれも判定を行わず、これは効果文を読んだ時点で Code 自身が読み取っていなければならなかった事実（ユーザーの提供ではない。ユーザー指摘「神業を行うことはないというのはあなたが自分で読み取らなければならなかったこと」）——ため到達しない死んだコードであり、効果文でなく器の側から書いた誤り。撤去した。あわせて既定消費の差し込みが宣言と判定の2か所に散っていて**治癒系の神業が通る治療（宣言形）の経路に届いていなかった**ため、用途が決まった直後の `_activateItemCheck` 1か所へ集約した（宣言・治療の宣言形・クリック待ち系のどれを通っても効く）。

**実機確認（隔離 Foundry v13.351・develop 複製・Playwright・創作データのみ）**:

- シートの神業ボタンが `startSkillCheck` で描画され、クリックで神業カード（タグ「神業」・ルビつき名前・効果文2段落・条件・残り使用回数 1 / 2）・消費済み1・`flags.miracle={itemId,name}`・発言者＝使用者・効果トレイなし。
- 2回目で消費済み2・`isUsed=true`・残り 0 / 2。再描画でボタンが disabled。起動関数を直接呼んでも「これ以上使用できません」の警告で中止しカードは出ず消費済みも増えない。
- 用途つき（宣言・消費先なし・自己効果）: 既定消費のダイアログ（「REPRO神業・用途」の使用回数を消費・残り 3/3）→実行→神業カード＋効果トレイ・消費済み1・残り 2 / 3・条件が空なら行が出ない・`usageEffects` フラグ。「効果を適用」で付与コピーに `fromMiracle=true`。
- 撤去した `isKill`/`isDefence`/`isAll` を作成データに混ぜても system に残らない。テスト2465件グリーン・eslint/stylelint クリーン。
- 是正後（`3c31361`）: 治療タイプ（宣言形・範囲=BS・消費先なし）の神業で、回復対象の確認→**既定消費のダイアログ（「REPRO神業・治療」の使用回数を消費・残り 2/2）**→BS 除去・消費済み1。クリック待ち系（宣言＋判定を修正・消費先なし）でも既定消費のダイアログ。A〜C の回帰も全件 OK。

### 2026-09-03: 17-2 器＝神業専用の用途タイプ5種と防御タイプの設定

**2026-09-03** / **関連実装** `scripts/module/usage-types.mjs`（5種・`isMiracleType`/`usageTypeLabelsFor`/`defaultUsageTypeFor`・`executionFormOf`）・`scripts/item/tnx-item-sheet.mjs`（作成ダイアログのフィルタと既定）・`scripts/actor/tnx-character-sheet-base.mjs`（神業タイプ→宣言経路）・`scripts/data/item/common/usage.mjs`（`defenceAction`/`defenceScope`/`defenceCategories`/`recoveryEffects`/`recoverySceneLimit`）・`scripts/module/tnx-usage-sheet.mjs`＋`templates/app/usage-sheet.hbs`（防御 fieldset・cure の治療設定共用・神業は目標値/対決なし）・テスト（usage-types +6・usage +5・confrontation の件数ガード 19→24） / **ステータス**: 実装済・隔離実機確認済（本体 `5c57d4a`）

**メカニクス監査**:

- ユーザー提案（verbatim）「神業アイテムにしか表示されず、神業アイテムではこれ以外表示されない用途を新規で作るのが良いと思います。メカニクスはそのまま流用できます」。種類は議論で確定（宣言・即死・防御・社会戦＋ユーザー「破壊は必要」）。派生タイプは作らず、他の神業になる神業はアイテム側の機能に一本化（設計判断6）。万能系は全種の用途を持たせれば足りる（ユーザー）。
- 17-1 の「用途タイプは新設しない」（Code 決定）を撤回した理由: タイプは技能の識別のためだけでなく**その用途で何が設定できるか**を決める。神業は判定を行わないのに判定・攻撃・リアクション等17種が並ぶ器は合っていなかった。神業タイプの用途に目標値・対決を出さないのも同じ理由。
- 防御タイプの動作4つ（適用前に防ぐ／打ち消し／回避／受けた後に消す）は Miracle_Rules「打ち消しの範囲」「防御神業」と効果文の再読で確定した区別に対応する。「受けた後に消す」が治療の回復設定を共用するのは、《腹心》《黄泉還り》《人命救助》《霧散》が「受けている状態を消す」という点で治療と同じ操作だから。足した2つ（スタイル技能の効果の解除・受けたシーンの制限）は効果文に根拠がある（《人命救助》「任意のスタイル技能の効果を解除する」・《腹心》《人命救助》「そのシーンで受けた完全死亡/精神崩壊」・《黄泉還り》「そのシーン中に受けたダメージしか」）。

**設計レビュー**:

- 5種は `USAGE_TYPE_DEFS` の `kind: "miracle"`。作成ダイアログの選択肢は `usageTypeLabelsFor(親型)` で親の型により分け、既存タイプ側は変更なし（件数ガードのテストだけ 24 へ）。`executionFormOf` は神業タイプを宣言に固定（判定経路へ入らない）。
- 起動関数の分岐は「宣言または神業タイプ」で 17-1 の宣言経路へ。即死・社会戦・破壊は 17-3 で専用の結果指定に切り替えるまで神業カードだけ出す（到達不能な用途を作らない）。防御の動作は次段で `_activateItemCheck` のクリック待ち分岐に足す。
- 防御の設定はフラグでなく **1つの動作セレクト**（排他が本質＝1用途1動作）。系統チェックは描画中の送信でだけ再構成し、未描画の送信で全消しにしない（消費行と同じ守り）。
- 隔離実機確認: 神業の作成ダイアログ＝5種だけ／スタイル技能＝5種を出さず既存19種／防御シート（既定=適用前に防ぐ・範囲・系統3つ）→社会を外して保存 `[physical,mental]`→動作を cure へ→治療設定が出て実行形式は出ない・2設定の保存・系統は保持／用途一覧の実効名「防御（REPRO神業・器）」／神業の用途シートに目標値・対決が出ない。17-1 の回帰も全件 OK。

### 2026-09-03: 17-2 防御「適用前に防ぐ」

**2026-09-03** / **関連実装** `scripts/module/miracle-logic.mjs`（`defencePreventPlan`/`unprotectedTargetIndices`・テスト+8）・`scripts/module/damage-flow.mjs`（`handleDamageProtectClick`・対象行クリック・防がれた行を台帳/状態領域/適用から外す）・`scripts/module/tnx-check-flow.mjs`（クリック待ち kind=protect/negate/evade の文言と達成値クリックの除外）・`scripts/actor/tnx-character-sheet-base.mjs`（防御タイプの分岐: cure→回復フロー／それ以外→クリック待ち） / **ステータス**: 実装済・隔離実機確認済（本体 `e26eee4`）

**メカニクス監査**:

- 効果文どおりの2粒度: 《難攻不落》「1回の判定、もしくは1発の神業によって発生したものすべて」＝まるごと／《守護神》《友情》「選択したひとりのキャラクター以外…助けることはできない」＝1人。系統は用途の設定（《難攻不落》「社会ダメージを除く」）。「受けてしまった後から治療することはできない」＝適用済みは拒否。
- 「消える」（ユーザー 2026-09-02「打ち消されたことでチャットカードから消えるような形式」）を防御にも適用: 防いだ対象は台帳の対象・最終ダメージ行・適用の対象から消える。理由の行は残さない（直前に神業カードが出ている）。データには `protectedBy`（神業の印）として残り、再描画で復活しない。

**設計レビュー**:

- 発動はクリック待ち（既存の「ダメージを修正」と同型・消費は待ち受け開始時に確定し発動時に適用）。発動点はダメージカードの対象行の名前。装飾クラスは攻撃側合計クリックと同じで、モード外のクリックは無視。
- 計画は純関数（範囲×系統×クリック行→添字 or 拒否理由）で、拒否はモードを維持したまま警告（別のカードを選び直せる＝再判定の警告と同じ振る舞い）。
- 表示と適用が同じ読み手（`unprotectedTargetIndices`）を使うことで「消えた行に適用される」ずれを構造的に防ぐ。
- 隔離実機確認: まるごと（3対象すべて消え・適用ボタンなし・台帳の対象行も消える・消費済み1）／1人（B だけ消え A/C は残る・対象（2体）・適用ボタンあり）／再描画後も消えたまま／社会ダメージへ社会を防げない神業＝警告しモード維持／同じ神業の再使用でキャンセル／適用済み＝警告。

### 2026-09-04: 17-2 打ち消し

**2026-09-04** / **関連実装** `scripts/module/miracle-logic.mjs`（`negateCheckGate`/`negatedCheckMods`・テスト+8）・`scripts/module/miracle-flow.mjs`（`handleNegateAchievementClick`/`handleNegateTrayClick`/`handleNegateMiracleCardClick`/`renderMiracleCard`）・`scripts/module/tnx-check-flow.mjs`（達成値クリックの negate 分岐）・`scripts/module/attack-flow.mjs`（failedReason=negated の帰属表示）・`scripts/module/usage-effects.mjs`（negatedBy でトレイを出さない・見出しクリックの発動点）・`scripts/tnx.mjs`（神業カードの描画フック） / **ステータス**: 実装済・隔離実機確認済（本体 `0710fed`）

**メカニクス監査**:

- ユーザー裁定（2026-09-02）「判定に対しては失敗させる／宣言に対しては効果の適用をキャンセルする」の2形で実装。判定を丸ごと無かったことにする形は作らない。効果文《チャイ》《平和》「1回の判定を失敗にさせる、使用に宣言が必要な特技（スタイル技能）の効果を打ち消す」「既に効果が適用された神業や判定に対して、時間をさかのぼって打ち消すことはできない」。
- **適用前**の線引き（Code の読み・効果文が根拠）: 成功時に効果が即座に適用される継続（回復の除去・修理・改造・購入の付与・登場・情報開示・制御打消・リアクション/カバーの対決解決・NPC取得の生成）は結果カードの投稿と同時に適用済みなので拒否。移動は段階の表示のみで適用は手動＝打ち消せる。攻撃はダメージカードが出ていても適用前なら打ち消せ、そのダメージカードは全対象を防いだ扱いで消える。宣言カードの効果トレイは適用状態を持たない（カードは状態を主張しない）ため、適用前に限るのは卓の運用。
- 「消える」の適用: 宣言トレイはトレイごと、神業カードは効果文・条件・残り回数・効果エリアが消えて見出しだけ残る。判定は失敗に**なる**ので消えない（既存の「修正後の成否」行に失敗が出る）。理由の行は残さない。

**設計レビュー**:

- 判定の失敗固定は既存の事後修正の器（`checkMods`）に修正 0 の「打ち消し（神業名）」行を積む形。達成値は変えず成否だけ失敗に固定し、既存の描画（修正後の達成値／修正後の成否）にそのまま乗る。判定要求由来は要求カードの結果も失敗へ追随。
- 攻撃は既存の「攻撃を失敗させる」と同じ着地（`state: "failed"`）に理由 `negated` と印を足し、帰属を「（神業名による打ち消し）」で表示。紐づくダメージカードは「適用前に防ぐ」と同じ `protectedBy` を全対象に刻む（1機構の再利用）。
- 神業カードの描画フックを新設（`renderMiracleCard`）。効果トレイの描画より前に呼び、打ち消し済みは効果エリアごと消す。
- 隔離実機確認: 山札からの実判定（目標値5）を打ち消し→成否失敗・差分なし・打ち消し行・達成値不変・カードに「修正後の成否 失敗」・消費済み1・二重は警告／合成の攻撃カード＋紐づくダメージカード→全体失敗（帰属表示）・ダメージカードは全対象が防がれ行と適用ボタンが消える／宣言トレイ消失／神業カードの中身消失／適用済みダメージつき攻撃は警告。「適用前に防ぐ」と 17-1 の回帰も全件 OK。

### 2026-09-04: 17-2 回避（《脱出》）

**2026-09-04** / **関連実装** `scripts/module/miracle-logic.mjs`（`evadePlan`・テスト+6）・`scripts/module/miracle-flow.mjs`（`handleAttackEvadeClick`）・`scripts/module/attack-flow.mjs`（対象行の捕捉クリック・`miracleEvade` の表示・`applyAttackTargetPatch` 公開） / **ステータス**: 実装済・隔離実機確認済（本体 `390bfdc`）

**メカニクス監査**:

- ユーザー指摘（2026-09-03）「脱出は防御ではなく回避です。少し性質が違います」。効果文「あなた、もしくはあなたの操縦するヴィークルへの物理攻撃をかわす（その場合、位置は変わらない）」。回避は**命中の段階**の動作で、攻撃カードの対象行を回避（miss）にする。ダメージが決まった後にダメージを無かったことにするのは防御神業の領分＝ダメージカードが出た後は拒否。
- 範囲は自分の行のみ・物理攻撃のみ（効果文）。ヴィークルへの攻撃はこのシステムでは操縦者を対象にするため「自分の行」に含まれる。「乗員も同時に回避」はシステムが同乗を持たないため手動（Code 整理・計画の 17-2 に明記済み）。

**設計レビュー**:

- 発動点は攻撃カードの対象行。既存の行クリックはリアクション入口（本人=決定/スキップ・他者=代理）なので、**捕捉段階でモードを同期判定して既存の入口へ流さない**（非同期の動的 import を挟むと既存リスナーが先に走るため、判定だけ同期にして実処理を後続にした）。
- 着地は既存の対象行更新（`applyAttackTargetPatch`）で `state: "miss"`・`resolution: "miracleEvade"`・`evadedBy`（印）。全体の状態は既存の全員回避時と同じく変えない（既存の対決解決の挙動に合わせた）。
- 隔離実機確認: 自分の行→miss・表示「REPRO脱出による回避」・消費済み1／他人の行・精神攻撃・ダメージ算出後は警告しモード維持／待ち受け外のクリックでは回避されない。

### 2026-09-04: 17-2 受けた後に消す（神業の治癒）

**2026-09-04** / **関連実装** `scripts/module/miracle-logic.mjs`（`recoveryCandidateAllowed`/`isSameReceivedScene`・テスト+8）・`scripts/module/recovery-flow.mjs`（候補列挙に神業の3条件・prebound のゲート）・`scripts/tnx.mjs`（`preCreateActiveEffect` で受けたシーンを刻む） / **ステータス**: 実装済・隔離実機確認済（本体 `24f2978`）

**メカニクス監査**:

- 効果文が根拠: 《人命救助》「任意のスタイル技能の効果を解除する」／《腹心》「［完全死亡］と［精神崩壊］は受けたシーン中に使用しなければ治癒できない」・《人命救助》「そのシーンで受けたものしか」／《黄泉還り》他人には「そのシーン中に受けたダメージしか治せない」。既存の回復機構は状態しか除去せず、状態は受けた時点を持っていなかった（Code の再読で発見した不足）。
- 「受けたシーン」は**上演中のシーン番号**（14-8 の実行時カウンタ）とアクト id の組。台本の行番号でなく入場のたびに +1 する番号なので巡回シーンでも一意。
- 不明（旧データ・アクト外で受けた状態）は制限で落とさない——検査できないものはゲートしない（ウェット無効の規約と同じ）。
- 印のゲートの受け側: 神業由来の状態は神業の治療でしか除去できない（神業の位置づけ＝下位のルールでは覆せない）。刻む側は 17-3。

**設計レビュー**:

- 3条件を純関数 `recoveryCandidateAllowed` に集約し、既存の範囲/除外の照合はそのまま（回復機構を作り直さない）。候補列挙は `listRecoverableEffects(patient, usage, {byMiracle, currentScene})` の1経路で、治療メニュー起点（prebound）は選択を飛ばすためゲートだけ別に置いた。
- 刻印は `preCreateActiveEffect` の一点（ダメージ・カスケード・RL 任意付与・手動のどの経路でも）。状態（`getConditionKinds` が空でない）だけに刻む。
- スタイル技能の効果の解除は「状態でない付与コピーで供給元がスタイル技能」。供給元が解決できないものは通す（解決できないことを理由に消せなくしない）。
- 隔離実機確認: 生成した状態に receivedScene（act・number=1）／神業の治癒で BS とスタイル技能の効果が消え一般技能の効果は残る・消費済み1／terminal で前シーンの完全死亡は候補外・今シーンは候補・all で前シーンの BS も候補外／神業由来の状態は神業以外の候補に載らず神業なら載る・治療メニュー起点でも神業以外は警告し状態は残る。回帰: 治療の既定消費・打ち消し・適用前に防ぐ・回避・17-1 すべて OK。

**17-2 の総括**: 用途タイプ「防御」の4動作（適用前に防ぐ／打ち消し／回避／受けた後に消す）が揃った。ユーザー実機確認は 17-6 で一括。

##### 追記 2026-09-07（ユーザー指示・その2）: 候補は出どころで絞らず、効果はダイアログで選ぶ

- ユーザー指示「ダイアログで任意の効果を選んで回復する形にしてください。**付与経路などの判断は全て邪魔なので無くしてください**。単純化してかまいません」。
- **ゲートを撤去**: `recoveryCandidateAllowed` の非状態枝は `byMiracle && usage.recoveryEffects === true` だけになった（付与コピーか／供給元がスタイル技能か／無効化済みか／転送コピーか、の判断をすべて削除。`grantedSourceIsStyleSkill` も削除）。**設定は《腹心》「バッドステータスなど不利になる効果を受けている場合、それらも同時に解除できる」の側に合わせた**——《人命救助》の「任意のスタイル技能の効果」はその部分集合として表せる。神業由来の印（`fromMiracle`）のゲートだけは残る（ルールであって出どころの判断ではない）。
- **効果は常に任意選択**（「該当すべてを回復」でも一括では消さない）。効果文がどちらも「**任意の**スタイル技能の効果を解除する」「それらも同時に**解除できる**」と任意にしているため、状態（＝自動で治癒される）と扱いを分けた。個数上限も置かない（`recoveryCount` は状態のみ）。ダイアログは「状態の群（一括確認 or N個まで）」＋「効果の群（チェックで選ぶ）」の2段。
- **言い方を分けた**: 状態＝「回復した」／効果＝「解除した」。両方あるときは「〈対象〉の…を回復し、…を解除した」。`buildRemovalPlan` が `conditionLabel`/`effectLabel` を別々に返す。
- **副産物で KI-052 を発見**（→ [Issues](../01_Wiki/Issues/Known_Issues_Detail.md)）: `DialogV2` は「コールバックの戻り値 ?? ボタンの action」で解決するため、キャンセルの `() => null` は文字列 "cancel" になる。回復対象の選択は**旧コードでもキャンセルが素通りしていた**（「該当すべて」設定では消費してカードまで出ていた）。ここは `typeof picked !== "object"` で修正済み。同型の取りこぼしが改造・FS・コンディション編集に残っており、全数掃引は指示待ち。
- 検証: 純ロジックのテストを差し替え（2616 件グリーン）。隔離実機で——出どころの違う効果3つ（スタイル技能の付与コピー／一般技能の付与コピー／手置き）がすべて並ぶ・「該当すべて」でも**チェックした効果だけ**解除され選ばなかった効果は残る・帰結行が「…を回復し、…を解除した」・効果だけなら「…を解除した」・設定オフでは効果が並ばない・キャンセルで消費もカードも起きない、をスクリーンショット目視込みで確認。

##### 追記 2026-09-07（ユーザー指示）: 設定名を「BS・ダメージ以外の不利な効果も解除」へ

- 用途シートのチェックボックス名を「スタイル技能の効果も解除できる」→ **「BS・ダメージ以外の不利な効果も解除」**（ユーザー指示「『解除できる』ではメモなので」）。旧名は《人命救助》の効果文の写しで、**設定欄が何をする欄かでなく1つの神業の言い回し**になっていた。
- **未決**: 新しい名前は**現在の挙動より広い**。ゲートは今も「状態でない**付与コピー**で**供給元がスタイル技能**」のままで、一般技能・アウトフィット由来の付与コピーや手置きの効果は候補に出ない。効果文を見ると《人命救助》は「任意のスタイル技能の効果」だが、**《腹心》は「バッドステータスなど不利になる効果を受けている場合、それらも同時に解除できる」**＝スタイル技能に限っていない。設定欄は両方の神業が使う共通の器なので、**候補の範囲を「状態でない効果すべて（無効化済み・アイテム狙いの転送コピーは除く）」へ広げるかはユーザー裁定待ち**。あわせて、システムは「不利な効果」を機械的に判別できない（AE に有利/不利の印が無い）ため、広げると有利な効果も候補に並ぶ——「該当すべてを回復」との組み合わせをどう扱うかも同じ裁定に含める。

### 2026-09-04: 17-3 即死・社会戦・破壊タイプ

**2026-09-04** / **関連実装** `scripts/module/miracle-logic.mjs`（`terminalKindFor`/`buildMiracleDamageFlag`/`miracleResultLabel`/`miracleTargetOutcome`・テスト+8）・`scripts/module/miracle-flow.mjs`（`useMiracleDamage`/結果の選択/`handleNegateMiracleDamageClick`/`useMiracleDestroy`/破壊の適用と描画）・`scripts/module/damage-flow.mjs`（`renderMiracleDamageCard`/`applyMiracleDamage`/`drawDamageValueFromDeck` 抽出・`openMitigationDialog` の分岐）・`scripts/module/condition-resolution.mjs`（`applyDamageChartResult` の `extraFlags`）・`scripts/tnx.mjs`（カスケードへの `fromMiracle` 伝播）・`scripts/data/item/common/usage.mjs`（`killCategory`/`socialDecide`）・用途シート・起動関数の分岐 / **ステータス**: 実装済・隔離実機確認済（本体 `20f897e`）

**メカニクス監査**:

- 効果文が根拠: 《死の舞踏》《とどめの一撃》「［完全死亡］させる…代わりに任意の肉体戦ダメージを与えても良い。神業以外の効果で防がれることも、治癒されることもない」《神の御言葉》「［精神崩壊］…任意の精神戦ダメージ」《制裁》「任意の社会戦ダメージ…チャートの中から好きなものを選ぶ…抹殺でもよい。トループならば全滅させてもよい」《暴露》「RL が任意に決定する。判断に迷った場合は、山札から2枚めくってカードの数字を合計し、社会戦ダメージチャートを参照」《天変地異》《突破》「アウトフィットをひとつ［破壊］」。
- **結果を直接指定し、算出を経ない**（設計判断5）。神業版のダメージカードは台帳が神業と結果だけで、防御力・受け値・軽減 AE・報酬点のどれも通らない＝「神業以外の効果で防がれることはない」の実装。「治癒されることもない」は付与した状態（とカスケード）への `fromMiracle` で 17-2 の受け側ゲートに乗る。
- **トループ壊滅は即死に含め、破壊はアウトフィットのみ**（ユーザー裁定 2026-09-04「トループ壊滅と破壊は別です。即死に含めるか、別用途にしてください」）。《天変地異》《突破》は破壊用途＋即死用途の2つを持つ。
- 派生ダメージ（社会9→精神 等）は既存のチャート効果として従来どおり適用し、そこには印を刻まない（派生は神業のダメージそのものではなくチャートの効果と読んだ。要すれば裁定）。
- 治癒側（《暴露》《制裁》の「1つ治癒」、《完全偽装》《不可触》の社会戦ダメージ消去）は社会戦タイプでなく防御タイプ「受けた後に消す」の別用途で表す（既存機構の流用）。

**設計レビュー**:

- 神業版のダメージカードは**既存のダメージカードの器**（`damageRoll` フラグ）に `miracle`/`miracleResult` を載せたもの。対象行の「防ぐ」クリック・見出しの「打ち消し」・全対象が消えたときの適用不能・適用ボタンの権限（対象の操作者/RL）がそのまま効く（1機構の再利用）。描画は `renderMiracleDamageCard` に分岐し、通常カードの軽減の段を一切通らない。
- 結果の選択は既存の選択ダイアログ（`TargetSelectionDialog`）で、終端状態＋チャート 1〜21（値と負傷名）。RL の「山札から2枚」は派生ダメージの引き処理を `drawDamageValueFromDeck` として抽出して共用（ジョーカーの数字宣言も既存どおり）。
- 適用は `applyMiracleDamage`（軽減ダイアログを挟まない）。対象の型で分岐（`miracleTargetOutcome`）: 終端状態の直接付与／`applyDamageChartResult` に `extraFlags` で印／トループの壊滅・減算／エキストラは適用なし。トークン uuid（未リンクのトループ）はそのトークンのアクターへ解決する（既存の `resolveTargetActor`＝最初の実機で人数が減らず発見・是正）。
- めくったカードの表示は1枚1行（狭い幅で語の途中で折れていたのを是正）。
- 破壊は神業カードに結果行と「破壊を適用」を足す形（`renderMiracleCard`）。適用後はボタンを消し「破壊済み」。
- 隔離実機確認: 終端状態（完全死亡・印・適用後の表示）／任意ダメージ 9（負傷「朦朧」とカスケード「酩酊(小)」に印・神業以外の治療の候補に載らず神業なら載る）／トループ壊滅（5→0）／社会戦の抹殺／山札2枚（2枚の名前と合計の表示）／難攻不落で防ぐ・チャイで打ち消す／破壊（候補・適用・isDestroyed・破壊済み表示）。17-1・防ぐ・打ち消しの回帰も全件 OK。

### 2026-09-04: 17-4 神業に干渉する神業（《ファイト！》《プリーズ！》）

**2026-09-04** / **関連実装** `scripts/module/miracle-logic.mjs`（`withoutConsumption`/`interferenceCandidates`/`addUseEffectSource`・テスト+5）・`scripts/module/miracle-flow.mjs`（`useMiracleInterference`/`applyMiracleAddUse`/`handleMiracleRequestClick`・`renderMiracleCard` の結果行/ボタン・`resolveSingleTarget`/`isTargetOperator`/`chatButton` を破壊と共用）・`scripts/actor/tnx-character-sheet-base.mjs`（`_activateItemCheck` の `miracleFree` 文脈と干渉の分岐・神業の分岐が「発動したか」を返す）・`scripts/data/item/common/usage.mjs`（`miracleInterference`）・用途シート / **ステータス**: 実装済・隔離実機確認済（本体 `a2f27e0`）

**メカニクス監査**:

- 効果文が根拠: 《ファイト！》「他のキャラクターの持つ神業の使用回数を、1回増やす。この神業はすでに使用されているものでも、まだ使用されていないものでもかまわない。…アクトが終了した後に持ち越すことはできない」「使用回数が増えた神業をどう使用するか、決定するのは対象となったキャラクター」「《ファイト！》を《ファイト！》することはできない」／《プリーズ！》「他人（キャストでもゲストでもよい）に、神業を使わせることができる」「相手の神業は使用済みにならない」「RL は判断に迷った場合、《プリーズ！》の効果を『神業の使用』に限定すること」（Miracle_Rules の一覧「使用済みも可」）。
- **使用回数+1 は「1つの神業に+1」**（使用者が選ぶ）。対象の神業が1つなら自動。同名（＝《ファイト！》）は候補外。使い切った神業も候補＝適用後は残り1でもう一度使える（残回数ゲートが実効値を読むことで成立）。
- **重ねがけ**: 2人（あるいは複数回）の《ファイト！》が同じ神業に乗れば +2。同名効果の重複排除（§4「同名効果の二重適用」禁止）は 1 つの効果の二重適用を防ぐ規約で、別々の神業の使用はそれぞれ独立した効果＝`stackable`（重ねがけ累積の例外）で表す。**Code の読み**＝要すれば裁定。
- 「持ち越せない」＝アクト終了の境界で失効（付与コピー）。使用回数の消費もアクト終了で 0 に戻るので、母数と消費が同じ境界で揃う。
- 《プリーズ！》で使わされる神業は**残回数ゲートも消費も無い**（使用済みも可＋使用済みにならない）。それ以外（対象・結果の選択・カード・印）は本人が使うときと同じ＝神業の挙動はそのまま。使うか・どう使うかは対象の操作者が決める（ボタンは対象の操作者/RL にだけ出る）。
- 「お願いの内容は神業の使用に限らなくても良い」は自動化しない（RL が迷ったときの限定形がこの器）。

**設計レビュー**:

- 設定は**用途の1欄**（神業の宣言タイプ専用の select）。既存の「適用効果」で書けない理由: アイテム着地の候補は素のキーだけだと**全アウトフィット**が既定（ユーザー確定）で神業が候補に入らず、相手のどの神業かは使用時にしか決まらない。
- +1 の効果は `addUseEffectSource`（純）を `buildGrantedEffectDataFrom` で付与コピー化して対象の神業に `createEmbeddedDocuments`。`grantedFrom`＝使った神業の uuid（由来の記録・同期の紐づけではない）→ `effectAutoApplies` が真・`planItemGrantExpiry` が境界で失効（KI-048 の器）。着地は `uses.max`→実効 `uses.maxTotal`（KI-038 で新設済み・miracle.mjs のコメントどおり）。実機で maxTotal 2→3→境界で 1 を確認。
- 《プリーズ！》の使用は**唯一の起動関数**に `openExtra.miracleFree` を載せるだけ（新しい起動関数は作らない）。用途なし＝ゲートと消費を飛ばす／用途あり＝既定消費を補わず `withoutConsumption` で消費先を空にする→以降の全分岐（宣言・防御の待ち受け・治療・即死/社会戦・破壊）が消費行ゼロで動き、`promptConsumption` は空行で `[]` を返す（ダイアログを出さない）。
- 「使用済み」の記録は**発動の事実**で行う: 神業の分岐が発動したかを返す（カード投稿／待ち受け開始／治療の除去）。キャンセルなら要求は残る（実機 F）。判定系の分岐は未定義のまま（神業は判定を行わない）。
- 破壊（17-3）と同じ形（神業カードに結果の段＋対象の操作者/RL のボタン）。対象1人の解決・操作者判定・ボタン生成を共通関数に抽出して破壊も同じ関数へ（1機構）。
- 隔離実機確認: 候補の同名除外／適用で効果1つ（uses.max +1・アクト中・神業由来・重ねがけ可・付与コピー）・maxTotal 2・残り 1／2回で +2／アクト終了で失効・消費 0／要求カード（3ボタン）→使い切った用途なしの神業が消費なしで発動・「使用: 《X》」／即死タイプ（使い切り）→消費ダイアログ無しで結果の選択→神業版ダメージカード（X の《死の舞踏》・完全死亡）・消費なし／キャンセルで要求が残る。17-1・17-2b/c・17-3 の回帰 ALL OK（17-3 の確認スクリプトはめくったカードの1行1枚化に合わせて更新）。

### 2026-09-04: 17-5 他の神業として使う神業（《万能道具》《突然変異》）

**2026-09-04** / **関連実装** `scripts/data/item/miracle.mjs`（`asOther`）・`scripts/item/tnx-miracle-sheet.mjs`＋`templates/item/miracle-sheet.hbs`（設定 fieldset）・`scripts/module/miracle-logic.mjs`（`miracleOriginOf(item, asOther)`/`asOtherRefCandidates`/`miracleLogCandidates`/`buildMiracleUseLogEntry`/`miracleUseFromMessageFlags`・テスト+7）・`scripts/module/miracle-flow.mjs`（`resolveAsOther`・各フローの `asOther` 伝搬・カードの「効果」行・打ち消し/回避の発動で神業カード）・`scripts/module/damage-flow.mjs`（防ぐの印と発動時の神業カード・台帳「効果」行）・`scripts/module/tnx-check-flow.mjs`（待ち受け状態に `usage`/`asOther`）・`scripts/module/recovery-flow.mjs`（神業の治癒で神業カード）・`scripts/module/session-state.mjs`（`miracleUseLog`・`registerMiracleUseLogging`）・`scripts/actor/tnx-character-sheet-base.mjs`（再入と参照先の用途） / **ステータス**: 実装済・隔離実機確認済（本体 `55960bf`）

> **後日の変更（この節の一部は現行ではない）**: 選んで固定する型（《万能道具》《神意》《半身》）は 2026-09-06 に**実体を写す方式**へ変えた（本体 `e041d81`・方針A）＝実行時に参照先を解決するのではなく、効果を決めた時点で参照先の**用途と経験点の取得条件**をその神業へ写す。写した条件の文中の神業名は写し先の名前に置き換える（KI-055・本体 `ad19ff6`・下の追記）。使用ログからのコピー（《突然変異》）は従来どおり実行時の解決。

**メカニクス監査**:

- 効果文が根拠: 《万能道具》「取得している〈フォルム〉によって、異なるスタイルの神業と同等の効果が発生する」＋対応表（ウェポン＝《死の舞踏》《とどめの一撃》…アイコン＝《電脳神》）／《突然変異》「そのアクト中に使用された神業をコピーして使用する」「コピーする神業は、あなたが登場したシーンで使用されたものに限られる」「登場していれば、その神業の効果が適用される前であっても、コピーすることは可能」「『うまく使った』条件は、コピーした神業のものと同じ」。
- **選び方は3つ・実行経路は1つ**（設計判断6）: 参照先（取得技能で決まる＝《万能道具》《神意》／指定の選択肢＝《半身》）と使用ログ（《突然変異》）。実行は参照先の**用途**を元の神業の名前・使用回数・消費・印で行う。カードの効果文と条件は参照先のもの（経験点条件も参照先と同じ＝効果文どおり）。
- **条件技能は識別キーで照合する**。〈フォルム：アーマー〉等は識別キーの異なる技能アイテムとして辞典に投入する前提（Code の前提・スタイル技能パックは未投入のためユーザー確認待ち＝17-6 一括）。
- **《突然変異》の候補**＝記帳時に自分が登場していたシーンの使用、または現在シーンの使用でいま登場している（同じシーンで使用の後に登場した場合も「登場したシーンで使用されたもの」と読む＝Code の読み・要すれば裁定）。他人の所持神業も参照先になる（uuid で解決）。《万能道具》の使用をコピーするときは解決後の神業（《難攻不落》等）をコピーする。
- 《万能道具》で防いだ/打ち消した/回避した印は「《万能道具》（《難攻不落》）」＝上位の神業由来の印はそのまま、効果の出どころを添える。

**設計レビュー**:

- **アイテム側の機能**（用途ではない・設計判断6/ユーザー明示）: DataModel `asOther` と神業シートの fieldset。参照行は対決欄と同じ辞典カスケード（`buildSkillCascadeSteps`・識別キー保存・現在名表示）＋スタイル→神業と同じ uuid 参照（ドロップ・`fromUuid` で現在名・右クリックで解除）。行は配列全体を送って更新（スタイル技能シートのコンボ行と同方式）・uuid は hidden で持ちフォーム送信で欠けない。閲覧モードでは追加/削除非表示・プルダウン読み取り専用。
- **唯一の起動関数への再入**: `asOther.mode` を持つ神業は `resolveAsOther` で参照先を決め `openExtra.asOther={uuid,name,source}` で再入。`usableUsages` を参照先の用途にするだけで、以降の全分岐（宣言・防御の待ち受け・治癒・即死/社会戦・破壊・干渉）が元の神業を `item` として動く（消費の既定行は self＝元の神業）。適用効果は参照先の効果から組む。待ち受けの状態に `usage`（参照先の用途）と `asOther` を載せ、防ぐの用途解決と印に使う。
- **印の拡張**: `miracleOriginOf(item, asOther)` に `uuid` と `asOther{uuid,name}`（source は載せない）。カードは「効果 《参照先》」の行（神業カードは本文先頭・ダメージカードは台帳「神業」の次の行＝名前に括弧で足すと狭い幅で語の途中で折れるため）。
- **使用ログ**はワールド設定（`sessionState`）に置き、`createChatMessage` をアクティブ GM が拾って直列に記帳（登場の記帳と同じ器）。記帳点＝神業カード/神業版ダメージカード。**防ぐ/打ち消し/回避の発動と神業の治癒は神業カードを出していなかった**ので出すようにした（神業の使用を卓に提示する 17-1 の原則にも合う）。
- 隔離実機確認: 《万能道具》＋アーマー→《難攻不落》自動解決・防ぐ（印・神業カード・消費は万能道具）／ウェポン追加で3候補→《死の舞踏》→即死（台帳）／使用ログ／条件不成立の警告／《突然変異》: 登場したシーンの使用だけ・後から登場でも候補・他人の所持神業のコピー／シート（選び方・3行・追加/ドロップ/カスケード/削除）。17-1〜17-4 回帰 ALL OK。

### 2026-09-04: 17-6 残りの神業の接続（《神出鬼没》《タイムリー》《買収》《不可知》）

**2026-09-04** / **関連実装** `scripts/data/item/common/usage.mjs`（`miracleEffect`・`miracleInterference` を改名）・`scripts/module/miracle-logic.mjs`（`conditionSwapPlan`）・`scripts/module/miracle-flow.mjs`（`useMiracleSwap`/`applyMiracleSwap`/`useMiracleAcquire`/`useMiracleInsensible`・`renderMiracleEffectRows`）・`scripts/module/purchase-flow.mjs`（神業の入手の再入）・`scripts/module/tnx-dictionary-browser.mjs`（神業の選択モード）・`scripts/module/tnx-check-flow.mjs`（印の消費）・`scripts/module/attack-flow.mjs`（リアクション不可・制御値で即確定）・`scripts/module/damage-flow.mjs`（印の伝搬・`applyDamageToTarget` の `extraFlags`）・`scripts/module/combat-progression.mjs`＋`scripts/combat/tnx-combat.mjs`（`buildMajorChargeUpdate`・`_chargeMajor`）・`scripts/item/tnx-outfit-sheet.mjs`＋テンプレート / **ステータス**: 実装済・隔離実機確認済（本体 `75afda1`）・ユーザー実機確認待ち

**メカニクス監査**:

- 効果文が根拠: 《神出鬼没》「“宿主”が受けたあらゆるダメージや状況はカゲムシャが引き受けることになるし、その逆も発生する」＝両者の状態を**丸ごと**入れ替える（時点の入力は持たない＝ユーザー指摘 2026-09-03）。／《タイムリー》「得たアウトフィットは［常備化］できない」《買収》（一覧「入手品は常備化不可」）＝購入判定の入手と同じ器（`isCheckAcquired`＝経験点計上なし）に**神業由来の印**を足す。購入値・常備化経験点・外界の条件は問わない（「ルールブックに載っているアウトフィットでもよいし…」）。／《不可知》「神業を使用しない限り、一切のリアクションやアウトフィットの使用などを行うことはできない」「防具など常に効果を発揮しているアウトフィットや、制御値のみが有効」「アクションランクを消費しない」「神業によってしか治療を行えない」。
- 《不可知》の実装形＝**次の行動に印を付ける**: 宣言で自分に印、次の判定の実行で消費。攻撃ではリアクション（カバー含む）の入口を出さず、対象は制御値で即確定（非対決と同じ経路）。神業の回避（対象行の捕捉）・打ち消し（達成値）はそのまま生きる。防具（恒久軽減）は効く。報酬点軽減はリアクション成立が条件なので自然に不可。非攻撃の判定に使った場合は印の消費のみ。
- 《完全偽装》《不可触》の「社会戦ダメージをすべて/1つ消去」は防御タイプ「受けた後に消す」（範囲＝負傷（社会））で表す＝17-2 の器。新規実装なし。

**設計レビュー**:

- 宣言（神業）の設定を **1 つの select「宣言の効果」** に統合（17-4 の「干渉」select を改名。効果はどれも排他的な 1 挙動のため）。起動関数の分岐は 1 か所で値ごとに専用フローへ。
- 入れ替えは純ロジック `conditionSwapPlan`（状態＝`conditionKind` を持つ効果・カスケードの子 `addedFrom` は消すだけ・移した親から `createActiveEffect` フックが再生）→ `toObject()` で複製し削除→生成。`receivedScene` は既存値があれば preCreate が上書きしないので保たれる。適用は RL か両者の操作者（相手のアクターへの書き込み権限）。
- 入手は既存の購入ピッカー（`openOutfitPicker`）に `miracle` を載せ、`startPurchaseWithUsage` が購入の分岐を通さず宣言の効果へ再入（他の神業として使う参照先 `asOtherUuid` も運ぶ）。付与は `grantPurchasedItem`（1 機構）＋複製に `fromMiracle`。アウトフィットシートの常備化経験点欄は印があれば「常備化できない（神業で入手）」。
- 《不可知》の AR 非消費: 実行時に `insensibleFree` を立て、プロセス終了の記帳 `_chargeMajor` が `buildMajorChargeUpdate(system, {arFree})`（CSカレント0 のみ）で読んで消す。純ロジックはテスト済み・**カット進行の実機は未確認**。
- 隔離実機確認: 入れ替え（宿主の負傷＋カスケード→カゲムシャ・宿主は無傷・カード「（適用済み）」）／入手（ブラウザ題名「神業: アウトフィットの選択」・全ボタン有効・複製の `isCheckAcquired`＋`fromMiracle`・カードの「入手」行・シートの表示）／不可知（印と「次の行動」行・攻撃カードの印/命中確定/注記/行クリック無し/印の消費・ダメージカードの印・適用した状態 phys-10/faint が `fromMiracle`）。17-1〜17-5 の回帰 ALL OK。

#### 追記 2026-09-04: 17-5 の作り直し（効果は選んで固定・取得技能から導かない）

ユーザー訂正（verbatim）:「フォルムや神格の種類に応じて変わるのではなく、フォルムとともに神業の効果を選ぶ形式にしてください。あなたのやり方ではスタイルや神業より先にスタイル技能を選んでいることになってしまいます。」

- **メカニクス**: キャラクター作成の順はスタイル → 神業 → スタイル技能。《万能道具》の効果を使用時に取得技能から導くと、技能が神業より先に決まっていることになる。正しくは**〈フォルム〉（神格）を選ぶときに神業側で効果を選んで固定する**。《半身》（リーダー決定時に1つ選ぶ）《神意》（複数属性はプレアクトに1選択）も同じ「選んで固定」の型。
- **設計**: `asOther.mode` は `"choice"`（指定の選択肢から選んで固定）／`"log"` の2種。`choices[{label, uuid}]`（label＝区分の表示用ラベル・uuid＝参照先）と `selected`（選んだ効果の uuid）。神業シートに「効果」の select。使用時は `asOtherSelection` が `selected` を返す（未選択なら警告して中止・選択肢が1つなら自動）。技能の識別キー照合と辞典カスケードは神業側から撤去。旧 `refs`（同日中の未リリース形）は `migrateData` で `choices` へ移す。
- 実機（本体 `1cfbc88`）: 選んだ効果《難攻不落》で防ぐ／シートの「効果」で《死の舞踏》に選び直して即死／未選択で警告／選択肢の追加・ドロップ・区分ラベル・削除。

#### 追記 2026-09-04: 《不可知》の割り込みは完全な割り込み（AR を消費しない挿入メイン）

ユーザー言語化（verbatim）:「不可知の割り込みはCSカレントも変わりません。完全な割り込みです。誰かがメインプロセスを行っていようと、例えば命中判定だけ済んでいてこれからリアクションの解決になるその直前にも割り込むことができます。」「一つ正確でない表現がありました。CSカレントが0になるのはARが消費されるからです。現に、追加行動技能による割り込みでは現状でも追加行動するキャラクターのARは消費されないはずです。」

- **メカニクス**: 《不可知》の行動はカット進行の任意の時点に差し込める割り込みで、AR を消費しない。CSカレント 0 は AR 消費の帰結（一般則 AR−1⟺CS0）なので、AR を消費しない以上 CSカレントにも触れない＝追加行動の割り込み（consumesAr=偽）と同じ扱い。Combat_Flow の「将来の神業による割り込みの特別措置も用途側で例外を明示できる」（2026-07-26）がここで使われた。
- **設計**: 宣言の効果 insensible は、印（次の行動）に加えて `applyInterruptGrantForUsage(actor, {grantsInterrupt: true, interruptConsumesAr: false}, {targetOverride: [自分]})` で自分の combatant に割り込み許可を立てる。開始・終了・AR 記帳は既存の挿入メイン（`startInterrupt`/`endInterrupt`・consumesAr）がそのまま担う。初版の「CSカレント 0 でプロセスを終える」独自処理（`insensibleFree`/`buildMajorChargeUpdate`/`_chargeMajor`）は撤去し、AR の記帳を一般則へ戻した（本体 `70fc65c`）。
- **実機**: 宣言で許可（consumesAr=偽）が立つ→`startInterrupt`＝自分がメイン行動者・退避 1 段・許可は消費→攻撃（印の消費・リアクション不可・状態に印）→`endInterrupt` で AR も CSカレントも不変・元の位置へ復帰。

#### 追記 2026-09-04: 《突然変異》の候補＝「見聞きした神業」／《ファイト！》の重ねがけ

ユーザー裁定（verbatim）:「ファイトの重ねがけは可能です。また、突然変異については『見聞きした神業』なので、基本的には自分が登場している間に使われたものだけだと考えます。ただし、効果文に書かれている通り、誰かが既に登場しているキャラクターを対象に死の舞踏を使用→登場→死の舞踏の効果が対象に適用、という順番なら、登場中に神技が使用されたと見做します。」

- **メカニクス**: 候補＝使用時に自分が登場していた神業。同一シーンで登場前に使われた神業は、登場した時点で効果が**まだ適用されていない**場合だけ候補（使用→登場→適用）。適用済みなら候補外（使用→適用→登場）。初版の「同じシーンで後から登場すれば無条件に候補」は緩すぎたので是正。
- **設計**: 使用ログの行に `messageId`・`applied`・`appliedAppeared` を足す。投稿時に `miracleUsePending`（適用ボタンを持つカード＝神業版ダメージ・破壊・使用回数+1・入れ替え・要求）なら `applied=false`、それ以外は投稿時に適用済み（宣言の適用効果トレイはカードに適用状態を持たないため投稿時扱い＝Code 決定）。`updateChatMessage`（アクティブ RL）で適用を検知し `markMiracleUseApplied` がその時点の登場者を刻む。`miracleLogCandidates`＝使用時に登場 or（適用済み→適用時に登場／未適用→現在シーンでいま登場）。
- **実機**（本体 `207e36b`）: ヒルコ不在で《電脳神》（投稿時適用）と《神の御言葉》（適用待ち）→ヒルコ登場→候補に《神の御言葉》あり・《電脳神》なし→適用でヒルコが刻まれ、次のシーンでも候補のまま。

#### 追記 2026-09-04: 《神出鬼没》の宿主はアクターに置く（RL がアクトごとに設定）

ユーザー（verbatim）:「そもそも神出鬼没の対象が何なのかあなたは理解していますか？」「そもそもこの場合は、カゲムシャスタイルを持つキャラクターに宿主を設定可能な何かを付加すべきです。これがスタイルに付加する者か、アクター自体に付加する者かの議論は必要ですが、少なくとも神業に付加するものではないと思います。それと、宿主の決め方はスタイルの説明に書いてあるはずです」「アクターに載せることに賛成です」。

- **メカニクス（正本＝カゲムシャのスタイル説明・特記事項）**:「カゲムシャはアクトごとに“宿主”を決定する。これはRLが決定する。もし適切なゲストがいない場合、キャストから選択してもよい」。《神出鬼没》の対象は使用時に選ぶ相手ではなく、この宿主。効果文「宿主が死んだあとで《神出鬼没》すれば、あなたが代わりに死亡し、宿主はふたたびアクトに参加できる」は、完全死亡の状態が宿主から自分へ移ることでそのまま成立する。
- **設計**: `host{uuid, name}` を **CharacterBase（cast/guest 共通）**に置く（カゲムシャのスタイルを持つキャラクター＝ゲストもあり得る。name は削除時のフォールバック表示のみ）。シート見出しの「所属」と同列に「宿主」の行（カゲムシャのスタイル＝識別キー `kagemusha` を持つときだけ・RL だけがドロップで設定／解除・トループの所有者欄と同方式）。アクト終了の境界で `planActorBoundaryUpdates` が消す。《神出鬼没》は対象を取らず、宣言した時点で `swapConditionsBetween(自分, 宿主)`（RL クライアント）か `miracleSwap` ソケットで RL へ委譲。カードは「宿主: ○○」の行のみ。使用ログの適用待ちからも外す（宣言で即時）。
- **Code の誤り（記録）**: 初版は「ターゲットした単体＋適用ボタン（RL か両者の操作者）」で、宿主の決め方を読まずに裁定を仰いだ。また B 案の短所に「スタイルは辞典データの写し」と書いたが、所有アイテムは複製時点でアクター固有の埋め込みドキュメントであり誤り（撤回・[[feedback-cite-source-before-claims]]）。
- **実機**（本体 `e240904`）: 宿主未設定で警告・消費なし／キャストシートに「宿主：未設定」とドロップ受け／RL のドロップで設定・名前表示／宣言で即時に宿主の負傷＋カスケードがカゲムシャへ移り宿主は無傷／カード「宿主 REPRO宿主」／アクト終了の境界で宿主が消える。RL 委譲ソケット（PL クライアント）の経路は単一クライアントの隔離環境では未確認。

### 2026-09-05: ユーザー実機確認による神業の是正（タイミング・カード表示・選択肢 UI・使用ログ・使用済みフラグ）

**2026-09-05** / **関連実装** `templates/app/usage-sheet.hbs`（タイミング欄の抑止）・`scripts/module/damage-flow.mjs`（`renderMiracleDamageCard` の再設計）・`scripts/item/tnx-miracle-sheet.mjs`＋`templates/item/miracle-sheet.hbs`＋`css/tnx2.css`（選択肢の2ドロップ・1行化）・`scripts/data/item/miracle.mjs`（`choices[{skillUuid, uuid}]`・`isUsed` 廃止）・`scripts/module/session-state.mjs`＋`scripts/module/miracle-flow.mjs`（登場者の出どころ）・`scripts/module/miracle-logic.mjs`・`scripts/tnx.mjs` / **ステータス**: 是正済・隔離実機確認済（本体 `44445fa`）

**メカニクス監査**:

- **神業にタイミングの概念は無い**（ユーザー確定 2026-09-05・verbatim「根本的なこととして、神業にタイミングの概念はありません。強いて言うなら『いつでも』です」）。用途の器はタイミングを持つが、神業専用タイプでは欄ごと出さない。カット進行への割り込み（《不可知》）はタイミングではなく割り込み許可（consumesAr=偽）で表す＝2026-09-04 の設計と整合。
- **使用回数管理の存在意義**（ユーザー質問への回答）: 神業は「アクト中にスタイルレベル回」使え、《ファイト！》が母数を +1 する。したがって**残量を数える器（uses.max/spent と実効 maxTotal）が実体**。`isUsed` は uses 一本化（2026-07-18）以前の名残で、リセットのトリガー以上の意味を持たず、「使用済み」は残り 0 から導ける。二重管理なので廃止した。
- 《突然変異》の「見聞きした」の判定材料は**登場状態**であって経験点の集計ではない。`appearedThisScene` はキャストの登場シーン数を数えるための配列（1シーン1点の重複防止）で、ゲスト・トループは入らない。ゲストが宿主や神業の使い手になる卓では候補が常に空になっていた。

**設計レビュー**:

- **神業版ダメージカードの表示**: 台帳は［神業］［効果］［カード n］［結果］の通常行のみ。`cr-total-row`/`cr-total-num`（1.8em）は**数値のための器**で、「完全死亡」のような状態名を載せるものではなかった。状態領域は対象ごとに［対象名］［結果］の通常行＋防がれた対象の説明行。**防がれた対象の行を消さない**のが要点——消すと「防げなかった」ように読める。全対象が防がれた/打ち消されたときは台帳の結果に取り消し線（`cr-calc-row--voided`）。打ち消しはカードごと無かったことになるので1行にまとめる。
- **選択肢 UI**: 1行＝[区分の技能][参照先の神業]の2ドロップ（flex・1行1選択肢）。〈フォルム〉9件で行の高さ 38px・一覧できることを実機で確認。区分だけ文字列入力だったのは非対称（神業はドロップ）で、ユーザーの事前の話とも違っていた。表示は uuid のライブ解決（名前キャッシュを持たない）。
- **使用ログの登場者**: `listAppearingActors()`（`flags.appearing`・全キャラクター種）を単一の出どころにし、記帳（使用時）と適用時の両方で使う。アクト未開始のゲートも外した（ログはアクトの読み込み・終了で消えるので実害がなく、テスト・準備中の使用が黙って落ちる方が害）。
- **実機**: 9件の選択肢シート／神業版ダメージカードの通常・防がれた・打ち消された／用途シートのタイミング有無（神業＝無し・一般技能＝有り）／**ゲスト同士の《突然変異》**（ゲスト A の使用がログに載り、ゲスト B がコピーできる＝ユーザーが踏んだ経路の再現）。17-1〜17-6 の回帰 ALL OK。

### 2026-09-05: 戦闘不能タグの ［］ 表記と「防いだ」の文言

**2026-09-05** / **関連実装** `scripts/module/conditions.mjs`（`conditionDisplayName`）・`condition-resolution.mjs`・`damage-flow.mjs`・`miracle-flow.mjs`・`miracle-logic.mjs`・`time-boundary-logic.mjs`・`tnx-usage-sheet.mjs`・`damage-chart-text-app.mjs`・`recovery-flow.mjs`・`tnx.mjs` / **ステータス**: 是正済・隔離実機確認済（本体 `7ed80fc`）

**メカニクス監査**:

- ユーザー指示（verbatim）:「完全死亡などの直接のダメージではない、ダメージによって与えられる戦闘不能状態のタグは、必ず［］をつけてください」。ルールブックの表記（［完全死亡］［精神崩壊］［抹殺］［支配］…）に合わせる。**対象は戦闘不能グループのみ**——BS と負傷（＝ダメージそのもの）は名前のまま。
- 「防がれた」の主語の誤り（verbatim）:「『（キャラクター名）』は《（防御神業名）》で防がれた」とはどういう意味ですか？ → 防御神業が防ぐのは**その対象への結果（ダメージ・状態の適用）**であって対象そのものではない。文言を「《難攻不落》が「対象X」への［完全死亡］を防いだ」に是正。

**設計レビュー**:

- **表示名の単一の出どころ**: `conditionDisplayName(kind, {quote})`。戦闘不能は ［］、それ以外は素の名前（`quote` で 「」）。`CONDITION_KINDS.label` は素の名前のまま（照合・並び・データ）で、**表示だけを一段挟む**。これにより ［］ の適用漏れが構造的に起きない。従来は表示側が `def.label` を直に読んでいて、`miracle-logic` には終端ラベルの重複定義まであった（撤去）。
- 適用範囲: チャットカード（神業版ダメージの結果・対象行・適用結果／通常ダメージの適用文）／制御判定の打診と結果（無効化・降格・継続）／付与する ActiveEffect の名前／`CONFIG.statusEffects`（トークン HUD・効果の要約）／用途シートの除外タグ・回復範囲／回復候補の一覧／ダメージチャート本文／結果の選択ダイアログ。未知のキーの落とし先（空文字・元の効果名）は従来どおり保つ。
- **防いだ表示は通常のダメージカードにも入れた**: 従来は防いだ対象の行が消えるだけで、防げたのか判断できなかった（9/5 の「防御神業の最終結果が分かりづらい」と同じ理由）。
- 隔離実機: タグの ［］ 表記（ヘルパー・ステータス効果・カード・適用結果・効果名・選択肢）／「《難攻不落》が「対象Y」への［完全死亡］を防いだ」／「《チャイ》がこの神業を打ち消した」／通常カードの「《守護神》が「対象X」へのダメージを防いだ」。17-1〜17-6 の回帰 ALL OK。

#### 追記 2026-09-05: カードの折り返しは機構で解決する（keepTogether）

ユーザー指摘（verbatim）:「意味不明に改行されるのもやめてもらって良いですか？何回目ですか？この手の修正。良いかげんにしてくれよ」（［完全死亡］が「［完／全死亡］」に割れていた）。

- **設計**: カードは幅が狭く（~250px）日本語はどこでも改行されるため、**文言を短くする対症療法では別の語で再発する**。`scripts/module/chat-text.mjs` の `keepTogether(html)` が**囲みで閉じた語**（［状態タグ］《ルール用語》「名前」）を `<span class="tnx-nobr">`（`white-space: nowrap`）でくくる。カード幅に収まらない長い塊（14文字超）は対象外＝従来どおり折り返す（nowrap にすると横にはみ出す）。**カードに文字列を組む箇所は必ずこの関数を通す**（ダメージカードの行・成否行、神業カードの段とボタン、攻撃カードの対象行・注記）。
- `.cr-result` は flex（語間 gap つき）なので、塊の span がそのまま flex アイテムになると語間が空く。本文を1要素（`.cr-result__text`）にまとめて回避。
- **確認の仕方**: 目視だけでなく、`Range#getClientRects()` で「塊が行をまたいでいないか」を自動判定する（`T:/pw/check-wrap.mjs`）。横はみ出しの自動検査と対で回す。
- **閾値は実測で決める（本体 `2d7ee55`）**: 折らない塊の上限は 10 文字。カード内で最も狭い段（対象名＋値の2列行）でも収まる長さで、状態タグ（完全死亡=4）やよくある神業名（難攻不落=4）は収まる。15文字級の固有名を nowrap にすると**横にはみ出す**（はみ出しの方が読めない）ので対象外にする。
- **文は受動で書く（本体 `a28ee85`）**: 行の値は**その対象に起きたこと**を述べる場所なので、神業を主語にすると神業自身が行為者になる（ユーザー指摘「《難攻不落》「が」はおかしくないですか？」）。「《難攻不落》で防がれた」「《チャイ》で打ち消された」。先の是正で問題だったのは**主題を対象名にしたこと**（「『対象』は…防がれた」）であって、受動そのものではない。
- **囲みで閉じていない決まり文句も塊**（`nowrap()`）: 「で防がれた」「で打ち消された」は `keepTogether` が拾えず「で防がれ／た」と割れる。組み立て側が塊と分かっている短い言い回しにだけ使う。
- **行の幅の取り合い**: 防いだ行の対象名は行の 60% までは折らない（`.cr-calc-row--prevented.cr-calc-row--wrap .cr-calc-label`＝後段の `--wrap` より強い2クラス指定）。放置すると値と幅を取り合って短い名前まで途中で折れる。
- **防いだ表示は対象行と同じ形に**（ユーザー指摘「『（キャラクター名）』への」は本当に必要ですか？いたずらにメッセージを長くしているようにしか見えません」・本体 `2d7ee55`）: 生きている対象の行が「名前＝ラベル／結果＝値」なので、防いだ対象も同じ形にして値は「《難攻不落》が防いだ」だけにする。**対象名は行のラベルに出るので繰り返さない**（何を防いだかは台帳の「結果」にある）。通常のダメージカードも同じ形。対象名のラベルは可変長なので `cr-calc-row--wrap` を付ける。
- 本体 `d4224b0`。テスト2544件（`keepTogether` の単体テスト+4）。あわせて確認スクリプトは開始時に山札を戻す（引ききっているとカードが引けず判定が進まない＝実行環境の落とし穴）。


### 2026-09-05: チャットカードの形式を統一規格にする

**2026-09-05** / **関連実装** `templates/chat/parts/card*.hbs`（新設）・`templates/chat/*.hbs`（19枚）・`scripts/module/chat-card.mjs`（新設）・`scripts/tnx.mjs`（別名パーシャル登録）・`css/tnx2.css` / **ステータス**: 実装済・隔離実機確認済（本体 `262176e`）

**設計レビュー**（ルール解釈は伴わない＝メカニクス監査の対象外）:

- ユーザー指示（verbatim）:「『同じ仕組みを入れる』のではなく、そもそもチャットカードの形式を完全に統一規格にしてください。別々の場所で別々に作っているからこういうことになるのではないですか？」。発端は「なぜ神業のチャットカードの効果文だけ折りたためないのですか？」。
- **全数調査の結果**（指摘は正しい）: カードは 19 枚。骨格は既に 2 系統に寄っていた（送信系 `tnx-check-request` 8 枚／結果系 `tnx-check-result` 10 枚／骨格外 1 枚）が、
  1. 同じ役割の段に複数の名前があった（ラベル＝値の行が `cr-req-field` と `cr-calc-row`、帰結の行が `cr-flow-outcome`/`cr-appearance-outcome`/`cr-info-outcome` の 3 つ）
  2. 「長文を畳めるか」が部品でなく**カードごとの判断**だった（畳めるのは判定内訳とアイテム解説だけ）
  3. テンプレートを通らず JS で素の HTML を組む経路が 5 つ残っていた
- **統一規格**: 骨格 `tnxCard`（器＋見出し＋本文）と段 5 種（`tnxCardField` ラベル＝値／`tnxCardRow` 値が組み立てを要する行／`tnxCardFold` 畳める内訳／`tnxCardText` カード本体の長文／`tnxCardResult` 結果行）を別名パーシャルで登録し、**カードを作る側はこの部品しか使わない**。JS からライブ描画する段（ダメージ台帳・対決の状態領域・状態の解決）は `chat-card.mjs` の同型の部品を通す（`keepTogether` もそこで一括して掛かる）。
- **見出しは全カードで「種別タグ＋名前」**。効果文などの長文を持つカードは**見出しそのものが開閉の起点**（`details`/`summary`）。これは解説カードが以前から採っていた作法で、**畳むための見出しを別に足さない**。
  - 経緯（重要）: 当初 Code は「効果文」という見出し行を新設した。ユーザー指摘（verbatim）「『効果文』とかいう謎見出し辞めれます？意味が分かんねぇよいつ私がそんな指示を出した？」「そもそもカードのヘッダーそのものが見出しで、折り畳み、展開もそこが起点だったはずだ」。**既存の作法を調べずに新しい機構を作ったのが誤り**（UI 文言の一斉規則＝「UI はメモ帳ではない」にも反する）。
- **神業カードの効果文は畳んで置く**（ユーザー指示 verbatim「神業は判定と違って組み合わせることもありません。そのため、基本的には何か具体的な効果があっても最上部に効果文を折りたたんでおいて置く形式に統一しましょう」）。使用の種類による出し分け（宣言だけ開く）は撤去。アイテム解説カードは**効果文を見せるためのカード**なので開いた状態（畳むと何も残らない）＝機能による違いであって、同じカード内での場当たりな出し分けではない。
- **過去ログの互換（設計上の落とし穴）**: チャットカードは**投稿時の HTML がそのまま保存される**ため、クラス名を変えるとログに残っている過去のカードが全て素の HTML に崩れる。実機で「全てのチャットカードのCSSが壊れている」と報告されて発覚（Code の隔離環境では新規カードしか描いていなかったため気づけなかった）。対処は**統一規格の規則に旧名を別名として並べる**（規則は 1 つのまま＝二重管理にしない）。骨格ごと変わったカード（`details` でくくっていた解説カード・独自枠のニューロカード）は当時の意匠を「過去ログ互換」の節に残す。**今後もカードのマークアップを変えるときは、保存済みのカードがどう見えるかを必ず確認する。**
- **パーシャルの文脈の落とし穴**: Handlebars のパーシャルは囲みの文脈を引き継ぐため、骨格に渡した `modifier` が中の段へそのまま流れ込んでいた（ニューロカードの行が枠ごと二重になって発覚）。**段ごとに別の指定名**（`cardClass`/`fieldClass`/`foldClass`…）にして構造的に防ぐ。
- **確認**: `T:/pw/check-cards.mjs`（新設）が全 23 種のカードを実機で描画し、(1) テンプレートが壊れていない (2) 見出しが「種別タグ＋名前」でそろっている (3) 横にはみ出さない (4) 囲みで閉じた語が行をまたがない (5) 畳める段が既定で閉じている を自動判定する。旧形式カード 4 種の再描画も確認。17-1〜17-6・タグ表記・折り返し・効果文の畳みの回帰 ALL OK。テスト 2546 件。

#### 追記 2026-09-05: 見出しの作法と、タグに内部分類を出さないこと（本体 `9aba074`）

- **畳みの起点は見出し**（ユーザー指摘 verbatim「『効果文』とかいう謎見出し辞めれます？意味が分かんねぇよいつ私がそんな指示を出した？頼むからいい加減にUIをお前のメモ帳扱いするのをやめてくれ」「そもそもカードのヘッダーそのものが見出しで、折り畳み、展開もそこが起点だったはずだ」）。既存の解説カードが `<details><summary><h3>名前</h3></summary>` で**カードの見出しを summary にしていた**のに、Code はそれを調べずに「効果文」という見出し行を新設した。撤去し、骨格 `tnxCard` が長文を持つとき見出しを `summary` にする形へ。長文は `{{#*inline "cardProse"}}` で渡す。
- **タグは卓に意味のある種別だけ**（ユーザー指摘 verbatim「何故『一般アウトフィット』というFVTT上の分類でしかないものがタグになっているのですか？何故分類表示のチップがこんなに大きいのですか？」）。Code は「見出し＝タグ＋名前」という自分の規格を満たすために `TYPES.Item.<type>`（FVTT の文書型）を引っ張り出した。**規格を満たすために存在しない情報を作らない**——タグ無しを許す形にし、アイテムの解説カードは名前だけ（従来と同じ）、スタイルの解説だけ在ゲームの区分として「スタイル」を出す（従来のカードも「スタイル: 名前」だった）。チップが大きく見えたのは長い文字列で見出しが圧迫され名前が2行に折れていたためで、寸法は他カードと同じ。
- 教訓（[[feedback_reuse_existing_ui_precedent]] / UI 文言の一斉規則）: **既存の作法を調べる前に新しい機構を作らない**。統一規格の作業中でも、規格に合わせるために UI へ語や分類を足してよいわけではない。

#### 追記 2026-09-05: 見出しの配置・タグの内容と、pre フックの落とし穴（本体 `3bf3c57`）

- **タグは残す。直すのは見た目**（ユーザー指摘 verbatim「タグを新設したことはあなたの過ちの本質ではありません。最悪なのは、タグが大きすぎること、単に配色が明度の高いベースに白文字で視認性が悪いこと、タグのせいで本体である見出しが折り返されて本末転倒と化していることです」「指摘されたから消す、などというしかられそうになって原因となるものを隠そうとする小学生のようなしぐさをやめてもらえませんか？」）。**指摘の原因を直さずに対象を消すのは対処ではない。**
- **見出しの配置は「名前（左）＋種別タグ（右寄せ）」**（ユーザー指摘「元々タグは左寄せではなく右寄せだったはずです」＝元の判定カードの配置）。タグに押されて名前が折り返すときはタグを次の行へ回す。
- **タグの意匠も元の判定カードのまま**（枠線＋シアン文字）。塗りつぶしにすると、チャットカードに本文色を敷く安全網（`.chat-message .message-content [class*="tnx-"]`＝詳細度 0,3,0）に負けて「明るい地に白文字」になり読めない。文字色はその安全網より詳細度を上げて置く。
- **タグの内容はアウトフィットの分類**（小分類→大分類。`itemKindLabel`）。FVTT の文書型（「一般アウトフィット」）は保管上の区分でしかない（ユーザー指摘「アイテムの分類を出しても意味がありません」）。
- **Foundry の pre 系フックを async にしない**（KI-051）。戻り値は同期で見られるため `return false` が効かず、**神業の母数-1が機能していなかった**。UI の作業中に見つけたコンソールエラーを「無関係な既存不具合」と切り分けた時点で、**症状の裏にルールの欠落がある可能性を確かめるべきだった**（影響度の見立てを誤った）。

#### 追記 2026-09-06: トループの壊滅（即死用途）の対象解決と呼び名（本体 未コミット→次コミット・KI-052 / KI-053）

**メカニクス監査**（ルール解釈の側）:

- 正本: 《天変地異》効果文「アウトフィットをひとつ［破壊］したり、トループ級のキャラクターを1体壊滅させることができる」／《制裁》「トループならば全滅させてもよい」。トループ壊滅は**破壊ではなく即死用途**で表す（ユーザー裁定 2026-09-04）。適用の実体は `heads.value → 0`（Troops.md の人数／エニグマポイント）。
- **監査の結果**: 適用（`miracleTargetOutcome` → 壊滅）はルールどおりだったが、**卓に見える呼び名がルールと食い違っていた**——カードは対象がトループでも ［完全死亡］ と書いていた。トループに完全死亡は起こらない（人数が 0 になる）ため、これは表示の誤りであってルールの誤りではないが、**卓からは「壊滅させられていない」と読める**。呼び名を対象の型で決めるよう是正（KI-053）。
- エキストラ（ダメージの概念が無い＝宣言死）の行の呼び名は未確定。ルール解釈が要るためユーザーの裁定を待つ（勝手に語を作らない）。

**設計レビュー**（技術的妥当性の側・ルール解釈は伴わない）:

- **同じ規約を 2 か所で持たない**。「対象欄が空でもレティクルを尊重する」（2026-09-06 裁定）を破壊・干渉の前段にだけ入れたため、即死・社会戦が取り残されていた（KI-052）。神業の対象解決は `resolveMiracleTargets` 1 つに集約した。**裁定を実装するときは、その規約を必要とする経路を全数で洗う**（今回は `resolveUsageTargetRefs` の呼び出し 3 か所を全数確認した）。
- **表示の権威も 1 か所**。終端結果の呼び名は `miracleResultLabel(result, category, targetTypes)` に集約し、結果の選択ダイアログ・カードの結果・対象行・適用後の記録がすべて同じ関数を通る。先行修正（`aebb9a3`）はダイアログにだけ壊滅を出しており、権威が分かれていたことが KI-053 の直接の原因。
- **調査の手順**（再現できない報告に対して）: ユーザーのワールド・コンペンディウムの**実データを読んで**（LevelDB を複製して走査）テスト対象を特定し、隔離 Foundry で**実データのまま**歩いた。結果、機構そのものは動いており、詰まるのは (a) 対象欄が空の経路 (b) 呼び名 の 2 点だと切り分けられた。机上の読みだけでは (a) に到達できなかった。
- **未リンクトークンの見え方**（仕様として記録）: トループのトークンは既定で非リンク（`tnx.mjs`・「複数部隊を並べる」）。壊滅は**そのトークンのアクター**に効くため、サイドバーのアクター（プロトタイプ側）の人数は変わらない。卓での確認はトークンのリソースバー／トークンのシートで行う。

#### 追記 2026-09-06: 写した経験点の取得条件の神業名（KI-055・本体 `ad19ff6`）

**メカニクス監査**（ルール解釈の側）:

- 正本: 《万能道具》効果文「**《万能道具》を「うまく使った」条件は、元となった神業と同じである**」。ユーザー指示（verbatim）:「万能道具でインポート時に効果を選択した際、経験点取得条件を書き換えるかと思いますが、その文中の元の神業の名前を《万能道具》に書き換えられませんか？」＋《ファイト！》の条件文を例示（「《ファイト！》を使用することで…ただし、《ファイト！》を《ファイト！》することはできない」の《ファイト！》を《万能道具》に置き換える）。
- **監査の結果**: 写す内容（条件は元の神業と同じ）はルールどおりだったが、**文の主語が写し元のままだった**。「元となった神業と同じ条件」を卓が読む文にすると、その文を名乗るのは《万能道具》の側なので、文中の神業名も《万能道具》になる。最後の一文（自分自身には使えない）も、写し先の名前で読んで意味が通る（《万能道具》＝いまは《ファイト！》なので、《万能道具》を《万能道具》することはできない）。
- **《突然変異》（使用ログからのコピー）は対象外**。あちらはアイテムへ写さず、使用のたびにコピー元の文をカードに出す（効果文「『うまく使った』条件は、コピーした神業のものと同じ」＝どの神業をコピーしたかが読み手に見える方がよい）。

**設計レビュー**（技術的妥当性の側）:

- 置き換えは純関数 `renameMiracleInCondition(html, fromName, toName)`（`miracle-logic.mjs`）。神業名は 《》 で囲って書かれるが、**囲みの内側（名前そのもの）で置き換える**ので、囲みの有無どちらの書き方にも届く。名前が空・同名・条件が空なら元のまま。
- 呼ぶのは写しの 1 か所（`asOtherCopyUpdate`）だけ＝**インポート時（作成フックの自動決定）も、シートで効果を選び直したとき（`updateItem` フック）も同じ経路**を通る。
- **既存データは書き直さない**（[[feedback_no_silent_data_rewrite]]）。すでに写し済みの神業は、シートで効果を選び直せば書き直る。
- テスト 3 件（例示の文の全置換／囲みなしの書き方・名前が出てこない条件／空・同名）。全 2569 件グリーン・eslint 0 error。
- **実機確認（隔離 Foundry・Playwright）**: 選択肢1つの《万能道具》をアクターへ追加 → 自動決定で条件が「《万能道具》を使用することで…ただし、《万能道具》を《万能道具》することはできない」・用途も写る。選択肢を2つに増やして選び直すと、条件が新しい参照先の文（名前は写し先）へ差し替わる。神業シートの「条件」欄の表示も置き換え後の文。

#### 追記 2026-09-06: 即死用途の系統に「トループの壊滅」を足す（KI-054）

**メカニクス監査**（ルール解釈の側）:

- 正本: 《天変地異》効果文「住居やヴィークルなどのアウトフィットをひとつ［破壊］したり、トループ1グループを全滅させることができる」「シーンに登場しているエキストラたちを退場させることもできる」「**この効果で、キャストやゲストに直接ダメージを与えることはできない**」／《突破》「アウトフィットをひとつ［破壊］したり、トループ級のキャラクターを1体壊滅させることができる」。
- **監査の結果（ユーザー指摘）**: 実装は即死用途を「誰に使ってもよい即死」とし、対象の型でだけ分岐していたため、《天変地異》でキャストを ［完全死亡］ にできた＝効果文の禁止を満たさない。ユーザー指摘 verbatim「えっと、ですから、トループの壊滅しかできない神業なのに完全死亡もできては意味が無くないですか？どういうつもりなのですか？」。
- **裁定**（verbatim）: 「**肉体・精神の選択肢と同様にトループの壊滅を選択できるようにする**」＝即死の「系統」を 肉体／精神／トループの壊滅 の3択にする。
- **《制裁》《神の御言葉》は従来どおり**。「1人（トループは1グループ＝全滅可）に任意の社会戦ダメージ」「同シーン・中距離の1人（トループは1グループ）を精神崩壊」＝**両方できる**神業なので、対象の型で分岐する作りが正しい。壊滅系統は「壊滅**しかできない**」神業のためのもので、両者は別物として並ぶ。
- **エキストラ**は壊滅系統でも従来どおり適用なし（宣言死）。「エキストラたちを退場させる」は RL の口頭処理。

**設計レビュー**（技術的妥当性の側）:

- **ルールが課す制限を運用に逃がさない**。17-3 では《天変地異》《突破》の制限を設定に落とさず、Miracle_Data_Setup に「即死用途はトループの壊滅にだけ使う（運用）」と注記して済ませていた。用途タイプが同じでも「その神業に何ができるか」は神業ごとに違い、そこは設定で表す。注記は撤回した。
- **既存の選択肢に足す形で表す**（新しいフラグを増やさない）。系統は既に「その用途が何を与えるか」を決める設定なので、壊滅はその3つ目の値。DataModel は `killCategory`（StringField）の値追加のみでスキーマ変更なし。
- **トループの壊滅はダメージであり、防御神業で防げる**（ユーザー指摘 verbatim「トループの壊滅はダメージなのだから防げるに決まってるだろ」）。壊滅は「防げる系統」の3値（肉体/精神/社会）に載らない値なので、`defencePreventPlan` は category が壊滅のとき系統照合を飛ばす（弾くと防げなくなる）。**この点は二重に誤った**——(1) 答えが正本（Miracle_Rules「防御神業」＝防ぐものはダメージに限らず悪影響を含み、何を防ぐかは使用者が選ぶ）にあるのに質問し（ユーザー verbatim「ルールを読め」）、(2) その後「壊滅はダメージではない」という誤った前提でカードの種別タグを「神業」に変えた（撤回済み）。**ルールの事実を推測で決めない。**
- **起こらない状況にガードを作らない**（ユーザー verbatim「意図が分かりません。用途で選択できるのになぜそのような状況になるのですか？」）。壊滅系統で対象にトループがいない場合の特別処理は入れない——対象ごとに「効果なし」と理由を出すだけで足りる。
- **種別タグは「ダメージ」のまま**（`damage-card.hbs` は無変更）。壊滅もダメージなので、タグを変える理由が無い。

#### 追記 2026-09-06: 破壊用途に「破壊できるアウトフィットの分類」を持たせる

**メカニクス監査**（ルール解釈の側）:

- 正本: 《天変地異》「**住居やヴィークルなどの**アウトフィットをひとつ［破壊］」／《突破》「**建物やヴィークルなどの**アウトフィットをひとつ［破壊］」（[Miracle_Effects.md](../01_Wiki/Game_Rules/Miracle_Effects.md)）。
- 17-3 の実装は「対象の未破壊アウトフィット全部」を候補にしていた。神業ごとに壊せる範囲は違うので、**その神業に何が壊せるかは用途の設定で表す**（即死の系統を3択にしたとき（KI-054）と同じ考え方＝ルールが課す制限を運用に逃がさない）。
- ただし効果文は「住居やヴィークル**など**」と例示止まりで、範囲を列挙して閉じてはいない。よって**既定は制限なし**とし、卓が狭めたいときだけ分類を選ぶ。ユーザー裁定（2026-09-06）: 「**空欄を許容せず、初期値を全てにしてください**」→ 初版が「全て」を大分類の列挙で表したのに対し「**『全てを対象にする』ということを表わす選択肢を用意してください**」＝**「全て」は専用の値（`all`／表示「全てのアウトフィット」）で表す**。列挙では「たまたま全部選んだ」状態と区別がつかず、分類が増えたときにも追随しない。個別の分類とは排他（「全て」を選べばそれだけになり、分類を選べば「全て」が外れる）。
- **サービス大分類は「全て」にも含まれない**。サービスは故障・破壊の免疫（2026-07-18 ユーザー確定）で、壊せないものを候補に並べる意味がない（選択肢にも出ない＝`buildCategoryKeyGroups` の既定）。免疫の除外はホワイトリストと独立に候補列挙で行う。結果として、破壊の候補にコネ等のサービスが並ばなくなった（従来は `isOutfitDestroyed` が免疫で常に false のため候補に出るが、選んでも何も起きなかった）。

**設計レビュー**（技術的妥当性の側）:

- 器は**修理用途 `repairableCategories` の踏襲**（大分類キー＝その大分類全体・小分類キーの混在した1配列）。照合も同じく分類集合（主分類＋副分類＝「両方の分類として扱う」）。新しいキー空間を作らない。
- **既存データの移行スクリプトは不要**。`actions` は `ArrayField(SchemaField)` で、欠けたフィールドは初期化時に `initial` が入る＝既に作ってある《天変地異》《突破》の破壊用途も「全て」で埋まり、挙動は従来どおり（隔離実機で実測：保存値を持たない破壊用途が `["all"]` になる）。[[feedback_no_silent_data_rewrite]] に触れない。
- **空欄の禁止は二重ガード**。行が1つのときは削除トリガーを共通機構 `applyTriggerDisable`（グレーアウト＋クリック不能・[[feedback_extract_reusable_mechanisms]]）で封じ、`_onDestroyCategoryDelete` でも最後の1つを弾く。純関数側は空を「候補なし」とし、**「全部壊せる」へフォールバックしない**（設定できない状態に第二の意味を与えない）。
- 選択済み行のラベル生成は用途シートのインライン実装を `categoryKeyLabel(key)`（`outfit-categories.mjs`）へ抽出し、**修理と破壊で共用**。未知のキーはキーをそのまま返す（旧実装は `／key` と大分類が空の形で出ていた）。
- 使用時の絞り込みは純関数 `listDestroyableOutfits(target, usage)`（`miracle-logic.mjs`）に置き、`miracle-flow.mjs` はそれを呼ぶだけ。テスト8件＋分類ラベル4件＋スキーマ2件を TDD で追加、全 2583 件グリーン・eslint 0 error・stylelint clean。
- **実機確認（隔離 Foundry＋Playwright）**: 破壊用途の「効果」タブに〔破壊〕枠が出て、保存値を持たない用途が「全てのアウトフィット」1行になる（× は淡色・`cursor: not-allowed`・`data-action` が外れ、クリックしても消えない）／「全て」の状態でプルダウンから「全て」が消える／ヴィークルを選ぶと「全て」と入れ替わり、白兵武器を足して「全て」を選ぶと1行へ畳まれる／小分類は「武器／白兵武器」の形で並ぶ／追加プルダウンにサービスが無い／破壊以外の用途タイプには枠が出ない／**分類をヴィークル＋住宅にして実際に使用すると選択ダイアログの候補が「新規地上車両」「新規住宅施設」だけになり、「全て」では対象の全アウトフィットが並ぶ一方、対象に持たせたサービス（コネ）は「全て」でも並ばない**。
- 併せて `.usage-icon-btn.tnx-trigger-disabled` の `cursor`/`hover` を追加（汎用 `.tnx-trigger-disabled` が CSS の後方にある `.usage-icon-btn` に上書きされ、無効化しても指カーソル＋赤ホバーのままだった）。

#### 追記 2026-09-07: 神業書き換え技能（17-7）

**メカニクス監査**（ルール解釈の側）:

- 正本: [Miracle_Rules.md](../01_Wiki/Game_Rules/Miracle_Rules.md)「神業書き換え技能」（2026-09-06 ユーザー言語化）。「神業と同じタイミングで使用することで、いままさに使用しようとしている神業の効果を書き換えるスタイル技能」で、効果は4種類——(1) 別の神業と同じ効果＋取得条件もその神業と同じ、(2) 別の神業と同じ効果＋取得条件は変わらない、(3) 独自の効果＋取得条件は変わらない、(4) 独自の効果＋取得条件も変わる。
- **4種類は2軸で尽くされる**（Code の読み・ユーザー承認済み）: 「効果の出どころ（別の神業／その技能自身）」×「経験点の取得条件を書き換えるか」。4つを別々の機構にすると同じものが4つできる。
- **書き換わるのはその1回の使用に限る**（ユーザー明示 verbatim「当然神業の効果はその一回の使用に限り書き換わります」）。よって《万能道具》の「実体を写す」（方針A・アイテムに焼き込む）ではなく、**実行時の差し替え**（《突然変異》のコピーと同じ側）に置く。
- **書き換えられるのは自分の神業だけ**（ユーザー裁定 2026-09-06）。《プリーズ！》で使わされた神業は、使う本人が自分の書き換え技能で書き換える形になる（索引表の「神業変更技能は両者同意で使用可」はこの形）。他人の神業を書き換える形は取らない＝待ち受けを他クライアントへ配る仕組みが要らない。
- **1つの神業に重ねられる書き換えは1つ**（ユーザー裁定 2026-09-06）。打ち消しのような多段の連鎖はしない（「どちらの効果になるか」「条件はどちらに従うか」という解決順の問題が生じないため、器も単一選択でよい）。
- 神業の使用回数は**元の神業から**通常どおり消費し、書き換え技能は技能自身の使用回数を消費する。「書き換える」を選んだ時点で技能は使用済みになる（宣言したため。以降の操作を中止しても戻らない＝戻すのは RL の裁量）。

**設計レビュー**（技術的妥当性の側）:

- **起動の形はユーザー提案を採用**（Code の初版＝「技能を先にロールして待ち受け、次の神業で発効」は撤回）。ユーザー案「**その神業をロールした時点で、対応する書き換え技能を使用回数を残して持っていれば書き換えるかを尋ねる**」の方が優れている: (1) 待ち受け状態を持たない（`_clickState` に種類を足さずに済む）、(2)「どの神業を書き換えるのか」が技能側の結線で明示される、(3) 出る場面が正しい。配線されないまま残っていた旧欄「書き換え対象の神業」は、**むしろ必要な欄だった**＝撤去案を取り下げてドロップの結線に作り直した。
- **実行は 17-5 の差し替えレールに合流**（新しい経路を作らない）。`asOther{uuid, name, source, kind, via}` の `source` が用途と効果文の出どころになる仕組みは既にあり、参照型は `source`=参照先の神業、独自型は `source`=技能自身。即死・防御・社会戦・破壊・宣言の**全分岐は無改修**で書き換え後の効果を実行する。
- **印は「名乗る神業」のまま**。`kind="skill"` のときは `asOther` を印に載せない——技能は神業ではないため、使用ログ（《突然変異》のコピー元）として解決できない。書き換えは `rewrite{itemId,name}` として別に載せ、カード（神業カード・神業版ダメージカード）に「書き換え 〈技能〉」の行を出す。
- **効果文と条件の出どころを分けた**のが唯一の構造変更。従来は `asOther.source ?? item` から効果文も条件もまとめて取っていたが、「効果だけ書き換えて条件は元のまま」を表せない。`miracleCardConditionPlan(asOther)` が `item`／`source`／`text` を返し、`postMiracleCard` がそれに従う。書き換えを伴わない従来の参照・コピー（《突然変異》）は `source` のままで**挙動不変**。
- **用途の器**: 独自効果型の書き換え技能では用途作成に**神業専用の5種**を出す（`usageTypeLabelsFor(parentType, {miracleOwnEffect})`）。その用途は技能自身の使い方ではなく書き換えた神業の効果そのものだから。神業アイテムの判定（`parentType === "miracle"`）に条件を足す形で、キー空間も既定タイプ（宣言）も共通。
- **候補の判定は区分（`unique === "miracleChange"`）と設定の両方**を見る。シートが設定欄を出す条件と揃え、区分を変えた技能の設定が裏で効き続けないようにした。純ロジック `miracleRewriteCandidates` はさらに「対応する神業（空欄＝どれでも・照合は `miracleIdentityMatches` と同じ識別キー優先の規則）」「使用回数が残っている（回数制限が無ければ常に可）」を見る。
- **UI は既存の前例を踏襲**: 参照はドロップで結線（1対1の結線＝ドロップ・[[feedback_dropdown_vs_drop]]）＋右クリックでリンク解除＋名前は `fromUuid` のライブ解決、条件文は他の本文欄と同じ枠付きエディタ（`tnx-editor-section`＝トグルがヘッダーへ移設される）、選択ダイアログは縦積みボタン（[[feedback_dnd_ui_conventions]]）。**技能単体のロールをグレーアウトする案はユーザー却下**（「見た目が悪くなる・余計なことをしない」）＝特別扱いを入れていない。
- 検証: 純ロジック16件を TDD で追加（全 2609 件グリーン・eslint 0 error）。**隔離 Foundry＋Playwright の実機で全4パターンを確認**——①参照＋条件も＝効果文は参照先／条件は参照先（文中の神業名が名乗る神業に置換）、②参照＋条件据置＝効果文は参照先／条件は元の神業、③独自＋条件据置＝効果文は技能／条件は元の神業（印に `asOther` 無し）、④独自＋条件も＝技能の条件文。あわせて「閉じる＝神業の使用ごと中止（消費なし）」「書き換えずに使用（技能は無消費）」「対応しない神業では尋ねない」「使い切った技能は候補から外れる」「参照先の即死用途が元の神業の名前・印で走り、神業版ダメージカードに『効果 《…》』『書き換え 〈…〉』が並ぶ」を実測。

##### 追記 2026-09-07（同日・ユーザー指摘）: 効果文の出どころも設定にする

- 初版は**効果文を出どころ（`asOther.source`）から無条件に出していた**——参照型なら書き換え先の神業の効果文、独自効果型なら**技能の解説**。ユーザー指摘「書き換えた場合の効果文が、現状恐らく神業書き換え技能の効果そのものに置き換わる形になっていますが、これも条件が書き換わる時と同様に設定欄で設定可能にしてください」。
- **技能の解説はその技能の説明であって、書き換えた後の神業の効果文ではない**。流用すると、神業カードに「この技能は神業と同じタイミングで使用し…」という技能の説明文が出る。書き換え後の文は**技能の設定欄に書く**。
- 条件と**対称**にした: `rewriteDescription`（効果文も書き換えるか）＋ `description`（独自効果型でオンのときの効果文）。純ロジックは `miracleCardConditionPlan` を **`miracleCardTextPlan`** に一般化し、効果文・条件それぞれ keep（元の神業）／source（書き換え先）／text（技能の設定欄）を返す1つの関数にまとめた（同じ形の関数を2つ置かない）。
- **既定はオン**（条件は既定オフ）。効果が別物になる以上、文も書き換え後のものを出すのが自然で、参照型の従来挙動（書き換え先の効果文）もそのまま保たれる。オフにすれば「効果だけ差し替えて、カードの文は元の神業のまま」も表せる。
- 検証: 純ロジック6件を差し替え/追加（全 2615 件グリーン）。隔離実機で4通り——①参照＋文も書換＝書き換え先の効果文、②参照＋文は据置＝元の神業の効果文、③独自＋文も書換＝技能の設定欄の文、④独自＋文は据置＝元の神業の効果文（条件だけ技能の文）——を実測。

##### 追記 2026-09-07（同日・ユーザー指摘2件）: 効果文に選択は要らない／写した文の神業名を置き換える

- **「効果文を書き換えるか」の選択を撤去**（ユーザー指摘 verbatim「元の効果文のままというのはありえなくないですか？」）。**正しい**——書き換え技能は効果そのものを差し替えるのだから、元の神業の効果文を出せば**起きないことの説明**をカードに載せることになる。選択が意味を持つのは取得条件だけ（ルールに「条件は変わらない」型が2つあるため）。前段で「条件と対称に」と考えたのは形の対称であって、**機能の対称ではなかった**（[[feedback_function_over_form_reuse]] と同根の誤り＝形で揃えて、その段が何を担うかを見ていない）。`rewriteDescription` を撤去し、効果文は参照型＝書き換え先の神業／独自効果型＝技能の設定欄、と常に書き換え後のものになる。
- **写した文の神業名を名乗る神業の名前へ置換する規則を効果文へ広げた**（ユーザー指示「《万能道具》の取得条件書き換えと同様に、効果文を書き換える場合は《（神業名）》の箇所を書き換え元の神業の名前で置き換えるべき」）。効果文《RW死の舞踏》「《死の舞踏》は、対象ひとりを完全死亡させる」を《チャイ》のカードに出すなら「《チャイ》は、…」でなければならない——文を名乗るのはその神業だから。関数は `renameMiracleInCondition` → **`renameMiracleInText`** へ改名（条件専用ではなくなったため）。
- **コピー（《突然変異》）にも同じ置換が効くようにした**（従来は書き換えのときだけ置換していた）。同じ「他の神業の効果を自分の名前で使う」状況であり、片方だけ置換しない理由が無い。**表示の変更なので、戻す判断があればユーザーの指示で戻す**。
- 検証: テスト2615件グリーン。隔離実機で、効果文に《RW死の舞踏》を2箇所書いた参照先を書き換えに使い、カードの効果文が両方とも《RWチャイ》に置き換わることを実測（条件も同じ）。独自効果型は設定欄の文がそのまま出る。

### 2026-09-07: 治療・修理・改造の帰結を「別カード」から使用カードの帰結行へ

**日付**: 2026-09-07 / **関連実装**: `scripts/module/chat-card.mjs`・`recovery-flow.mjs`・`repair-flow.mjs`・`modification-flow.mjs`・`tnx-check-flow.mjs`・`miracle-flow.mjs`・`scripts/item/item.mjs`・`scripts/tnx.mjs`・`css/tnx2.css` / **ステータス**: 実装済・隔離 Foundry で検証済（ユーザー実機確認待ち）

#### 発端（ユーザー指摘）

「黄泉還りなどの復活系神業を含む、用途の治療セクションの設定によりダメージを回復した際、効果文があるわけでもなく、場合によっては用途名がタイトルの『○○を回復』だけの短いチャットカードが別で出てしまう。別のチャットカードを出すのではなく、通常通りその神業の効果として回復したことが分かるようにできないか」。**修理・改造も含める**指示、および**ダメージ名を ［］ でくくらない**（戦闘不能と読み違えるため）の指示を追加で受けた。

#### 原因

治療（`useRecovery`）・修理・改造は、適用のあとに必ず `postConditionOutcome`（`condition-outcome.hbs`）で**帰結だけの別カード**を投稿していた。見出しは `usage.name || item.name` のため神業でも**用途名**が出て、効果文も持たない薄いカードが本体カードの隣に並ぶ。判定形では判定結果カードの直後に、宣言形（神業）では神業カードの直後に出ていた。神業以外の宣言形にいたっては、このカードだけが出て解説カードが出ていなかった。

#### 設計判断

- **帰結行は既にある段**（`tnx-card__outcome`＝判定結果カードの「シーンに登場した」「入手した」「情報を開示した」）。新しい表現を作らず、治療・修理・改造もこの段に載せた。着地先は**その使用を表しているカード**——神業＝神業カード／宣言（神業以外）＝アイテムの解説カード（他の宣言用途と同じ `postDescriptionCard`）／判定＝判定結果カード。
- **後から刻む口は共通機構**にした（[[feedback_extract_reusable_mechanisms]]）: `stampCardOutcome(message, {icon, text})` がフラグ `cardOutcome` を置き、`renderCardOutcome`（`renderChatMessageHTML`）が描く。**本文の書き換えでは刻まない**——投稿済み content を組み直すと再判定の置き換え等と競合し、非作者の更新はソケット委譲で非同期に着地するため古い本文で上書きしうる。フラグ駆動なら再判定でカードが組み直されても行が消えない（`infoDisclosed` のようなフラグ引き継ぎの作り込みも不要）。
- **失敗時は帰結行を出さない**（Code 提案・ユーザー黙認）。判定結果カード自身が「失敗」と示すため、「回復しませんでした」の行は重複になる。**「何を対象にしたか」は失敗時に卓へ出なくなる**——気になるようなら失敗の帰結行を足す（要ユーザー判断）。
- **状態名の表記を規約に載せ直した**: 旧カードは `condition-outcome.hbs` が `label` を無条件に「」でくくっており、戦闘不能の ［］ 規約から外れていた。帰結行は `conditionDisplayName(kind, {quote:true})` を1状態ずつ通す＝戦闘不能だけ ［］・負傷/BS は「」。**負傷（ダメージ）を ［］ でくくらない**のはユーザー指示（戦闘不能と読み違えるため）。
- **文字組み**: 「の故障を修理した」が「の故 / 障を修理した」で割れたため、決まった言い回しを `nowrap` で塊にした（[[feedback_fix_wrapping_structurally]]。囲みで閉じた語は `keepTogether` が描画時に一括で拾う）。
- `postConditionOutcome` と `condition-outcome.hbs` は**残す**——効果決定カードの旧形式（フラグを持たない保存済みカード）からの呼び出しが現役で、過去ログのカードも投稿時の HTML のまま読める（実機で確認）。

#### 検証

- テスト 2615 件グリーン・eslint 0 error・stylelint clean。
- 隔離 Foundry＋Playwright（`T:/pw/verify-outcome-row.mjs`）で5経路を実測——A 神業の治癒（宣言）＝神業カード1枚に「〈対象〉の「動脈切断」・［気絶］を回復した」／B 神業以外の治療（宣言）＝解説カード1枚に帰結行／C 治療（判定）＝判定結果カードに帰結行・状態が実際に消える／D 修理（判定）＝帰結行・故障が実際に直る／E 改造（判定）＝「…の受け値＋2を適用した」。あわせて**別カードが出ないこと（カードは常に1枚）**・横はみ出しなし・囲みで閉じた語の行またぎなし・旧形式カードの描画をスクリーンショット目視込みで確認。

### 2026-09-07: 神業の用途から「適用される効果」を削除

**日付**: 2026-09-07 / **関連実装**: `templates/app/usage-sheet.hbs`・`scripts/actor/tnx-character-sheet-base.mjs`・`scripts/module/miracle-flow.mjs` / **ステータス**: 実装済（ユーザー実機確認待ち）

#### 発端（ユーザー指示）

「神業用途から『適用される効果』セクションを削除してください。何故か残ったままになっています」。

#### メカニクス（ルール解釈）

神業の効果は**効果文**と、神業専用タイプが持つ設定（宣言の効果・即死・防御・社会戦・破壊）で表す。ActiveEffect を用途に紐づけて付与する一般用途の仕組みは神業には当てはまらない——神業由来の状態は「神業でしか防げない・治せない」印つきの専用経路（17-2/17-6）で扱うため、AE の一般付与と二重の入口になっていた。神業専用タイプ5種の新設（17-2）でタイミング・目標値・対決を落としたときに、このセクションだけが落とし漏れて残っていた。

#### 実装

- 用途シート: 「適用される効果」の fieldset を `{{#unless isMiracleUsage}}` で囲む。**「ウェットの対象には効果がない」トグルも同じ fieldset の中**なので一緒に消える——神業のダメージカード（`buildMiracleDamageFlag`）は `noEffectVsWet` を読まず、このトグルの唯一の効き先が付与効果のペイロードだったため、神業では機能を持たない設定だった。データ（`usage.noEffectVsWet`・`usage.effects`）は**消さない**（欄が出ないフォーム送信は既存値を保つ＝[[feedback_no_silent_data_rewrite]]）。
- `_useDeclarationUsage`: 神業は `prepareUsageEffectPayload` を呼ばない（UI から設定できないものが古いデータ由来で動く「幽霊設定」を作らない）。
- `postMiracleCard`: 到達しなくなった `usageEffects` 引数・`tnx-usage-use-card` の器・`usageEffects` フラグを削除し、`handleNegateMiracleCardClick` の `usageEffects.negatedBy` 分岐も落とした。**`handleNegateTrayClick` は残す**——神業以外の宣言用途の解説カードに出る効果トレイの打ち消しは現役。過去ログのカードは投稿時の HTML とフラグのまま読める（クラス名は変えていない）。

#### 検証

- テスト 2616 件グリーン・eslint 0 error。Handlebars のブロック対応も検算（balanced）。
- 開発ワールド（`worlds/develop`）の保存データを走査し、`usage.effects` が非空の用途は**1件も無い**ことを確認（神業に限らず全用途）＝この削除で失われる既存設定は無い。

##### 追記 2026-09-07（同日・ユーザー指摘3件目）: 「書き換える神業」は識別キーで指定する

- ユーザー提案「『書き換える神業』に関してはドロップボックスではなく識別キーで指定にしませんか？」→ **採用**。ドロップ（1対1の結線）が向くのは**その文書の実体を使う**参照——書き換え先の神業は用途と効果文をそこから読むので uuid のドロップのまま。一方「書き換える神業」は**「どの神業か」の照合**であり、卓のキャラクターが持つのは辞典の写し（uuid は本体と別・名前は変わりうる）だから、識別キーで持つのが正しい。[[feedback_dropdown_vs_drop]] の基準（選ぶ操作＝プルダウン／1対1の結線＝ドロップ）にも、こちらが合う。
- 実装: `target{uuid,key,name}` → **`targetKey`**（StringField）。シートは辞典の神業を並べたプルダウン（先頭「すべての神業」＝指定なし・値は識別キー・表示は名前・`getIndex` で読む＝KI-026）。照合は識別キーの一致だけを見る（名前フォールバックは廃止——キーで指定する以上、キーの無い神業とは対応しない）。用途シートの「打ち消せる神業」のプルダウンと同じ作り。
- **未リリースの形の差し替えなので移行は書かない**。17-7 自体がまだリリース前で、旧 `target` を設定していた場合は次回保存で捨てられ「すべての神業」に戻る（設定し直しが要る）。
- 検証: テスト2616件グリーン。隔離実機で、プルダウンに32件（すべての神業＋辞典の31件）が名前順で並び、選ぶと識別キーが保存され、対応しない神業では候補から外れ、「すべての神業」に戻すと候補に戻ることを実測。

- **是正（同日）**: 上記の実装を辞典のプルダウンにしたのは読み違いで、ユーザーの求めは「**識別キーを直接入力する形**」（verbatim「いえ、ですから、識別キーを直接入力する形にしませんか？とお話したのですが」）。入力欄（アイテム自身の識別キー欄と同じ形）に直した——辞典に無い神業も指定でき、空欄がそのまま「どの神業でも」になる。辞典パックを読む処理（`_miracleKeyOptions`）は撤去。

- **再是正（同日・ユーザーが思い直して確定）**: 「プルダウンの方が良いかもしれません。プルダウンに戻せますか？」→ **辞典の神業を並べたプルダウンに戻した**（識別キーを保存する点は変わらない）。31件は辞典に揃っていて打ち間違いの余地が無い方が良い、という判断。あわせて**辞典に無い識別キーが入っていた場合はその行を足して選択を保つ**（黙って「すべての神業」に落ちて次の保存で消えるのを防ぐ。名前に解決できないのでキーをそのまま出す）。

- **「すべての神業」を撤去（同日・ユーザー指示「『全ての神業』は必要ありません。撤去してください」）**: 対象の指定を**必須**にした——書き換え技能はどの神業を書き換えるかが決まっており、「どの神業でも」は Code が「組み合わせの可否は卓が決める」の一般規範から足したもので、ルールに求めが無かった。プルダウンの先頭は中立の「—」（未設定の枠）だけを残す——選択肢から空を消すと、未設定の技能を開いたときにブラウザが先頭の神業を選んだ形になり、次の保存でその値が黙って書き込まれる。純ロジックは `targetKey` が空なら候補にしない。

### カード設定のタブ保持・GM 所有キャストの経験点同期（フェーズ18、KI-057・KI-058）

**日付**: 2026-09-10 / **ステータス**: コード修正・自動検証完了、実機確認待ち。

- 根拠は当日のユーザー不具合報告。GM が所有者として設定されたキャストもレコード同期対象にする。
- 経験点の付与・消費・由来分離の計算規則は変更しない。非 GM の既存優先順位を保ち、個別 OWNER の GM を候補に加える。暗黙の GM 権限だけでは紐付けない。
- カード設定は表示タブの保持のみ。カード生成・配布ルールに変更なし。
- 全119ファイル・2793テスト通過。Foundry の権限設定画面を含む実操作は未確認。

### チャット通知の既定値（フェーズ18、KI-059）

**日付**: 2026-09-10 / **ステータス**: 実装・本体スキーマ検証完了、ブラウザ操作未確認。

ユーザーの要求は新規アクセス時の既定値を通知バッジにすること。ゲームルールへの影響はない。各ブラウザの未設定値と設定リセットに適用し、個人が保存した選択は保持する。GM の保存値の配布やログインごとの強制上書きは行わない。技術的根拠は KI-059 の公式 API とローカル本体コード参照。

### 本体の役割名をルーラーへ変更（フェーズ18）

**日付**: 2026-09-10 / **ステータス**: 実装・自動検証完了、ブラウザ表示未確認。

ユーザーの指定に従い、本体の役割表示を日本語では「ルーラー」「アシスタントRL」、英語では「Ruler」「Assistant RL」、略称は共通で「RL」とする。役割番号・isGM・所有権・実ユーザー名は変更しない。TNX 自身の通知文に残っていた GM 表記も RL に統一した。ゲームルールへの変更はない。

### 初期GMアカウント名をRulerへ変更（フェーズ18）

**日付**: 2026-09-10 / **ステータス**: 実装・自動検証完了、実機未確認。

ユーザーの「ワールド作成者のデフォルトユーザーネームもRulerに」という依頼に対応。ゲームルール・権限・ユーザーIDは変えず、既定名 Gamemaster の完全GM（role=GAMEMASTER）の名前だけを一度 Ruler に変更する。別名へ変更済み、同名Rulerが存在する場合は変更しない。既存ワールドも更新後の初回起動で対象となる。

### 固有名詞技能の名前保存（フェーズ18、KI-060）

**日付**: 2026-09-11 / **ステータス**: 修正・自動検証完了、実機未確認。

既存裁定の「保存形はフル名、編集表示だけ区分と固有名詞に分割」に適合させる修正。区分を選んだ時点で接頭辞を合成して保存し、閲覧・辞典・参照先が同じ名前を読めるようにする。技能分類や識別キーの意味、判定ルールは変更しない。原因と検証は KI-060 を参照。

### 2026-09-11 アクターの立ち絵・コマ画像

共通サイドバーの画像保存先を修正し、立ち絵とコマ画像の設定操作を分離。既存のコア画像フィールドと標準設定画面を利用し、データモデルの追加なし。詳細は Issues の KI-061。既存2806テスト・ESLint・Stylelint通過、ブラウザ実機未確認。

#### 2026-09-11 KI-061 表示操作の訂正

ユーザー提示の D&D 画面に合わせ、画像を並べる方式から単一画像枠の切り替え方式へ変更。スイッチは編集モードのみ表示し、閲覧モードでは選択中の画像のみ表示。切り替えは DOM の表示変更のみで、未保存フォーム入力を保持する。選択状態はシートインスタンス内に保持。実際の切り替えメソッドを Node で実行して往復・編集モード外・編集権限なしを検証。ESLint・Stylelint通過、ブラウザ実機未確認。

#### 2026-09-11 KI-061 コマ画像の選択操作を訂正

コマ画像クリック時にトークン設定全体を開いていた実装を訂正し、FilePicker の画像選択を直接開く。通常アクターは prototypeToken.texture.src、合成アクターは token.texture.src のみ更新する。サイズ等の設定・立ち絵は変更しない。編集モードと権限を選択開始時・保存時に確認する。4件の回帰テストを追加し、全121ファイル2810テスト・ESLint通過。ブラウザ実機未確認。
