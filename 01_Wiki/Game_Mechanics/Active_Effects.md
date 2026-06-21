# ActiveEffect 適用モデル（確定設計 v2・実装済み）

フェーズ9 の中核。**標準の ActiveEffect 設定 UI（変更タブ＝キー＋モード＋値）だけで完結**させ、マクロや非存在フィールドに依存しない。2026-06-21 に effectMod 方式から**「total 直接適用」方式**へ再設計し、同日に実装完了（ユーザー確定）。

ルール前提: [Character_Creation.md](../Game_Rules/Character_Creation.md)（0clamp）・[Damage_Rules.md](../Game_Rules/Damage_Rules.md)・[Time_Management.md](../Game_Rules/Time_Management.md)。フェーズ計画: [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md)。

> **用語**: 「判定」は英語で **check**（TRPG 標準。judge/judgment ではない）。キーも `check.` / `controlCheck.` を用いる。

---

## 0. 大原則

1. **base（UI 編集値）と total（実効値）を分離**。UI が編集するのは base（value / growth / mod 等）、表示・判定・ダメージが読むのは total。
2. **エフェクトは total に直接、ネイティブのモードで効く**（ADD / OVERRIDE / MULTIPLY / UPGRADE / DOWNGRADE）。effectMod のような加算専用の中間枠は持たない（OVERRIDE/MULTIPLY が表現できないため**全廃済み**）。
3. **対象・パラメータは「変更（changes）のキー」で表す**。標準 UI に存在するのはキー（テキスト）＋モード（選択）＋値（テキスト）のみ。これ以外のフィールドは前提にしない（例外: 重複可チェックボックスは詳細タブに自前注入＝§5）。
4. Foundry の Item は自己効果を適用せず、change.key は他アイテムへ解決できない。よって**アクターの単一適用パスが全 buff を集めて total へ適用**する（`CONFIG.ActiveEffect.legacyTransferral=false`、ネイティブ applyActiveEffects には依存しない）。横断バフは **transfer（効果をキャラクターに適用）OFF でも反映**される。

## 1. 注入点は2つ

| 注入点 | 対象 | タイミング | キー接頭 |
|---|---|---|---|
| **恒常（適用パス）** | キャラ値・アイテム値 | prepareDerivedData で base→total を出した後、アクターの適用パスが total を改変 | `system.` |
| **判定時（check）** | 判定の達成値・制御値への加算 | 判定実行時にエフェクト単位で評価 | `check.` / `controlCheck.` |

## 2. 変更キーの文法（標準 UI で打つ・実装済み）

`parseEffectTargetKey(key)`（`scripts/data/item/helpers.mjs`）が解析する。

### 値バフ（恒常・`system.<名前空間>`）

| キー | 対象 → 着地 total |
|---|---|
| `system.ability.reason` | アクター能力値 → `reason.total` |
| `system.control.reason` | アクター制御値 → `reason.totalControl` |
| `system.self.attack` | 効果が乗るアイテム自身 → `attack.total` |
| `system.parent.attack` | 効果が乗るアイテムの親アウトフィット → `attack.total` |
| `system.category.melee.attack` | 小分類 melee のアイテム群 → `attack.total` |
| `system.category.weapon.attack` | 大分類 weapon のアイテム群（大小キーは衝突しない） |
| `system.skill.melee.level` | 識別キー melee の技能 → `levelTotal` |
| `system.skill.society_*.level` | 識別キー前方一致の技能群 |
| `system.handMaxSizeMod` | ネイティブキー（KI-020 特例。所有 cast に着地→User へ合算） |

- パスは内部で total 系へ解決（`resolveItemTotalPath`）: `attack`→`attack.total`、`defence.S`→`defence.S_total`、`level`→`levelTotal`、`FAValue`→`FAValueTotal`、その他→`<param>.total`。base は不変。
- 識別キー（技能）と `category`（アウトフィット）で住み分け（アウトフィットは識別キーで個体指定しない方針）。

### 判定バフ（判定時・`check.` / `controlCheck.`）

| キー | 対象 |
|---|---|
| `check.melee` | 技能判定（識別キー完全一致） |
| `check.society_*` | 技能判定（プレフィックス前方一致） |
| `check.reason` | 能力値判定（reason/passion/life/mundane） |
| `controlCheck.reason` | 制御判定 |

- 末尾にパラメータを付けない技能キー＝その技能の判定への加算（`.judgment` 等の接尾は不要）。

### 条件付き（標準キーに畳み込む）

`<セレクタ>[<条件>].<パス>`。角括弧内に `パス 比較 値`、複数は `;` 区切り。`parseEffectConditions` / `evalEffectConditions`。

