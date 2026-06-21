# コンディション（BS・戦闘不能）の ActiveEffect 表現（フェーズ9-4 設計・確定）

ダメージや技能で付与される**状態（condition）**を ActiveEffect として表現するモデル。バッドステータス（[Bad_Status.md](../Game_Rules/Bad_Status.md)）と戦闘不能ダメージ状態（[Damage_Rules.md](../Game_Rules/Damage_Rules.md) §戦闘不能ダメージ）を同一モデルで扱う。AE 適用の土台は [Active_Effects.md](Active_Effects.md)（v2）。

> **ステータス**: 設計確定（2026-06-22、ユーザー承認）。付与 UI は Foundry 標準のステータスエフェクト（`CONFIG.statusEffects` / トークン HUD）。

---

## 0. 大原則（condition は自己完結する実体）

D&D と違い TNX の BS は**効果が一意固定でなく、効果値がインスタンスごとに可変**（邪毒n・衰弱-n・電子妨害n・重圧の対象能力値・萎縮/憎悪の対象…）。よって「定義が効果を固定所有するレジストリ」では不足。一方、効果値を**汎用の変更（changes）タブ**で設定すると、アイテムバフと区別がつかず condition の意味が失われる。

このため condition は次の二層で表す。**設定はすべて condition 自身に閉じる**。

| 層 | 中身 | 置き場所 |
|---|---|---|
| **同一性** | status id / `conditionKind` / 回復トリガ / 除去可否 / 「BS扱い」 | condition 定義（貼付で確定） |
| **効果値（可変）** | -n / 強度 n / 対象 ability・UUID 等 | **condition 専用フラグ**（詳細タブで設定） |

- **condition は汎用 changes タブを使わない。** 効果値は**詳細タブの condition 専用フィールド**（重複可と同じ場所）または貼付時ダイアログで設定し、`flags.tokyo-nova-axleration` に保持する。
- **効果はコードが `conditionKind` ＋効果値フラグを読んで適用**する（changes 経由でない）。これにより condition（status＋値フィールド）が**設定の唯一の出所**となり、手打ち changes で replicate できず、アイテムバフ（v2・changes 手打ち）と完全に住み分く。
- **アイテムバフ（v2）と condition は別系統**:
  - アイテムバフ … 変更タブで `key+mode+value` を手打ち（自由・Active_Effects v2）。
  - condition … status で貼り、詳細タブの専用欄で効果値を設定、適用はコード（kind 駆動）。
- 重複は既存の重複可フラグ（`stackable`）。**重複明記の BS のみ true**。非重複は最大採用（強度違いは最大 n）。
- 効果文はルール本文を焼き込まず**ユーザー入力**（AE の description を流用、§4.1）。

## 1. condition のフラグ・スキーマ（`flags.tokyo-nova-axleration`）

既存の `effectId` / `stackable` に加える:

- `conditionKind`（string）… condition の種別。コードの適用分岐・回復・表示・除去判定の鍵。
- `magnitude`（number）… 可変の効果量（酩酊 2/5・衰弱 n・邪毒 n・電子妨害 n 等）。kind ごとに意味づけ。
- `targetAbility`（"reason"|"passion"|"life"|"mundane"）… 重圧の対象能力値。
- `targetUuid`（string）… 萎縮/憎悪の対象キャラ。
- `targetMode`（"include"|"exclude"）… 対象を含む時/含まない時に -5（萎縮=include / 憎悪=exclude）。
- `durationUnit`（enum）… 回復トリガ（mainProcessStart / mainProcessEnd / cleanup / sceneEnd / cutEnd / treated / actEnd 等）。9-4 はフラグ保持のみ。発火は 13/14/15。

status id（`CONFIG.statusEffects`）が condition の一意性を担い、`conditionKind` がコード分岐を担う（同義になる想定でよいが、id は Foundry 連携・kind は適用ロジックの鍵として分ける）。

## 2. 詳細タブ UI（condition 専用フィールド）

`renderActiveEffectConfig` フック（重複可チェックボックスと同じ）で、AE が condition（`conditionKind` を持つ）のとき詳細タブに kind 別フィールドを出す:

- 数値修正型（酩酊・衰弱）: 「効果量」欄（`magnitude`）
- 強度型（邪毒・電子妨害）: 「強度 n」欄（`magnitude`）
- 重圧: 能力値ドロップダウン（`targetAbility`）
- 萎縮・憎悪: 対象選択（`targetUuid` ＋ `targetMode` は kind で固定）
- 共通: 回復トリガ（`durationUnit`）／重複可（`stackable`）

## 3. 効果分類（4型）と適用ロジック

適用はアイテムバフと同じ**2注入点**（[Active_Effects.md](Active_Effects.md) §1）に condition 用コードを追加する。

### ① 数値修正型 — コードが派生値／判定に加算
酩酊・衰弱。`conditionKind`＋`magnitude` を読み、対象へ -magnitude。
- 酩酊(小/大) = 全（上方）判定の達成値 -magnitude ＋ 全制御値 -magnitude。
- 衰弱(-n) = 全制御値 -magnitude。
- 制御判定への達成値ペナルティは無し（論点1: 上方判定のみ）。

