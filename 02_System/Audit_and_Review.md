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
**ステータス**: ✅ 実装完了（本体 `68b0d3f`・全テスト green 1147・lint 0 error）・実機確認待ち

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
**ステータス**: ✅ 実装完了（本体 `a76c7fe`・全テスト green 1152・lint 0 error）・実機確認待ち

#### ルール解釈（2026-07-10 言語化）

- **攻撃対象が特定のスタイルを持つ場合にダメージが増加する能力が複数存在する**（例: 〈アヤカシ〉持ちに +N）。これを**データ（AE 一行）で再現したい**というのがユーザーの主目的。
- 対象依存の値は**値バフ（prepareDerivedData 焼き込み）では表せない**（対象未定）。判定バフ `check.*` が既に「実行時にアクション文脈を見る」パターンを持つので、その**ダメージ・対象参照版**として実装するのが正しい設計（ユーザー確定＝AE で表現）。

#### 監査結果

✅ `damage.vsStyle.<スタイル識別キー>` / `damage.vsWorks.<組織識別キー>` を追加。恒常適用パスから除外し、ダメージフローが攻撃対象を解決→対象の style（`type:"style"` アイテム）/works（ワークス技能の組織）で照合→合致分を用途のダメージ修正と同じ台帳行としてダメージへ合算。値はリテラル・重複規約は判定バフと同一（`_gatherBonusSources` を共用）。

#### 課題・申し送り

- AE `damage.vs*` はフラット +N（キャラ恒常）。**式ベース・用途固有の対象依存は用途のダメージ修正欄で `@target.*` を参照する形で実装済み**（下記「用途ダメージ修正の @target 参照」・`f66db00`）。「組み合わせた物理攻撃が〈スタイル名〉を対象とする場合～」等の用途固有効果はそちらで表現する。
- 対象の「ワークス所属」は当面ワークス技能の組織で近似（キャラクター自身のワークス所属フラグは未整備＝[Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md) §6）。


### 用途ダメージ修正の @target 参照（式で攻撃対象を参照・フェーズ12 補）

**日付**: 2026-07-10
**関連実装**: `scripts/module/tnx-formula.mjs`（`buildFormulaData` の target 引数・`buildTargetData`・`evaluateBonusRows`）、`scripts/data/item/helpers.mjs`（`targetStyleWorksKeys`）、`scripts/module/damage-flow.mjs`（対象解決と配線）。正本 [Formula_Reference.md](../01_Wiki/Game_Mechanics/Formula_Reference.md)「@target.*」
**ステータス**: ✅ 実装完了（本体 `f66db00`・全テスト green 1155・lint 0 error）・実機確認待ち

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
**ステータス**: ✅ 実装完了（本体 `705a6b4`・全テスト green 1160・lint 0 error）・実機確認待ち

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
**ステータス**: ✅ 修正完了（全テスト green 1161・lint 0 error）・実機確認待ち

#### ルール解釈（2026-07-10 ユーザー指摘）

- **トーキョーN◎VA の対決判定は受動有利**：能動（攻撃）側は受動（リアクション）側を**上回れば**勝利、**同値は受動側の勝利**。従来 `resolveOpposed` は相手の達成値を「目標値」として扱い `達成値 ≥ 相手` で命中していた（同値で攻撃側勝利）＝**バグ**（ユーザー診断「『目標値』というワードが悪さをしている」どおり）。

#### 監査結果

✅ `hit = margin > 0`（`>=` → `>`）に修正。同値は攻撃側敗北＝攻撃回避。①固定値・②制御値は**目標値モードのまま** `達成値 ≥ 目標値`（`resolveNoReaction` は据え置き）。

#### 課題・申し送り

- **要確認**: 「リアクションしない」＝②対象の制御値受け（`resolveNoReaction`・`≥`）も受動有利（`達成値 > 制御値`）にすべきか。現状は目標値モードとして `≥` のまま（ユーザー指摘は対決＝リアクション同値に限定）。


### ヴィークル搭乗中のドッジ＝操縦上書き（回避ブロック・フェーズ12 補）

**日付**: 2026-07-10
**関連実装**: `scripts/module/attack-flow.mjs`（`startReaction` の dodge 分岐・`promptVehicleDodgeFallback`）。正本 [Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md)「ヴィークルの操縦」・[Outfits.md](../01_Wiki/Game_Rules/Outfits.md)・[Damage_Rules.md](../01_Wiki/Game_Rules/Damage_Rules.md)「ドッジ」
**ステータス**: ✅ 実装完了（本体 `340ac6e`・全テスト green 1161・lint 0 error）・実機確認待ち

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
**ステータス**: ✅ 実装完了（本体 `e6b92e2`・全テスト green 1165・lint 0 error）・実機確認待ち

