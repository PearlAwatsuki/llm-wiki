# フェーズ3: 既存シート ApplicationV2 移行

**目標**: ApplicationV1 を完全廃止。プロジェクト内の全 Application を V2 に移行する。**V2 移行で生じた CSS 回帰バグ修正と、旧フェーズ5（CSS リファクタ + デザイン刷新）の全作業をフェーズ3 に統合して並走させる**（2026-06-06 方針変更）。

**方針確定日**: 2026-06-06

---

## スコープ

### 移行対象シート

| クラス | 現基底 | ファイル | 備考 |
|---|---|---|---|
| `TokyoNovaCastSheet` | `ActorSheet` | `scripts/actor/tnx-cast-sheet.mjs` | 最複雑 |
| `TokyoNovaItemSheet` | `ItemSheet` | `scripts/item/tnx-item-sheet.mjs` | 基底クラス |
| `TokyoNovaStyleSheet` | `TokyoNovaItemSheet` | `scripts/item/tnx-style-sheet.mjs` | 派生 |
| `TokyoNovaMiracleSheet` | `TokyoNovaItemSheet` | `scripts/item/tnx-miracle-sheet.mjs` | 派生 |
| `TokyoNovaGeneralSkillSheet` | `TokyoNovaItemSheet` | `scripts/item/tnx-general-skill-sheet.mjs` | 派生 |
| `TokyoNovaStyleSkillSheet` | `TokyoNovaItemSheet` | `scripts/item/tnx-style-skill-sheet.mjs` | 派生 |
| `TokyoNovaOrganizationSheet` | `ItemSheet` | `scripts/item/tnx-organization-sheet.mjs` | |
| `TnxScenarioSheet` | `JournalSheet` | `scripts/journal/tnx-scenario-sheet.mjs` | Journal 系 |

### 移行対象 Application

| クラス | 現基底 | ファイル | 備考 |
|---|---|---|---|
| `TnxHud` | `Application` | `scripts/module/tnx-hud.mjs` | popOut: false、常駐 |
| `TnxScenarioSettingWizard` | `FormApplication` | `scripts/module/tnx-scenario-setting-wizard.mjs` | マルチステップフォーム |

### 移行対象ダイアログ (8 クラス、全て `tnx-dialog.mjs`)

| クラス | 用途 |
|---|---|
| `DeckCreationDialog` | 山札作成オプション |
| `AmountInputDialog` | 数値入力 |
| `TargetSelectionDialog` | 複数選択肢から一択 |
| `CardSelectionDialog` | カード複数選択 |
| `RichConfirmDialog` | 画像付き確認 |
| `DealTrumpDialog` | 切り札配布先選択 |
| `UnlinkConfirmDialog` | リンク解除確認 |
| `UsageCreationDialog` | usage 追加 |

### V2 化済み（対象外）

- `TnxRecordSheet` — フェーズ2-0 で完了。参照実装として使う。

---

## V1 → V2 API 対応表 (3-0 で確定)

| V1 | V2 |
|---|---|
| `static get defaultOptions()` | `static DEFAULT_OPTIONS = {...}` |
| `getData()` | `async _prepareContext(options)` |
| `activateListeners(html)` | `static DEFAULT_OPTIONS.actions` + `_onRender()` |
| `options.tabs` | V2 組み込みの `static TABS` + `tabGroups` プロパティ |
| `ActorSheet` 継承 | `HandlebarsApplicationMixin(ActorSheetV2)` (`foundry.applications.sheets`) |
| `ItemSheet` 継承 | `HandlebarsApplicationMixin(ItemSheetV2)` (`foundry.applications.sheets`) |
| `JournalSheet` 継承 | `HandlebarsApplicationMixin(DocumentSheetV2)` + `DocumentSheetConfig.registerSheet` |
| `Application` 継承 | `HandlebarsApplicationMixin(ApplicationV2)` |
| `FormApplication` 継承 | `HandlebarsApplicationMixin(ApplicationV2)` |
| `new Dialog({...})` | `foundry.applications.api.DialogV2.wait({...})` |
| `render(true)` | `render({ force: true })` |
| `jQuery` 前提のハンドラ | ネイティブ DOM `addEventListener` |
| `_getHeaderButtons()` | `static DEFAULT_OPTIONS` の `window.controls` |

