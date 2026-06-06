# HUD・ダイアログ・その他 UI

## TnxHud

| 項目 | 内容 |
|---|---|
| クラス | `TnxHud` extends `Application` (ApplicationV1) |
| ファイル | `scripts/module/tnx-hud.mjs` |
| テンプレート | `templates/hud/hud.hbs` |
| 配置 | `position: fixed` 独立レイヤー |

### レイアウト (v13 調整済 — フェーズ1-2)

| エリア | 位置 | 内容 |
|---|---|---|
| `.hud-top-right` | 右上 (top:50px, right:360px) | シーン/ニューロカード操作 |
| `.hud-main-cards` | 右下 (bottom:80px, right:360px) | 山札・捨て札 |
| `.hand-area` | 下部中央 | 手札 (max-width:680px ≒ 5枚) |

### 将来構造 (フェーズ2 V2 化時)

- 左上: TokenActionHUD 用に空け
- 右側三層: 上=シーン/ニューロ、中段=情報項目 (予定地)、下=山札/捨て札
- 他 PC 手札小型表示 (将来)

### 手札取得 (移行中)

- **現状**: player Actor の handPileId 等
- **フェーズ2-3 予定**: `game.user` の User flag 経由

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

`css/tnx.css` — 全 UI スタイル。フェーズ5 で構造リファクタ + サイバーパンクデザイン刷新予定。

HUD デザイン: 構造=V2 化 (フェーズ2)、見た目=フェーズ5

## 削除済み機能

- ホットバー自動折りたたみ (v13 コアから削除 — KI-012)
- ジャーナル2択作成メニュー → コア createEntry + 独自1ボタンに簡素化
