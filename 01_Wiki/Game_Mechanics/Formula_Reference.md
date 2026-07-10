# 式（フォーミュラ）で使えるキー一覧

判定ボーナス・ダメージ修正・`damageBoost`/`damageReduce`（および将来の FS 進行判定）の**式**で参照できる値の一覧（2026-07-10 作成・ユーザー要望）。正本は DataModel（`scripts/data/`）＝フィールドの増減はコードが権威。ここは実務用の索引。

---

## 参照の3系統

式は Foundry の Roll を**決定的評価**する（ダイス `d` を含む式は機械適用しない＝表示のみ）。参照キーは `@` で始める。

| 系統 | 何を指すか | 例 |
|---|---|---|
| **`@system.*`** | **判定を行うアクター**の system（＝AE がアクターに乗せる `system.*` キーと対応） | `@system.life.total` |
| **`@item.<識別キー>.system.*`** | アクターが持つ**任意のアイテム**を**識別キー**で引いた system（＝AE がそのアイテムに乗せる `system.*` と対応） | `@item.<武器の識別キー>.system.attack.total` |
| **`@diff` / `@achievement`** | 判定結果（差分値 / 達成値）。**ダメージ算出時のみ**確定（判定ボーナスは判定前のため不可） | `floor(@diff / 10)` |

- **`.total`（実効値・AE 込み）を使う**のが原則。`value` などの素フィールドは AE 適用前の入力値。**アクターの値もアイテムの値も同じ**——各 `prepareDerivedData` で `total=base` を置いた後、**アクターの単一適用パス（`_applyEffectBuffs`・`legacyTransferral=false`）が全 buff を集めて total を改変**する（AE モデル v2）。判定/ダメージは実行時に走るので、その時点で `.total` は AE 込みの最終値になっている。
  - アクター: 能力値 `reason.total`/`reason.totalControl`。 アイテム: `attack.total`・`defence.S_total`・`levelTotal`・`<param>.total`（`system.self.attack`／`system.category.weapon.attack` 等の AE がここへ着地）。
  - ※ **アイテムのパラメータが「キャラクターへ」与える修正**（制御値修正・CS 修正・危険値など）は AE でなく派生パイプラインが直接読む設計（[Active_Effects.md](Active_Effects.md) 大原則5）。ただしこれは「アイテム→キャラ」の話で、`@item.<識別キー>.system.<param>.total` が指す**アイテム自身の値**は上記どおり AE 込みで取れる。
- `@item.<識別キー>` は**識別キーを持つアイテムだけ**引ける（辞典由来の技能・アウトフィットは持つ）。識別キーはアイテムの `system.identificationKey`。
- どこで使えるか: 判定ボーナス（アクター）・ダメージ修正/`damageBoost`（攻撃者）・`damageReduce`（防御側）・**AE の「値」**（change.value にも式を書ける・2026-07-10）。`@diff`/`@achievement` はダメージ算出時のみ（AE 値・判定ボーナスでは不可）。

### AE の「値」の式（＋相対参照）

AE の change の**値**にも式を書ける（`@system.*`・`@item.<識別キー>.system.*`）。加えて **AE 値だけの相対参照**が使える:

| キー | 意味 |
|---|---|
| `@item.self.system.*` | その効果が**乗っているアイテム自身** |
| `@item.parent.system.*` | その親＝**装備先ホスト**（`parentItemId`。例: オプション→装備先の武器） |

- 例: オプションの AE 値を `@item.parent.system.attack.total` にすると、装備先アイテムの攻撃力を参照できる。
- **AE 値は `prepareDerivedData` 内で評価**されるため、`.total` は「リテラル値の AE を先に適用した後」の値を参照する（リテラル先→式後の2フェーズ。式値どうしは相互参照しない前提）。判定/ダメージの用途式は実行時評価なので、参照先 `.total` は AE 完全適用済み（[Active_Effects.md](Active_Effects.md) §3）。