### ② 行動制限型 — kind による遮断（数値でない）
恐慌・重圧・捕縛・気絶/失神/仮死/昏睡。消費側が active condition を見て遮断/警告。
- 重圧 = `targetAbility` を使う上方判定を不可（スート→能力値一致の技能判定＋能力値判定。制御は可）。**9-4 で判定時に配線**。
- 捕縛 = 指定武器（識別キー）で攻撃不可。**9-4 で攻撃判定時に配線**（対象武器の保持方法は実装時詳細）。
- 恐慌（リアクション不可）・気絶等（メインプロセス不可）は **13 申し送り**。

### ③ 計算型 — kind＋強度をコードが算出
電子妨害n。所持アイテムを `category∈{武器・アーマーギア・サイバーウェア・トロン・サイコアプリ}` かつ電制(hack)≤`magnitude` で数え（上限10）、全（上方）判定にマイナス。ウェット/全身義体/該当タップでゴースト等の特例は段階値。

### ④ 継続ダメージ型 — kind＋強度（発火は後続）
邪毒n。`magnitude=n` を保持。実発火（クリンナップごとに山札1枚＋n の肉体ダメージ）は**カット進行(13)＋ダメージ算出(12)**。9-4 は器のみ。

### 条件付き攻撃判定デバフ（萎縮・憎悪）
攻撃判定時（〈白兵〉〈射撃〉〈心理〉〈圧力〉を識別キーで検出）に `game.user.targets` を読み、`targetUuid` と照合して達成値 -5（`targetMode`）。戦闘(13)非依存。

## 4. BS 一覧の kind 割り当て

| BS | 型 | 効果値フラグ | 重複 |
|---|---|---|---|
| 恐慌 | ② | （なし） | 非 |
| 邪毒 | ④ | magnitude=n | 非（最大n） |
| 重圧 | ② | targetAbility | 非 |
| 衰弱 / 衰弱(-n) | ① | magnitude=n（全制御値 -n） | **重複** |
| 捕縛 | ② | 対象武器（識別キー） | **武器ごと** |
| 酩酊(小/大) | ① | magnitude=2/5 | 小大別・**重複** |
| 萎縮 | ①+対象 | targetUuid・targetMode=include | **対象ごと** |
| 憎悪 | ①+対象 | targetUuid・targetMode=exclude | 対象ごと存在・ペナルティ非重複 |
| 電子妨害 | ③ | magnitude=n | 非（n違いも非重複） |

## 5. 戦闘不能ダメージ状態

| 状態 | 型 | 効果 | 回復（発火は15/13） |
|---|---|---|---|
| 気絶 / 失神 | ② | メインプロセス不可（移行判定も） | カット進行終了 |
| 仮死 / 昏睡 | ②＋終端予約 | メインプロセス不可＋「シーン終了までに未治療→完全死亡」 | 治療(目標値20)→2シーン後 |
| 完全死亡 / 精神崩壊 / 抹殺(社会) | 終端マーカー | キャラロスト（社会はセッション終了後） | 原則不可（神業等） |

「この効果は BS として扱う」スタイル技能効果も `conditionKind` を持たせ、BS 解除で解除可能にする。

## 6. 5論点 裁定（2026-06-21）

1. `check`系の「あらゆる判定」は**上方判定のみ**。制御判定は「制御判定」を指定した効果のみ対象。
2. 重圧 = 指定能力値を使う判定が不可（そのままの意味。制御は可）。
3. 萎縮/憎悪 = 対象 UUID ＋ `game.user.targets` 照合で **9-4 完結**（戦闘13不要）。
4. 重複明記なしの BS は全て非重複。強度違いは最大 n。
5. 効果値は **condition 自身の専用フィールド**で設定（汎用 changes でなく）。付与 UI は既存ステータスエフェクト。

## 7. 9-4 スコープ / 申し送り

**9-4 で実装**: condition フラグスキーマ＋詳細タブ UI（kind 別効果値）／kind 別適用（①数値・②重圧捕縛の遮断・③電子妨害算出・萎縮憎悪の対象照合）／11 BS＋戦闘不能の `CONFIG.statusEffects` 定義・kind 割り当て／持続単位フラグ（基盤のみ）／効果文ユーザー入力。

**申し送り**: ダメージ算出本体→12 ／ チャート参照→付与トリガ→12 ／ 邪毒・恐慌・メインプロセス不可の発火→13 ／ 回復タイミングの発火→15 ／ 仮死昏睡の死亡確定→15。

## 参照

- [Active_Effects.md](Active_Effects.md)（v2 適用モデル）
- [Bad_Status.md](../Game_Rules/Bad_Status.md) / [Damage_Rules.md](../Game_Rules/Damage_Rules.md)
- [Time_Management.md](../Game_Rules/Time_Management.md)（回復タイミングの時間単位）
- [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md)
</content>