`TabsApplicationMixin` は v13 に**存在しない**。タブは V2 基底クラス (`ActorSheetV2` 等) に組み込みサポートがある。

---

## フェーズ3 で解消する KI

| KI | 内容 | 難易度 | 対応サブフェーズ |
|---|---|---|---|
| KI-009 | `EffectsSheetMixin` の適用方式二系統(インスタンス代入 vs 静的呼び出し) | 中 | 3-1 |
| KI-017 | cast シートのデッドコード (`_getHistoryData` 等) | 低 | 3-3 |
| KI-018 | `RewritedTarget` タイポ → `RewrittenTarget` | 低 | 3-3 |
| KI-019 | `RewritingMiracle_ID` 命名規約違反 → `rewritingMiracleId` | 低 | 3-3 |
| (軽微) | `tnx-hud.mjs` サイドバー連動 JS 死コード | 低 | 3-4 |

---

## サブフェーズ

### 3-0: 移行方針・設計確定 ✅ (2026-06-06)

- [x] `ActorSheetV2` / `ItemSheetV2` の確認 → `foundry.applications.sheets` に存在
- [x] `JournalSheet` の V2 相当確認 → `DocumentSheetV2` 直継承 + `registerSheet`
- [x] `foundry.applications.api.DialogV2` の API 確認 → `DialogV2.wait()` で移行
- [x] `EffectsSheetMixin` の V2 統一設計決定 → `ACTIONS` オブジェクト方式で KI-009 解消
- [x] Item シート継承ツリーの設計決定 → V2 でも `ItemSheetV2` 基底の継承ツリーを維持
- [x] `TabsApplicationMixin` の確認 → v13 に存在しない。V2 組み込みの `TABS`/`tabGroups` を使用
- [x] 設計内容を wiki / Design_Review_Entries に記録 → DR エントリ「3-0」追記済み

### 3-1: Item シート基底 + EffectsSheetMixin V2 化 ✅ (2026-06-06)

- [x] `TokyoNovaItemSheet` を `ItemSheetV2` 基底に移行 (`HandlebarsApplicationMixin(ItemSheetV2)`)
- [x] `EffectsSheetMixin` を V2 対応に統一 (KI-009 解消、ACTIONS オブジェクト方式)
- [x] `active-effects-list.hbs` の data-action 値を V2 用に更新 (`createEffect` 等)
- [x] テスト全通過確認 (625/625)

### 3-2: Item 派生シート 5 本 V2 化 ✅ (2026-06-06)

- [x] `TokyoNovaStyleSheet` — DEFAULT_OPTIONS / PARTS / TABS / ContextMenu / dragDrop V2 化
- [x] `TokyoNovaMiracleSheet` — EffectsSheetMixin.activateEffectListListeners 呼び出しを削除(基底 ACTIONS で代替)
- [x] `TokyoNovaGeneralSkillSheet` — select/isInitial checkbox に stopPropagation パターン適用
- [x] `TokyoNovaStyleSkillSheet` — getData() 依存を _normalizeSystem() 静的ヘルパーで代替、スピナーを static actions に変換
- [x] `TokyoNovaOrganizationSheet` — ItemSheet 直継承 → TokyoNovaItemSheet 継承に統一
- [x] 各テンプレート: `<form>` → `<section>`、nav `<a>` に `data-group="primary"`、`data-action="editImage"`、action 名キャメルケース化
- [x] テスト全通過確認 (625/625)

### 3-2b: アイテムシート CSS 修正 ✅ (2026-06-06)

> フェーズ5 CSS 作業のフェーズ3 統合に伴い新設。V2 移行で生じた回帰バグ修正が中心。

