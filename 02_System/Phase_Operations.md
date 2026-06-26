# フェーズ運用

フェーズベース開発の進め方。

## フェーズの単位

1フェーズ = 1つの明確な目標 + 完了条件 + (通常) MINOR バージョンアップ

各フェーズはサブフェーズ (例: B-0〜B-10, 2-0〜2-5) に分割。

## 作業ファイル

フェーズ別の作業・調査記録は **`01_Wiki/Phases/`** に置く（旧 `docs/PHASE_*_TASKS.md` は廃止）。

| ファイル | 内容 |
|---|---|
| `01_Wiki/Phases/Phase_N_Tasks_Detail.md` | 各フェーズの作業記録（サブフェーズ・完了条件） |
| `01_Wiki/Phases/Completed_Phases.md` | 初期フェーズ（0/A/B/1/2）の概要 |

新フェーズ着手時に `01_Wiki/Phases/Phase_N_Tasks_Detail.md` を作成する。

## 進行ルール

1. **1タスク = 1コミット**
2. タスク完了時に TASKS ファイルで `- [x]` チェック
3. 大きな変更前はプラン → 承認 → 実装
4. フェーズ完了時:
   - メカニクス監査・設計レビューのエントリ最新化（`02_System/Audit_and_Review.md` / `Design_Review_Entries.md`）
   - CHANGELOG 更新
   - バージョンアップ + git tag
   - `02_System/Roadmap.md` の進行状況を更新（CLAUDE.md はフェーズを保持しない）

## サブフェーズ設計原則

- **可逆を先、不可逆を最後** (例: 2-5 player 廃止)
- 中途半端な状態でコミットしない (1 サブフェーズ内で完結)
- 不可逆操作の直前でユーザー報告

## フェーズスコープの厳守

各 PHASE_* ファイルに「やってはいけないこと」セクションあり。

例 (フェーズB):
- 既存シートの構造書き換え → フェーズ3
- EXP 計算の意味的変更 → 禁止
- データマイグレーション機構の先回り → 禁止

## 設計フェーズ vs 実装フェーズ

| 種別 | 主体 | 成果物 |
|---|---|---|
| 設計 (7, 9 等) | Claude Code | ルール言語化、データ構造設計、調査記録 |
| 実装 (8, 10 等) | Claude Code | コード、テスト、設計レビューエントリ |

## 調査メモ

着手前の制約・トレードオフは該当フェーズの `Phase_N_Tasks_Detail.md`（または調査ページ）に記録。

例: フェーズ2 — User は DataModel 不可 → flag 運用確定

## 将来検討

確定前の項目は Roadmap の再編メモ等に記し、確定したらロードマップ本体へ移す（旧 `docs/FUTURE_CONSIDERATIONS.md` は廃止）。

## 関連

- [Release_Process.md](Release_Process.md)
- [Roadmap.md](Roadmap.md)
