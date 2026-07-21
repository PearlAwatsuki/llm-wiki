# Phase 13: 戦闘システム（カット進行）

> 🚧 **計画確定・実装未着手（2026-07-21）**。本ページは設計/計画の記録。実装はユーザーの明示指示（サブフェーズごとの承認）で開始する。ルールの正本は [Combat_Flow.md](../Game_Rules/Combat_Flow.md)、時間境界は [Time_Management.md](../Game_Rules/Time_Management.md)。

FVTT デフォルトのコンバットトラッカーを上書きし、TNX の**カット進行**（戦闘準備→セットアップ→(イニシアチブ→メイン)×n→クリンナップ＝1カット、繰り返し）を実装する。判定・攻撃・ダメージ・リアクションの**実行機構**はフェーズ12 で完成済み。13 は「いつ・誰が」の**進行管理**と、CS/AR の自動記帳、トラッカー UI、境界イベントの発火、FS判定の実接続を担う。

## スコープ

**含む**

| # | 内容 | 出所 |
|---|---|---|
| 1 | `Combat`/`Combatant` サブクラス化・コンバットトラッカー上書き | Roadmap／Combat_Flow 申し送り |
| 2 | プロセス進行の状態機械（戦闘準備/セットアップ/イニシアチブ/メイン/クリンナップ） | Combat_Flow §2-6 |
| 3 | CSカレントのカット中自動管理（カット開始でCS実効値／セットアップ末で確定／メジャー後0／待機1） | Combat_Flow CS節・申し送り |
| 4 | AR 管理（カット開始で付与／メジャー−1／イニシアチブ行動不能−1／クリンナップ全回復） | Combat_Flow AR節 |
| 5 | プロセスのタイミング ↔ 用途 `timing` 値域の対応（プロセス別に出せる行動） | Combat_Flow 申し送り |
| 6 | リアクションの宣言タイミング・回数の進行管理（トラッカー上の割り込み制御） | Combat_Flow §5c・申し送り |
| 7 | カット境界／カット進行（＝シーン）境界イベントの発火 | Time_Management 申し送り |
| 8 | FS判定の実接続（下記「FS実接続」） | Focus_System_Mechanics 申し送り |

**含まない（申し送り）**

- 効果の失効・使用回数リセットの**適用本体**（Times-Up 的持続管理）→ フェーズ15（13 が発火する境界イベントを購読する側）
- シーン管理の一般化（非戦闘シーン・シーン切替の抽象化・バックエンド差し替え）→ フェーズ14
- 行動回数制限の強制（メジャー1回/マイナー1回等）→ **恒久的に非スコープ**（Time_Management 確定・RL 裁定）
- タクティクス（マップ）戦闘 → 選択ルールとして後日
- 登場判定（セットアッププロセス）→ フェーズ18

## 確定した設計判断（2026-07-21 ユーザー承認）

1. **自動化の度合い＝RL駆動＋自動記帳＋手番提示**: プロセス遷移は RL の操作で進める。システムは CS/AR の記帳を自動化し、「次の手番＝CSカレント最大かつ AR≥1」を算出してハイライトするが、割り当て・カット終了・行動可否は強制しない。「表示・補助に留め強制しない」方針・Time_Management「行動回数は自動化しない」と一致（案A採用・案B自動進行/案C純表示は不採用）。
2. **カット終了の境界**: クリンナップ完了→次カットのセットアップ開始の遷移（＝ラウンド遷移）で前カットが終了。効果持続・使用回数のカット境界（15）もこの遷移を基準にする。→ [Combat_Flow.md](../Game_Rules/Combat_Flow.md)「カット境界とシーンの関係」。
3. **カット進行終了とシーン終了は強連動させない（案1）**: カット進行を終える操作で確認ダイアログ「このシーンも終了しますか？」を出す。「終了する」→ シーン終了イベント発火。「シーンは継続」→ トラッカーは閉じるがシーンは開いたまま（治療などをシーン内で認める運用に対応）。継続時のシーン終了はシーンコントロール側の明示導線。開始側は combat開始＝シーン開始（案2完全独立/案3設定切替は不採用・必要になれば案3を後から追加可）。
4. **トラッカーUI＝サイドバー統合**: Foundry `CombatTracker` をサブクラス化して再テンプレート化。加えて**現在のプロセス・手番を常に見える表示**（トラッカーヘッダー or 常駐HUD側の小表示・形は13-4で確定）。対抗＝独自 Application は不採用。
5. **Foundry モデルへの写像**: 1 Foundry Combat ＝ 1 カット進行 ＝ 1（戦闘）シーン。カット＝round 相当。手番/プロセスは `TnxCombat` 側で独自管理し、round/turn は参照用に整合させる（詳細は13-2/13-3で確定）。
6. **13/14 の分担**: 13 は「戦闘シーン1種」を実装、14 が非戦闘シーン・シーン切替を一般化。シーン連動を14へ丸投げしない。

