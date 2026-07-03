# Actor 型定義

全 4 type（cast / guest / troop / extra）が `scripts/data/actor/` に DataModel 化済。
**player type は v0.3.1（フェーズ2）で廃止済み**。

## 共通 Template

| Template | ファイル | 主なフィールド |
|---|---|---|
| biography | `common/biography.mjs` | charaname_ruby, handle, post, citizenRank, age, gender, 身体情報, **description** (HTMLField) |
| attributes | `common/attributes.mjs` | reason/passion/life/mundane (value, control, styleA〜C, growth, mod 等), combatSpeed（3層: value/base/current/freeMod — 下記）, actionRank（付与型: value/freeMod のみ保存。実効付与値 maxTotal=定数1+freeMod+AE は派生・フェーズ11 追補。正本 [Combat_Flow.md](../Game_Rules/Combat_Flow.md)「アクションランク」）, 各ダメージ |
| actorBase | `common/actor-base.mjs` | （フィールドなし）手札・切り札の所有（handPileId / trumpCardPileId）・手札上限（handMaxSize）は **User flag の権威へ一本化済み**（フェーズ2 積み残し解消） |

## 各 Type

### cast — プレイヤーキャラクター

**mixin**: biography + attributes + actorBase

**シート**: `TokyoNovaCastSheet`（ApplicationV2、フェーズ3 で移行完了）

| 固有フィールド | 型 | 備考 |
|---|---|---|
| ownerUserId / syncWithOwner | String / Boolean | User UUID — EXP 同期用（フェーズ2-1）と同期の有効化 |
| history | ObjectField | オブジェクトマップ `{ [id]: { date, title, exp, rl, players } }` |
| exp | SchemaField | value, spent, total, additional |
| lifePath | SchemaField | origin, experience, encounter — 各 `{itemUuid, name}` SchemaField。フェーズ6-5 実装済み |
| partSlots | ArrayField | 部位スロット集合（フェーズ10-1）。`{value, count, occupiesOther, targetPart, targetCount}` — 占有計算の母数。新規キャストにはゲーム設定のプリセットを流し込む |
| bounty / bountyBase | NumberField | 報酬点の現在値／基準値（外界値から手動確定） |
| baseAttack / baseDefence / baseGuard | SchemaField | **生身のデータ**（フェーズ10-6 で正式化・2026-07-02 裁定）。生身はアイテムでなくアクターデータ（攻撃力 I+0・受け値 0）。全身義体・生身変更装備（weapon.isFleshChange）はこれを「書き換える」（正本 [Outfits.md](../Game_Rules/Outfits.md)「生身の変更」） |
| weaponRefs | SchemaField | `{attackItemId, parryItemId}` — 攻撃／パリー別の生身書き換え元・使用武器選択（フェーズ10-6）。空欄=未変更の生身。便宜的事前設定（非強制） |
| appearanceModifier | NumberField | 携帯中アウトフィットの危険値合計（自動集計） |
| isGhost | BooleanField | ゴースト登場中フラグ（CS 修正の読み飛ばしに影響） |
| handMaxSizeMod | NumberField | 手札上限への AE 着地点（KI-020・権威は User flag のまま） |
| outfitMod | SchemaField | 装備集計値 `{control, combatSpeed, reason, passion, life, mundane}` |

> `player_name` フィールドはフェーズ6-0 で削除済み（デッドフィールド確定）。

### guest — NPC ゲスト

**継承**: `CharacterBaseDataModel`（フェーズ11-3 新設・`common/character-base.mjs`。biography + attributes + actorBase の mixin に、cast と共有するフィールド＝partSlots・isGhost・報酬点・生身・weaponRefs・appearanceModifier・handMaxSizeMod・outfitMod と派生値パイプラインを持つ）。**guest は固有フィールドなし**（cast との差分＝セッション履歴クラスタ4フィールド＋lifePath の計5フィールドのみ。テストで不変条件化）。cast は同基底を継承しこの5フィールドを追加する構成。

