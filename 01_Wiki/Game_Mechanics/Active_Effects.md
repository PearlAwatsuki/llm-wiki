# ActiveEffect 適用モデル

フェーズ9 の中核設計。**AE（技能効果・アイテム効果・バッドステータス）が実効値にどう流れ込むか**の正本。実装着手（9-1 以降）前にユーザー承認を得る対象（§4.2/§4.3）。

ルール前提は [Character_Creation.md](../Game_Rules/Character_Creation.md)（0clamp）・[Damage_Rules.md](../Game_Rules/Damage_Rules.md)（BS＝AE）・[Time_Management.md](../Game_Rules/Time_Management.md)（持続＝進行単位）。フェーズ全体計画は [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md)。

---

## 0. バフの適用先（最重要分類）

バフは**着地先**で2つに分かれる。これが第一の軸であり、後述のモードA/B は「アイテムへのバフをどう適用するか」の下位の実装手段にすぎない。

| 分類 | 着地先 | 例 |
|---|---|---|
| **キャラクターへのバフ** | `actor.system.*` | 「判定に＋n」「○○ダメージに＋n」など、特定アイテムでなくキャラ本体に乗る修正 |
| **アイテム自体へのバフ** | `item.system.*` | 「武器の攻撃力に＋n」「準備したアウトフィットの危険値に＋n」など、アイテムのパラメータそのものを書き換える修正 |

アイテムの実効値（base＋AE）は**アイテム内で完結する基盤表現**であり、その算出はアイテム自身の `prepareDerivedData` の責務。武器/防具の表示（フェーズ10）・ダメージ式（フェーズ12）はこの実効値を**読むだけ**の消費者であるため、**実効値の表現はすべてフェーズ9 で確定させる**（§11）。

---

## 1. 基本原則

1. **AE は base を破壊しない。** authored 値（スタイル基本値・武器の素の攻撃力など）には触れず、**`effectMod` 系の専用着地フィールドに ADD で積む**。これにより改造前/改造後・バフ有無を可逆に保てる。
2. **実効値は `prepareDerivedData` で一本算出。** `total = base + Σ(各種mod) + effectMod`。能力値/制御値は最後に **0clamp**（[Character_Creation.md](../Game_Rules/Character_Creation.md)）。
3. **着地点が無いパラメータには `effectMod` 相当を新設する**（§4.3 承認案件 → §10）。アクター能力値は既存の `effectMod`/`controlEffectMod` を流用。
4. **OVERRIDE が要る稀なケース**（「攻撃力を5にする」等）は Foundry の change mode（OVERRIDE/UPGRADE/DOWNGRADE）で base 値を直接狙う。既定は **ADD→effectMod**。

> 「アウトフィット効果を直接書き換える」改造・エンチャント系効果も、内部実装は**素の値の上書きでなく effectMod への ADD**。プレイヤーから見た実効値は書き換わるが、authored 値は保持される（D&D のエンチャント＝アイテムに AE を付与する方式と同型）。

---

## 2. 2つの適用モード

Foundry の AE は `change.key` を**その AE が乗るドキュメント自身**にしか解決しない。アクターの AE は `actor.system.*` のみ、アイテムの AE は（transfer に応じ）自分かアクターに効く。**埋め込みアイテム横断は素では不可**。TNX のアイテムバフはこれを 2 モードで吸収する。

### モードA — アイテム自己適用（Foundry ネイティブ）

AE をアイテム自身に乗せ `transfer: false` にすると、そのアイテムの prepareData 中に自分の `system.*.effectMod` へ適用される。

- 用途: **改造**（恒久。改造技能の結果としてアイテムに AE を1枚足す）、**そのアイテム固有の常時ボーナス**。
- 例: ある武器に「攻撃力＋2」AE → `system.attack.effectMod += 2`。
- 実装コスト低（ネイティブ）。9-3 で着地フィールドさえ用意すれば成立。

### モードB — アクター起点の他アイテム／カテゴリ指定（カスタム層）

アクター（やスタイル技能）が**他のアイテム群／カテゴリ**を対象にするバフ。`change.key` では表現できないため**カスタム照合・注入層**を要する。

- 用途: スタイル技能「〈白兵〉武器すべてに＋1」、**区分バフ**（対象社会技能の identificationKey 列挙、論点3）、判定時の達成値ボーナスなど。
- target spec（対象指定）は **identificationKey 列挙 / カテゴリ / 技能種別**などを AE flag に持たせ、照合層が解決する。
- 実装は §3 の順序問題に注意。

---

## 3. prepare 順序とモードB 注入タイミング（重要）

Foundry の Actor 準備順:

```
Actor.prepareData()
  ├ prepareBaseData()            … 素の値
  ├ prepareEmbeddedDocuments()
  │    ├ 各 item.prepareData()    … item.prepareDerivedData まで完了（モードA はここで適用済み）
  │    └ applyActiveEffects()     … アクター自身の AE を actor.system.* へ適用
  └ prepareDerivedData()         … アクター実効値を算出
```

**問題**: アイテムの prepareDerivedData は、アクター AE 適用より**前**に終わる。よってモードB（アクター→アイテム）を素直にアイテム側で読むと間に合わない。

**採用方針**: モードB は **アクターの `prepareDerivedData` 内で「クロスターゲット注入パス」**として実装する。アクター AE 適用後に、所有アイテムを走査し、target spec で一致する AE 寄与を**各アイテムの `system.<param>.total` へ後追いで加算**する（`item.system` はライブオブジェクトなので後調整可能）。これにより `item.system.<param>.total` が常に単一の真実になる（dnd5e 流の actor 側後処理と同型）。

- モードA（自己適用）はアイテムの prepareDerivedData で `base + effectMod` を算出。
- モードB はアクターの prepareDerivedData で各アイテムの total を再調整。
- 判定エンジン・ダメージ算出・シート表示はすべて `item.system.<param>.total` を読むだけ。

---

## 4. バフ対象パラメータ 棚卸し

「現状フィールド」は着地点（effectMod 相当）の有無。**無し→新設が §10 の承認案件**。

| 着地先 | パラメータ | バフ源の例 | 着地（提案） | モード | 現状着地 | 表現/消費フェーズ |
|---|---|---|---|---|---|---|
| キャラ | reason/passion/life/mundane 値・制御 | スタイル技能・状況修正 | `effectMod`/`controlEffectMod`（既存） | A(自) / B | **有** | 9 |
| キャラ | 判定達成値（「判定に＋n」） | 技能効果・コンボ | 判定時収集 or actor の判定 mod | A(自) / B | 無 | 9（基盤）/判定連携 |
| キャラ | ダメージ修正（「○○ダメージに＋n」） | damageBoost 等 | actor 側 damageMod | A(自) / B | 無 | 9（表現）/12（消費） |
| キャラ | handMaxSize（User権威） | 手札上限バフ（KI-020） | `handMaxSizeMod`（要設計） | B＋User合流 | 無 | 9 |
| キャラ | baseAttack/baseDefence/baseGuard・combatSpeed | 地力・素手攻撃 | 各 effectMod 相当（CS はスナップショット方針） | A(自) / B | 一部 | 9（表現）/10・12（消費） |
| アイテム(weapon) | attack（攻撃力） | 武器強化・改造・スタイル技能 | `attack.effectMod` | A / B | 無 | **9** |
| アイテム(weapon) | guardValue（受け値）・FAValue | 改造 | 各 `effectMod` | A | 無 | **9** |
| アイテム(armor) | defence S/P/I（防御力） | 防具強化・改造・スタイル技能 | `defence.*_effectMod` | A / B | 無 | **9** |
| アイテム(armor) | controlMod（制御修正） | 改造 | `controlMod.effectMod` | A | 無 | **9** |
| アイテム(各アウトフィット) | appearancePenalty（危険値） | 「準備したアウトフィットの危険値に＋n」 | `appearancePenalty.effectMod`（条件: isPrepared 等） | A / B | 無 | **9** |
| アイテム(cyborg/vehicle) | attack / defence | 改造 | weapon/armor と同様 | A / B | 無 | **9** |
| アイテム(skill) | level（=使用スート数） | 稀（条件達成用） | `level.effectMod` 等 | B | 無 | 9（必要時） |

> 「表現/消費フェーズ」: **実効値の表現（base＋AE→total）はすべてフェーズ9**。表示（10）・ダメージ式（12）はその total を消費するだけ（§11）。
> アウトフィットの `controlMod` / `combatSpeedMod` / `appearancePenalty` は**アクターの outfitMod / appearanceModifier に集約**される派生入力（B-2 でライブ算出）。集約（9-2）は**アイテムの AE 込み実効値 total を読む**必要があるため、アイテム側 effectMod（9-3）が前提（9-2 と 9-3 は連動）。アイテム自体へのバフはモードA でアウトフィット側 effectMod に積み、集約は調整後の値を拾う。

---

## 5. 改造・エンチャントモデル（モードA／恒久）

- **改造技能による改造**は、対象アウトフィットに **AE を1枚追加**する操作（`transfer:false`、effectMod へ ADD）。アイテムの authored 値は不変。
- 改造の取り外し＝その AE の削除。可逆。
- スタイル技能が持つ「この武器の攻撃力を上げる」恒常効果も同型（アイテムに AE）。
- 付与 UI（改造技能の判定→AE 生成）は購入・改造判定 UI（フェーズ17）と連携。**フェーズ9 では「アイテムに乗る effectMod 着地 AE が実効値に反映される器」までを作る**。

