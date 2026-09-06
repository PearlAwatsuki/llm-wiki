# Item 型定義

全 17 type が `scripts/data/item/` に DataModel 化済。

## 共通 Template

| Template | ファイル | 内容 |
|---|---|---|
| base | `common/base.mjs` | description (HTMLField) |
| usage | `common/usage.mjs` | actions[] — `{ type, name, description }` |
| skillBase | `common/skill-base.mjs` | level, suits (spade/heart/diamond/club), isAction |
| outfitBase | `common/outfit-base.mjs` | 下記「outfitBase フィールド」参照 |
| extensible | `common/extensible.mjs` | slots[] — `{ kind, count }` プール方式（kind 6 種、count は `{mode,value}`） |

## outfitBase フィールド

| フィールド | 型 | 備考 |
|---|---|---|
| isPrepared | BooleanField | 準備済み（＝部位への装備。即使用可能） |
| isCarrying | BooleanField | 携帯中（アクトに持参） |
| isCyber | BooleanField | サイバーウェア |
| `isPre-play` | BooleanField | プレアクト購入 |
| isCheckAcquired | BooleanField | 判定取得 |
| isOption | BooleanField | （残置）オプション判定はフェーズ10 で部位行 `kind=option` から派生する方式へ吸収 |
| isConsumption | BooleanField | 消費アイテム |
| noPrepareRequired | BooleanField | 準備不要で使用可の例外（部位「-」品など・2026-06-26） |
| fromStyleSkillKey | StringField | スタイル技能由来マーク（フェーズ10-2・自動取得の重複防止。プレフィックス照合） |
| isMutantOrgan / isShiki / shikiType | Boolean/String | 特性フラグ（フェーズ10-2）: 変異器官・式神装備（タイプ= attack/defense/drive/independent） |
| isDerivedData / hasDerivedData / derivedDataRefs | Boolean/Array | 派生データ（フェーズ10-2）: 本体＝常備化経験点を消費しない／派生元＝取得時に `derivedDataRefs`（uuid+name）から自動生成 |
| quantity | SchemaField | `{value, max}` — 消費アイテムの現在/最大個数 |
| buy | SchemaField | `{mode: none/value/reference, value}` — 購入値 |
| preserveExp | SchemaField | `{mode: none/value, value}` — 常備化経験点 |
| hide | SchemaField | `{mode: none/value/reference/control, value}` — 隠匿値 |
| appearancePenalty | SchemaField | `{mode: none/value, value}` — 危険値 |
| hack | SchemaField | `{mode: none/value, value}` — 電脳制御値 |
| exclusive | ArrayField | 専用（フェーズ10-2 で自由記述→**辞典参照の配列** `{type: style/organization, key}` へ。指定のみで enforcement なし） |
| part | ArrayField | 部位行（フェーズ10-1 で種別ベースへ拡張）: `{kind: none/bodyPart/option/reference/other, value, slots, hostMajor, hostMinor, hostMinorExclude, hostFeature, hostName, refSubKind}`。兄弟フィールド `partRelation`（and/or）・`partOrChoice`（or 時の占有行）・`partOptional`（部位全体に効く任意フラグ・占有非カウント） |
| uses | SchemaField | `{isLimit, type, max, spent}` — 使用回数（spent=消費済み・D&D 方式）。**max は StringField で数値も式も受ける**（2026-08-09「1シーンにレベル回」）。実効値は派生 `uses.maxTotal`＝式を解いた結果に AE が乗ったもので、**消費・表示は必ず実効値を読む**（`usesMaxTotalOf`・`scripts/data/item/uses.mjs`） |
| majorCategory | StringField | 大分類（choices 付き）|
| minorCategory | StringField | 小分類（choices 付き、大分類と Item type で絞込） |
| identificationKey | StringField | 識別キー（全アイテム型共通） |
| parentItemId / parentSlotKind | StringField | 親アウトフィットの ID／スロット種別（オプション品用） |
| combineGroupId | StringField | コンバイン連携グループ ID |

> `isBiological` はフェーズ6-4 で削除済み。生体装備はカテゴリ「生体装備」で識別。

## Type 一覧

