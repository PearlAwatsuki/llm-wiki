# ドキュメントマップ

プロジェクト内ドキュメントの所在と役割。

## llm-wiki 構成

```
llm-wiki/
├── 01_Raw/          原本アーカイブ (docs/ のコピー)
├── 02_Wiki/         FVTT 開発向け Wiki (再構成)
│   ├── 00_Index.md
│   ├── Architecture/
│   ├── Documents/
│   ├── UI/
│   ├── Game_Mechanics/
│   ├── Migration/
│   ├── Issues/
│   └── Phases/
└── 03_System/       全体方針・運用 (本フォルダ)
    ├── 00_Index.md
    ├── Project_Overview.md
    ├── Development_Rules.md
    ├── Roadmap.md
    └── ...
```

## ルートファイル

| ファイル | 用途 | 更新頻度 |
|---|---|---|
| `CLAUDE.md` | Claude Code 向け指針。**セッション開始時に最初に読む** | フェーズ完了時 |
| `CHANGELOG.md` | リリース履歴 | リリース時 |
| `README.md` | プロジェクト説明 | 随時 |

## docs/ (作業原本)

開発中の作業記録。**01_Raw と同期を維持** (llm-wiki 更新時に docs/ も更新推奨)。

| ファイル | 種別 | Wiki 対応 |
|---|---|---|
| INVENTORY_REPORT.md | 棚卸し (静的) | 02_Wiki/Architecture/* |
| KNOWN_ISSUES.md | 既知課題 (生きた文書) | 02_Wiki/Issues/* |
| V13_MIGRATION_NOTES.md | v13 移行 | 02_Wiki/Migration/ |
| PHASE_0_TASKS.md | フェーズ0 タスク | 02_Wiki/Phases/ |
| PHASE_B_TASKS.md | フェーズB タスク | 02_Wiki/Phases/, Architecture/DataModel |
| PHASE_5_TASKS.md | フェーズ5 予定 | 03_System/Roadmap |
| PHASE_2_INVESTIGATION.md | フェーズ2 設計 | 02_Wiki/Phases/Phase_2_* |
| MECHANICS_AUDIT.md | 監査記録 | 03_System/Audit_and_Review |
| DESIGN_REVIEW.md | 設計レビュー | 03_System/Audit_and_Review, Architecture/DataModel |
| B4_INVESTIGATION.md | B-4 調査 | 02_Wiki/Game_Mechanics/Experience_Points |
| FUTURE_CONSIDERATIONS.md | 将来検討 | 03_System/Phase_Operations |
| RELEASE_CHECKLIST.md | リリース手順 | 03_System/Release_Process |

## 更新方針

| 変更種別 | 更新先 |
|---|---|
| 新 KI 発見 | docs/KNOWN_ISSUES + 02_Wiki/Issues/Active |
| フェーズ完了 | CLAUDE.md + docs/PHASE_* + 02_Wiki/Phases + 03_System/Roadmap |
| 設計決定 | docs/DESIGN_REVIEW + 該当 Wiki |
| ルール監査 | docs/MECHANICS_AUDIT |
| 原本変更 | docs/ → 01_Raw/ 再コピー |

## 03_System 目次

| ファイル | 内容 |
|---|---|
| [Project_Overview.md](Project_Overview.md) | プロジェクト概要・著作権 |
| [Development_Rules.md](Development_Rules.md) | 実装ルール・禁止事項 |
| [Roadmap.md](Roadmap.md) | 全フェーズロードマップ |
| [Phase_Operations.md](Phase_Operations.md) | フェーズ運用手順 |
| [Audit_and_Review.md](Audit_and_Review.md) | 監査・レビュー運用 |
| [Release_Process.md](Release_Process.md) | バージョン・リリース |
| [Roles_and_Workflow.md](Roles_and_Workflow.md) | 3者体制 |
| [Documentation_Map.md](Documentation_Map.md) | 本ファイル |

## 02_Wiki 目次

→ [../02_Wiki/00_Index.md](../02_Wiki/00_Index.md)