---

## 6. target spec / change.key 規約（モードB）

- モードB の AE は flag（例: `flags.tokyo-nova.aeTarget`）に**対象指定**を持つ:
  - `byIdentificationKey: [...]`（技能・アイテムを identificationKey で名指し。区分バフはここに区分内の公式キーを列挙）
  - `byItemType: ["weapon", ...]` / `bySkillTrait: "action"` 等のカテゴリ指定
  - `param: "attack" | "value" | ...`（どのパラメータに効くか）と `value`（加算量）
- 照合層（アクター prepareDerivedData）が spec を解決し、一致アイテムの total へ加算。
- 素の Foundry `changes[]` は**アクター自身・自己アイテム（モードA）にのみ使う**。モードB は changes でなく flag ＋カスタム照合で表現（changes はキー解決できないため）。

---

## 7. 持続時間フラグ（論点4 連携）

- 各 AE は進行単位フラグを持つ（`flags.tokyo-nova.duration = { unit: "act"|"scene"|"cut"|"mainprocess" }`）。
- 各単位の「開始」でカウンタを進め、AE は生成時カウンタを記録、進めば失効（Foundry duration 同型）。
- **フェーズ9 はこのフラグ（スキーマ）の確定まで**。境界の実発火・カウンタ増加は 13/14/15。

---

## 8. バッドステータス（論点2 連携・9-4）

- 11 ステータス（`CONFIG.statusEffects`）を **changes を持つフル AE** 化し、全自動化を目指す。
- 機構前提の効果（AR 減少・タイミング制限・カット終了回復）は**実発火が 13/14/15**。9-4 は器＋現機構で表現可能な分を実装し、残りは積み増す。
- 効果文は**ルール本文でなくユーザー入力テキスト**として AE に持たせる（§4.1）。

---

## 9. 0clamp の適用点

- 能力値・制御値の `total` は `max(0, …)`（全 mod 合算後・順序非依存、[Character_Creation.md](../Game_Rules/Character_Creation.md)）。
- 増減ペアの矛盾（clamp でデメリットだけ消える等）は**自動化しない**（手動 AE 調整）。

---

## 10. DataModel 変更提案（§4.3 承認案件）

実装前にユーザー承認が必要な**着地フィールド新設**の一覧（すべてフェーズ9 で配線。アイテム実効値は表現＝9 のため、§11）:

- weapon: `attack.effectMod`、`guardValue.effectMod`
- armor: `defence.S/P/I_effectMod`、`controlMod.effectMod`
- cyborg / vehicle: 上記に準ずる
- skill-base: `level.effectMod`（必要時）
- cast: `baseAttack/baseDefence/baseGuard` の effectMod 相当、`handMaxSizeMod`（KI-020）

> 命名は `effectMod` 接尾辞で能力値（既存 `effectMod`）と一貫させる。helpers の `attackField()`/`defenceField()` に effectMod を足すか、派生専用の別構造にするかは実装時に確定（承認後）。

---

## 11. フェーズ分担（どこまでが9）

**原則: 「実効値の表現（base＋AE）」はすべてフェーズ9。「その実効値の消費（表示・式）」が後続フェーズ。** アイテムの実効値（AE 込み）はアイテム自身に完結する基盤表現であり、武器/防具のデータ表示（10）・ダメージ式（12）はそれを**読むだけ**になる。表現を後回しにすると表示・戦闘を素の値で組んでしまい AE を後付けする逆順になるため、フェーズ9 で確定させる（2026-06-20 ユーザー指摘により旧「歩調を合わせる」案を撤回）。

- **フェーズ9（実効値の表現を確定）**: 派生パイプライン、AE→effectMod 着地、モードA/B の適用機構、クロスターゲット注入パス、0clamp、**全 buff 対象アイテムパラメータ（weapon.attack/guardValue, armor.defence/controlMod, cyborg/vehicle, outfit.appearancePenalty 等）の effectMod 着地＋実効値 total**、KI-020、BS の AE 表現と持続フラグ、効果文入力。
- **フェーズ10（戦闘タブ）**: 武器・防具の実効値 total を**表示**。
- **フェーズ12（ダメージ）**: ダメージ式が攻撃力・防御力の total を**消費**。
- **フェーズ13/14/15**: 持続時間の境界発火・カウンタ。
- **フェーズ17**: 改造判定 UI から改造 AE（モードA）を生成。

---

## 参照

- [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md)
- [Character_Creation.md](../Game_Rules/Character_Creation.md) / [Damage_Rules.md](../Game_Rules/Damage_Rules.md) / [Time_Management.md](../Game_Rules/Time_Management.md)
- [DataModel.md](../Architecture/DataModel.md)
- [Judgment_System.md](Judgment_System.md) — 判定時のバフ収集連携
</content>
