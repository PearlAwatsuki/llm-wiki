# TNX Foundry システム Wiki

トーキョーN◎VA THE AXLERATION 用 Foundry VTT システム (`tokyo-nova-axleration`) の開発向けリファレンス。

## この Wiki の目的

- FVTT システム開発時に必要な**実装情報**をトピック別に参照できるようにする
- 全体方針・ロードマップは `llm-wiki/02_System/` を参照

## クイックリファレンス

| 項目 | 値 |
|---|---|
| 対象 FVTT | v13.351 動作確認済 (`minimum: 11`) |
| 現在フェーズ・版数 | [../02_System/Roadmap.md](../02_System/Roadmap.md) が正本（本表は固定値を持たない） |
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
| [Actor_Types.md](Documents/Actor_Types.md) | Actor 型のフィールドと役割 |
| [Item_Types.md](Documents/Item_Types.md) | Item 型・共通 template |
| [Card_Types.md](Documents/Card_Types.md) | Card 型・Cards API 連携 |

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
| [Check_Rules.md](Game_Rules/Check_Rules.md) | 判定ルール（能力・技能・カード値・スート対応） |
| [Damage_Rules.md](Game_Rules/Damage_Rules.md) | ダメージ算出・適用（物理/精神/社会・チャート・戦闘不能・治療） |
| [Combat_Flow.md](Game_Rules/Combat_Flow.md) | カット進行（プロセス・アクション・AR・コンバットスピード） |
| [Troops.md](Game_Rules/Troops.md) | トループ・トループ級キャラクター（heads・エニグマポイント・分身・共通制約） |
| [Scenario_Progress.md](Game_Rules/Scenario_Progress.md) | アクト進行（アンダー/プレ/メイン/ポスト・シーン進行・舞台裏・経験点） |
| [Time_Management.md](Game_Rules/Time_Management.md) | 時間単位（メインプロセス/カット/シーン/アクト）・使用回数リセット・持続失効 |
| [Miracle_Rules.md](Game_Rules/Miracle_Rules.md) | 神業（構造・神業一覧・派生神業の参照対応） |
| [Appearance_Check.md](Game_Rules/Appearance_Check.md) | 登場判定（タイミング・使用技能・エリア別目標値・チーム免除・ゴースト） |
| [Purchase_and_Modification.md](Game_Rules/Purchase_and_Modification.md) | 購入判定・改造判定（信用・購入値・改造技能×製作技能・分類別改造項目） |
| [Bad_Status.md](Game_Rules/Bad_Status.md) | バッドステータス（状態異常の効果・回復・重複・強度。AE 設計の前提） |

### Game_Mechanics — ゲームロジック実装

| ファイル | 内容 |
|---|---|
| [Card_Operations.md](Game_Mechanics/Card_Operations.md) | 山札・手札・切り札・ニューロ操作 |
| [Experience_Points.md](Game_Mechanics/Experience_Points.md) | EXP 計算・同期・User 移行 |
| [Usage_System.md](Game_Mechanics/Usage_System.md) | usage テンプレート (判定基盤) |
| [Check_System.md](Game_Mechanics/Check_System.md) | 判定システム実装設計（UIフロー・データ構造） |
| [Active_Effects.md](Game_Mechanics/Active_Effects.md) | AE 適用モデル（派生値・モードA/B・棚卸し。フェーズ9 中核設計） |
| [Formula_Reference.md](Game_Mechanics/Formula_Reference.md) | 式で使えるキー一覧（`@system.*`／`@item.<識別キー>.system.*`／`@diff`/`@achievement`） |
| [Focus_System_Mechanics.md](Game_Mechanics/Focus_System_Mechanics.md) | FS判定の実装設計（設定ページ・正本＝ワールド設定・パネル・進行修正・判定要求プリセット） |

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
| [Phase_8_Tasks_Detail.md](Phases/Phase_8_Tasks_Detail.md) | フェーズ8 完了記録（判定システム 実装） |
| [Phase_9_Tasks_Detail.md](Phases/Phase_9_Tasks_Detail.md) | フェーズ9 完了記録（ActiveEffect / 派生値計算） |
| [Phase_10_Tasks_Detail.md](Phases/Phase_10_Tasks_Detail.md) | フェーズ10 完了記録（既存シート関連のやり残し総仕上げ） |
| [Phase_11_Tasks_Detail.md](Phases/Phase_11_Tasks_Detail.md) | フェーズ11 完了記録（guest / troop / extra シート） |
| [Phase_12_Tasks_Detail.md](Phases/Phase_12_Tasks_Detail.md) | フェーズ12 完了記録（攻撃判定 / ダメージ算出 / FS判定基幹） |
| [Phase_12_Implementation_Plan.md](Phases/Phase_12_Implementation_Plan.md) | フェーズ12 残作業の実装計画（T1〜T11・作業順序と依存） |
| [Phase_13_Tasks_Detail.md](Phases/Phase_13_Tasks_Detail.md) | フェーズ13 完了記録（戦闘システム＝カット進行） |
| [Phase_14_Tasks_Detail.md](Phases/Phase_14_Tasks_Detail.md) | フェーズ14 完了記録（セッション進行） |
| [Phase_15_Tasks_Detail.md](Phases/Phase_15_Tasks_Detail.md) | フェーズ15 計画（時間・使用回数管理・設計確定/実装未着手） |

## 関連

- 開発方針・ロードマップ → `../02_System/`
- Claude Code 向け指針 → プロジェクトルート `CLAUDE.md`
