# シート一覧

## Application API 状況

| API | 使用状況 |
|---|---|
| ApplicationV1 | フェーズ3 で**全廃予定**。現在も全既存シートが V1 のまま |
| ApplicationV2 | `TnxRecordSheet` のみ完了 (フェーズ2-0)。フェーズ3 で全面移行 |

**方針**: このプロジェクトでは ApplicationV1 は一切使用しない。フェーズ3 完了後は V1 クラス (ActorSheet, ItemSheet, JournalSheet, Application, FormApplication, Dialog) をコード内から完全排除する。

## 実装済みシート

| クラス | 継承 | 対象 type | ファイル |
|---|---|---|---|
| TokyoNovaCastSheet | ActorSheet | cast | `scripts/actor/tnx-cast-sheet.mjs` |
| TokyoNovaPlayerSheet | ActorSheet | player | `scripts/actor/tnx-player-sheet.mjs` | (v0.3.0 廃止)
| TokyoNovaItemSheet | ItemSheet | 基底 (直接登録なし) | `scripts/item/tnx-item-sheet.mjs` |
| TokyoNovaStyleSheet | TokyoNovaItemSheet | style | `scripts/item/tnx-style-sheet.mjs` |
| TokyoNovaMiracleSheet | TokyoNovaItemSheet | miracle | `scripts/item/tnx-miracle-sheet.mjs` |
| TokyoNovaGeneralSkillSheet | TokyoNovaItemSheet | generalSkill | `scripts/item/tnx-general-skill-sheet.mjs` |
| TokyoNovaStyleSkillSheet | TokyoNovaItemSheet | styleSkill | `scripts/item/tnx-style-skill-sheet.mjs` |
| TokyoNovaOrganizationSheet | ItemSheet | organization | `scripts/item/tnx-organization-sheet.mjs` |
| TnxScenarioSheet | JournalSheet | JournalEntry (アクト) | `scripts/journal/tnx-scenario-sheet.mjs` |
| TnxRecordSheet | ApplicationV2 | User (レコード) | `scripts/module/tnx-record-sheet.mjs` |

## 未登録 (コアデフォルト)

**Actor**: guest, troop, extra — フェーズ6 で追加予定

**Item**: weapon, armor, ianus, cyborg, tron, tap, vehicle, residence, housingArea, combiner, general, lifePath — フェーズ6 ( cast 含む全既存シートはフェーズ3 で ApplicationV2 移行予定)

## 共通 UI パターン

- **view/edit モード**: `_isEditMode` + `_render()` オーバーライド
- **UUID コピー**: 全シートヘッダー
- **テキスト自動スケール**: CastSheet, PlayerSheet (`_applyTextSqueezing()`)

## アクトシート (TnxScenarioSheet)

- `JournalEntry` はサブタイプを持たない → 作成ダイアログでの型選択不可
- 導線: ジャーナルディレクトリの「アクトシートを作成」ボタン (フェーズ1-1)
- 構造: flag + 独自タブ UI (ハンドアウト・情報項目・シーン・テキスト)
- **JournalEntryPage カスタムサブタイプ化は不採用** (通常ジャournal に混入するため)

## ApplicationV2 移行計画

| 対象 | フェーズ |
|---|---|
| TnxHud | フェーズ2 |
| TnxRecordSheet | 完了 (2-0) |
| 既存 Actor/Item/Journal シート | フェーズ3 |
| 未実装シート | フェーズ6 (フェーズ5 デザイン踏襲) |

## 参照

- テンプレート: [Templates.md](Templates.md)
- v13 移行: [Foundry_v13.md](../Migration/Foundry_v13.md)
