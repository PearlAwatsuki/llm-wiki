# フェーズ3: 既存シート ApplicationV2 移行

**目標**: ApplicationV1 を完全廃止。プロジェクト内の全 Application を V2 に移行する。
V2 移行と並走して **CSS リファクタリング**（デッドコード除去・構造整理・コンポーネント化・再利用性向上）を実施する。
視覚デザイン（カラー・タイポグラフィ・装飾）の刷新は**フェーズ4 に分離**する。

**方針確定日**: 2026-06-06（CSS リファクタと視覚デザインの分離は同日再確定）

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

### 3-CSS-0: CSS リファクタリング（構造整理・再利用性向上）✅ (2026-06-07)

> 2026-06-06 方針再確定。**視覚デザインの決定はフェーズ4 に分離**。
> 本サブフェーズはコード構造の整理と、フェーズ6（未実装シート追加）を見据えた再利用性の向上に特化する。
> デッドコード除去・構造整理が目的であり、見た目の変更を伴わない。

**スコープ（構造・整理）**:
- [x] デッドコード除去（`.abilities-section`、▼▼▼ ステールコメント等）
- [x] 重複・矛盾ルールの整理（分類C: `.array-row select` 等 15+ 重複ペア解消）
- [x] スコープ付与（分類D: `.tokyo-nova.sheet.item`, `.tnx-hud`, `.tokyo-nova` 親スコープ追加）
- [x] CSS カスタムプロパティの棚卸し・整理（分類B: 4変数追加、ハードコードカラー変数化）
- [x] ベンダープレフィックス互換問題の解消（分類E: `.edit-mode-toggle` に `appearance: none` 追加）
- [x] `!important` 削減（分類F: `.small-button` / `.usage-limit-group` を詳細度引き上げで解消）

**スコープ（再利用性）**:
- [x] コンポーネントパターンの確立（グループ1・3・4・6・7: セレクターリスト統合・スタイル統一）
- [x] 外見同一・クラス別パターンの整理（`.texts-header`/`.info-header`、`.text-item-header`/`.info-item-header`、アイコンコントロール共通化）

**フェーズ4 に持ち越すもの**:
- カラーパレット・アクセントカラーの変更
- タイポグラフィ設計（フォント種類・サイズスケール）
- 装飾要素方針（グリッチ・スキャンライン・アニメーション等）
- 分類L（ライトテーマ値）、分類M（スートパターン統一）、分類I（変数名整理）
- 残留 hardcoded color 9 件（単一箇所のみの設計値、DRY メリットなし）
- `rgba(0, 170, 255, 0.08)` → `--tnx-color-primary-glow`（0.1）に統一（L2044）

**追加作業（2026-06-07）**:
- CSS 開発ツール整備: `stylelint` + `.stylelintrc.json`、`postcss` + `postcss-combine-duplicated-selectors`（build: 8917c4f）
- `npm run lint:css`: 警告 0 を確認（重複セレクタ・空ブロックなし）
- `npm run css:merge`: 構造変更 0 を確認（分散セレクタなし）

### 3-3: CastSheet V2 化 + CSS デザイン適用 ✅ (2026-06-07)

- [x] `TokyoNovaCastSheet` を `HandlebarsApplicationMixin(ActorSheetV2)` 基底に移行
- [x] KI-017: デッドコード (`_getHistoryData` / `_saveHistoryData` / `_updateExpTotal`) 削除
- [x] KI-018: `RewritedTarget` → `RewrittenTarget` 修正（前コミットで完了）
- [x] KI-019: `RewritingMiracle_ID` → `rewritingMiracleId` 修正（前コミットで完了）
- [x] テスト全通過確認 (625/625)
- [x] 各テンプレート: `<form>` → `<section>`、nav `<a>` に `data-group="primary"`、`data-action="editImage"`、historyリスト/BSリストに `data-action` 追加

**実装コミット**: a3110e5 (2026-06-07)

**追加コミット**: 1d9bcc5 / 3a9ad64 (2026-06-07)

- [x] タブ切替不動作修正: nav `<a>` に `data-action="tab"` 追加・コンテンツ div に `data-group` 追加（V2 組み込みタブハンドラの要件）
- [x] プレイヤーアクター廃止に伴う関連コード完全削除: unlinkPlayer ボタン・action・`_onDrop` override・`_onUnlinkPlayer`・`_performHistoryUpdate` B ブランチ・`playerActor` 参照
- [x] スートチェックボックス CSS 特異度修正: `.suit-checkbox-tiny input` → `input[type="checkbox"]`（グローバルルールとの競合解消）
- [x] セッション履歴タブ sync チェックボックスレイアウト修正: `.sync-toggle-row` → `.checkbox-row`
- [x] 再描画時スクロールリセット対策: `_preRender` でスクロール位置保存・`_onRender` の `requestAnimationFrame` で復元
- [x] 再描画時サイドバーチラツキ対策: `_preRender` で `.tnx-no-transitions` 付与・rAF コールバックで除去（CSS トランジション再生抑制）

