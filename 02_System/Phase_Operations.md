# フェーズ運用

フェーズベース開発の進め方。

## フェーズの単位

1フェーズ = 1つの明確な目標 + 完了条件 + (通常) MINOR バージョンアップ

各フェーズはサブフェーズ (例: B-0〜B-10, 2-0〜2-5) に分割。

## 作業ファイル

| ファイル | フェーズ |
|---|---|
| `docs/PHASE_0_TASKS.md` | フェーズ0 |
| `docs/PHASE_B_TASKS.md` | フェーズB |
| `docs/PHASE_5_TASKS.md` | フェーズ5 (予定) |
| `docs/PHASE_2_INVESTIGATION.md` | フェーズ2 調査・設計 |

新フェーズ着手時に `PHASE_N_TASKS.md` または `PHASE_N_INVESTIGATION.md` を作成。

## 進行ルール

1. **1タスク = 1コミット**
2. タスク完了時に TASKS ファイルで `- [x]` チェック
3. 大きな変更前はプラン → 承認 → 実装
4. フェーズ完了時:
   - MECHANICS_AUDIT / DESIGN_REVIEW のエントリ最新化
   - CHANGELOG 更新
   - バージョンアップ + git tag
   - CLAUDE.md §2.2 更新

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
| 設計 (7, 9 等) | Opus チャット | ルール言語化、データ構造、TASKS/INVESTIGATION |
| 実装 (8, 10 等) | Claude Code | コード、テスト、DESIGN_REVIEW エントリ |

## 調査メモ

着手前の制約・トレードオフは `PHASE_N_INVESTIGATION.md` に記録。

例: フェーズ2 — User は DataModel 不可 → flag 運用確定

## 将来検討

確定前の項目は `docs/FUTURE_CONSIDERATIONS.md`。
確定したらロードマップに移し、FUTURE から削除。

## 関連

- [Release_Process.md](Release_Process.md)
- [Roadmap.md](Roadmap.md)
