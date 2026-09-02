# 未解消の既知課題

[Known_Issues_Detail.md](Known_Issues_Detail.md) から**現在有効**な項目のみ抽出。

## 未解消

- **KI-049**: 複数アイテムを一括で追加すると、アイテム狙い AE の転送コピーが個数分だけ多重に作られる（値が黙って多重に乗る。辞典からの複数ドラッグ・アクターの複製で起こる）
- **KI-050**: 新規に追加したアイテム自身が持つアイテム狙い AE が、その時点で転送されない（別操作が起きるまで効果が乗らない）

いずれも 2026-09-02 起票。原因はどちらも転送の実体化（`materializeItemTransfers` と `createItem` フック）にあるが、別の原因である。詳細は [Known_Issues_Detail.md](Known_Issues_Detail.md)。

（KI-040〜KI-042 は解消、KI-043 は前提の誤認により撤回、KI-044〜KI-048 は解消 → [Resolved_Issues.md](Resolved_Issues.md)）

## 軽微な残存 (機能的支障なし)

- Font Awesome 5/6 混在 (v13 実機未確認)

## 更新ルール

新規発見時は本ファイルに KI-NNN として起票し、[Known_Issues_Detail.md](Known_Issues_Detail.md) に詳細を追記する。解消・分割済みは [Resolved_Issues.md](Resolved_Issues.md) へ移動する（KI の正本は Issues 一箇所のみ。他所での重複管理は廃止）。
