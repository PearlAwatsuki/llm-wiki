# フックとモジュール

## 主要モジュール

| モジュール | ファイル | 役割 |
|---|---|---|
| TnxActionHandler | `scripts/module/tnx-action-handler.mjs` | カード操作 (判定・ドロー・プレイ・切り札・ニューロ) |
| TnxHud | `scripts/module/tnx-hud.mjs` | 常駐 HUD (**ApplicationV2**) |
| TnxDialog | `scripts/module/tnx-dialog.mjs` | 各種ダイアログ (Promise ベース `prompt()`) |
| 判定 | `tnx-judgment-engine.mjs` / `tnx-judgment-flow.mjs` / `tnx-judgment-dialog.mjs` / `tnx-rl-request-app.mjs` | 達成値計算・判定フロー・ダイアログ・RL 要求 (フェーズ7-8) |
| usage 編集 | `tnx-usage-sheet.mjs` | 用途シート (ApplicationV2、フェーズ8) |
| コンディション | `conditions.mjs` / `condition-resolution.mjs` / `condition-resolution-core.mjs` | BS・戦闘不能の AE 表現・カードドロー解決 (フェーズ9-4) |
| ダメージチャート | `damage-chart.mjs` / `damage-chart-text-app.mjs` | チャート定義・効果文設定アプリ (フェーズ9-4) |
| ソケット | `tnx-socket-handler.mjs` | emit/on 集約 (KI-006 解消・フェーズ8-1) |
| EffectsSheetMixin | `scripts/module/effects-sheet-mixin.mjs` | ActiveEffect CRUD・ダメージ由来 AE のリスト除外 |
| カードセットアップ | `tnx-card-setup-app.mjs` | 山札/手札/切り札の設定アプリ (ゲーム設定メニュー、フェーズ5) |
| TnxHistoryMixin | `scripts/module/tnx-history-mixin.mjs` | EXP 履歴テーブル操作 |
| TnxSkillUtils | `scripts/module/tnx-skill-utils.mjs` | 技能関連ユーティリティ |
| カード生成 | `tnx-playing-cards.mjs` / `tnx-neuro-cards.mjs` / `tnx-access-cards.mjs` | デッキ定義 |
| User flag | `user-flag-schema.mjs` | User flag スキーマ・読み書き (フェーズ2) |
| レコードシート | `tnx-record-sheet.mjs` | User EXP 表示 (ApplicationV2) |
| cast 所有 | `cast-ownership.mjs` | ownership → `ownerUserId` 記録 (フェーズ2-1) |
| EXP 同期 | `exp-sync.mjs` | cast ↔ User EXP 同期 (フェーズ2-2) |

## init フック (`scripts/tnx.mjs`)

| フック | 処理 |
|---|---|
| `preCreateActor` | cast 作成時の所有権 OBSERVER/OWNER 設定 |
| `preCreateItem` | 非 GM 作成時 OWNER 権限。generalSkill は `check` 用途を1件自動挿入 |
| `preCreateCard` | 切り札置き場上限 (1枚) チェック |
| `createActor` | cast→prototypeToken 既定設定 + `setupDefaultSkills`（初期技能） |
| `preDeleteItem` | miracle usageCount 削除抑制、style→連動 miracle 削除 |
| `preUpdateItem` | miracle isUsed リセット、style レベル→miracle usageCount 増減、level3→ペルソナ/キー強制 |
| `getSceneControlButtons` | GM のシーンコントロールに「判定要求」ボタン追加（`TnxRlRequestApp`、フェーズ8-5） |
| `createCard` / `deleteCard` | `game.tnx.refreshSheets()` (50ms delay) |
| `renderJournalDirectory` | 「アクトシートを作成」ボタン挿入 (v13 対応済) |
| `getUserContextOptions` | レコードシートを開くメニュー (フェーズ2-0) |

モジュール読込時（init 外）に `renderActiveEffectConfig`（重複可/コンディション詳細タブ注入）・`preUpdateActiveEffect` / `createActiveEffect`（コンディションのカスケード等）・`renderChatMessageHTML` も登録。

**現行コードに無し**: `renderHotbar`（v13 でホットバー折りたたみ廃止 — KI-012）、`preDeleteActor`（player 廃止）、`renderSettingsConfig`（カード設定は `TnxCardSetupApp`＝ゲーム設定メニューへ移行）

## ready フック

`ready` で HUD 生成（`TnxHud`）、判定フローのダイアログ注入（`game.tnx.judgment`）、ソケット受信登録（`TnxSocketHandler.onMessage`）を行い、以下を登録する。

| フック | 処理 |
|---|---|
| `createItem` / `deleteItem` / `updateItem` | cast 親の場合 `recalcActorExp()` で EXP 再計算 |
| `updateActor` | cast EXP 再計算、cast→User 同期 (2-2)、手札上限更新 |
| `updateUser` | User flag 変更→紐づく cast へ配布 (2-2) |
| `updateSetting` | `revealPlayerHands` 変更で HUD 再描画 |
| `renderPlayerList` / `renderSidebar` | プレイヤー一覧・サイドバーの再描画連動 |

## ダイアログ一覧

すべて `TnxDialog` 経由の静的 `prompt()` パターン:

- `DeckCreationDialog`, `AmountInputDialog`, `TargetSelectionDialog`
- `CardSelectionDialog`, `RichConfirmDialog`, `DealTrumpDialog`
- `UnlinkConfirmDialog`, `UsageCreationDialog`

## Mixin 適用方式

`EffectsSheetMixin.ACTIONS` を各シートの `DEFAULT_OPTIONS.actions` にスプレッド展開して統一済み（KI-009 解消・フェーズ3）。V1 の二系統（インスタンス代入／静的呼び出し）は解消した。

## UI 更新

```javascript
game.tnx.refreshSheets()  // 自クライアントのシート/HUD 再描画
```

- マルチプレイヤー同期は `TnxSocketHandler` で実装済み（KI-006 解消・フェーズ8-1）。