### AE で「判定そのもの」にボーナスを足す（キー側）

上記は式の**値参照**。AE の**キー**で判定の達成値/制御値そのものにボーナスを足すこともできる（値バフとは**別経路**＝判定時に `_computeCheckBonus` が集計。正本 [Active_Effects.md](Active_Effects.md)「判定バフ」）。

| AE change キー | 対象 |
|---|---|
| `check.<能力値>` | 能力値判定（例 `check.life`） |
| `check.<技能識別キー>` | その技能の技能判定（例 `check.operate_car`） |
| `check.<プレフィックス>*` | 識別キー前方一致の技能判定（例 `check.operate*`＝操縦全般） |
| `controlCheck.<能力値>` | 制御判定（能力値のみ） |

- モード ADD・値＝達成値/制御値への加算。末尾にパラメータ接尾は不要。条件は `[…]` で付けられる。
- ※現状、**この判定バフの「値」はリテラル数値のみ**（式参照は値バフ＝`system.*` キー側でのみ有効）。判定用途の「判定ボーナス」欄（式・供給元つき）とは別枠。

---

## アクター `@system.*`

### 能力値・制御値（`reason` / `passion` / `life` / `mundane`）

| キー | 意味 |
|---|---|
| `@system.<能力>.total` | **能力値の実効値**（例 `@system.life.total`） |
| `@system.<能力>.totalControl` | **制御値の実効値**（＝生命の制御値など。例 `@system.reason.totalControl`） |

（`value`/`control`/`growth`/`mod`/`styleA_value`… は素の入力。実効値は `.total`/`.totalControl`。能力＝理性 reason・感情 passion・生命 life・外界 mundane）

### コンバットスピード（`combatSpeed`）・アクションランク（`actionRank`）

| キー | 意味 |
|---|---|
| `@system.combatSpeed.displayTotal` | **表示CS**（カット中はカレント/それ以外はCS。initiative と同値） |
| `@system.combatSpeed.valueTotal` / `.currentTotal` / `.baseTotal` | CS 各層の実効値 |
| `@system.actionRank.maxTotal` | **付与AR（実効）** |
| `@system.actionRank.value` | 現在AR（カット進行中のみ意味を持つ） |

### ダメージ・報酬点・生身・その他

| キー | 意味 |
|---|---|
| `@system.physicalDamage.value` / `.min` / `.max` | 肉体ダメージ（同様に `mentalDamage`・`socialDamage`。max 初期 21） |
| `@system.bounty` / `@system.bountyBase` | 報酬点（現在 / 基本） |
| `@system.baseAttack.value` / `.mod` / `.damageType` | 生身の攻撃力 |
| `@system.baseGuard.value` / `.mod` | 生身の受け値 |
| `@system.baseDefence.S_defence` / `.P_defence` / `.I_defence` | 生身の防御力（S/P/I） |
| `@system.appearanceModifier` | 危険値の合計（携帯中アウトフィット由来） |
| `@system.handMaxSizeMod` | 手札上限への AE 修正 |
| `@system.isGhost` | ゴースト状態か（真偽） |
| `@system.outfitMod.control` / `.reason` / `.passion` / `.life` / `.mundane` / `.combatSpeed` | アウトフィット由来の制御値/能力値/CS 修正 |

---

## アイテム `@item.<識別キー>.system.*`

`<識別キー>` はそのアイテムの `system.identificationKey`（辞典由来）。

### 技能（generalSkill / styleSkill）

| キー | 意味 |
|---|---|
| `@item.<key>.system.levelTotal` | 技能レベルの実効値（`.level` は素値） |
| `@item.<key>.system.suits.spade` / `.heart` / `.diamond` / `.club` | 使用可能スート（真偽） |
| `@item.<key>.system.isAction` | アクション技能か（真偽） |

### アウトフィット共通（全 outfit 型）

