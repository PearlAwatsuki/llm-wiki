# Foundry VTT v13 移行

v13 実機観測に基づく移行メモの要約。詳細は `01_Raw/V13_MIGRATION_NOTES.md`。

## 互換性

```json
"compatibility": {
  "minimum": 11,
  "verified": "13.351"
}
```

## 致命的エラー (フェーズ0 で解消)

すべて **render 系フックの第2引数が jQuery → HTMLElement** に変更されたことが原因。

| フック | 対応 | 状態 |
|---|---|---|
| renderSettingsConfig | jQuery → DOM API (P0-08) | ✅ 完了 |
| renderHotbar | html[0] → html (P0-09) | ✅ エラー解消 → 機能削除 (KI-012) |
| renderJournalDirectory | DOM API + ボタン再設計 (1-1) | ✅ 完了 |

## deprecation 警告 (フェーズ0 では許容)

v15 まで動作保証。フェーズ3/6 で ApplicationV2 移行時に対処:

- ApplicationV1 全般 (ActorSheet, ItemSheet, Dialog 等)
- TextEditor.enrichHTML `{ async: true }`
- Dialog.confirm
- loadTemplates
- Actors/Items.unregisterSheet
- ContextMenu jQuery オプション

## 個別項目

| # | 項目 | 状態 | 優先度 |
|---|---|---|---|
| 1 | system.json compatibility | ✅ verified 更新済 | — |
| 2 | ApplicationV1 deprecation | 許容 (フェーズ3/6) | 低 |
| 3 | app._closed | 未確認。代替: app.rendered | 中 |
| 4 | renderJournalDirectory | ✅ 1-1 で再設計 | — |
| 5 | renderSettingsConfig | ✅ P0-08 | — |
| 6 | renderHotbar | ✅ 死コード削除 (1-0) | — |
| 7 | HUD サイドバー連動 | CSS 死ルール削除済。JS 残存 | 低 (フェーズ2) |
| 8 | TextEditor async | 影響なし | 低 |
| 9 | Dialog.confirm | 影響なし | 低 |
| 10 | Font Awesome 5/6 混在 | 未確認 | 中 (フェーズ5) |
| 11 | loadTemplates | 影響なし | 低 |
| 12 | unregisterSheet | 影響なし | 低 |
| 13 | Cards API | 致命的エラーなし | 要実機確認 |
| 14 | preCreateCard | 未確認 | — |
| 15 | ContextMenu | 影響なし | 低 |

## v13 UI 変更への対応 (フェーズ1)

| サブフェーズ | 内容 | KI |
|---|---|---|
| 1-0 | ホットバー折りたたみ死コード削除 | KI-012 解消 |
| 1-1 | アクトシート作成ボタン復活 | KI-011 解消 |
| 1-2 | HUD CSS 座標調整 | KI-013 解消 |

## JournalDirectory ApplicationV2 化

- 旧 `.create-document` セレクタは v13 に存在しない
- 採用: `[data-action="createEntry"]` 直後にボタン挿入
- 不採用: getHeaderControls (API 未確定), JournalEntryPage サブタイプ (通常 journal に混入)

## 今後の v13 対応

| 対象 | フェーズ |
|---|---|
| TnxHud ApplicationV2 化 | フェーズ2 |
| 全シート ApplicationV2 化 | フェーズ3 |
| Font Awesome 統一 | フェーズ5 |
| Dialog → ApplicationV2 | フェーズ6 |