- 例: `system.category.melee[hack>=3].attack` = 電制 total が 3 以上の白兵武器の攻撃力。
- 比較演算: `>= <= > < == !=`。左辺は対象アイテムの system パス（`hack`→`hack.total` 等に解決）。
- 条件はキー文字列内で完結（標準 UI のみで書ける）。

### モード・値

- モード = 標準の Add / Override / Multiply / Upgrade / Downgrade。値 = 数値。
- 複数効果は `change.priority`（既定 mode×10）順に適用。

## 3. 恒常適用パス（`CastDataModel.prepareDerivedData` → `_applyEffectBuffs`）

```
1. 各アイテム prepareDerivedData: computeItemEffectiveValues で total=base を算出（エフェクト抜き）
   能力値: total = growth + Σ(style×level) + mod + outfitMod（effectMod 項は廃止）
2. _applyEffectBuffs:
   a. アクター自身＋全所有アイテムの effects から有効な変更を集める（transfer 非依存）
   b. parseEffectTargetKey でセレクタ/条件/パス解析。check 系（abilityCheck/controlCheck/skillCheck）は除外
   c. _resolveBuffApplications で対象（ability/control/self/parent/category/skill）と total パスへ展開、条件評価
   d. 同一効果の重複排除（§4）→ priority 順に effect.apply で total へモード適用
3. 能力値・制御値は最後に 0clamp
```

- `effect.apply(targetDoc, {...change, key: "system.<path>"})` でモード処理をネイティブ再利用。
- self/parent はそのエフェクトの親アイテム（と parentItemId）で解決。所持時のみ反映。

## 4. 同一効果の重複適用不可（値・判定 共通の一般原則）

トーキョーN◎VA では、**明記されない限り同一効果は重複しない**（ごく稀に重複可と明記される）。値バフ・判定バフの**両方**に適用する。

- **同一性の判定子 = 効果の安定 ID**: `flags["tokyo-nova-axleration"].effectId` があればそれ、無ければ ActiveEffect の document `_id`。名前変更に耐える（辞典が effectId を振れば、コピー同士も同一効果として重複排除可能）。
- **重複排除の規則**:
  - 非 stackable の効果は **identity ごとに1回だけ**適用。同一 identity 内で複数の変更が該当しても**最も有利な値（最大）を1回**。
  - **stackable=true** の効果は重複排除しない（常に加算）。
- **値バフ**（`_applyEffectBuffs`）: 非 stackable は `(対象ドキュメント, total パス, モード, identity)` ごとに最大値1つだけ適用。
- **判定バフ**（`computeCheckBonus`）: 非 stackable は identity ごとに最大値。別 identity はスタック。
  - 「バサラ/マヤカシに+1、組み合わせても+1」= 同一エフェクトが複数技能に該当 → 1回（プレフィックスが2技能に当たっても +1）。

### 判定フローへの配線

`tnx-judgment-flow.mjs` `_computeCheckBonus(actor, ctx, abilityKey)` が effects を集め、判定種別で criteria を作る:
- controlCheck → `{type:"control", ability}`、skillCheck → `{type:"skill", skillKeys:[参加技能の識別キー]}`、それ以外 → `{type:"ability", ability}`。
- `calcSkillCheck` は達成値に `+checkBonus`、`calcControlCheck` は制御値を `+checkBonus`（成功条件が緩む）。報酬点プレビューにも反映。

## 5. 重複可（stackable）チェックボックス UI

標準 ActiveEffectConfig の**詳細タブ**に「重複可」チェックボックスを注入（`renderActiveEffectConfig` フック、`scripts/tnx.mjs`）。change 時に `flags["tokyo-nova-axleration"].stackable` へ `setFlag`（フォーム送信に依存しない）。未チェック＝重複適用不可、チェック＝スタック。

## 6. 構造化の前提（別途・後続）

スタイル/ワークス述語には、スタイル技能の「スタイル」欄・ワークス技能の「組織」欄・専用欄の**構造化（辞典読み取りドロップダウン・キー化）**が前提。ワークス装備フラグも要追加。これらは AE 本体の後の作業。当面は識別キー/プレフィックス/カテゴリで賄う。

分類（カテゴリ）の**コードキー化は実装済**（`category.melee` 等。`outfit-categories.mjs` が `{majorKey:{minors:{minorKey}}}`）。

## 参照

- [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md) / [Judgment_System.md](Judgment_System.md)
- [Character_Creation.md](../Game_Rules/Character_Creation.md)（0clamp）
</content>
