# 未解消の既知課題

[Known_Issues_Detail.md](Known_Issues_Detail.md) から**現在有効**な項目のみ抽出。



（KI-040〜KI-042 は解消、KI-043 は前提の誤認により撤回、KI-044〜KI-056 は解消 → [Resolved_Issues.md](Resolved_Issues.md)）

**2026-09-11 時点でコード上未解消の KI はない。** KI-057〜KI-068 は修正と自動検証を完了、Foundry のブラウザでの実機確認は未実施（→ [Resolved_Issues.md](Resolved_Issues.md)）。

## 軽微な残存 (機能的支障なし)

- Font Awesome 5/6 混在 (v13 実機未確認)

## 更新ルール

新規発見時は本ファイルに KI-NNN として起票し、[Known_Issues_Detail.md](Known_Issues_Detail.md) に詳細を追記する。解消・分割済みは [Resolved_Issues.md](Resolved_Issues.md) へ移動する（KI の正本は Issues 一箇所のみ。他所での重複管理は廃止）。


## KI-069: 通常判定ダイアログの追加技能から数値修正を再導出しない

2026-11-21、本体44fcc5cで再現。`TnxCheckFlow._rebuildDialogSkillContext` の数値再導出が `ctx.isRecheck` 内にあるため、通常判定では追加した技能の判定・攻撃ダメージ修正が入らない。参加技能自体は更新される。`skill-numeric-provision-review.test.mjs` の期待失敗2件で再現。今回の第一段階は調査のみで未修正。再導出と再判定専用の消費処理を分離し、入口の追加修正も保全する案。参照: [再検証報告](../Game_Mechanics/Skill_Numeric_Provision_Review.md)。

## KI-070: 部分フォーム送信で用途固有の受け値修正行が失われる

2026-11-21、本体44fcc5cで再現。`submitUsageForm` はreaction用途で `collectBonusRows(raw, "guardBonus")` を無条件に代入するため、guard行を含まない部分送信で保存済み `guardBonuses` が空になる。通常の全フォーム送信で常に発生するという意味ではない。`skill-numeric-provision-review.test.mjs` の期待失敗1件で再現。第一段階では未修正。入力欠落と意図的な全行削除を区別して保全する案。参照: [再検証報告](../Game_Mechanics/Skill_Numeric_Provision_Review.md)。
