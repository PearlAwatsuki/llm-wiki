# 未解消の既知課題

[Known_Issues_Detail.md](Known_Issues_Detail.md) から**現在有効**な項目のみ抽出。



（KI-040〜KI-042 は解消、KI-043 は前提の誤認により撤回、KI-044〜KI-056 は解消 → [Resolved_Issues.md](Resolved_Issues.md)）

**2026-09-11 時点でコード上未解消の KI はない。** KI-057〜KI-068 は修正と自動検証を完了、Foundry のブラウザでの実機確認は未実施（→ [Resolved_Issues.md](Resolved_Issues.md)）。

## 2026-11-24 再監査

| ID | 優先度 | 未解消事項 |
|---|---|---|
| KI-071 | 高 | GitHub Actions の請求ロックによりジョブが起動しない |
| KI-072 | 高 | 複数クライアントの報酬点決済、GM委譲時の軽減記帳 |
| KI-073 | 高 | 複数対象へのダメージ適用が途中で失敗した場合の再開 |
| KI-074 | 中 | 入力後に資源が不足した用途の確定、複数資源の部分消費 |
| KI-075 | 中 | 派生ダメージのドロー中断後に、残りの処理だけを再開する手段 |

根拠と受入条件は [課題詳細](Known_Issues_Detail.md)。検証結果は [CI・ルール再監査](../../02_System/CI_and_Rules_Audit_2026-11-24.md)。

## 軽微な残存 (機能的支障なし)

- Font Awesome 5/6 混在 (v13 実機未確認)

## 更新ルール

新規発見時は本ファイルに KI-NNN として起票し、[Known_Issues_Detail.md](Known_Issues_Detail.md) に詳細を追記する。解消・分割済みは [Resolved_Issues.md](Resolved_Issues.md) へ移動する（KI の正本は Issues 一箇所のみ。他所での重複管理は廃止）。


2026-11-21: KI-069・KI-070は修正と自動検証を完了し、[Resolved_Issues.md](Resolved_Issues.md)へ移動。
