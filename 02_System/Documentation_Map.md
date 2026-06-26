# ドキュメントマップ

プロジェクト内ドキュメントの所在と役割。

## llm-wiki 構成

```
llm-wiki/
├── 01_Wiki/         FVTT 開発向け Wiki
│   ├── 00_Index.md
│   ├── Architecture/
│   ├── Documents/
│   ├── UI/
│   ├── Game_Rules/      ルール解釈の正本
│   ├── Game_Mechanics/
│   ├── Migration/
│   ├── Issues/
│   └── Phases/
└── 02_System/       全体方針・運用 (本フォルダ)
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

## 更新方針

llm-wiki が運用の正本。更新は該当 Wiki ページを直接編集する。

| 変更種別 | 更新先 |
|---|---|
| 新 KI 発見 | 01_Wiki/Issues/Active（正本は Issues 一箇所・§6.3） |
| フェーズ完了 | 01_Wiki/Phases + 02_System/Roadmap（CLAUDE.md はフェーズを保持しない） |
| 設計決定 | 02_System/Audit_and_Review + 該当 Wiki |
| ルール監査 | 02_System/Audit_and_Review |

## 02_System 目次

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

## 01_Wiki 目次

→ [../01_Wiki/00_Index.md](../01_Wiki/00_Index.md)