## 既存資産（実装済み・流用）

- CS 3層 `system.combatSpeed.{value,base,current,freeMod}`／AR `system.actionRank.{value,freeMod}`（付与定数 `ACTION_RANK_GRANT=1`）＝`scripts/data/helpers.mjs`。派生 totals・`inCombat`・`resolveCombatSpeedDisplayTotal` あり。
- `timing` enum（processes: setup/initiative/clean-up、actions: move/minor/major/reaction/auto 他）＝`scripts/module/tnx-skill-utils.mjs`。戦闘タブのタイミング別行動表示＝`templates/actor/parts/tab-combat.hbs` + `scripts/actor/tnx-character-sheet-base.mjs`。
- AE 仮想名前空間 `system.cs.*`/`system.ar.max`、AR 消費機構（`consumeTargets` type=actionRank）＝`scripts/module/usage-consumption.mjs`。
- 純ロジック `reaction-logic.mjs`／`vehicle-move.mjs`（フェーズ12）。
- **カット開始シード** `seedCombatStartValues`（`scripts/tnx.mjs`・`combatStart`/`createCombatant` フック。per-process/turn/cleanup は13送りと明記）→ 13-2 で `TnxCombat` へ統合。

## サブフェーズ

### 13-1: 設計確定・記録（2026-07-21 ✅）

本計画のレビュー確定・llm-wiki 記録（本ページ・Combat_Flow「カット境界とシーンの関係」）。設計レビュー/メカニクス監査の起票は各実装サブフェーズで行う。

### 13-2: `TnxCombat`/`TnxCombatant` サブクラス＋CONFIG登録＋シード集約（2026-07-21 ✅ 実装済・実機確認は13-4以降）

カット進行の「器」＝クラス新設・`CONFIG.Combat/Combatant.documentClass` 登録・カット開始シードのロジック集約・純ロジック（手番順/シード算出）。プロセス自動遷移・UI・CS/AR 自動記帳は載せない。

- 純ロジック（TDD）: `combat-turn-order.mjs`（`resolveTurnOrder`＝CSカレント降順＋同値時①CSベース②種別③ユーザー順・`nextActiveMain`＝CSカレント最大かつAR≥1）／`combat-seed-logic.mjs`（`buildCombatSeedUpdate`）。
- **エキストラは手番を持たない**（2026-07-21 ユーザー裁定）＝手番順・次の手番から種別で除外（→ Combat_Flow §3）。当初「原文に無いので最下位に仮置き」は誤りとして是正。
- サブクラス: `scripts/combat/tnx-combat.mjs`（flag 状態 getter＋シード static）・`tnx-combatant.mjs`（最小）。`scripts/tnx.mjs` で登録＋シードのフックを static へ委譲（挙動不変）。`eslint.config.js` に `Combat`/`Combatant` を追加。
- **シード統合の範囲を一部 13-3 へ**: ロジックは `TnxCombat` へ集約したが、トリガーは既存フックのまま（挙動不変・ゼロ回帰）。ライフサイクル（`startCombat`/`_onEnter`）への完全移行は CS/AR 自動記帳とまとめて 13-3。
- 検証: テスト 1682 green・eslint 0 error。実機確認はトラッカー UI が入る 13-4 以降（ユーザー了承）。

