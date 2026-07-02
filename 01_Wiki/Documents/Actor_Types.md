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

**mixin**: biography + attributes + actorBase。固有フィールドなし。

**シート**: コアデフォルト（**フェーズ11** でカスタムシート追加予定）

> ゲスト = キャストからライフパス・報酬点・セッション履歴（EXP 含む）を除いたもの（設計方針確定）。

### troop — 部隊

**mixin**: attributes + actorBase

| 固有フィールド | 型 | 備考 |
|---|---|---|
| memo | StringField | 自由記述 |
| heads | SchemaField | `{value, max}` — HP のように機能。トークンリソースバー連携は**フェーズ11** |

**シート**: コアデフォルト（**フェーズ11** でカスタムシート追加予定）

### extra — エキストラ

**mixin**: biography のみ。固有フィールドなし。

> エキストラは基本は名前のみ。スタイル・神業・技能・アウトフィットを持てる（シート実装はフェーズ11）。

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