**シート**: `TokyoNovaGuestSheet`（フェーズ11-3。TnxCharacterSheetBase の薄い派生・SHEET_FEATURES exp/history=false・guest-sheet.hbs は履歴タブなしのシェル）

**新規作成時**: 部位プリセット・基本13技能・リンクトークンをキャスト同様に流し込む（所有権自動設定・disposition 変更はキャストのみ）。handMaxSizeMod は GM ユーザーの実効手札上限へ全ゲスト合算（[Card_Operations.md](../Game_Mechanics/Card_Operations.md)「手札上限」）。

> ゲスト = キャストから**セッション履歴（EXP・User 同期）とライフパス**を除いたもの（2026-07-03 確定・同日再訂正）。**報酬点は通常どおり持つ**（外界値の報酬点＋報酬点を取得するスタイル技能の効果等。「報酬点を持たない」だけが勘違い＝ルールに記載なし）。**ライフパスは持たない**（フェーズ6 記録どおり。「セッション履歴以外差異なし」の言はライフパスの失念＝再訂正）。

### troop — 部隊

**継承**: `CharacterBaseDataModel`（フェーズ11-4 で再構成。biography も持つ＝2026-07-03 ユーザー意向。判定・派生値パイプラインはキャストと同じ＝[Troops.md](../Game_Rules/Troops.md)）。基底由来の報酬点・partSlots・生身・isGhost・handMaxSizeMod はトループのルール上未使用（スキーマに残るが死蔵・シート非表示）。

| 固有フィールド | 型 | 備考 |
|---|---|---|
| troopMode | StringField | 種別ドロップダウン `troop/enigma/bunshin`（正本 [Troops.md](../Game_Rules/Troops.md)「種別と名前の規則」。旧 isEnigmaMode は誤設計として migrateData で移行・廃止） |
| hasWorks | BooleanField | 「ワークスを設定」。ON＋組織アイテムありで名前が「(組織名)（(スタイル名)(トループレベル)レベル）」に。所属ブロックは ON のときだけ表示 |
| sourceName | StringField | 分身元キャラクター名（bunshin の固定名「(分身元)の分身」に使用。11-6 で自動設定へ拡張予定） |
| troopLevel | NumberField | トループレベル。**能力値＝スタイルの基本値＋トループレベル**の決定項（成長なし・`_prepareAbilityTotals` オーバーライド） |
| heads | SchemaField | `{value, max}` — 人数（enigma ではエニグマポイント・フィールド共用）。HP のように機能しダメージ分減少（チャート不参照）。トークンリソースバー割当（trackableAttributes・新規作成時 bar1 既定）。bunshin はリソース管理なし（シート非表示） |

> 旧 `memo` はフェーズ11-4 で廃止（`migrateData` で biography.description へ移行）。分身も troop として作成し、AR=1・CS/CSカレント=0 はデータ入力で表現。名前はトループ=「(スタイル名)・トループ」／分身=「(分身元)の分身」で固定（tnx.mjs `syncTroopName`・自由入力はエニグマのみ）。

**シート**: `TokyoNovaTroopSheet`（フェーズ11-4。TnxCharacterSheetBase の薄い派生・神業/報酬点/部位管理/成長欄/ハンドル/市民ランク/パーソナルデータなし・トループレベル＋種別/heads パネル付き）

### extra — エキストラ

**mixin**: biography のみ。固有フィールドなし。

> エキストラは**名前・パーソナルデータ・説明のみ**のキャラクター（2026-07-04 確定）。~~スタイル・神業・アウトフィットを持てる~~はフェーズ6 記録以来の勘違い＝**神業以外を持たないゲストとの取り違え**（ユーザー確認）。神業を使うエキストラ的キャラクターが必要なら口頭宣言か、データを制限したゲストとして作る。
> **例外＝固定値判定**: エキストラ（アウトフィットとして取得できるものを含む）は**固定値の判定が行える**ため、**一般技能のみ持てる**——スート・レベルの概念は持たず（読み取りもしない）、達成値は check 用途の「固定達成値」で表す（設定方法は用途＝2026-07-04 ユーザー確定 → [Check_Rules.md](../Game_Rules/Check_Rules.md)「固定値判定」）。

