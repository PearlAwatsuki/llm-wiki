# コンディション（BS・戦闘不能）の ActiveEffect 表現（フェーズ9-4 設計・承認待ち）

ダメージや技能で付与される**状態（condition）**を ActiveEffect として表現するモデル。バッドステータス（[Bad_Status.md](../Game_Rules/Bad_Status.md)）と戦闘不能ダメージ状態（[Damage_Rules.md](../Game_Rules/Damage_Rules.md) §戦闘不能ダメージ）を同一モデルで扱う。AE 適用の土台は [Active_Effects.md](Active_Effects.md)（v2）。

> **ステータス**: 設計（2026-06-21、5論点ユーザー裁定済み）。実装着手は本ページ承認後。
> **付与 UI**: Foundry 標準のステータスエフェクト（`CONFIG.statusEffects` / トークン HUD）を用いる。コンディション種別を選ぶ専用 UI は新設しない。

---

## 0. 大原則

- condition = actor 上の ActiveEffect。**効果の数値修正は標準の changes（キー＋モード＋値）**で表す（Active_Effects v2 と同じ文法）。
- 数値で表せない効果（行動制限・計算型・継続ダメージ）は、**標準キーの拡張（block キー）またはマーカー＋コード**で表す。
- 重複は既存の **重複可フラグ**（`flags.tokyo-nova-axleration.stackable`）に従う。**重複すると明記された BS のみ stackable=true**。それ以外は非重複（同一効果＝最大採用）。
- 強度 n（邪毒n・電子妨害n）は AE のフラグ `intensity` に保持。強度違いは非重複なら**最大 n を採用**。
- 効果文テキストはルール本文を焼き込まず**ユーザー入力**（§4.1）。
- 回復タイミングは**持続単位フラグ**として保持（9-4 はフラグのみ。発火＝失効/回復処理は 13/14/15）。

## 1. 効果分類（4型）と表現

### ① 数値修正型 — 標準 changes ＋ ワイルドカード拡張
全判定・全能力値など「広い対象」を指すため、`parseEffectTargetKey` にワイルドカード `*` を追加する。

| 追加キー | 対象 |
|---|---|
| `check.*` | **あらゆる上方判定（技能・能力値）の達成値**。制御判定は含まない（論点1） |
| `controlCheck.*` | あらゆる制御判定（明示時のみ。論点1） |
| `system.ability.*` | 全能力値 total |
| `system.control.*` | 全制御値 totalControl |

- 例: 酩酊(小) = `check.*` ADD -2 ＋ `system.control.*` ADD -2。酩酊(大) = -5。
- 例: 衰弱(-n) = `system.control.*` ADD -n。
- 既存の重複排除（identity 単位・最大採用・stackable）に乗る。

### ② 行動制限型 — block キー（数値でない）＋消費側チェック
キーの**存在**が禁止を表す（mode/value は無視、または強度に流用）。消費側が該当 block を見て遮断/警告する。

| block キー | 意味 | 消費側 | 9-4 で配線 |
|---|---|---|---|
| `block.check.<ability>` | その能力値を使う上方判定が不可（重圧）。スート→能力値が一致する技能判定＋能力値判定。制御は可（論点2） | 判定フロー | ○（既存 abilityKey 判定を流用） |
| `block.reaction` | リアクション不可（恐慌） | リアクション系 | △ 13 |
| `block.mainProcess` | メインプロセス不可（気絶/失神/仮死/昏睡） | カット進行 | △ 13 |
| `block.attackWith.<識別キー>` | 指定武器で攻撃不可（捕縛・武器ごと） | 攻撃判定 | ○（攻撃技能＋使用武器の識別キー照合） |

- 9-4 では**現存する経路だけ配線**（重圧・捕縛は判定/攻撃判定時に評価可能）。恐慌・メインプロセス不可は戦闘フェーズ(13)で配線。

### ③ 計算型 — マーカー＋強度をコードが算出
電子妨害n。所持アイテムを数えるため純粋 changes で書けない。AE は `intensity=n` のマーカーとし、判定/派生時にコードが個数を算出して `check.*` 相当の修正を生成。

- 算出: category∈{武器・アーマーギア・サイバーウェア・トロン・サイコアプリ} かつ 電制(hack) ≤ n の**所持数**（上限10）を全（上方）判定にマイナス。ウェット/全身義体/該当タップでゴースト等の特例は段階値（-1 / -10）。
- n 違いでも別 BS 扱いせず非重複（最大 n）。

### ④ 継続ダメージ型 — マーカー＋強度（発火は後続）
邪毒n。AE は `intensity=n` ＋「クリンナップ継続ダメージ」記述子を保持。**実発火（クリンナップごとに山札1枚＋n の肉体ダメージ）はカット進行(13)＋ダメージ算出(12)**。9-4 は器のみ。

