# ActiveEffect キーリファレンス

2026-10-09の作業ツリーの実装に対応。AEの変更キーは変更先を指定し、値に書くFormulaは参照元を指定する。変更キーに `@` は付けない。式は [Formula_Reference.md](Formula_Reference.md) を参照。

## 種類と適用先

| 種類 | 用途 |
|---|---|
| 通常AE | キャラクターの能力値・所持技能・判定・ダメージ・状態 |
| 付与AE（type: enchantment） | アウトフィットの性能・構造属性 |

付与AEはアウトフィットに適用する。Actor標準転送・判定／ダメージ補正収集から除外され、ステータスと状態フラグも保存時に除去される。

- 自動適用オンの付与AEは「準備先に適用」「対象を選択して適用」のどちらか一つを指定する。選択対象は同じActorが所持する別のアウトフィット。
- 自動適用オフでは、準備先・選択対象・自身のいずれか一つ、または未指定。未指定なら用途カードのドロップ欄で付与先アウトフィットを指定する。
- 自動適用オフの通常AEは使用時付与用の定義。アウトフィット由来のパッシブ通常AEは準備済みまたは準備不要の場合に有効。
- 転送・付与済みコピーは元定義の自動適用設定と区別する。無効化・抑制された効果は適用しない。

## 通常AE：能力値・技能

`<能力>` は `reason`、`passion`、`life`、`mundane`。`<key>` は所持技能の `system.identificationKey`。

| 変更キー | 内容 |
|---|---|
| `system.abilities.<能力>.value` / `.control` | 能力値／制御値の実効値 |
| `system.cs.base` / `system.cs.value` / `system.cs.current` | CSベース／CS／CSカレント |
| `system.ar.max` | 付与ARの実効値 |
| `system.bodySlot.<部位キー>.capacity` | 身体部位の容量 |
| `system.baseAttack.damageType` | 生身のダメージ種別（S/P/I/X） |
| `system.handMaxSizeMod` | 手札上限修正（ネイティブ適用の例外） |
| `system.skill.<key>.level` | 一般技能・スタイル技能の実効レベル |
| `system.skill.<key>.usesBounty` | 技能の報酬点使用可否 |
| `system.skill.<key>.suits.spade` / `.heart` / `.diamond` / `.club` | 技能の使用可能スート |

技能キーは末尾 `*` による前方一致も受理する。`system.skill` は技能型に限定され、同じ識別キーのアウトフィットには適用しない。値バフは原則として実効値を変更し、編集用の素値を保持する。

## 通常AE：判定

| 変更キー | 適用範囲 |
|---|---|
| `system.check.all` | 能力値判定・技能判定全般（制御判定を除く） |
| `system.check.<能力>` / `system.controlCheck.<能力>` | 能力値判定／制御判定 |
| `system.check.<key>` | 指定技能判定（末尾 `*` 可） |
| `system.check.style.<key>` / `system.check.works.<key>` | スタイル／組織の技能判定 |
| `system.check.society.nation` / `.city` / `.industry` / `.organization` | 社会技能の下位区分 |
| `system.check.all.appearance` / `.information` | 登場判定／情報収集判定 |
| `system.check.all.appearance.<area>` | 指定エリアへの登場判定 |
| `system.check.<key>.appearance` / `.appearance.<area>` / `.information` | 指定技能の特殊判定 |
| `system.check.<key>.<action>` | 指定技能のアクションタイミング限定判定 |
| `system.check.<usageType>` | 指定用途タイプの判定 |
| `system.check.meleeAttack` / `.rangedAttack` | 白兵攻撃／射撃攻撃の判定 |
| `system.check.cardValue` | 使用カードの数字を選択値で上書き |
| `system.check.suitChange` | スート変更許可マーカー（値不要） |

`<action>` は `major`、`minor`、`reaction`、`move`、`auto`、`explanation`、`other`。アクションタイミングの場合のみ一致する。

`<usageType>` は `physicalAttack`、`mentalAttack`、`socialAttack`、`dodge`、`parry`、`mentalReaction`、`socialReaction`、`moveBlockReaction`、`escapeBlockReaction`、`reaction`、`move`、`escape`、`treatment`、`repair`、`modification`、`purchase`、`covering`、`declaration`。