- [x] `.edit-mode-toggle` に `display: inline-block; flex-shrink: 0;` 追加（V2 フレックスヘッダー対応）
- [x] `.editor` / `.editor-edit` / `.editor-content` 向け CSS を削除し `prose-mirror` カスタム要素向けに書き換え
- [x] `min-width: 513px` 除去（オーバーフロー原因）
- [x] `.tnx-editor-section .prosemirror` (小文字) dead CSS ルール削除
- [x] `.suit-selection input:checked + .suit-icon` → `+ i` 修正（セレクタ構造不一致）
- [x] Design_Review_Entries.md に 3-2 / 3-2b エントリ追記

### 3-CSS-0: デザインシステム基盤確立（設計セッション）

> 旧フェーズ5-0 + 5-1 相当。コード実装前にユーザーが設計方針を確定する。

- [ ] 基調トーン確定（クラシックサイバーパンク / モダン / ミニマル）
- [ ] カラーパレット確定（既存 `--tnx-color-*` 変数を継承・拡張）
- [ ] タイポグラフィ確定（見出し・本文・特殊用途フォント）
- [ ] 装飾要素方針確定（ボーダー・グリッチ・スキャンライン・アイコン体系）
- [ ] アニメーション方針確定（あり / なし / 最小限）
- [ ] CSS ファイル構成確定（単一 `tnx.css` 維持 or 分割）
- [ ] 上記決定を受けてコード実装: CSS 変数整理、dead CSS 削除（`.hotbar-is-collapsed` 等）

### 3-3: CastSheet V2 化 + CSS デザイン適用

- [ ] `TokyoNovaCastSheet` を `DocumentSheetV2` 基底に移行
- [ ] KI-017: デッドコード (`_getHistoryData` / `_saveHistoryData` / `_sortHistory`) 削除
- [ ] KI-018: `RewritedTarget` → `RewrittenTarget` 修正
- [ ] KI-019: `RewritingMiracle_ID` → `rewritingMiracleId` 修正
- [ ] キャストシートに 3-CSS-0 デザインシステムを適用
- [ ] テスト更新・通過確認

### 3-4: TnxHud V2 化 + HUD CSS デザイン

- [ ] `TnxHud` を `HandlebarsApplicationMixin(ApplicationV2)` に移行
- [ ] サイドバー連動 JS 死コード削除
- [ ] `popOut: false` 相当を V2 で実現
- [ ] HUD に 3-CSS-0 デザインシステムを適用
- [ ] テスト更新・通過確認

### 3-5: TnxScenarioSheet V2 化 + CSS デザイン適用

- [ ] `TnxScenarioSheet` を V2 相当に移行
- [ ] JournalSheet 特有機能(ドラッグ&ドロップ等)の V2 対応
- [ ] シナリオシートに 3-CSS-0 デザインシステムを適用
- [ ] テスト更新・通過確認

### 3-6: ダイアログ系 V2 化 + CSS デザイン適用

- [ ] `TnxScenarioSettingWizard` (FormApplication → ApplicationV2)
- [ ] `tnx-dialog.mjs` の 8 ダイアログ (`Dialog` → `DialogV2`)
- [ ] ダイアログ群に 3-CSS-0 デザインシステムを適用
- [ ] テスト更新・通過確認

### 3-7: 最終検証・CSS 最終調整・記録

- [ ] 全テスト通過確認
- [ ] 実機確認 (各シートの基本操作)
- [ ] CSS 最終調整・全シートの視覚的一貫性確認（旧フェーズ 5-4 相当）
- [ ] KI-009/017/018/019 → Resolved_Issues.md に移動
- [ ] Design_Review_Entries.md に各サブフェーズエントリ追記
- [ ] Roadmap.md のフェーズ3 完了マーク + Completed_Phases.md 追記

---

## 参照

- V2 参照実装: `scripts/module/tnx-record-sheet.mjs`
- [Sheets.md](../UI/Sheets.md)
- [HUD_and_Dialogs.md](../UI/HUD_and_Dialogs.md)
- [Design_Review_Entries.md](../../02_System/Design_Review_Entries.md)
- [Active_Issues.md](../Issues/Active_Issues.md)