`{mode, value}` 系はいずれも `.total`（AE 込み実効値）を持つ。

| キー | 意味 |
|---|---|
| `@item.<key>.system.buy.total` | 購入値 |
| `@item.<key>.system.hide.total` | 隠匿値 |
| `@item.<key>.system.appearancePenalty.total` | **危険値** |
| `@item.<key>.system.hack.total` | **電脳制御値** |
| `@item.<key>.system.quantity.value` / `.max` | 数量 |
| `@item.<key>.system.uses.max` / `.spent` | 使用回数（最大 / 消費済み。残り＝max−spent） |
| `@item.<key>.system.isPrepared` / `.isCarrying` | 準備済み / 携帯中（真偽） |

### 武器（weapon）

| キー                                                     | 意味                                                  |
| ------------------------------------------------------ | --------------------------------------------------- |
| `@item.<key>.system.attack.total`                      | **攻撃力の実効値**（`.value` は素値・`.damageType` は種別 S/P/I/X） |
| `@item.<key>.system.guardValue.total`                  | 受け値                                                 |
| `@item.<key>.system.FAValue`                           | フルオート値（`.isFullAuto` が真のとき有効）                       |
| `@item.<key>.system.ammo.mode` / `.value` / `.current` | 残弾（mode=-/value/arbitrary・current=現在残弾）             |
| `@item.<key>.system.range.min` / `.max`                | 射程                                                  |

### 防具（armor）・義体（cyborg）・ヴィークル（vehicle）

| キー | 意味 |
|---|---|
| `@item.<key>.system.defence.S_total` / `.P_total` / `.I_total` | 防御力の実効値（S/P/I。armor / cyborg / vehicle） |
| `@item.<key>.system.attack.total` | 攻撃力（cyborg / vehicle） |
| `@item.<key>.system.guardValue.total` | 受け値（cyborg） |
| `@item.<key>.system.controlMod.total` | 制御値修正（armor / vehicle / ianus） |
| `@item.<key>.system.speedFactor.total` / `.passenger.total` | 移動力（SF） / 乗員（vehicle） |
| `@item.<key>.system.operateSkillKey` | 対応する操縦の識別キー（vehicle） |

### タップ（tap）

| キー | 意味 |
|---|---|
| `@item.<key>.system.cycle.total` | サイクル |
| `@item.<key>.system.combatSpeedMod.total` | CS 修正 |

---

## 使い方の例

- 生命値をダメージに加算: ダメージ修正の式に `@system.life.total`。
- 「選んだ武器の攻撃力」をダメージに加算: `@item.<武器の識別キー>.system.attack.total`（供給元にそのスタイル技能を選べば、チャットにその技能名で帰属表示される）。
- 差分値の 1/10 を進行に（FS 進行判定・将来）: `floor(@diff / 10)`。

---

## 注意

- **正本は DataModel**（`scripts/data/actor/*` ・ `scripts/data/item/*`）。フィールドは各フェーズで増減するため、迷ったらスキーマを見る。
- 実効値は `.total`（能力値は `.total`/`.totalControl`）。素フィールド（`value` 等）は AE 適用前。
- `@item.<識別キー>` は識別キーを持つアイテムのみ。手作りアイテム（識別キー空）は式から引けない。
- 評価は決定的（ダイスを含む式は機械適用されない＝表示のみ・手動修正で対応）。

## 参照

- 式の評価: `scripts/module/tnx-formula.mjs`（`buildFormulaData` / `evaluateFormula` / `evaluateBonusRows`）
- 判定ボーナス/ダメージ修正: [Check_Rules.md](../Game_Rules/Check_Rules.md)「技能判定」 / [Damage_Rules.md](../Game_Rules/Damage_Rules.md)「ダメージ修正」 / [Usage_System.md](Usage_System.md)
- AE モデル: [Active_Effects.md](Active_Effects.md)