`system.check.melee` は〈白兵〉技能、`system.check.meleeAttack` は白兵攻撃を指定する。判定補正は実行時に評価する。

## 通常AE：ダメージ・状態

`<系統>` は `physical`、`mental`、`social`。省略可能なキーで系統を省略すると全系統。

| 変更キー                                                               | 意味                               |
| ------------------------------------------------------------------ | -------------------------------- |
| `system.damage.dealt` / `system.damage.dealt.<系統>`                 | 与えるダメージへの加算                      |
| `system.damage.dealt.meleeAttack` / `.rangedAttack`                | 白兵／射撃攻撃のダメージへの加算                 |
| `system.damage.taken` / `system.damage.taken.<系統>`                 | 受けるダメージへの加算（負なら軽減）               |
| `system.damage.taken.S` / `.P` / `.I` / `.X`                       | 肉体ダメージ種別を指定した加算                  |
| `system.damage.vsStyle.<key>` / `system.damage.vsWorks.<key>`      | 対象のスタイル／組織を条件とする加算               |
| `system.damage.fromStyle.<key>` / `system.damage.fromWorks.<key>`  | 攻撃者のスタイル／組織を条件とする被ダメージ加算         |
| `system.damage.vsWet` / `system.damage.vsNotWet`                   | 対象がウェット／非ウェットなら加算（末尾に `.<系統>` 可） |
| `system.damage.replaceTag.<元タグ>` / `system.damage.addTag.<元タグ>`    | タグ置換／追加。値はタグキー                   |
| `system.ignore.all` / `system.ignore.bs` / `system.ignore.bs.<種別>` | 状態の効果を無視するマーカー                   |
| `system.ignore.damage` / `system.ignore.damage.<系統>`               | ダメージ由来の状態の効果を無視するマーカー            |

無視キーは値不要。状態・負傷の記録自体は削除しない。

## 付与AE：数値・特性

適用先はAE設定で指定する。以下は付与先アウトフィット自身の属性。対象型に存在しない属性は新設しない。

| 変更キー | 内容 |
|---|---|
| `system.attack.value` / `system.guard.value` | 攻撃力／受け値 |
| `system.attack.damageType` | ダメージ種別（S/P/I/X） |
| `system.defence.S` / `.P` / `.I` | 防御力 |
| `system.buy` / `system.hide` / `system.preserveExp` | 購入値／隠匿値／常備化経験点 |
| `system.appearancePenalty` / `system.hack` / `system.controlMod` | 危険値／電脳制御値／制御値修正 |
| `system.FAValue` | フルオート値 |
| `system.speedFactor` / `system.passenger` | 移動力／乗員数 |
| `system.cycle` / `system.combatSpeedMod` | サイクル／CS修正 |
| `system.appearanceTarget` / `system.cyberSecurity` / `system.analogSecurity` | 住宅施設の各値 |
| `system.uses.max` / `system.ammo.max` | 使用回数／装弾数の実効最大値 |
| `system.slot.capacity` / `system.slot.<種別>.capacity` | 通常／名前付きスロットの容量 |
| `system.equip.required.<部位キー>` / `system.equip.alternative.<部位キー>` | 必須／代替部位の追加。値は0以上の整数となる式（0は追加なし） |
| `system.name` | 名前装飾。値の `{}` は現在名に置換。なければ全置換 |

名前付きスロットには `surface`、`deep`、`unconscious`、`software`、`hardware` などを使い、存在する種別だけを変更する。

真偽の特性キーは `system.isFullAuto`、`system.isLaser`、`system.isFleshChange`、`system.isCyber`、`system.isMutantOrgan`、`system.isShiki`、`system.noPrepareRequired`、`system.isConsumption`、`system.isMalfunction`、`system.isDestroyed`。オン／オフを選ぶ。`isAction`・`noCombo` は対象外。

## 付与AE：構造属性

