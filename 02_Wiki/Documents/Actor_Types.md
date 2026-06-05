# Actor 型定義

全 5 type が `scripts/data/actor/` に DataModel 化済。

## 共通 Template

| Template | ファイル | 主なフィールド |
|---|---|---|
| biography | `common/biography.mjs` | charaname_ruby, handle, post, citizenRank, age, gender, 身体情報, **description** (HTMLField) |
| attributes | `common/attributes.mjs` | reason/passion/life/mundane (value, control, styleA〜C, growth, mod 等), combatSpeed, 各ダメージ |
| actorBase | `common/actor-base.mjs` | handMaxSize, handPileId, trumpCardPileId |

## 各 Type

### cast — プレイヤーキャラクター

**mixin**: biography + attributes + actorBase

| 固有フィールド | 型 | 備考 |
|---|---|---|
| player_name | String | 未使用デッドフィールド (将来判断) |
| playerId | String | player Actor UUID (2-5 廃止予定) |
| ownerUserId | String | User UUID — EXP 同期用 (フェーズ2-1) |
| history | ObjectField | オブジェクトマップ `{ [id]: { date, title, exp, rl, players } }` |
| exp | SchemaField | value, spent, total, additional |
| lifePath | SchemaField | origin, experience, encounter — **UI 未実装** |

**シート**: `TokyoNovaCastSheet` (ApplicationV1)

### player — プレイヤー単位データ (廃止予定)

**mixin**: actorBase

| 固有フィールド | 備考 |
|---|---|
| history | 共有履歴の権威 (→ User flag へ移行中) |
| exp | total (独立), spent/value (派生) |

**シート**: `TokyoNovaPlayerSheet`

**フェーズ2-5 で廃止予定**。EXP・履歴・手札は User flag へ移行。

### guest — NPC ゲスト

**mixin**: biography + attributes + actorBase。固有フィールドなし。

**シート**: コアデフォルト (カスタム未登録)

### troop — 部隊

**mixin**: attributes + actorBase

| 固有 | memo (String) |

**シート**: コアデフォルト

### extra — エキストラ

**mixin**: biography のみ。固有フィールドなし。

**シート**: コアデフォルト

## attributes.combatSpeed

| フィールド | 用途 |
|---|---|
| value, base, current, mod, freeMod | イニシアチブ計算 (`@system.combatSpeed.value`) |

**UI 未実装** (KI-010) — シートに入力欄がなく常に 0。

## 所有権モデル (移行中)

| データ | 現状 | 移行先 (フェーズ2) |
|---|---|---|
| EXP・履歴 | player Actor / cast ローカル | User flag |
| 手札・切り札 | player Actor (handPileId 等) | User flag |
| cast↔主体 | playerId (player UUID) | ownerUserId (User UUID) |

## 参照ファイル

- DataModel: `scripts/data/actor/*.mjs`
- cast シート: `scripts/actor/tnx-cast-sheet.mjs`
- EXP 調査: [B4_Investigation_Detail.md](../Phases/B4_Investigation_Detail.md)
