# 01_Raw — 原本アーカイブ

`docs/` フォルダ内ドキュメントの**原本コピー**。

## 用途

- 作業記録の一次情報 (編集は `docs/` 側で行い、必要に応じて再コピー)
- Wiki 再構成前の全文参照
- 詳細な調査レポート・設計レビュー全エントリ

## 収録ファイル

| ファイル | 内容 |
|---|---|
| INVENTORY_REPORT.md | 初期棚卸し (静的) |
| KNOWN_ISSUES.md | 既知課題 (KI-001〜) |
| V13_MIGRATION_NOTES.md | v13 移行観測 |
| PHASE_0_TASKS.md | フェーズ0 作業リスト |
| PHASE_B_TASKS.md | フェーズB 作業計画 |
| PHASE_5_TASKS.md | フェーズ5 予定 |
| PHASE_2_INVESTIGATION.md | フェーズ2 設計・進捗 |
| MECHANICS_AUDIT.md | メカニクス監査 (テンプレート) |
| DESIGN_REVIEW.md | 設計レビュー (B-0〜2-2 等) |
| B4_INVESTIGATION.md | B-4 事前調査 |
| FUTURE_CONSIDERATIONS.md | 将来検討 (不採用含む) |
| RELEASE_CHECKLIST.md | リリース手順 |

## 読みやすい版

要点を抽出・再構成した版は [../02_Wiki/00_Index.md](../02_Wiki/00_Index.md) を参照。

## 更新

`docs/` を更新したら、本フォルダへ再コピーして同期を保つ:

```powershell
Copy-Item -Path "docs\*" -Destination "llm-wiki\01_Raw\" -Force
```