**シート**: `TokyoNovaExtraSheet`（フェーズ11-5。ポートレート＋パーソナルデータ＋名前＋技能欄＋説明のタブなし1ページ。技能欄は「技能名＋達成値」のみ表示・ドロップは一般技能のみ許可。新規作成はリンクトークンのみ設定）

**シート**: コアデフォルト（**フェーズ11** でカスタムシート追加予定）

## attributes.combatSpeed（CS 3層モデル・フェーズ10-5 で KI-010 解消）

| フィールド | 用途 |
|---|---|
| base | CSベース＝能力値項スナップショット（「プレアクト初期化」で floor((理性+感情+生命)÷2) を焼き込み・以後追従しない） |
| value | CS（中段・プレアクト初期化で CSベース実効値から設定） |
| current | CSカレント（カット開始時に CS が自動セット。カット中の自動管理はフェーズ13） |
| freeMod | CSベースへの手動修正（ライブ加算） |

- 各層とも「決定値（再計算不可・保存）＋修正値（ライブ）」で表現。旧 `mod` フィールドは削除済み。
- 実効値 `baseTotal / valueTotal / currentTotal / displayTotal` は派生算出（保存しない）。AE は仮想名前空間 `system.cs.base|value|current` に着地する（[Active_Effects.md](../Game_Mechanics/Active_Effects.md) 大原則6）。
- シート表示は単一「CS」で**自動制御**: カット進行中＝カレント／それ以外＝CS。initiative は `@system.combatSpeed.displayTotal`。guest / troop は `attributes.mjs` のフォールバック派生で同式を解決（extra は combatSpeed 自体を持たない）。
- UI はキャストシート戦闘タブの CS ボックス（正本 [Combat_Flow.md](../Game_Rules/Combat_Flow.md)「コンバットスピード」）。

## アウトフィット集計（cast）

`CastDataModel.prepareDerivedData` → `_prepareOutfitAggregates` が `computeOutfitAggregates(items)`（`scripts/data/helpers.mjs`）で**都度算出**する（永続化しない＝B-2、フェーズ9-2。isGhost 引数はフェーズ10-5 でフラグ駆動化に伴い廃止——ゴースト読み飛ばしは呼び出し側）。旧来のフック書き戻し（`updateCastOutfitMods` / `updateCastAppearanceModifier` / `recalcOutfitAggregates`）・起動時スキャン・isGhost 再集計は撤去済み（`scripts/tnx.mjs` L264-267 に明記）。

| 集計対象 | 条件 |
|---|---|
| `appearanceModifier` | 携帯中（isCarrying）アウトフィットの `appearancePenalty.value` 合算 |
| `outfitMod.control` | 準備済み（isPrepared）かつ携帯中の `controlMod.value` 合算 |
| CS 修正 | `combatSpeedModGhostIgnore` フラグで一元化（フェーズ10-5・2026-07-02 裁定）: **OFF**＝準備起点で `combatSpeed` に加算（ゴースト無関係）／**ON**＝携帯起点で別枠 `combatSpeedGhostIgnorable` に加算し、ゴースト登場中は CastDataModel 側が読み飛ばす（決定値に触れない「修正項の条件付き除外」）。シート表示は（）括り |

> `outfitMod` の reason/passion/life/mundane は常に 0（現行ルール上アウトフィットは制御値・CS のみ修正）。値バフ（AE）は `_applyEffectBuffs` が `total` へ直接適用する（フェーズ9-3 v2）。

## 参照ファイル

- DataModel: `scripts/data/actor/*.mjs`、集計の純関数: `scripts/data/helpers.mjs`（`computeOutfitAggregates`）
- cast シート: `scripts/actor/tnx-cast-sheet.mjs`
