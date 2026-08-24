# 未解消の既知課題

[Known_Issues_Detail.md](Known_Issues_Detail.md) から**現在有効**な項目のみ抽出。

## KI-042: activeGM 不在だと情報収集判定の自動開示が消失し、結果カードは「情報を開示した」と表示する

- **起票**: 2026-08-25（「開示結果が HUD・ツールチップに出ない」報告の隔離実機調査で確定）
- **症状**: GM クライアント未接続で PL が情報収集判定に成功すると、判定は完走し結果カードに「情報を開示した」と出るが、開示の正本（アクトジャーナルの infoItems）は開かず、HUD チップ・ツールチップにも反映されない。
- **原因**: 開示の書き込みは GM 所有ジャーナルのため `emitInfoDisclose` で activeGM に委譲する（info-gathering.mjs）。activeGM 不在では受信者がおらず開示は静かに消失。結果カードの帰結行は `result.success` のみから組まれ（tnx-check-flow.mjs `_renderResultContent` の infoCheck）、実適用と無関係＝虚偽表示になる。
- **詳細**: [Known_Issues_Detail.md](Known_Issues_Detail.md) KI-042

## 軽微な残存 (機能的支障なし)

- Font Awesome 5/6 混在 (v13 実機未確認)

## 更新ルール

新規発見時は本ファイルに KI-NNN として起票し、[Known_Issues_Detail.md](Known_Issues_Detail.md) に詳細を追記する。解消・分割済みは [Resolved_Issues.md](Resolved_Issues.md) へ移動する（KI の正本は Issues 一箇所のみ。他所での重複管理は廃止）。
