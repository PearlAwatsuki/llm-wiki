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

- 左上: TokenActionHUD 用に空け（フェーズ20・1.0.0 後）
- 右カラム: シーン/ニューロカード・情報項目・山札/捨て札（折りたたみ可）
- 下バー（2026-08-22 改修）: 上段に**参加者パネル**と**アクセスカードパネル**を横並び
  （各々アコーディオン開閉。設定 `hudParticipantsCollapsed` / `hudAccessCollapsed` で永続化）、
  下段に**ステータスパネル**（自分の状態の自動表示・14-7。手札の左隣・カードは手札/切り札と
  **同じ高さ 150px**＝幅はアクセスカード素材 179x249 の比のまま・アクセント強調）・手札・切り札

### 参加者パネル（2026-08-22 新設）

- 自分以外の**接続中**ユーザー（手札所持）を**常時**表示（`revealPlayerHands` に依存しない）。
  並びは受け渡しダイアログと同じ規則（PL 名前順 → GM 名前順）。
- 行 = 名前 + 左にステータス列（常設・アクセスカード意匠・手札のカードと**同じ高さ 69px**・
  アクセント強調。組み立ては自分のステータスパネルと共通の `TnxHud._buildUserStatusCards(user)`。
  **RL は常に RL（"RULER"+「RL」リボンの CSS カード・2026-08-22 意匠統一）**）+
  右に手札（幅は実枚数分・4枚を超えると**手札列の中で**折り返す=`max-width`）。
- プレイヤーのステータスは**登場三態を排他表示**（2026-08-22 裁定）:
  シーンプレイヤー（scene_player）／**登場中**＝登場していてシーンプレイヤーでない
  （専用画像が無いため "ON STAGE"+「登場中」リボンの CSS カード）／**舞台裏**（behind_the_scene）＝
  **文字通り舞台裏にいる＝登場していない**ことの表示（舞台裏枠の開始〜終了とは無関係）。
  ゴースト／抹殺は担当キャラクターの状態から追加表示。三態で基本埋まるが、
  万一空のときは破線の器で空を明示する（列は消さない）。
  行・パネルは実内容の幅で描き、幅の予約や flex 自動縮小をさせない
  （2026-08-22 是正: 固定幅予約+自動縮小で手札が見切れていた）。
- 手札の表裏: `revealPlayerHands` オフは全員裏面のみ・**RL の手札は設定によらず常に裏面**。
  裏面カードは title/alt にカード名を載せない（ツールチップからの内容漏れ防止）。
- 行はカードのドロップ先（そのユーザーへ渡す）。→ [Card_Operations](../Game_Mechanics/Card_Operations.md) の D&D 表
- 旧「プレイヤーの手札」パネル（右カラム・`revealPlayerHands` オン時のみ・行クリックで
  受け渡しダイアログ起動）はこのパネルに置換した。

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