### 13-3: プロセス状態機械＋CS/AR自動記帳

セットアップ/イニシアチブ/メイン/クリンナップの遷移。CSカレント（カット開始でCS実効値／セットアップ末確定／メジャー後0／待機1）と AR（メジャー−1／行動不能−1／クリンナップ全回復）の自動記帳。

### 13-4: トラッカーUI上書き

`CombatTracker` サブクラス再テンプレート化。プロセス・手番ハイライト（CSカレント最大かつAR≥1）・CS/AR・timing別行動起動・現在プロセス表示。

### 13-5: リアクションの宣言タイミング・回数のトラッカー統合

フェーズ12 の実行機構（reaction-logic 等）を進行管理へ接続。メジャーへの割り込み制御・宣言タイミング・回数。

### 13-6: カット境界／カット進行（＝シーン）境界イベントの発火

プロセス遷移イベント・カット境界イベント（クリンナップ完了→次セットアップ）・カット進行（＝シーン）開始/終了イベント。カット進行終了時のシーン終了は案1の確認ダイアログ。適用（失効・回復・リセット）はフェーズ15 が購読。

### 13-7: FS判定の実接続

下記「FS実接続」。

### 13-8: 仕上げ・実機確認・v0.13.0

記録の最新化・実機確認・版数同期（タグ付け・push はユーザー）。

> 各実装サブフェーズは、着手前にチャットで詳細仕様を提示→ユーザー承認→実装、の順で進める。

## FS実接続（13-7・[Focus_System_Mechanics.md](../Game_Mechanics/Focus_System_Mechanics.md) 申し送りより）

- 進行判定の実行 → `floor(@diff/10)+進行修正` の自動加算
- 支援判定の AR 消費と、対象の次の進行判定への +1
- 支援判定要求の宛先を AR 残量で絞り込み
- 進行判定要求の宛先を「今のメインプロセスのキャスト」で自動判別（12 は RL 手動選択）
- `progressMod.source==="outfit"` の実解決（判定者の準備済みアイテムから種別を探し、複数なら選択）
- コンバットトラッカーのカット進行と `cut` の連動

## 完了条件（暫定）

- [ ] Combat 開始＝カット進行開始でトラッカーが TNX 表示になり、プロセス（セットアップ〜クリンナップ）を RL 操作で進められる
- [ ] CSカレント・AR がプロセス遷移で自動記帳される（カット開始/セットアップ末/メジャー後/待機/行動不能/クリンナップ全回復）
- [ ] プロセス別に出せる行動（timing）がトラッカーから起動できる
- [ ] リアクションの宣言タイミング・回数が進行管理される
- [ ] カット境界／カット進行（＝シーン）境界イベントが発火する（適用は15）・カット進行終了時のシーン終了は案1の確認ダイアログ
- [ ] FS実接続（進行加算・支援AR消費+1・AR絞り込み・宛先自動判別・outfit進行修正・cut連動）
- [ ] 版数 0.13.0 を同期（タグ付け・push はユーザー）
- [ ] 実機確認

## 申し送り（後続フェーズ）

- 効果失効・使用回数リセットの適用（本境界イベントの購読）→ フェーズ15
- シーン管理の一般化 → フェーズ14
- タクティクス（マップ）戦闘 → 後日（選択ルール）

## 関連

- [Roadmap.md](../../02_System/Roadmap.md)
- [Combat_Flow.md](../Game_Rules/Combat_Flow.md)（ルール正本）
- [Time_Management.md](../Game_Rules/Time_Management.md)
- [Focus_System_Mechanics.md](../Game_Mechanics/Focus_System_Mechanics.md)
- [Phase_12_Tasks_Detail.md](Phase_12_Tasks_Detail.md)（前フェーズ）
