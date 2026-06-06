# フックとモジュール

## 主要モジュール

| モジュール | ファイル | 役割 |
|---|---|---|
| TnxActionHandler | `scripts/module/tnx-action-handler.mjs` | カード操作 (判定・ドロー・プレイ・切り札・ニューロ) |
| TnxHud | `scripts/module/tnx-hud.mjs` | 常駐 HUD (ApplicationV1) |
| TnxDialog | `scripts/module/tnx-dialog.mjs` | 各種ダイアログ (Promise ベース `prompt()`) |
| EffectsSheetMixin | `scripts/module/effects-sheet-mixin.mjs` | ActiveEffect CRUD |
| TnxHistoryMixin | `scripts/module/tnx-history-mixin.mjs` | EXP 履歴テーブル操作 |
| TnxSkillUtils | `scripts/module/tnx-skill-utils.mjs` | 技能関連ユーティリティ |
| カード生成 | `tnx-playing-cards.mjs`, `tnx-neuro-cards.mjs`, `tnx-access-cards.mjs` | デッキ定義 |
| User flag | `user-flag-schema.mjs` | User flag スキーマ・読み書き (フェーズ2) |
| レコードシート | `tnx-record-sheet.mjs` | User EXP 表示 (ApplicationV2) |
| cast 所有 | `cast-ownership.mjs` | ownership → `ownerUserId` 記録 (フェーズ2-1) |
| EXP 同期 | `exp-sync.mjs` | cast ↔ User EXP 同期 (フェーズ2-2) |

## init フック (`scripts/tnx.mjs`)

| フック | 処理 |
|---|---|
| `preCreateActor` | cast 作成時の所有権 OBSERVER/OWNER 設定 |
| `preCreateItem` | 非 GM 作成時 OWNER 権限 |
| `preCreateCard` | 切り札置き場上限 (1枚) チェック |
| `createActor` | player→手札/切り札自動作成、cast→初期技能+手札確認 |
| `preDeleteItem` | miracle usageCount 削除抑制、style→miracle 連動削除 |
| `preDeleteActor` | cast/player リンクカード削除確認 |
| `preUpdateItem` | miracle isUsed リセット、style レベル→miracle usageCount |
| `updateActor` | player カード権限名前同期、cast ownership→ownerUserId (2-1) |
| `createCard` / `deleteCard` | `game.tnx.refreshSheets()` (50ms delay) |
| `renderSettingsConfig` | 設定画面ドロップダウン化 (v13 DOM API 対応済) |
| `renderJournalDirectory` | 「アクトシートを作成」ボタン挿入 (v13 対応済) |
| `getUserContextOptions` | レコードシートを開くメニュー (フェーズ2-0) |

**削除済**: `renderHotbar` (v13 でホットバー折りたたみ廃止 — KI-012)

## ready フック

| フック | 処理 |
|---|---|
| `createItem` / `deleteItem` / `updateItem` | cast 親の場合 `updateCastExp()` |
| `updateActor` | cast EXP 再計算、cast→User 同期 (2-2)、手札上限更新 |
| `updateUser` | User flag 変更→紐づく cast へ配布 (2-2) |

## ダイアログ一覧

すべて `TnxDialog` 経由の静的 `prompt()` パターン:

- `DeckCreationDialog`, `AmountInputDialog`, `TargetSelectionDialog`
- `CardSelectionDialog`, `RichConfirmDialog`, `DealTrumpDialog`
- `UnlinkConfirmDialog`, `UsageCreationDialog`

## Mixin 適用方式 (設計課題 KI-009)

| 対象 | 方式 |
|---|---|
| CastSheet | インスタンスプロパティへ Mixin メソッド代入 |
| ItemSheet | `EffectsSheetMixin.activateEffectListListeners()` 静的呼び出し |

フェーズ3 (ApplicationV2 移行) で統一予定。

## UI 更新

```javascript
game.tnx.refreshSheets()  // 自クライアントのシート/HUD 再描画
```

- `app._closed` 使用箇所あり (v13 未確認 — 代替: `app.rendered`)
- マルチプレイヤー同期は未実装 (KI-006)
