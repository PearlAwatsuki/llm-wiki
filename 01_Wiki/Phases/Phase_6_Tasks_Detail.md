# Phase 6: 未実装シート追加

## 目的

シート未実装のドキュメント型（outfit 系 Item 11 種 + lifePath、guest / troop / extra Actor 3 種）に
ApplicationV2 シートを新規作成する。フェーズ3 の CSS 構造（tnx2.css 変数体系）とフェーズ4 のデザインを踏襲する。
あわせてキャストシート「アウトフィット」タブを整備する。

現状、`tnx.mjs` でコアの ActorSheet / ItemSheet を unregister しているため、
これらの型は**シートを開くことすらできない**状態にある。

## スコープ外

- 判定システムの設計・実装（フェーズ7-8）。用途タブの完成は判定システム依存
- 神業システム（フェーズ9-10）
- ActiveEffect パイプライン整備（ActiveEffect フェーズ。本フェーズで対象フィールドが固まる）
- KI-010（combatSpeed UI）の配置先確定（フェーズ11 までに決める）
- シナリオ作成ウィザード再設計（フェーズ11）

## 設計決定事項（2026-06-12 ユーザー確認済み）

- **cast の `player_name` は削除**（B-4 申し送りのデッドフィールド判断に決着）
- **`identificationKey` は全アイテム型に追加**（フェーズ5-5 では実装済み 5 種のみだった）
- **ライフパスは出自・経験・邂逅の 3 種のみ**。life-path.mjs 旧コメントの「運命」は誤り
- **キャスト詳細タブのライフパススロットは型と対応させる**（出自スロットには lifePathType=origin のみ）
- **ライフパスの概要は参照先アイテムの description を表示**（アイテム側に summary フィールドは追加しない）
- **参照表示は毎回 fromUuid で現物解決**。キャッシュ名はアイテム削除時のフォールバック表示のみ。
  ドロップ時の名前を保持し続ける方式は今後一切採用しない（全アイテム共通の恒久方針）
- **ゲスト = キャストからライフパス・報酬点・セッション履歴（EXP 含む）を除いたもの**
- **トループ**: スタイル・能力値・技能・アウトフィット・状態を持ち、神業なし。
  `heads`（人数）が HP のように機能し、トークンのリソースバーに割り当てる
- **エキストラ**: 基本は名前のみだが、スタイル・神業・技能・アウトフィットを持てる
- **装備品共通フィールドのルール解釈**は [Outfits.md](../Game_Rules/Outfits.md) を正本とする
- **majorCategory / minorCategory は選択リスト化**。シートでは大分類→小分類の連動ドロップダウンとし、
  その Item type に有効な小分類のみを提示する

## サブフェーズ

### 6-0: DataModel 変更・設計記録 ▶ 進行中

- [ ] cast: `player_name` 削除（テスト更新含む）
- [ ] 未実装 12 種（weapon / armor / ianus / cyborg / tron / tap / vehicle / residence / housingArea / combiner / general / lifePath）に `identificationKey` 追加
- [ ] outfitBase: `isCarrying`（Boolean）追加
- [ ] outfitBase: `buy` / `hide` を `{mode, value}` SchemaField 化（mode = none / value / reference）
- [ ] outfitBase: `preserveExp` / `appearancePenalty` を NumberField 化
- [ ] outfitBase: `majorCategory` / `minorCategory` を choices 付き StringField 化（カテゴリ定数モジュール新設）
- [ ] troop: `heads`（`{value, max}` SchemaField）追加
- [ ] lifePath: `lifePathType` を choices 付き（origin / experience / encounter）に変更 + 「運命」誤コメント修正
- [ ] wiki: Game_Rules/Outfits.md 新設、監査・設計レビューエントリ追加

### 6-1: 装備品シート共通基盤 + general

- [ ] outfitBase 部分の共通レイアウト（カテゴリ連動ドロップダウン、購/隠/危険値表示、部位、設定タブ）
- [ ] extensible のスロット追加/削除 UI
- [ ] general シートで共通基盤を確立

### 6-2: 戦闘系装備シート（weapon / armor / cyborg）

- [ ] 着手時に固有フィールド（attack 構造・defence・FAValue 等）のルール解釈を確認

### 6-3: 電脳系装備シート（ianus / tron / tap）

- [ ] 着手時に固有フィールド（controlMod / cycle / combatSpeedMod 等）のルール解釈を確認

### 6-4: 生活系装備シート（vehicle / residence / housingArea / combiner）

- [ ] 着手時に固有フィールド（speedFactor / セキュリティ値 / Mod 群 / combinedOutfitID 等)のルール解釈を確認

### 6-5: lifePath シート + キャスト詳細タブ連携

- [ ] lifePath シート（lifePathType ドロップダウン / skillName / description）
- [ ] キャスト詳細タブ: 型対応ドロップ検証・fromUuid live 解決・description の概要表示

### 6-6: キャストシート「アウトフィット」タブ整備

### 6-7: guest シート

- [ ] キャストシートの共通部品化（キャストの挙動は変えない範囲）

### 6-8: troop シート

- [ ] heads のトークンリソースバー連携

### 6-9: extra シート

### 6-10: 仕上げ

- [ ] wiki 整合化（Sheets.md / Actor_Types.md / Item_Types.md の V1 時代記述の更新）
- [ ] 監査・レビュー全エントリ通読・最新化
- [ ] CHANGELOG 更新・v0.7.0 バージョンアップ

## 実装サブフェーズ着手時に確認する残課題

- `hack`（電脳制御値）の形式: なし/数値の 2 状態でよいか
- `timing` の選択肢: スタイル技能の timing と共通リストか
- `exclusive`（スタイル/ワークス専用）の入力形式: 自由記述か
- 各種別の固有フィールドの意味と UI 形式（各サブフェーズ冒頭で確認）

## 完了条件

- 全ドキュメント型がカスタムシートを持ち、フェーズ4 デザインで表示・編集できる
- キャストシートにアウトフィットタブがあり、装備品の管理ができる
- v0.7.0 タグ

## 参照

- [Roadmap.md](../../02_System/Roadmap.md)
- [Outfits.md](../Game_Rules/Outfits.md)
- [Item_Types.md](../Documents/Item_Types.md) / [Actor_Types.md](../Documents/Actor_Types.md)
- フェーズ5 申し送り → [Phase_5_Tasks_Detail.md](Phase_5_Tasks_Detail.md)、Design_Review_Entries の 5-6 エントリ
