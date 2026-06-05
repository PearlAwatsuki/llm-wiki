# DataModel 設計

フェーズB (v0.2.0) で `template.json` から Foundry v13 推奨の **TypeDataModel** へ完全移行済。

## 概要

| カテゴリ | type 数 | 権威 |
|---|---|---|
| Actor | 5 | `system.json` → `documentTypes.Actor` + `CONFIG.Actor.dataModels` |
| Item | 17 | 同上 |
| Card | 3 | 同上 |
| **合計** | **25** | `template.json` は廃止 |

## 命名・配置 (B-0 決定)

### クラス命名

プレフィックスなし: `CastDataModel`, `StyleSkillDataModel` 等 (dnd5e 流派)

### ファイル配置

```
scripts/data/
  abstract.mjs              # SystemDataModel + mixin()
  helpers.mjs               # attributeField(), damageField() 等 (Actor 側)
  actor/
    common/
      biography.mjs
      attributes.mjs
      actor-base.mjs
    cast.mjs, player.mjs, guest.mjs, troop.mjs, extra.mjs
  item/
    common/
      base.mjs, usage.mjs, skill-base.mjs, outfit-base.mjs, extensible.mjs
    helpers.mjs             # defenceField(), attackField() 等 (Item 側)
    style.mjs, weapon.mjs, ...
  card/
    common/base.mjs         # CardBaseTemplate (Item とは独立)
    playing-cards.mjs, neuro-cards.mjs, other.mjs
```

## Mixin パターン

1 type が複数 template を合成する構造 (例: cast = biography + attributes + actorBase)。

```javascript
// scripts/data/abstract.mjs
export class SystemDataModel extends foundry.abstract.TypeDataModel {
  static mixin(...mixins) { /* class extends this + プロトタイプ合成 */ }
}

// scripts/data/actor/cast.mjs
export class CastDataModel extends SystemDataModel.mixin(
  BiographyTemplate, AttributesTemplate, ActorBaseTemplate
) {
  static defineSchema() {
    return { ...super.defineSchema(), /* cast 固有 */ };
  }
}
```

登録 (`scripts/tnx.mjs` の `init` フック):

```javascript
CONFIG.Actor.dataModels.cast = CastDataModel;
```

## defineSchema 記述方針

- 基本は**インライン**定義
- 繰り返しパターンはヘルパー関数化 (`damageField()`, `attackField()` 等)
- 1 type 専用の小さな繰り返しはファイル内ローカル関数 (例: `style.mjs` の `abilityField()`)

## HTMLField 化 (B-9)

`description` (トップレベル) を `StringField` → `HTMLField` に変更。

`system.json` の `documentTypes.*.htmlFields: ["description"]` で宣言:

- Item 全 17 type / Card 全 3 type
- Actor: cast, guest, extra (biography 保有 type のみ)
- troop, player は biography 非保有のため対象外

## シートとの接続

**DataModel 移行とシート書き換えは分離** (B-0 決定):

- シートは `actor.system.xxx` でアクセスする方式を維持
- シートロジックを DataModel に寄せない (フェーズ3 ApplicationV2 移行時に整理)
- `prepareDerivedData` への EXP 計算移行は将来検討 (現状はフック駆動)

## マイグレーション

実運用前のため**データマイグレーション機構は未導入** (B-6 判断)。

破壊的変更が必要になった時点で設計する。

## 設計上の注意点

| 項目 | 内容 |
|---|---|
| `history` | `ObjectField` — ランダム ID キーのオブジェクトマップ。EmbeddedDocument ではない |
| `styleSkill` 配列 | template.json の `["blank"]` ではなく、シート実装を真実の源として SchemaField 定義 |
| タイポ維持 | `RewritedTarget`, `RewritingMiracle_ID` — 一貫性のため現状維持 (KI-018/019) |
| User | **DataModel 不可**。システムデータは flag のみ (フェーズ2) |

## 参照

- 詳細レビュー: [Design_Review_Entries.md](../../03_System/Design_Review_Entries.md) (B-0〜B-9 エントリ)
- 移行計画: [Phase_B_Tasks_Detail.md](../Phases/Phase_B_Tasks_Detail.md)
