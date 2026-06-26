# Actor 型定義

全 4 type（cast / guest / troop / extra）が `scripts/data/actor/` に DataModel 化済。
**player type は v0.3.1（フェーズ2）で廃止済み**。

## 共通 Template

| Template | ファイル | 主なフィールド |
|---|---|---|
| biography | `common/biography.mjs` | charaname_ruby, handle, post, citizenRank, age, gender, 身体情報, **description** (HTMLField) |
| attributes | `common/attributes.mjs` | reason/passion/life/mundane (value, control, styleA〜C, growth, mod 等), combatSpeed, 各ダメージ |
| actorBase | `common/actor-base.mjs` | handMaxSize, handPileId, trumpCardPileId |

## 各 Type

### cast — プレイヤーキャラクター

**mixin**: biography + attributes + actorBase

**シート**: `TokyoNovaCastSheet`（ApplicationV2、フェーズ3 で移行完了）

| 固有フィールド | 型 | 備考 |
|---|---|---|
| ownerUserId | String | User UUID — EXP 同期用（フェーズ2-1） |
| history | ObjectField | オブジェクトマップ `{ [id]: { date, title, exp, rl, players } }` |
| exp | SchemaField | value, spent, total, additional |
| lifePath | SchemaField | origin, experience, encounter — 各 `{itemUuid, name}` SchemaField。フェーズ6-5 実装済み |
| bountyBase | NumberField | 報酬点の基準値（外界値から手動確定） |
| baseGuard | NumberField | 基本ガード値 |
| appearanceModifier | NumberField | 携帯中アウトフィットの危険値合計（自動集計） |
| isGhost | BooleanField | ゴースト登場中フラグ（CS 修正に影響） |
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

## attributes.combatSpeed

| フィールド | 用途 |
|---|---|
| value, base, current, mod, freeMod | イニシアチブ計算（`@system.combatSpeed.value`） |

**UI 未実装**（KI-010）— シートに入力欄がなく常に 0。配置先はフェーズ10（戦闘タブ）で扱う（10-5 で CS 入力 UI を実装し KI-010 解消予定）。

## アウトフィット集計（cast）

`CastDataModel.prepareDerivedData` → `_prepareOutfitAggregates` が `computeOutfitAggregates(items, isGhost)`（`scripts/data/helpers.mjs`）で**都度算出**する（永続化しない＝B-2、フェーズ9-2）。旧来のフック書き戻し（`updateCastOutfitMods` / `updateCastAppearanceModifier` / `recalcOutfitAggregates`）・起動時スキャン・isGhost 再集計は撤去済み（`scripts/tnx.mjs` L264-267 に明記）。

| 集計対象 | 条件 |
|---|---|
| `appearanceModifier` | 携帯中（isCarrying）アウトフィットの `appearancePenalty.value` 合算 |
| `outfitMod.control` | 準備済み（isPrepared）かつ携帯中の `controlMod.value` 合算 |
| `outfitMod.combatSpeed` | 携帯中の `combatSpeedMod.value` 合算（isGhost 時は無効化） |

> `outfitMod` の reason/passion/life/mundane は常に 0（現行ルール上アウトフィットは制御値・CS のみ修正）。値バフ（AE）は `_applyEffectBuffs` が `total` へ直接適用する（フェーズ9-3 v2）。

## 参照ファイル

- DataModel: `scripts/data/actor/*.mjs`、集計の純関数: `scripts/data/helpers.mjs`（`computeOutfitAggregates`）
- cast シート: `scripts/actor/tnx-cast-sheet.mjs`
