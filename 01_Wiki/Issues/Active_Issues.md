# 未解消の既知課題

[Known_Issues_Detail.md](Known_Issues_Detail.md) から**現在有効**な項目のみ抽出。

## KI-039: キャストシートの「登場」列が住宅施設の実効値を読んでいない

- **起票**: 2026-08-09（フェーズ14-8 で住宅施設を舞台にする実装を入れた際に発見）
- **症状**: キャストシートのアウトフィット表の「登場」列が、住宅施設の**素値** `appearanceTarget` に住宅エリアの修正を足して表示する。AE の着地先である実効値 `appearanceTargetTotal` を見ていないため、**AE で登場判定目標値を上書き／加算しても列に反映されない**。
- **場所**: `scripts/actor/tnx-character-sheet-base.mjs` の `effectiveValues`（住宅施設の分岐）。同種の表示を行う `scripts/item/tnx-outfit-sheet.mjs` は `appearanceTargetTotal ?? appearanceTarget` の順で読んでおり、**2箇所で読み方が食い違っている**。
- **影響**: 表示のみ（判定側は 14-8 の `residence-area.mjs` で実効値を読むため正しい）。「表示は全箇所実効値」の規範に反する。
- **対応**: 未着手。14-8 の作業に無関係な不具合を混ぜないため分離した。修正は `tnx-outfit-sheet.mjs` と同じ読み方に揃えるだけ。

## 軽微な残存 (機能的支障なし)

- Font Awesome 5/6 混在 (v13 実機未確認)

## 更新ルール

新規発見時は本ファイルに KI-NNN として起票し、[Known_Issues_Detail.md](Known_Issues_Detail.md) に詳細を追記する。解消・分割済みは [Resolved_Issues.md](Resolved_Issues.md) へ移動する（KI の正本は Issues 一箇所のみ。他所での重複管理は廃止）。