| 変更キー | 値・操作 |
|---|---|
| `system.outfitCategories` | 分類の行操作（追加・指定行の変更・削除・全体置換） |
| `system.equip.rows` | 装備部位の行操作（追加・指定行の変更・削除・全体置換） |
| `system.majorCategory` / `system.minorCategory` | 主分類のみ変更。通常の小分類変更時は大分類も補完 |
| `system.additionalCategories` | 旧形式。副分類を追加／副分類配列全体を上書き |
| `system.equip.parts` | 旧形式。部位行の追加／部位配列全体を上書き |
| `system.equip.relation` / `system.equip.optional` | 旧形式。全部位行の requirement／optional にも反映 |
| `system.range.min` / `system.range.max` / `system.attackArea` | 射程／攻撃範囲 |
| `system.isMeleeWeapon` / `system.isRangedWeapon` / `system.canStun` | 白兵武器／射撃武器／スタン可能 |
| `system.shikiType` / `system.combatSpeedModGhostIgnore` | 式神タイプ／ゴースト時のCS修正無視 |
| `system.buy.mode` / `system.hide.mode` / `system.guard.mode` | 値の方式（属性ごとの選択肢） |
| `system.<属性>.mode` | preserveExp、appearancePenalty、hack、defence、controlMod、speedFactor、passenger、cycle、combatSpeedMod の方式（none/value/reference） |
| `system.uses.isLimit` / `system.uses.type` / `system.ammo.isLimit` | 使用回数制限／回復区分／弾数制限 |
| `system.exclusive` | 専用条件の行を追加／上書き |
| `system.description` / `system.img` | 説明（追加／上書き）／画像（上書き） |
| `system.operateSkillKey` / `system.housingArea` / `system.useHousingAreaDrop` | 操縦技能キー／住宅エリア参照文字列／住宅エリア利用 |

選択・真偽・上書き専用文字列は上書きに固定し、保存時も検証する。構造属性は派生計算前の実効データへ適用し、解除時に素値へ戻す。

## 分類・部位の行操作（2026-10-09）

分類キーの全一覧と保存先は [Outfit_Category_Keys.md](../Documents/Outfit_Category_Keys.md)。部位の必須／選択候補・任意・表示は [Outfits.md](../Game_Rules/Outfits.md)。

`system.outfitCategories` と `system.equip.rows` は変更タイプを「上書き」に固定し、操作フォームで次を選ぶ。AEの「上書き」は変更タイプであり、行操作の「全体置換」とは別。

| operation | 分類 | 部位 |
|---|---|---|
| `add` | 副分類を追加 | 行を追加 |
| `update` | 指定行の項目を変更 | 指定行の項目を変更 |
| `remove` | 指定副分類を削除（主分類は削除不可） | 指定行を削除 |
| `replace` | 全体を置換。先頭が主分類、残りが副分類。空配列不可 | 全体を置換。空配列も可 |

対象は固定ID、または項目の一致条件で指定する。主分類は予約ID `primary`、副分類と部位は保存された `rowId` を使用する。既存行には読込時の互換移行でIDを補い、新規行にもIDを付け、編集・並べ替え時は保持する。配列番号を対象IDにしない。部位の削除・置換時は、占有候補 `partOrChoice` の番号を行IDから引き直す。

条件は指定した全項目の完全一致。対象なし、または複数一致で「一致する全行」がオフの場合は、その操作を適用しない。無効な項目・分類の矛盾も停止し、対象Item Sheetに理由を表示する。他のAE変更まで一括取消しする処理ではない。

フォームの「変更しない」は元の値を保持する。自由記述は変更チェックを付けてから編集し、未変更と空文字への変更を区別する。「行を選ぶための参照装備」は候補確認用であり、実際の付与先はAEの適用先設定で決まる。

保存値はJSON。例えば「防具の副分類を持つ行」の小分類を変更する場合：

```json
{"operation":"update","selector":{"mode":"condition","match":{"major":"armor"},"all":false},"changes":{"minor":"armorGear"}}
```

この条件は主分類にも一致し得る。特定の副分類だけを指定する場合は `{"mode":"id","id":"対象のrowId"}` を使用する。`add` / `replace` は `rows`、`update` は `selector` と `changes`、`remove` は `selector` を持つ。

分類の変更項目は `major` / `minor` / `majorOther` / `minorOther`。通常の小分類から親大分類を補完し、`reference` / `other` は既存の大分類を保持する。大分類だけの変更で既存小分類と矛盾する場合は、小分類も指定するか未選択を明示する。

部位では `kind`、`requirement`、`optional`、`value`、`partKey`、`slots`、`refSubKind`、`hostMajor`、`hostMinor`、`hostMinorExclude`、`hostFeature`、`hostKey` を行ごとに変更できる。`rowId` 自体は変更項目にしない。

