# Formula 参照キーリファレンス

2026-09-27の作業ツリーの `buildFormulaData` に対応。Formulaは「値を読む式」で、参照には `@` を付ける。AEの変更先は [Active_Effects.md](Active_Effects.md) を参照。

## 公開名前空間

| 参照 | 内容 | 例 |
|---|---|---|
| `@abilities.<能力>.*` | 行為者の能力値・制御値 | `@abilities.life.value` |
| `@skill.<key>.*` | 所持する一般技能・スタイル技能 | `@skill.melee.level` |
| `@outfit.<key>.*` | 所持アウトフィット | `@outfit.weapon_key.attack.value` |
| `@item.*` | 式の保持元アウトフィット自身（bearer） | `@item.attack.value` |
| `@parent.*` | 保持元の準備先（parentItemId） | `@parent.attack.value` |
| `@achievement` / `@diff` / `@card` | 評価時に渡された判定結果 | `@diff` |
| `@target.*` | 対象が渡されるダメージ式の対象情報 | `@target.abilities.life.value` |
| `@system.*` | 専用別名のないActorデータ等 | `@system.combatSpeed.displayTotal` |

`<key>` は `system.identificationKey`。名前・Document IDではない。一意な識別キーを使い、例の `weapon_key` 等は実際のキーに置き換える。FormulaではAEの前方一致 `*` を使わない。

新しい短縮形 `@item` はアウトフィット用。技能自身のレベルは `@skill.<key>.level` で参照する。識別キーがない技能の旧相対参照 `@item.self.system.levelTotal` は互換経路として残るが、`@item.level` への単純置換はできない。

## 能力値・技能

`<能力>` は `reason`、`passion`、`life`、`mundane`。

| 参照キー | 意味 |
|---|---|
| `@abilities.<能力>.value` | 能力値の実効値（旧total） |
| `@abilities.<能力>.base` | 能力値の素値（旧value） |
| `@abilities.<能力>.control` | 制御値の実効値（旧totalControl） |
| `@skill.<key>.level` | 技能の実効レベル（levelTotal、未生成ならlevel） |
| `@skill.<key>.base` | 技能の素のレベル（level） |
| `@skill.<key>.uses.maxTotal` / `.spent` | 使用回数の実効最大値／消費済み回数（対応する技能のみ） |
| `@skill.<key>.suits.spade` / `.spadeTotal` | スートの素値／生成済みの実効値（heart、diamond、clubも同様） |

その他のsystemフィールドも供給するが、すべてが実効値に置換されるわけではない。AE変更先 `system.abilities.life.value` とFormula `@abilities.life.value` の対応も、保存スキーマのパスが同じという意味ではない。

## アウトフィット

下表の `@outfit.<key>` は、保持元アウトフィットなら `@item`、準備先なら `@parent` に置き換えられる。保持元・準備先が渡されていない場面では相対参照を使わない。

| 参照キー | 意味 |
|---|---|
| `@outfit.<key>.attack.value` / `.base` | 攻撃力の実効値／素値 |
| `@outfit.<key>.guard.value` / `.base` | 受け値の実効値／素値 |
| `@outfit.<key>.defence.S` / `.P` / `.I` | 防御力の実効値 |
| `@outfit.<key>.slot.capacity` | 通常スロット容量の実効値 |
| `@outfit.<key>.slot.<種別>.capacity` | 名前付きスロット容量の実効値 |
| `@outfit.<key>.buy.total` / `.hide.total` / `.preserveExp.total` | 購入値／隠匿値／常備化経験点 |
| `@outfit.<key>.appearancePenalty.total` / `.hack.total` | 危険値／電脳制御値 |
| `@outfit.<key>.controlMod.total` / `.combatSpeedMod.total` | 制御値修正／CS修正 |
| `@outfit.<key>.speedFactor.total` / `.passenger.total` / `.cycle.total` | 移動力／乗員／サイクル |
| `@outfit.<key>.FAValueTotal` | フルオート値の実効値 |
| `@outfit.<key>.uses.maxTotal` / `.spent` | 使用回数の実効最大値／消費済み回数 |
| `@outfit.<key>.ammo.maxTotal` / `.spent` | 装弾数の実効最大値／消費済み弾数 |
| `@outfit.<key>.quantity.value` / `.max` | 数量 |

短縮時に組み替えるのは `attack`・`guard`・`defence`・`slot`。その他は元のsystemの形を引き継ぐ。たとえば `.hack.value` は素値で、実効値は `.hack.total`。`uses.max` は式文字列を含み得るため、数値参照には `uses.maxTotal` を使う。

用途の式の `@item` は用途を持つアウトフィット、AEの値では評価に渡される保持元アウトフィット。付与・移設で保持元が変わる場合、元アイテムを読み続けたい式は一意な `@outfit.<key>` で指定する。

## 判定結果と評価時点

| 参照キー | 意味 | 有効な時点 |
|---|---|---|
| `@card` | 判定に用いたカードの数値 | カードプレイ後の判定補正・判定結果を渡すダメージ式 |
| `@achievement` | 達成値 | 判定確定後 |
| `@diff` | 差分値 | 判定確定後 |

