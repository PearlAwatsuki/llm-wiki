# 未解消の既知課題

[Known_Issues_Detail.md](Known_Issues_Detail.md) から**現在有効**な項目のみ抽出。

## 機能的支障のある未解消 KI

- **KI-043**: ニューロカードが既にドロー済みだと切り札の自動配布が例外を投げ、**アクト開始が途中適用のまま沈黙して中断する**（2026-08-27 起票・フェーズ14-8 の一括実機確認で検出）。報酬点/CS は適用済み・`actStarted` は false・境界イベント未発火。アクト終了がニューロデッキを回収しないため、2アクト目の開始でほぼ必ず踏む。是正方針は未裁定（→ [Known_Issues_Detail.md](Known_Issues_Detail.md) KI-043）。

## 軽微な残存 (機能的支障なし)

- Font Awesome 5/6 混在 (v13 実機未確認)

## 更新ルール

新規発見時は本ファイルに KI-NNN として起票し、[Known_Issues_Detail.md](Known_Issues_Detail.md) に詳細を追記する。解消・分割済みは [Resolved_Issues.md](Resolved_Issues.md) へ移動する（KI の正本は Issues 一箇所のみ。他所での重複管理は廃止）。
