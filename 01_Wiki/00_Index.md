# TNX Foundry システム Wiki

トーキョーN◎VA THE AXLERATION 用 Foundry VTT システム (`tokyo-nova-axleration`) の開発向けリファレンス。

## この Wiki の目的

- FVTT システム開発時に必要な**実装情報**をトピック別に参照できるようにする
- 全体方針・ロードマップは `llm-wiki/02_System/` を参照

## クイックリファレンス

| 項目 | 値 |
|---|---|
| 対象 FVTT | v13.351 動作確認済 (`minimum: 11`) |
| 現バージョン | v0.8.0（フェーズ8 部分完了） |
| 現在フェーズ | **フェーズ9** — ActiveEffect / 派生値計算（設計準備。2026-06-20 フェーズ再編） |
| 判定方式 | カード (Cards API)。ダイスロールなし |
| データ定義 | DataModel。`template.json` 廃止済 |

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
| [Sheets.md](UI/Sheets.md) | シート一覧・ApplicationV2 状況 |
| [HUD_and_Dialogs.md](UI/HUD_and_Dialogs.md) | TnxHud・ダイアログ・レコードシート |
| [Templates.md](UI/Templates.md) | Handlebars テンプレート構成 |

### Game_Rules — ゲームルール解釈

| ファイル | 内容 |
|---|---|
| [Terminology.md](Game_Rules/Terminology.md) | 用語定義・名称表記マーカー |
| [Character_Creation.md](Game_Rules/Character_Creation.md) | キャラクター作成・技能体系・スタイル・経験点・能力値clamp |
| [Bounty_Points.md](Game_Rules/Bounty_Points.md) | 報酬点ルール |
| [Outfits.md](Game_Rules/Outfits.md) | 装備品ルール・フィールド解釈 |
| [Judgment_Rules.md](Game_Rules/Judgment_Rules.md) | 判定ルール（能力・技能・カード値・スート対応） |
| [Damage_Rules.md](Game_Rules/Damage_Rules.md) | ダメージ算出・適用（物理/精神/社会・チャート・戦闘不能・治療） |
| [Combat_Flow.md](Game_Rules/Combat_Flow.md) | カット進行（プロセス・アクション・AR・コンバットスピード） |
| [Scenario_Progress.md](Game_Rules/Scenario_Progress.md) | アクト進行（アンダー/プレ/メイン/ポスト・シーン進行・舞台裏・経験点） |
| [Time_Management.md](Game_Rules/Time_Management.md) | 時間単位（メインプロセス/カット/シーン/アクト）・使用回数リセット・持続失効 |

### Game_Mechanics — ゲームロジック実装

| ファイル | 内容 |
|---|---|
| [Card_Operations.md](Game_Mechanics/Card_Operations.md) | 山札・手札・切り札・ニューロ操作 |
| [Experience_Points.md](Game_Mechanics/Experience_Points.md) | EXP 計算・同期・User 移行 |
| [Usage_System.md](Game_Mechanics/Usage_System.md) | usage テンプレート (判定基盤) |
| [Judgment_System.md](Game_Mechanics/Judgment_System.md) | 判定システム実装設計（UIフロー・データ構造） |

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
| [Completed_Phases.md](Phases/Completed_Phases.md) | フェーズ0/A/B/1/2 の完了内容 |
| [Phase_3_Tasks_Detail.md](Phases/Phase_3_Tasks_Detail.md) | フェーズ3 完了記録 |
| [Phase_4_Tasks_Detail.md](Phases/Phase_4_Tasks_Detail.md) | フェーズ4 完了記録 |
| [Phase_5_Tasks_Detail.md](Phases/Phase_5_Tasks_Detail.md) | フェーズ5 完了記録 |
| [Phase_6_Tasks_Detail.md](Phases/Phase_6_Tasks_Detail.md) | フェーズ6 完了記録 |
| [Phase_7_Tasks_Detail.md](Phases/Phase_7_Tasks_Detail.md) | フェーズ7 完了記録（判定システム 設計） |
| [Phase_8_Tasks_Detail.md](Phases/Phase_8_Tasks_Detail.md) | **現在フェーズ8** — 判定システム 実装 |

## 関連

- 開発方針・ロードマップ → `../02_System/`
- Claude Code 向け指針 → プロジェクトルート `CLAUDE.md`