この3つは判定結果が渡されるときに供給される。結果が渡されても未確定の項目は0になるため、0が存在しても、その場面で結果を参照できるという意味ではない。恒常AE・使用回数の式に確定後の値を使わない。判定ボーナスでは達成値・差分値は未確定。

目標値「その他」の回復判定では、呼出側が `@condition.magnitude`（状態の強度）・`@condition.woundValue`（負傷のダメージ値）を補う。全Formulaに常設される参照ではない。

## ダメージ式の対象情報

| 参照キー | 意味 |
|---|---|
| `@target.abilities.<能力>.value` / `.base` / `.control` | 対象の能力値・制御値 |
| `@target.style.<key>` | 対象が指定スタイルを持てば1、なければ0 |
| `@target.works.<key>` | 対象が指定組織のワークスを持てば1、なければ0 |
| `@target.isWet` | 対象がウェットなら1、そうでなければ0 |

対象を渡す呼出側でのみ利用できる。`@target.skill` / `@target.outfit` を一括公開する処理はない。対象条件による修正行の有効／無効も用途側で設定できる。

## その他のActor参照

| 参照キー | 内容 |
|---|---|
| `@system.combatSpeed.displayTotal` | 表示CS |
| `@system.combatSpeed.baseTotal` / `.valueTotal` / `.currentTotal` | CS各層の実効値 |
| `@system.actionRank.maxTotal` / `.value` | 付与ARの実効値／現在AR |
| `@system.physicalDamage.value` / `@system.mentalDamage.value` / `@system.socialDamage.value` | ダメージ値 |
| `@system.bounty` / `@system.bountyBase` | 現在／基本報酬点 |
| `@system.appearanceModifier` / `@system.handMaxSizeMod` | 危険値の集計／手札上限修正 |

AE変更キーに機械的に `@` を付けない。CSの変更先は `system.cs.value`、参照先は `@system.combatSpeed.valueTotal`。

## 使用回数の最大値

`uses.max` 欄は数値または式を受け付ける。技能レベル回なら `@skill.<key>.level`。

この式はAE適用前に評価され、AEで増えたレベルを使用回数へ加算しない。その後、最大値へのAEが `uses.maxTotal` に適用される。端数は切り捨て、負値・評価不能は0。神業の母数はスタイルレベル連動で機械維持されるため、一般の式入力と同じ運用にはしない。

## 式の例と重複

| 目的 | 式 |
|---|---|
| 生命の実効値を加える | `@abilities.life.value` |
| 保持元武器の攻撃力を加える | `@item.attack.value` |
| 指定武器の攻撃力を加える | `@outfit.weapon_key.attack.value` |
| 指定技能のレベルに2を加える | `@skill.skill_key.level + 2` |
| 差分値の10分の1を切り捨てる | `floor(@diff / 10)` |
| ウェット以外の対象に5を加える | `5 * (1 - @target.isWet)` |

数値・算術式・参照を決定的に評価する。ダイス・自由文・構文エラーなど評価不能な式は数値として適用しない。追加行の「供給元」は表示上の帰属であり、参照アイテムを自動で切り替えない。

判定・ダメージの合流処理では直接参照キーの重複を照合する。基礎能力値・カード等の基礎値を優先し、競合する修正欄は有利なものを採用。不採用欄は式全体を0にして理由を残す。新旧キー・相対参照は同じ参照先へ揃え、別Actorの値は区別する。「重複可」は指定した欄／効果にだけ適用する。差分値・達成値の内部依存を展開したり、同じ欄の中の繰り返し参照を禁止したりする処理ではない。

判定結果の正式な入力表記は `@achievement`・`@diff`・`@card`。実装には別名 `@check.achievement`・`@check.margin`・`@check.card` も残るが、この3つを置き換える必要はない。

## 旧表記との対応

旧参照も互換用に残るが、新規入力は上記を使う。素値と実効値を取り違えない。

| 旧Formula | 現在の表記 |
|---|---|
| `@system.life.total` / `.value` / `.totalControl` | `@abilities.life.value` / `.base` / `.control` |
| `@item.<技能キー>.system.levelTotal` / `.level` | `@skill.<技能キー>.level` / `.base` |
| `@item.<武器キー>.system.attack.total` / `.value` | `@outfit.<武器キー>.attack.value` / `.base` |
| `@item.<武器キー>.system.guardValue.total` | `@outfit.<武器キー>.guard.value` |
| `@item.<防具キー>.system.defence.S_total` | `@outfit.<防具キー>.defence.S` |
| `@item.self.system.attack.total` | `@item.attack.value` |
| `@item.parent.system.attack.total` | `@parent.attack.value` |

技能・アウトフィット以外の旧Item参照に新しい専用名前空間を仮定しない。識別キー重複・参照先不明・移設時の相対参照などがある式は、移行処理でも推測で変換しない。

## 実装の参照先

- データ構築・評価: `scripts/rules/tnx-formula.mjs`
- 能力値の公開データ: `scripts/rules/ae-public-keys.mjs`
- 重複照合: `scripts/rules/formula-duplicates.mjs`
- 旧式の変換計画: `scripts/core/formula-key-migration.mjs`
- 使用欄: [Usage_System.md](Usage_System.md)、[Check_Rules.md](../Game_Rules/Check_Rules.md)、[Damage_Rules.md](../Game_Rules/Damage_Rules.md)
