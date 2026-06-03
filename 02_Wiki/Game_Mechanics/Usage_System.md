# usage (用途) システム

判定・効果発動の**データ基盤**。D&D 5e の「用途」設計思想を踏襲。

## データ構造

`usage` template (`scripts/data/item/common/usage.mjs`):

```javascript
actions: ArrayField of SchemaField {
  type: StringField,
  name: StringField,
  description: StringField  // プレーンテキスト (HTMLField ではない)
}
```

## 使用 type

miracle, generalSkill, styleSkill が usage template を mixin。

## UI コンポーネント

| コンポーネント | ファイル | 状態 |
|---|---|---|
| usage-list.hbs | `templates/parts/usage-list.hbs` | edit ボタン未接続 (KI-002) |
| UsageCreationDialog | `scripts/module/tnx-dialog.mjs` | 動作 |

## 設計上の位置づけ

- フェーズ7 (判定システム設計) / フェーズ8 (実装) の中心
- 「用途を起動する」共通インタフェースを上位に置く構造を目指す
- 神業 (miracle) の複雑な効果も usage 経由で表現する方向性

## 変更制約

**設計議論なしに以下を改変しない** (CLAUDE.md §4.3):

- usage template 構造
- usage-list.hbs
- UsageCreationDialog

## miracle 連動

既存の usageCount 管理 (棚卸し確認済み範囲):

- style レベル変更 → miracle usageCount 増減 (`preUpdateItem`)
- style 削除 → 連動 miracle 削除 (`preDeleteItem`)
- miracle 削除 → usageCount チェックで抑制 (`preDeleteItem`)

神業の自動化処理はユーザー明示的合意なしに踏み込まない。

## 参照

- DataModel: `scripts/data/item/common/usage.mjs`
- 未解決: [Active_Issues.md](../Issues/Active_Issues.md) KI-002
