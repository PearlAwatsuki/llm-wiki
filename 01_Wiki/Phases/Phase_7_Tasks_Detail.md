# Phase 7: 判定システム 設計

## 目的

TNX の中核メカニクスである**判定システム**の設計を確定させる。
実装はフェーズ8 で行う。本フェーズは「ルールの言語化 → データ構造設計 → ユーザー承認」のサイクルで完結する。

CLAUDE.md §2.2 の原則: 「設計フェーズではルールの言語化・データ構造設計を先に確定させ、ユーザーの承認を得てから実装フェーズに入る。」

## スコープ外

- コードの実装（フェーズ8）
- 神業システム（フェーズ9-10）
- 戦闘シート UI の実質内容（フェーズ11）

## 前提知識

TNX の判定はすべて **Foundry Cards API** 経由のカードドロー。ダイスロール (`Roll`) は使用しない。

既存実装（Card_Operations.md 参照）:
- `checkFromDeck()`: 山札→捨て札へ 1 枚表向き移動 + チャット通知（達成値計算なし）
- `UsageCreationDialog`: 用途起動ダイアログ（フェーズ7 の設計対象）
- `usage.mjs` template: `actions[]`（type / name / description）がデータ基盤

未解決: KI-002（`usage-list.hbs` の edit ボタン未接続）

## サブフェーズ

### 7-1: 判定ルールの言語化 ✅ 確定（2026-06-14）

確定内容は [Judgment_System.md](../Game_Mechanics/Judgment_System.md) を正本とする。

#### 確定済み事項

- [x] 判定の種類: 能力判定 / 技能判定
- [x] 達成値計算式（通常: カード値 + 能力値実効値 + 報酬点）
- [x] A の特殊処理（11 or 達成値21完全固定）
- [x] Joker（スート・値とも自由宣言）
- [x] 切り札（手札1枚を Joker 化、消費後シーンカードへ）
- [x] 技能レベルの役割（使用可能スート数・発動条件のみ、達成値加算なし）
- [x] 報酬点（1点=+1達成値、上限なし）
- [x] スート↔能力値マッピング（♠=reason/♣=passion/♥=life/♦=mundane）
- [x] 能力値実効値の計算式（`totalValue` = growth + styleTotalValue + mod + effectMod + abilityOutfitMod）
- [x] 山札ファンブル（絵札→致命的失敗）
- [x] 誤スート使用（自動失敗、UI では拒否しない）
- [x] UI フロー（チャット駆動型。SW2.5 / Monk's Token Bar パターン踏襲）
- [x] RL 主導判定要求のフロー（左サイドバーボタン → チャットカード → ボタンで応答）
- [x] socket は KI-006 を**フェーズ8 で解決**（判定システムと同時実装）

#### 残確定事項（全解決 2026-06-14）

- [x] RL 判定要求は複数プレイヤーへ同時送信可能（選択制、Monk's Token Bar 方式）
- [x] 情報収集項目: JournalEntry flags の infoItems（実装済み）。TN は `skills[].tn`
- [x] チャット結果の公開範囲: 秘匿ルールなし。RL 要求ダイアログに roll mode 選択肢を設ける
- [x] 能力判定トリガー: キャストシートの能力値数値クリック（RL 主導が基本のため稀。一応残す）
- [x] PL による TN 手入力フローは採用しない（データ駆動設計。能力値クリック時のみ例外として許容）
- [x] 購入判定: 辞典ブラウザ実装後に接続（フェーズ8 スコープ外）
- [x] 登場判定: アクトシートにシーン設定フィールドが必要。フェーズ11 候補

### 7-2: データ構造設計 ✅ 確定（2026-06-15）

- [x] `usage.mjs` の `actions[]` に `skillRefs: [{ itemId }]` を追加（組み合わせ技能）
- [x] `actions[].type` に `"controlCheck"` / `"attack"` を追加。リアクションは timing 駆動（D&D 方式）
- [x] チャットカードフラグ設計確定（`tokyo-nova-axleration` スコープ、Monk's Token Bar 方式）
- [x] 達成値計算アルゴリズム確定（カード値 → スート → 実効値 + 報酬点）
- [x] 制御判定（下方判定）: 出したカードのスート対応制御値が基準
- [x] KI-002: usage edit ボタン → UsageCreationDialog（skillRefs ドロップダウン含む）
- [x] KI-006: socket 実装をフェーズ8 で同時実施
- [x] AE ボーナス集計はフェーズ8 スコープ外（ActiveEffect Phase 後）
- [x] チャット表示: 技能名を「+」連結・制御判定は「達成値 ≤ 制御値 ▶ 成功」
- [x] 詳細: [Judgment_System.md](../Game_Mechanics/Judgment_System.md)

### 7-3: 設計書ドラフトとユーザー承認 ✅ 完了（2026-06-15）

- [x] 設計書: [Judgment_System.md](../Game_Mechanics/Judgment_System.md)（実装設計）
- [x] ルール正本: [Judgment_Rules.md](../Game_Rules/Judgment_Rules.md)
- [x] フェーズ8 実装計画: [Phase_8_Tasks_Detail.md](Phase_8_Tasks_Detail.md)
- [ ] メカニクス監査・設計レビューエントリ → フェーズ8 実装後に記録

## 完了条件

- 判定システムの全設計判断がユーザー確認済み
- 実装仕様（データ構造・API 設計・UI フロー）が文書化済み
- メカニクス監査・設計レビューエントリが記録済み
- フェーズ8 で Code が迷わず実装できる状態

## 参照

- [Roadmap.md](../../02_System/Roadmap.md)
- [Card_Operations.md](../Game_Mechanics/Card_Operations.md)
- [Usage_System.md](../Game_Mechanics/Usage_System.md)
- [Active_Issues.md](../Issues/Active_Issues.md) — KI-002
- フェーズ6 完了記録 → [Phase_6_Tasks_Detail.md](Phase_6_Tasks_Detail.md)
