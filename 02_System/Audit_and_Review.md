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
