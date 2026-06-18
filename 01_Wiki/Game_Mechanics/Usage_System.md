# usage (用途) システム

判定・効果発動の**データ基盤**。D&D 5e の「用途」設計思想を踏襲。

## データ構造

`UsageTemplate` (`scripts/data/item/common/usage.mjs`):

```javascript
actions: ArrayField(SchemaField({
    _id:         StringField,           // ランダム ID (randomID())。インデックス参照を廃止
    type:        StringField,           // 種別（作成時固定。下記参照）
    name:        StringField,
    description: StringField,

    timing: SchemaField({
        value:       StringField,       // "blank" | "action" | "process" | "reaction" | "other" | ...
        actionName:  StringField,       // timing.value === "action" の場合のサブ選択肢
        processName: StringField,       // timing.value === "process" の場合のサブ選択肢
        timingOther: StringField,       // timing.value === "other" の場合の自由入力
    }),

    target: StringField,                // "blank" | "self" | "single" | "team" | ...

    effects: ArrayField(SchemaField({
        effectId: StringField,          // この用途使用時に付与する ActiveEffect の id
    })),

    baseSkillRef: SchemaField({
        itemId: StringField,            // ベース技能の Item ID。generalSkill 新規作成時に親 ID で自動設定
    }),

    skillRefs: ArrayField(SchemaField({
        itemId: StringField,            // 組み合わせ技能（ベース技能は含まない）の Item ID
    })),

    // attack タイプのみ
    weaponRef: SchemaField({ itemId: StringField }),
    damageType: StringField,            // "S" | "P" | "I"

    // damageBoost / damageReduce タイプのみ
    formula: StringField,               // 効果量（計算式または固定値文字列）
    damageCategory: StringField,        // "physical" | "mental"

    // modification タイプのみ
    modifiableParams: ArrayField(StringField),
}))
```

### タイプ（type）値域

| type | 説明 | 主な固有フィールド |
|---|---|---|
| `check` | 技能判定（情報収集・登場など） | baseSkillRef, skillRefs |
| `attack` | 攻撃判定（物理・精神・社会戦） | baseSkillRef, skillRefs, weaponRef, damageType |
| `declaration` | 宣言・神業を含む | — |
| `damageBoost` | ダメージ増加 | formula, damageCategory |
| `damageReduce` | ダメージ軽減 | formula, damageCategory |
| `modification` | 改造 | modifiableParams |

- `controlCheck` は独立した判定種別であり、usage タイプには含まない（timing で区別）
- 神業（miracle）は当面 `declaration` に統合。専用システム分離は将来フェーズ
- タイプは作成時固定（UI 上で切り替え不可）

### タイミング / 対象の "-"（blank）の意味

`timing.value = "blank"` / `target = "blank"` は「指定なし」= **どのタイミング・対象とも組み合わせ可能**を意味する（自由タイミング・自由対象）。空欄でも判定は正常に起動できる。

### migrateData

既存データに `_id` が無い場合は `randomID()` で自動付与し、`baseSkillRef` が無い場合は `{ itemId: "" }` を追加する（`Usage_Template.migrateData`）。

## 使用 Item type

`miracle`, `generalSkill`, `styleSkill` が UsageTemplate を mixin。

## UI コンポーネント

| コンポーネント | ファイル | 状態 |
|---|---|---|
| usage-list.hbs | `templates/parts/usage-list.hbs` | `data-usage-id` ベース。種別タグ表示 |
| TnxUsageSheet | `scripts/module/tnx-usage-sheet.mjs` | ApplicationV2、タブ構成、KI-002 解決 |
| usage-sheet.hbs | `templates/app/usage-sheet.hbs` | TnxUsageSheet 本体テンプレート |

### TnxUsageSheet の設計

- **タブ**: 基本 / 起動 / 種別固有（check・attack・damageBoost / damageReduce・modification） / エフェクト
- `submitOnChange: true` — フォーム変更で即時 `_patchUsage()` を呼び自動保存
- `_patchUsage()` は `foundry.utils.setProperty` でドット記法キーを処理（`mergeObject` は配列でバグあり）
- **skillRefs 追加**: ドロップダウン選択で即時追加（「追加」ボタン廃止）
- **アクション技能**: ベース技能を親技能で表示固定（セレクター非表示）
- **ベース技能候補** (`availableBaseSkills`): アクション技能を含む全技能（自身以外）。非アクション技能もベースになりうる
- **コンボ候補** (`availableSkills`): アクション技能を除外（アクション技能は必ずベースになるため）
- **スタンドアロンアイテム対応**: `actor` が null の場合も `baseId === parentItemId` なら親アイテムの名前を表示

## generalSkill 新規作成時の自動用途挿入

`preCreateItem` フック（`scripts/tnx.mjs`）:
- `data.type === "generalSkill"` かつ `system.actions` が空の場合に `check` 用途を1件自動挿入
- `baseSkillRef.itemId` には `item._id` を設定（親技能が自身をベースとして保持）

## 判定起動時のコンボ解決

`_onStartSkillCheck` (`scripts/actor/tnx-cast-sheet.mjs`):

1. 技能行クリック → `check` タイプの用途を解決
2. `baseSkillRef.itemId` からベース技能を決定（未設定時は親アイテム ID にフォールバック）
3. `skillRefs` からコンボ技能を収集
4. 用途所持技能がベース技能でない場合は自動的にコンボに追加（非アクション技能からの起動）
5. 全技能スートの積集合で `validSuits` を計算し `TnxJudgmentFlow.open()` へ渡す

## 変更制約

CLAUDE.md §4.3 より:

- `usage.mjs` のフィールド追加・改名は事前承認を得る
- `usage-list.hbs`・`TnxUsageSheet` の構造変更は設計議論を経てから

## miracle 連動（既存動作、改変禁止）

- style レベル変更 → miracle usageCount 増減 (`preUpdateItem`)
- style 削除 → 連動 miracle 削除 (`preDeleteItem`)
- miracle 削除 → usageCount チェックで抑制 (`preDeleteItem`)

## 参照

- DataModel: `scripts/data/item/common/usage.mjs`
- 編集シート: `scripts/module/tnx-usage-sheet.mjs`
- 判定起動: `scripts/actor/tnx-cast-sheet.mjs` `_onStartSkillCheck`
- 解決済み: KI-002（usage edit ハンドラ不在） → [Resolved_Issues.md](../Issues/Resolved_Issues.md)
