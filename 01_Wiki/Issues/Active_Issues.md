# 未解消の既知課題

[Known_Issues_Detail.md](Known_Issues_Detail.md) から**現在有効**な項目のみ抽出。

- **KI-038**: `system.uses.max` 宛の ActiveEffect が効かない（使用回数に実効値フィールドが無く、着地先 `uses.max.total` を誰も読まない）。神業の母数バフ（ファイト！等）が事実上不可。→ 使用回数の式対応で `uses.maxTotal` を新設し AE 着地点として登録する（2026-08-09 裁定・設計確定／実装未着手）。詳細は [Known_Issues_Detail.md](Known_Issues_Detail.md)

## 軽微な残存 (機能的支障なし)

- Font Awesome 5/6 混在 (v13 実機未確認)

## 更新ルール

新規発見時は本ファイルに KI-NNN として起票し、[Known_Issues_Detail.md](Known_Issues_Detail.md) に詳細を追記する。解消・分割済みは [Resolved_Issues.md](Resolved_Issues.md) へ移動する（KI の正本は Issues 一箇所のみ。他所での重複管理は廃止）。
