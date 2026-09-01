# 未解消の既知課題

[Known_Issues_Detail.md](Known_Issues_Detail.md) から**現在有効**な項目のみ抽出。

- **KI-047**: 複数対象攻撃で、対象依存のダメージ修正（対象条件・`@target.*` の式・`damage.vs*` AE）が**先頭の命中対象**の条件で全対象に適用される。単体対象では正しい。用途トグル「ウェットの対象には効果がない」は対象ごとに評価するため影響なし。**設計変更を要するためユーザー裁定待ち**（2026-09-01 起票・詳細は [Known_Issues_Detail.md](Known_Issues_Detail.md)）。

（KI-040〜KI-042 は解消、KI-043 は前提の誤認により撤回、KI-044〜KI-046 は解消 → [Resolved_Issues.md](Resolved_Issues.md)）

## 軽微な残存 (機能的支障なし)

- Font Awesome 5/6 混在 (v13 実機未確認)

## 更新ルール

新規発見時は本ファイルに KI-NNN として起票し、[Known_Issues_Detail.md](Known_Issues_Detail.md) に詳細を追記する。解消・分割済みは [Resolved_Issues.md](Resolved_Issues.md) へ移動する（KI の正本は Issues 一箇所のみ。他所での重複管理は廃止）。
