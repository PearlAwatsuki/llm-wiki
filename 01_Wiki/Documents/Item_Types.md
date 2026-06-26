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
| isPrepared | BooleanField | 準備済み（即使用可能） |
| isCarrying | BooleanField | 携帯中（アクトに持参） |
| isCyber | BooleanField | サイバーウェア |
| `isPre-play` | BooleanField | プレアクト購入 |
| isOption | BooleanField | 別アウトフィットに装備するオプション品 |
| isConsumption | BooleanField | 消費アイテム |
| quantity | SchemaField | `{value, max}` — 消費アイテムの現在/最大個数 |
| buy | SchemaField | `{mode: none/value/reference, value}` — 購入値 |
| preserveExp | SchemaField | `{mode: none/value, value}` — 常備化経験点 |
| hide | SchemaField | `{mode: none/value/reference/control, value}` — 隠匿値 |
| appearancePenalty | SchemaField | `{mode: none/value, value}` — 危険値 |
| hack | NumberField | 電脳制御値（nullable。null = なし） |
| timing | SchemaField | `{value, actionName, processName, timingOther}` — タイミング単一（6-3 で配列から変更） |
| exclusive | StringField | 専用（自由記述） |
| part | ArrayField | `[{value, slots}]` — 装備部位（複数可） |
| uses | SchemaField | `{type, isLimit, max, value}` — 使用回数 |
| majorCategory | StringField | 大分類（choices 付き）|
| minorCategory | StringField | 小分類（choices 付き、大分類と Item type で絞込） |
| identificationKey | StringField | 識別キー（全アイテム型共通） |
| parentSlotKind | StringField | 親アウトフィットのスロット種別（オプション品用） |
| combineGroupId | StringField | コンバイン連携グループ ID |

> `isBiological` はフェーズ6-4 で削除済み。生体装備はカテゴリ「生体装備」で識別。

## Type 一覧

| type | mixin | 固有フィールド（主要） |
|---|---|---|
| **style** | base | isPersona, isKey, level(1-3), miracle, reason/passion/life/mundane (value/control) |
| **miracle** | base + usage | furigana, usageCondition, isKill/Defence/All/Used, usageCount |
| **generalSkill** | base + usage + skillBase | generalSkillCategory, initialSkill, onomasticSkill |
| **styleSkill** | base + usage + skillBase | unique, style, comboSkill[], timing[], target, range, confrontation[], uses, special/performance/secret/mystery |
| **weapon** | base + outfitBase + extensible | attack `{damageType, value}`, guardValue `{mode,value}`, range `{min,max}` |
| **armor** | base + outfitBase | defence `{mode,S_defence,P_defence,I_defence}`, controlMod `{mode,value}` |
| **ianus** | base + outfitBase + extensible | controlMod `{mode,value}` |
| **cyborg** | base + outfitBase | defence, attack `{damageType, value}`, guardValue `{mode,value}` |
| **tron** | base + outfitBase + extensible | なし |
| **tap** | base + outfitBase + extensible | cycle `{mode,value}`, combatSpeedMod `{mode,value}` |
| **vehicle** | base + outfitBase + extensible | speedFactor `{mode,value}`, passenger `{mode,value}`, controlMod `{mode,value}`, attack, defence |
| **residence** | base + outfitBase + extensible | appearanceTarget, cyberSecurity, useHousingAreaDrop |
| **housingArea** | base | area（セキュリティランク）, buyRatingMod, preserveExpMod, slotMod, *Mod 群（hideMod は廃止） |
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
- `RewrittenTarget` / `rewritingMiracleId` — 旧 typo 名（`RewritedTarget` / `RewritingMiracle_ID`）から正規化済み（KI-018/019 解消・参照ゼロのため移行不要）
- `unique` の initial は `"none"`（template.json の `""` は誤りを修正済み）
- `timing[]` はアウトフィットの `timing`（単一 SchemaField）と**別物**（styleSkill は複数可の配列）

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

## 参照

- DataModel: `scripts/data/item/*.mjs`
- カテゴリ定義: `scripts/data/item/outfit-categories.mjs`
- 装備品ルール: [Outfits.md](../Game_Rules/Outfits.md)
- 設計レビュー: [Design_Review_Entries.md](../../02_System/Design_Review_Entries.md)（B-5〜B-7 / 6-0〜6-6）
