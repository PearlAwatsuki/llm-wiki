# TNX Foundry システム Wiki

トーキョーN◎VA THE AXLERATION 用 Foundry VTT システム (`tokyo-nova-axleration`) の開発向けリファレンス。

## この Wiki の目的

- FVTT システム開発時に必要な**実装情報**をトピック別に参照できるようにする
- 全体方針・ロードマップは `llm-wiki/02_System/` を参照

## クイックリファレンス

| 項目 | 値 |
|---|---|
| 対象 FVTT | v13.351 動作確認済 (`minimum: 11`) |
| 現バージョン | v0.3.0 (フェーズ2 完了時点) |
| 現在フェーズ | **フェーズ3** — 既存シート ApplicationV2 移行 |
| 判定方式 | カード (Cards API)。ダイスロールなし |
| データ定義 | DataModel 25 type。`template.json` 廃止済 |

## 目次

### Architecture — システム構造

| ファイル | 内容 |
|---|---|
| [System_Overview.md](Architecture/System_Overview.md) | エントリポイント・モジュール構成・技術スタック |
| [DataModel.md](Architecture/DataModel.md) | TypeDataModel・Mixin パターン・ファイル配置 |
| [Hooks_and_Modules.md](Architecture/Hooks_and_Modules.md) | 登録フック一覧・主要モジュール |

### Documents — ドキュメント型定義

| ファイル | 内容 |
|---|---|
| [Actor_Types.md](Documents/Actor_Types.md) | Actor 5 種のフィールドと役割 |
| [Item_Types.md](Documents/Item_Types.md) | Item 17 種・共通 template |
| [Card_Types.md](Documents/Card_Types.md) | Card 3 種・Cards API 連携 |

### UI — シートとインターフェース

| ファイル | 内容 |
|---|---|
| [Sheets.md](UI/Sheets.md) | シート一覧・ApplicationV1/V2 状況 |
| [HUD_and_Dialogs.md](UI/HUD_and_Dialogs.md) | TnxHud・ダイアログ・レコードシート |
| [Templates.md](UI/Templates.md) | Handlebars テンプレート構成 |

### Game_Mechanics — ゲームロジック実装

| ファイル | 内容 |
|---|---|
| [Card_Operations.md](Game_Mechanics/Card_Operations.md) | 山札・手札・切り札・ニューロ操作 |
| [Experience_Points.md](Game_Mechanics/Experience_Points.md) | EXP 計算・同期・User 移行 |
| [Usage_System.md](Game_Mechanics/Usage_System.md) | usage テンプレート (判定基盤) |

### Migration — 移行・互換

| ファイル | 内容 |
|---|---|
| [Foundry_v13.md](Migration/Foundry_v13.md) | v13 移行で判明した API 変更と対応状況 |

### Issues — 既知課題

| ファイル | 内容 |
|---|---|
| [Active_Issues.md](Issues/Active_Issues.md) | 未解消の既知課題 |
| [Resolved_Issues.md](Issues/Resolved_Issues.md) | 解消済み課題の要約 |

### Phases — フェーズ別作業

| ファイル | 内容 |
|---|---|
| [Completed_Phases.md](Phases/Completed_Phases.md) | フェーズ0/A/B/1 の完了内容 |
| [Phase_2_User_Migration.md](Phases/Phase_2_User_Migration.md) | 現在フェーズの設計・進捗 |

## 関連

- 開発方針・ロードマップ → `../02_System/`
- Claude Code 向け指針 → プロジェクトルート `CLAUDE.md`