| type | mixin | 固有フィールド（主要） |
|---|---|---|
| **style** | base | isPersona, isKey, level(1-3), miracle, reason/passion/life/mundane (value/control) |
| **miracle** | base + usage | furigana, usageCondition, uses（isKill/isDefence/isAll はフェーズ17 で撤去・usageCount は uses へ一本化済み・**isUsed は 2026-09-05 に廃止**＝残り使用回数と二重管理だった）, **asOther** `{mode: ""\|choice\|log, selected=選んだ効果の uuid, choices[{skillUuid=区分の技能, uuid=参照先の神業}]}`（17-5・他の神業として使う: 《万能道具》《半身》《神意》=choice＝〈フォルム〉等を選ぶときに選んで固定・《突然変異》=log。区分と参照先は**どちらもドロップ**で指定） |
| **generalSkill** | base + usage + skillBase | generalSkillCategory, initialSkill, onomasticSkill |
| **styleSkill** | base + usage + skillBase | unique, style, comboSkill[], timing[], target, range, confrontation[], uses, special/performance/secret/mystery, 自動取得（フェーズ10-2: acquiresOutfit, autoAcquireItems[], autoAcquireActors[], weaponUseMandatory）, 取得・経験点（フェーズ10-3: expFree, excludeFromCount, levelRef `{enabled, key}`）, 神業書き換え（フェーズ17-7: miracleRewrite） |
| **weapon** | base + outfitBase + extensible | attack `{damageType, value}`, guardValue `{mode,value}`, range `{min,max}`, **isFleshChange**（生身変更装備・フェーズ10-6） |
| **armor** | base + outfitBase | defence `{mode,S_defence,P_defence,I_defence}`, controlMod `{mode,value}` |
| **ianus** | base + outfitBase + extensible | controlMod `{mode,value}` |
| **cyborg** | base + outfitBase | defence, attack `{damageType, value}`, guardValue `{mode,value}` |
| **tron** | base + outfitBase + extensible | なし |
| **tap** | base + outfitBase + extensible | cycle `{mode,value}`, combatSpeedMod `{mode,value}`, **combatSpeedModGhostIgnore**（ゴースト時無視・フェーズ10-5） |
| **vehicle** | base + outfitBase + extensible | speedFactor `{mode,value}`, passenger `{mode,value}`, controlMod `{mode,value}`, attack, defence |
| **residence** | base + outfitBase + extensible | appearanceTarget, cyberSecurity, analogSecurity, housingArea（UUID・エリアランクをバッジ表示＝フェーズ10-4）, useHousingAreaDrop |
| **housingArea** | base | area（セキュリティランク: なし/レッド/**イエロー**(10-4追加)/グリーン/ホワイト/サンクチュアリ）, buyRatingMod, preserveExpMod, slotMod, *Mod 群（hideMod は廃止） |
| **combiner** | base + outfitBase | combine `{source1, source2, appearance, params}`, isCombineActive, combatSpeedMod |
| **general** | base + outfitBase | なし |
| **organization** | base | なし |
| **lifePath** | base | lifePathType（origin/experience/encounter/空文字）, skillName |

## カスタムシート有無（フェーズ6 完了時点）

| カスタムシートあり | 備考 |
|---|---|
| style, miracle, generalSkill, styleSkill, organization | フェーズ3 以前 |
| weapon, armor, ianus, cyborg, tron, tap, vehicle, residence, combiner, general | TokyoNovaOutfitSheet（フェーズ6-1〜6-4） |
| housingArea | TokyoNovaHousingAreaSheet（フェーズ6-4） |
| lifePath | TokyoNovaLifePathSheet（フェーズ6-5） |

## styleSkill 特有の注意

- template.json 未定義の ★ フィールド 8 個を DataModel で明示定義
- `miracleRewrite` — 神業書き換え技能（区分 `unique = miracleChange`）の設定。`target{uuid,key,name}`（書き換える神業・空欄はどの神業でも）／`effect`（`ref`=別の神業と同じ・`own`=この技能の用途）／`refUuid`／`description`（独自効果型の効果文＝効果文は必ず書き換わるため選択は持たない）／`rewriteCondition`＋`condition`（経験点の取得条件・既定オフ）。フェーズ17-7 で、配線されないまま残っていた旧 `RewrittenTarget` / `rewritingMiracleName` / `rewritingMiracleId`（KI-018/019 で正規化した3つ）を置き換えた
- `unique` の initial は `"none"`（template.json の `""` は誤りを修正済み）
- `timing[]` は styleSkill 固有（複数可の配列）。アウトフィットの単一 `timing` は 2026-07-19 に廃止（タイミングは用途側で管理）

## usage テンプレート

判定システム（フェーズ7/8）の基盤。**構造変更は設計議論必須**。

- `usage-list.hbs` — edit ハンドラ実装済み（`_onUsageEdit`・KI-002 解消・フェーズ8-6）
- `UsageCreationDialog` — 用途作成 UI

## アウトフィット系の modeValueField

`{mode: "none"|"value", value: Number}` 構造を `modeValueField()` ヘルパー（`helpers.mjs`）で統一。
「0」と「そもそもない」を区別するため、none 時はシートで「-」表示。

## extensible スロット

`slots[] = [{kind, count: {mode,value}}]` プール方式（フェーズ6-2 で再設計）。
kind 6 種（`SLOT_KINDS`、`scripts/data/item/common/extensible.mjs`）：`normal`（スロット）/ `surface`（表層意識）/ `deep`（深層意識）/ `unconscious`（無意識）/ `software`（ソフトウェア）/ `hardware`（ハードウェア）。
装備側はスロット数のみを設定。オプション品との連携先 kind は `parentSlotKind` で記録。

## ヘルパー関数（`scripts/data/item/helpers.mjs`）

- `defenceField()` — armor/cyborg/vehicle 共用
- `attackField()` — weapon/cyborg 共用
- `modeValueField(choices?)` — `{mode,value}` SchemaField（全アウトフィット型で共用）
- 部位まわりの純関数は `scripts/data/item/part-helpers.mjs`（フェーズ10-1）: `computePartOccupancy` / `computeHostOccupancy` / `formatPartDesignation` / `joinPartDesignations`（コンバイン併記は「部位1+部位2」）等

## 参照

- DataModel: `scripts/data/item/*.mjs`
- カテゴリ定義: `scripts/data/item/outfit-categories.mjs`
- 装備品ルール: [Outfits.md](../Game_Rules/Outfits.md)
- 設計レビュー: [Design_Review_Entries.md](../../02_System/Design_Review_Entries.md)（B-5〜B-7 / 6-0〜6-6）