AEの結果は派生計算前の実効データに適用する。元データへ書き込まず、無効化・解除で戻る。編集フォームにも実効値を焼き込まない。旧形式の分類・部位キーも互換用に残るが、新規設定は行操作を使用する。

実装は `scripts/rules/outfit-row-operations.mjs`、`scripts/rules/enchantment-attributes.mjs`、`scripts/ui/outfit-row-operation-editor.mjs`。純関数・スキーマ・DOMフォームはローカルテスト済み。Foundry実機での操作・再保存は未検証。

## 条件・評価・重複・持続

条件はキーの角括弧に書き、複数条件は `;` で区切る。解析・比較は `parseEffectConditions` / `evalEffectConditions` に従う。数値変更は定数または決定的な式と、加算・上書き・乗算・上限／下限のモードを使う。選択値・文字列・マーカーは各表の規約を優先する。

恒常の数値変更はリテラル値を先に、式を後に適用する。式同士の循環参照を前提にしない。使用回数の最大値の式はAE適用前に評価され、その後最大値へのAEが乗る。容量変更で消費済み回数・弾数を巻き戻さない。

同一AEの重複と式の参照重複は別の照合。式の重複は [Formula_Reference.md](Formula_Reference.md) を参照。

効果時間 `nextCheck`（判定1回）は使用時に付与された実体を対象とし、通常判定は完了処理後、攻撃はダメージ解決後に失効する。パッシブ定義・転送コピーを消費しない。中止では消費せず、シーンをまたいで保持し、アクト終了で失効する。

## 設定例

| 目的 | 種類 | 変更キー | モード | 値 |
|---|---|---|---|---|
| 生命を2増やす | 通常 | `system.abilities.life.value` | 加算 | `2` |
| 〈白兵〉判定を2増やす | 通常 | `system.check.melee` | 加算 | `2` |
| 白兵攻撃ダメージを3増やす | 通常 | `system.damage.dealt.meleeAttack` | 加算 | `3` |
| Pダメージを3軽減 | 通常 | `system.damage.taken.P` | 加算 | `-3` |
| 指定先の攻撃力を2増やす | 付与 | `system.attack.value` | 加算 | `2` |
| 指定先の名前を装飾 | 付与 | `system.name` | 上書き | `{}（強化）` |

## 旧キーとの対応

| 旧表記 | 現在の公開キー／設定 |
|---|---|
| `check.*` / `controlCheck.*` / `damage.*` / `ignore.*` | 先頭に `system.` を付ける |
| `system.ability.life` / `system.control.life` | `system.abilities.life.value` / `.control` |
| `system.partSlot.<部位>` | `system.bodySlot.<部位>.capacity` |
| `system.attack` / `system.attack.total` | `system.attack.value` |
| `system.guardValue` / `.value` / `.total` | `system.guard.value` |
| `system.guardValue.mode` | `system.guard.mode` |
| `name` | `system.name` |
| `system.slot.normal.capacity` | `system.slot.capacity` |
| `item.<技能キー>.system.levelTotal` | `system.skill.<技能キー>.level` |
| `item.<アウトフィットキー>.system.*` | 付与AEの適用先設定＋対象属性の `system.*` |

旧キーの受理と保存データの移行完了は別。`system.category.<分類>.<属性>` や旧識別キー指定は互換経路として存在するが、新規の個別付与は型と適用先設定で表す。`system.self.*`、`system.parent.*`、`item.self.system.*`、`item.parent.system.*` はAE変更キーとして使わない。

## 実装の参照先

- 公開キー・候補: `scripts/rules/ae-public-keys.mjs`、`scripts/core/effect-config-inject.mjs`
- キー解析・適用可否: `scripts/data/item/helpers.mjs`
- 型・付与先: `scripts/rules/ae-kind.mjs`
- 構造属性: `scripts/rules/enchantment-attributes.mjs`
- 実効値の適用: `scripts/data/actor/common/character-base.mjs`
- 用途からの付与: [Usage_System.md](Usage_System.md)

過去の設計・移行経緯は `02_System/AE_*_2026-09-20.md` を参照。本ページは現在の入力リファレンス。