#### ルール解釈（2026-07-10 ユーザー確定）

- **用途を解決した際、ターゲットしたキャラクターへ AE を付与**（攻撃に限らず全用途・自己バフも表現可）。
- ターゲットは**居るか居ないかだけで判断**（数・種別は問わない）。原則ターゲット必須で、居なければ**確認ダイアログ**（続行＝自分対象・キャンセル＝中止）。誤って自分をターゲット中の攻撃等は**ブロックしない**（過度な自動化は進行を妨げる＝ユーザー明言）。
- 付与は**対象所有者/RL がボタン**で実行（自動付与しない＝命中/成功や誤爆の判断は卓に委ねる）。

#### 監査結果

✅ 判定/攻撃/用途使用の結果カードに「効果を適用」ボタン。対象所有者/GM が押すと供給元 AE を対象へ複製生成。ノーターゲットは確認ダイアログで自分対象/中止。

#### 課題・申し送り

- 命中/成功の条件付けは自動化せず卓判断（ボタンをいつ押すか）。非攻撃の判定用途も `game.user.targets` で対象を取る＝射程/対象記述の検証はしない（卓運用）。
- item-sheet の旧「供給元アイテム上で効果を有効化」（フェーズ13 待ちの暫定）は本モデル（対象へ付与）に置換。


### 用途の「無視する指定技能」（指定技能を無視して単体参加・フェーズ12 補）

**日付**: 2026-07-10（同日中に2度の再設計＝下記経緯）
**関連実装**: `scripts/data/item/common/usage.mjs`（用途フィールド `ignoreComboSkills`＝辞典識別キーの配列）、`scripts/module/skill-chain-resolution.mjs`（`resolveUsageSkills` 第4引数 `ignoreComboKeys`）、`scripts/module/tnx-usage-sheet.mjs`（設定 UI・追加判定・auto-fill 反映）、`templates/app/usage-sheet-combo.hbs`。正本 [Usage_System.md](../01_Wiki/Game_Mechanics/Usage_System.md)「組み合わせ解決」
**ステータス**: ✅ 実装完了（本体 `c79e55f`→再設計 `c4611e3`・全テスト green 1169・lint 0 error）・実機確認待ち

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
**ステータス**: ✅ 修正完了（本体 `47fe6ba`＋`592214f`・全テスト green 1171・lint 0 error）・実機確認待ち

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
**ステータス**: ✅ 実装完了（本体 `d409992`＋`fbfbb75`・全テスト green 1171・lint 0 error）・実機確認待ち

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
**ステータス**: ✅ 実装完了（本体 `4c9ade1`・全テスト green 1173・lint 0 error）・実機確認待ち

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
**ステータス**: ✅ 実装完了（本体 `dd15abe`・全テスト green 1174・lint 0 error）・実機確認待ち

#### ルール解釈（2026-07-11 ユーザー確定）

- **事後的に判定へボーナス/ペナルティを与える技能**が存在し、導線は再判定の付与と同じ「技能をクリック→達成値をクリック」（ユーザー案＝達成値クリックの流用）。
- **再判定との共存**: 再判定→事後修正は可能。**事後修正→再判定は修正がリセット**（再判定は元の構成から再実行するため）——スナップショットに事後修正を含めないことで自然に成立。
- 修正値＝用途の「この用途の判定修正値」（式）を使用者のデータで評価・空なら手入力（Code 設計・ユーザー黙認）。ダメージカード算出後の攻撃は修正不可（唯一の強制・設計時に提示済み）。

#### 監査結果

✅ modifyCheck 用途の使用→クリック待ちモード（recheck と排他）→達成値クリックで式評価（空は手入力）→フラグ達成値更新＋目標値つきは成否/差分再計算＋解決済み攻撃は対決再解決（受動有利の `resolveOpposed` 再利用）＋要求カード追随。内訳は `checkMods` フラグからライブ描画（元の内訳は残す）。非作者の修正は GM へソケット委譲。

#### 課題・申し送り

- 攻撃（解決済み）の再解決はリアクション達成値/制御値の**保存値**に対して行う（リアクション側の巻き戻しはしない）。
- 修正のリセット（事後修正→再判定）はスナップショット非包含による自然な帰結＝checkMods はカード表示専用で再判定に影響しない。
