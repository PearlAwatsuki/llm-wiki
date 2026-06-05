# Item 型定義

全 17 type が `scripts/data/item/` に DataModel 化済。

## 共通 Template

| Template | ファイル | 内容 |
|---|---|---|
| base | `common/base.mjs` | description (HTMLField) |
| usage | `common/usage.mjs` | actions[] — `{ type, name, description }` |
| skillBase | `common/skill-base.mjs` | level, suits (spade/heart/diamond/club), isAction |
| outfitBase | `common/outfit-base.mjs` | isPrepared, isCyber, buy, preserveExp, hide, exclusive, uses 等 |
| extensible | `common/extensible.mjs` | slot[] — `{ slotlabel, value, optionId[] }` |

## Type 一覧

| type | mixin | 主な固有フィールド |
|---|---|---|
| **style** | base | isPersona, isKey, level(1-3), miracle, reason/passion/life/mundane (value/control) |
| **miracle** | base + usage | furigana, usageCondition, isKill/Defence/All/Used, usageCount |
| **generalSkill** | base + usage + skillBase | generalSkillCategory, initialSkill, onomasticSkill |
| **styleSkill** | base + usage + skillBase | unique, style, comboSkill[], timing[], target, range, confrontation[], uses, special/performance/secret/mystery |
| **weapon** | base + outfitBase + extensible | attack, guardValue, range, isthrow, isLaser 等 |
| **armor** | base + outfitBase | defence (S/P/I_defence), controlMod |
| **ianus** | base + outfitBase + extensible | controlMod |
| **cyborg** | base + outfitBase | defence, attack, guardValue |
| **tron** | base + outfitBase + extensible | なし |
| **tap** | base + outfitBase + extensible | cycle, combatSpeedMod |
| **vehicle** | base + outfitBase + extensible | speedFactor, passenger, controlMod |
| **residence** | base + outfitBase + extensible | appearanceTarget, cyberSecurity, *Mod 群 |
| **housingArea** | base | buyRatingMod, preserveExpMod, *Mod 群 |
| **combiner** | base + outfitBase | combinedOutfitID[] |
| **general** | base + outfitBase | なし |
| **organization** | base | なし |
| **lifePath** | base | lifePathType, skillName |

## カスタムシート有無

| カスタムシートあり | コア ItemSheet (未実装) |
|---|---|
| style, miracle, generalSkill, styleSkill, organization | weapon, armor, ianus, cyborg, tron, tap, vehicle, residence, housingArea, combiner, general, lifePath |

未実装 11 種はフェーズ6 で追加予定。

## styleSkill 特有の注意

- template.json 未定義の ★ フィールド 8 個を DataModel で明示定義
- `RewritedTarget`, `RewritingMiracle_ID` — タイポ・命名揺れを意図的に維持
- `unique` の initial は `"none"` (template.json の `""` は誤り)

## usage テンプレート

判定システム (フェーズ7/8) の基盤。**構造変更は設計議論必須**。

- `usage-list.hbs` — edit ボタンのハンドラ未実装 (KI-002)
- `UsageCreationDialog` — 用途作成 UI

## ヘルパー関数 (`scripts/data/item/helpers.mjs`)

- `defenceField()` — armor/cyborg 共用
- `attackField()` — weapon/cyborg 共用

## 参照

- DataModel: `scripts/data/item/*.mjs`
- 設計レビュー: [Design_Review_Entries.md](../../03_System/Design_Review_Entries.md) (B-5〜B-7)