## 2. 条件付き攻撃判定デバフ（萎縮・憎悪）— 対象 UUID 方式

戦闘フェーズに依存せず実装する（論点3）。

- AE が **対象キャラの UUID** と適用条件（`includeTarget`＝対象を含む時 / `excludeTarget`＝含まない時）を保持。
- 攻撃判定時（〈白兵〉〈射撃〉〈心理〉〈圧力〉を識別キーで検出）に **`game.user.targets`** を読み、対象 UUID と照合して達成値 -5。
- 萎縮 = 対象を含む攻撃判定で -5（`includeTarget`）。憎悪 = 対象を含まない攻撃判定で -5（`excludeTarget`）。
- 萎縮/憎悪は**対象ごとに別 BS**（対象 UUID が identity の一部）。憎悪のペナルティ自体は複数受けても非重複（-5 は最大1回＝既存の最大採用）。
- 将来 13 がターゲティングを精緻化したらそこへ寄せる。

## 3. BS 一覧の型割り当て

| BS | 型 | キー / 機構 | 重複 |
|---|---|---|---|
| 恐慌 | ② | `block.reaction` | 非 |
| 邪毒n | ④ | intensity=n・継続ダメージ記述子 | 非（最大n） |
| 重圧(能力値) | ② | `block.check.<ability>` | 非 |
| 衰弱 / 衰弱(-n) | ① | `system.control.*` ADD -n | **重複**（stackable） |
| 捕縛 | ② | `block.attackWith.<識別キー>` | **武器ごと** |
| 酩酊(小/大) | ① | `check.*` ＋ `system.control.*` ADD -2/-5 | 小大は別 BS で**重複** |
| 萎縮 | ①+対象 | 攻撃判定 -5・`includeTarget`・対象UUID | **対象ごと** |
| 憎悪 | ①+対象 | 攻撃判定 -5・`excludeTarget`・対象UUID | 対象ごと存在・ペナルティ非重複 |
| 電子妨害n | ③ | intensity=n・計算型 | 非（n違いも非重複） |

## 4. 戦闘不能ダメージ状態

| 状態 | 型 | 効果 | 回復トリガ（発火は15/13） |
|---|---|---|---|
| 気絶 / 失神 | ② | `block.mainProcess`（移行判定・メインプロセス不可） | カット進行終了 |
| 仮死 / 昏睡 | ②＋終端予約 | メインプロセス不可＋「シーン終了までに未治療→完全死亡」 | 治療（目標値20）→2シーン後 |
| 完全死亡 / 精神崩壊 / 抹殺(社会) | 終端マーカー | キャラロスト（社会はセッション終了後適用） | 原則不可（神業等） |

「この効果は BS として扱う」スタイル技能効果も BS 化し、BS 解除効果で解除可能にする。

## 5. AE が持つフラグ（スキーマ案）

`flags.tokyo-nova-axleration` に集約（既存 effectId / stackable に追加）:

- `intensity`（number）… 強度 n（邪毒・電子妨害）
- `conditionKind`（string）… 計算型/継続ダメージ等の識別（電子妨害・邪毒）。コード分岐に使う
- `targetUuid`（string）… 萎縮/憎悪の対象キャラ
- `targetMode`（"includeTarget" | "excludeTarget"）
- `durationUnit`（enum）… mainProcess / cut / scene / act / 治療 等（回復トリガ。9-4 はフラグのみ）

block キーは標準 changes として打つ（`block.*`）。`parseEffectTargetKey` に block スコープと `*` を追加する。

## 6. 9-4 スコープ / 申し送り

**9-4 で実装**: ワイルドカード文法（`check.*`/`system.ability.*`/`system.control.*`）、block キー（`block.*`）＋重圧・捕縛の判定時遮断、計算型（電子妨害）算出、萎縮/憎悪の対象UUID照合（Foundry targets）、強度/重複の整合、持続単位フラグ（スキーマのみ）、効果文ユーザー入力、11 BS＋戦闘不能の `CONFIG.statusEffects` 化。

**申し送り**: ダメージ算出本体→12 / チャート参照→付与トリガ→12 / 邪毒・恐慌・メインプロセス不可の発火→13 / 回復タイミングの発火→15 / 仮死昏睡の死亡確定→15。

## 参照

- [Active_Effects.md](Active_Effects.md)（v2 適用モデル）
- [Bad_Status.md](../Game_Rules/Bad_Status.md) / [Damage_Rules.md](../Game_Rules/Damage_Rules.md)
- [Time_Management.md](../Game_Rules/Time_Management.md)（回復タイミングの時間単位）
- [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md)
</content>
