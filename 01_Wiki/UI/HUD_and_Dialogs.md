# HUD・ダイアログ・その他 UI

## TnxHud

| 項目 | 内容 |
|---|---|
| クラス | `TnxHud` extends `HandlebarsApplicationMixin(ApplicationV2)`（フェーズ3 で V2 化） |
| ファイル | `scripts/module/tnx-hud.mjs` |
| テンプレート | `templates/hud/hud.hbs` |
| 配置 | `position: fixed` 独立レイヤー |

### レイアウト (v13 調整済 — フェーズ1-2)

| エリア | 位置 | 内容 |
|---|---|---|
| `.hud-top-right` | 右上 (top:50px, right:360px) | シーン/ニューロカード操作 |
| `.hud-main-cards` | 右下 (bottom:80px, right:360px) | 山札・捨て札 |
| `.hand-area` | 下部中央 | 手札 (max-width:680px ≒ 5枚) |

### 構造

- 左上: TokenActionHUD 用に空け（フェーズ19）
- 右側: シーン/ニューロカード・アクセスカード表示エリア（折りたたみ可）・山札/捨て札
- カラム折りたたみ・他プレイヤー手札表示（`revealPlayerHands` 設定）を実装済み（フェーズ5）

### 手札取得

- **User flag 経由**（フェーズ2 完了・player Actor 廃止）。`game.user` の flag（`handPileId` / `trumpCardPileId`、`user-flag-schema.mjs`）からカード置き場を解決する。

## レコードシート (ApplicationV2)

| 項目 | 内容 |
|---|---|
| クラス | `HandlebarsApplicationMixin(ApplicationV2)` |
| 導線 | プレイヤーリスト右クリック → 「レコードシートを開く」 |
| 表示 | User flag の EXP サマリ + 履歴テーブル |
| 編集 | 履歴追加・編集・削除 (2-0 で実装) |

## セットアップウィザード

`TnxScenarioSettingWizard` (FormApplication) — アクト設定一括作成

## ダイアログ

`scripts/module/tnx-dialog.mjs` — すべて静的 `prompt()` → Promise

| ダイアログ | 用途 |
|---|---|
| DeckCreationDialog | 山札作成 |
| AmountInputDialog | 数値入力 |
| TargetSelectionDialog | 対象選択 |
| CardSelectionDialog | カード選択 |
| RichConfirmDialog | 確認 (リッチ) |
| DealTrumpDialog | 切り札配布 |
| UnlinkConfirmDialog | リンク解除確認 |
| UsageCreationDialog | 用途 (usage) 作成 |

## チャット表示

- `templates/chat/scene-card.hbs` — シーン/ニューロカードのチャット HTML
- カスタム ChatMessage クラスなし。`ChatMessage.create()` で生 HTML

## CSS

`css/tnx2.css` — 全 UI スタイル（フェーズ3 で構造リファクタ、フェーズ4 でデザイン刷新。旧 `tnx.css` は廃止）。

## 削除済み機能

- ホットバー自動折りたたみ (v13 コアから削除 — KI-012)
- ジャーナル2択作成メニュー → コア createEntry + 独自1ボタンに簡素化