### 3-4: TnxHud V2 化 + HUD CSS デザイン ✅ (2026-06-07)

- [x] `TnxHud` を `HandlebarsApplicationMixin(ApplicationV2)` に移行
- [x] サイドバー連動 JS 死コード削除（sidebar collapse リスナー・`accessCardPileId` / `gmTrumpDiscardId` 未使用変数を除去）
- [x] `popOut: false` 相当を V2 で実現（`window: { frame: false, positioned: false }`）
- [x] jQuery 依存を完全排除（ContextMenu / dragDrop はネイティブ DOM、`data-action` キャメルケース化）
- [x] hud.hbs: `id="tnx-hud"` 削除（DEFAULT_OPTIONS で付与）、data-action を camelCase 化
- [x] tnx.mjs: `render(true)` → `render({ force: true })` 更新
- [x] テスト全通過確認 (625/625)
- [x] HUD CSS: 3-CSS-0 で実施済み（スコープ付与・重複解消完了）

**実装コミット**: ab03084 (2026-06-07)

### 3-5: TnxScenarioSheet V2 化 + CSS デザイン適用 ✅ (2026-06-07)

- [x] `TnxScenarioSheet` を `HandlebarsApplicationMixin(DocumentSheetV2)` 基底に移行
- [x] JournalSheet 特有機能（dragDrop / _onDrop）の V2 対応
- [x] `getData()` → `_prepareContext()`、`this.object` → `this.document` に全面置換
- [x] jQuery 依存を完全排除（ContextMenu / change リスナーはネイティブ DOM）
- [x] `Dialog.confirm()` → `DialogV2.confirm()` に移行
- [x] 24 アクションを `_handleActionClick()` switch → 静的メソッド群に分割
- [x] 動的 import（user-flag-schema）をトップレベル静的 import に移動
- [x] `_onSendInfoToChat()` をヘルパー関数で重複排除
- [x] scenario-sheet.hbs: `<form>` → `<section>`、nav `<a>` に `data-action="tab"` 追加、全 data-action camelCase 化
- [x] シナリオシート CSS: 3-CSS-0 で実施済み（スコープ付与・重複解消完了）
- [x] テスト全通過確認 (625/625)

**実装コミット**: a5a51a2 (2026-06-07)

### 3-6: ダイアログ系 V2 化 + CSS デザイン適用 ✅ (2026-06-07)

- [x] `TnxScenarioSettingWizard` を `HandlebarsApplicationMixin(ApplicationV2)` に移行
  - `static DEFAULT_OPTIONS / PARTS / ACTIONS` 構成
  - ステップナビゲーション・スピナーをすべて `static ACTIONS` ベースに変換
  - `_updateObject()` → `_onFinishSetup()` + `_executeSetup()` に分割
  - 動的 `import` をトップレベル静的 `import` に移動
  - scenario-setting-wizard.hbs: 全ボタンに `data-action` 付与、`type="submit"` → `type="button" data-action="finishSetup"`
- [x] `tnx-dialog.mjs` の 8 ダイアログを `DialogV2.wait()` に移行
  - `DeckCreationDialog` / `AmountInputDialog` / `TargetSelectionDialog` / `CardSelectionDialog`
  - `RichConfirmDialog` / `DealTrumpDialog` / `UnlinkConfirmDialog` / `UsageCreationDialog`
  - `AmountInputDialog`: バリデーションをコールバック後に移動（V1 はダイアログを閉じず再入力可、V2 は null 返却で閉じる — 仕様変更として記録）
- [x] `CardSelectionDialog` の jQuery 依存を完全排除
  - card-selection-dialog.hbs: `<div>+<label>` 構造 → `<label class="card-selection-item">` ラップ構造に変更
  - CSS: `:has(input:checked)` セレクタで選択状態を表現（`.selected` クラス JS 付与不要）
- [x] `npm run lint` エラー 0 確認

**実装コミット**: 4a19780 (2026-06-07)

### 3-7: 最終検証・CSS 最終調整・記録 ✅ (2026-06-07)

- [x] 全テスト通過確認 (625/625)
- [x] 実機確認 — アクトシートタブ動作確認済み（ユーザー確認）
- [x] CSS 最終調整・全シートの視覚的一貫性確認（3-CSS-0 で実施済み）
- [x] KI-009/017/018/019 → Resolved_Issues.md に移動
- [x] Design_Review_Entries.md に 3-6 / 3-7 エントリ追記（タブバグ事後検討含む）
- [x] Roadmap.md のフェーズ3 完了マーク + Completed_Phases.md 追記

**修正コミット（3-5 後のタブバグ対応）**: 932da9a, 8587b82, 00b16c0 (2026-06-07)

---

## 参照

- V2 参照実装: `scripts/module/tnx-record-sheet.mjs`
- [Sheets.md](../UI/Sheets.md)
- [HUD_and_Dialogs.md](../UI/HUD_and_Dialogs.md)
- [Design_Review_Entries.md](../../02_System/Design_Review_Entries.md)
- [Active_Issues.md](../Issues/Active_Issues.md)
