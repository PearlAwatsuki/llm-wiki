# usage (用途) システム

判定・効果発動の**データ基盤**。D&D 5e の「用途」設計思想を踏襲。

## データ構造

`UsageTemplate` (`scripts/data/item/common/usage.mjs`):

```javascript
actions: ArrayField(SchemaField({
    _id:         StringField,           // ランダム ID (randomID())。インデックス参照を廃止
    type:        StringField,           // 種別（作成時固定。下記参照）
    name:        StringField,
    description: StringField,

    timing: SchemaField({
        value:       StringField,       // "blank" | "action" | "process" | "reaction" | "other" | ...
        actionName:  StringField,       // timing.value === "action" の場合のサブ選択肢
        processName: StringField,       // timing.value === "process" の場合のサブ選択肢
        timingOther: StringField,       // timing.value === "other" の場合の自由入力
    }),

    // 対象・射程・目標値（スタイル技能と同形の flat フィールド。発動タブで設定）
    target:        StringField,         // "blank" | "self" | "single" | "team" | ...
    targetOther:   StringField,         // target === "other" の自由入力
    isFixedTarget: BooleanField,        // 変更不可（※）: AE による対象変更を抑止（実機能はフェーズ13）
    range:        StringField,          // "blank" | "close" | "short" | ... | "weapon"
    rangeOther:   StringField,
    isFixedRange: BooleanField,         // 変更不可（※）
    targetValue:       StringField,     // "blank" | "number" | "control" | "total" | ...
    targetValueNumber: NumberField,
    targetValueOther:  StringField,

    isUnopposable: BooleanField,        // 対決不可: 対象がこの判定に対決できない（実機能はフェーズ13、現状は保持のみ）

    effects: ArrayField(SchemaField({
        effectId: StringField,          // この用途使用時に付与する ActiveEffect の id
    })),

    baseSkillRef: SchemaField({
        itemId: StringField,            // ベース技能の Item ID。generalSkill 新規作成時に親 ID で自動設定
    }),

    skillRefs: ArrayField(SchemaField({
        itemId: StringField,            // 組み合わせ技能（ベース技能は含まない）の Item ID
    })),

    // attack タイプのみ
    weaponRef: SchemaField({ itemId: StringField }),
    damageType: StringField,            // "S" | "P" | "I"

    // damageBoost / damageReduce タイプのみ
    formula: StringField,               // 効果量（計算式または固定値文字列）
    damageCategory: StringField,        // "physical" | "mental"

    // modification タイプのみ
    modifiableParams: ArrayField(StringField),
}))
```

### タイプ（type）値域

| type | 説明 | 主な固有フィールド |
|---|---|---|
| `check` | 技能判定（情報収集・登場など） | baseSkillRef, skillRefs |
| `attack` | 攻撃判定（物理・精神・社会戦） | baseSkillRef, skillRefs, weaponRef, damageType |
| `declaration` | 宣言・神業を含む | — |
| `damageBoost` | ダメージ増加 | formula, damageCategory |
| `damageReduce` | ダメージ軽減 | formula, damageCategory |
| `modification` | 改造 | modifiableParams |

- `controlCheck` は独立した判定種別であり、usage タイプには含まない（timing で区別）
- 神業（miracle）は当面 `declaration` に統合。専用システム分離は将来フェーズ
- タイプは作成時固定（UI 上で切り替え不可）

### タイミング / 対象の "-"（blank）の意味

`timing.value = "blank"` / `target = "blank"` は「指定なし」= **どのタイミング・対象とも組み合わせ可能**を意味する（自由タイミング・自由対象）。空欄でも判定は正常に起動できる。

### migrateData

既存データに `_id` が無い場合は `randomID()` で自動付与し、`baseSkillRef` が無い場合は `{ itemId: "" }` を追加する（`Usage_Template.migrateData`）。

## 使用 Item type

`miracle`, `generalSkill`, `styleSkill` が UsageTemplate を mixin。

## UI コンポーネント

| コンポーネント | ファイル | 状態 |
|---|---|---|
| usage-list.hbs | `templates/parts/usage-list.hbs` | `data-usage-id` ベース。種別タグ表示 |
| TnxUsageSheet | `scripts/module/tnx-usage-sheet.mjs` | ApplicationV2、タブ構成、KI-002 解決 |
| usage-sheet.hbs | `templates/app/usage-sheet.hbs` | TnxUsageSheet 本体テンプレート |

### TnxUsageSheet の設計

- **タブ（3構成、2026-06-20 改編）**: 基本 / 発動 / 効果
  - **基本**: 説明
  - **発動**（旧「起動」）: タイミング / 対象（＋変更不可※）/ 射程（＋変更不可※）/ 目標値 / 対決不可。技能ベース用途（check・attack）では参加技能の対決を読み取り「対決可能技能」を読み取り専用表示し、「参加技能から自動入力」ボタンを置く
  - **効果**: 種別固有設定（check=コンボ / attack=武器・ダメージ種別・コンボ / damageBoost・Reduce=計算式・カテゴリ / modification=改造パラメータ）＋「適用される効果」（ActiveEffect）。旧「種別固有」タブと「エフェクト」タブを統合
  - ベース技能＋組み合わせ技能の UI は `usage-sheet-combo.hbs` パートに切り出し（check / attack で共用。`tnx.mjs` の `loadTemplates` に登録）
- **発動パラメータの自動入力**（`_onAutoFill`）: 参加技能（ベース＋コンボ）の**固有値**（スタイル技能の skill レベル `target`/`range`/`targetValue`/`timing`/`confrontation`。一般技能は固有値を持たず制約なし＝寄与しない）を読み、優先度で合成して用途へ書き込む。対象は対象優先度、射程は射程優先度（※複数は最短）、目標値は数値最大、タイミングはベース技能の先頭、対決不可は参加技能に「対決不可」があれば true。常時上書きせずボタン式（誤更新防止）。手編集で上書き可能
- **対決（情報表示）**: 参加技能の `confrontation`（値=技能名/技能名※）を読み取り専用で表示。相手側リアクションの自動解決・対決値統合はフェーズ13。書き込み用の対決フィールドは usage に持たせない（`isUnopposable` フラグのみ保持）
- `submitOnChange: true` — フォーム変更で即時 `_patchUsage()` を呼び自動保存
- `_patchUsage()` は `foundry.utils.setProperty` でドット記法キーを処理（`mergeObject` は配列でバグあり）
- **skillRefs 追加**: ドロップダウン選択で即時追加（「追加」ボタン廃止）
- **effects 管理（エフェクトタブ）**: D&D の「適用される効果」方式。セレクトボックスでアイテムの ActiveEffect（未追加のもの）を選ぶと `usage.effects` に `{effectId}` を追加。追加済みは行表示＋「解除」ボタンでデタッチ（効果自体は削除しない＝アイテムシートのエフェクトタブの責務）。アイテムに ActiveEffect が無い場合も空のセレクトを表示。`addedEffects` / `availableEffects` / `hasAnyEffect` をコンテキストに供給（旧チェックボックス羅列方式を 2026-06-20 に置換）
- **アクション技能**: ベース技能を親技能で表示固定（セレクター非表示）
- **ベース技能候補** (`availableBaseSkills`): アクション技能を含む全技能（自身以外）。非アクション技能もベースになりうる
- **コンボ候補** (`availableSkills`): アクション技能を除外（アクション技能は必ずベースになるため）
- **スタンドアロンアイテム対応**: `actor` が null の場合も `baseId === parentItemId` なら親アイテムの名前を表示

## generalSkill 新規作成時の自動用途挿入

`preCreateItem` フック（`scripts/tnx.mjs`）:
- `data.type === "generalSkill"` かつ `system.actions` が空の場合に `check` 用途を1件自動挿入
- `baseSkillRef.itemId` には `item._id` を設定（親技能が自身をベースとして保持）

## 判定起動時のコンボ解決

`_onStartSkillCheck` (`scripts/actor/tnx-cast-sheet.mjs`):

1. 技能行クリック → `check` タイプの用途を解決
2. `baseSkillRef.itemId` からベース技能を決定（未設定時は親アイテム ID にフォールバック）
3. `skillRefs` からコンボ技能を収集
4. 用途所持技能がベース技能でない場合は自動的にコンボに追加（非アクション技能からの起動）
5. 全技能スートの積集合で `validSuits` を計算し `TnxCheckFlow.open()` へ渡す

## スタイル技能の「技能」欄(comboSkill)の意味（2026-06-28 ユーザー言語化／解決は下記「組み合わせ解決」で実装済み）

スタイル技能の「技能」欄(`comboSkill`)は、その技能が組み合わせる相手（指定技能）を表す。値の種類と意味：

- **なし**: 単体で使用できる（が、他技能との組み合わせも可能）。
- **単独**: 他技能と組み合わせできない。ただし効果文で「他技能と組み合わせできない」と書く技能（＝`noCombo`）と異なり、「［技能：単独］とも組み合わせ可能」とされる技能が時折あるため、**単独からの組み合わせ不可の自動化は非推奨**。
- **宣言**: 宣言のみで使用、判定を行わない＝組み合わせも無い。
- **ドッジ**: ドッジ判定と組み合わせ可能。基本は〈回避〉だが「この判定でドッジを行える」技能もあるため、ドッジでありさえすれば技能は問わない。
- **パリー**: 組み合わせ先がパリーである以外はドッジと同じ。
- **技能名**: 特定の技能を指定（identificationKey で格納・辞典カスケードで選択）。
- **任意**: 任意の一般技能と組み合わせる（組み合わせの中に一般技能を1つ以上含む必要がある）。

### カテゴリ全体指定（whole-category）

「技能」欄は単一技能名でなく**カテゴリ全体**を指定することがある。単一技能名でないので**ベース自動固定はしない**が、**ベース／組み合わせ技能選択の絞り込み対象**になる。

- **固有名詞技能のカテゴリ全体**（〈社会〉〈芸術〉等）: カスケードの各小分類リスト（社会/操縦/製作/芸術/コネ）の**先頭**に「カテゴリ全体」エントリを置く（ユーザー入力で表現可能）。
- **スタイル技能の例外的カテゴリ全体**（コードにハードコード）: 〈元力〉＝識別キープレフィックス `element_`（バサラ）、〈血脈〉＝`bloodline_`（アヤカシ）。スタイル選択（P2b）後、**そのスタイルの技能リストに該当プレフィックスの技能があれば、そのスタイルのリスト先頭**に置く（リスト全体の先頭ではない）。

### 組み合わせ解決（2026-06-29 実装済み）

解決は**判定起動時でなく用途設定時**に行い、結果（baseSkillRef/skillRefs）を保存する。判定起動は解決済みの用途を使う（ルール解釈は [Audit_and_Review.md](../../02_System/Audit_and_Review.md)）。純関数は `scripts/module/skill-chain-resolution.mjs`、用途エディタ配線は `scripts/module/tnx-usage-sheet.mjs`。

- **技能チェーン**: 「技能」が単一技能名 X を指す間 X を辿る（例 S(技能:守護天使)→守護天使(技能:心理)→心理）。「&」結合（両方必須）も単一名と同様に自動固定対象。「、」結合（いずれか1つ）は候補制限で扱う。
- **ベース**: チェーン（起点＋足した組み合わせ技能の連鎖）全体に**アクション技能があればそれがベース**（アクション技能の「技能」(組み合わせ相手)も辿る）。無ければ末端がベース。&全員非アクションは手動。**アクション技能はベース欄以外に絶対入らない**（コンボから一律除外）／**アクション同士は組み合わせ不可**（参加が2つ以上になる追加はブロック）。
- **ベース自動固定はハードロックでなく候補制限**: 単一技能名（カテゴリ全体は対象外）は「指定技能＋その代用」候補から選択（本体優先で既定・候補が増えれば選べる）。
- **代用**: 各段で本体優先・無ければ代用で解決（代替先プロパティは引かない＝自前のスート・報酬点フラグ）。代用がアクションの代替ならベース候補。
- **スート**: 連鎖全体の積（共通スート無し＝判定不可）。候補絞り込みは共通スート。
- **個数制限**: 組み合わせ技能数 ≤ ベース技能のレベル（ベース込みで level+1）。追加時に連れ込み分＋アクション連れ込みでのベース入れ替わりを反映して事前ブロック。ベース変更での超過はトリムダイアログで調整（必須は外せない・取り消しで元のベースへ）。
- comboSkill / confrontation の技能名表示は識別キーから**都度逆引き**（`tnx-skill-utils.mjs`・保持名キャッシュなし）。

## 使用回数の消費（2026-06-20）

判定起動時に参加技能の使用回数を消費する（ルール: [Check_Rules.md](../Game_Rules/Check_Rules.md#使用回数の消費確定-2026-06-20)）。

- 使用回数は D&D 方式の **`uses.spent`（消費済み）** で保持（残り = `max − spent`）。旧 `uses.value`（残り回数）から `migrateUsesValueToSpent()` で移行（`helpers.mjs`、style-skill / outfit-base 共用）。
- `TnxCheckFlow.planUsesConsumption(actor, skillIds)`: `_onStartSkillCheck` が `open()` の前に呼ぶ。`uses.isLimit` を持つ参加技能があれば D&D 方式の消費ダイアログを表示。チェック済みで残り（`max − spent`）0なら起動不可（null 返却）。確定した消費対象を `open()` の `consumeUses` として渡す
- `TnxCheckFlow._consumeUses(actor, skillIds)`: `_execute` 内（カードプレイ直後）で `updateEmbeddedDocuments` により各技能の `uses.spent` を +1（max 上限）。組み合わせ技能は別アイテムのため遠隔消費
- 消費の確定は起動時、減算は実行時。カード選択キャンセル時は `_execute` 未到達＝非消費

## 技能の役割(skillRoles)と用途モデルの再設計（2026-07-09 ユーザー方針・進行中）

「デフォルトの使用技能が決まっている行動」(治療=医療・ドッジ=回避・パリー=白兵・各リアクション・各攻撃)を、**技能名一致でなく技能側の役割フラグ**で扱う。

- **概念(ユーザー確定)**: スタイル技能の効果は「**所定の行動を、規定以外の技能をベースにして行える**」ことであり、対象技能に「その行動が可能になる効果を付与」するのではない。よって規定の技能‑行動対応は技能の恒常属性＝役割で持つ。代替ベースのスタイル技能は**そのスタイル技能自身が役割を持つ**(他技能のフラグは立てない)。
- **`skillRoles`**(役割キー配列・SkillBaseTemplate)を技能に追加。`skill-roles.mjs`: SKILL_ROLES(dodge/parry/mentalReaction/socialReaction/meleeAttack/rangedAttack/mentalAttack/socialAttack/treatment)・`getSkillRoles`(フィールドが権威・未設定の既定技能は正準名の既定値をブリッジ)・`actorSkillsWithRole`。役割付き技能は基本アクション技能、例外は医療(非アクションだが治療役割)。
- **ヴィークルの操縦**: 搭乗中はドッジ=操縦のみで回避を**ブロック**する必要がある。役割の宣言(加算的)だけでは表せず、**文脈(搭乗状態)による差し替え/ブロック**は状態管理=フェーズ13。操縦に dodge 役割は持たせておき、ブロック層は状態が入ってから。
- **用途モデルの一本化(方針・未実装)**: 「他者を攻撃する白兵判定」も「別用途の白兵判定」もルール上同じ判定であり、*attack* は判定の種類でなく**帰結(攻撃として使うか)**。よって attack 用途タイプを廃し、**判定(check)に一本化**する予定(攻撃固有=武器/ダメージは check の任意プロファイル・役割で行動を決定)。**declaration(宣言)は判定なしで使える能力を表すため残す**(ユーザー明言)。
- **用途エディタのレイアウト刷新(予定)**: 狭いウィンドウに設定項目が縦積みで見辛い。フィールド構成の変更に合わせて組み直す。

段階: ① skillRoles＋役割ベース検出(治療・リアクション)=**実装済み `ad34b3a`** → ② attack→check 一本化＋データ移行 → ③ 用途エディタ刷新＋skillRoles 編集UI。

## 変更制約

CLAUDE.md §4.3 より:

- `usage.mjs` のフィールド追加・改名は事前承認を得る
- `usage-list.hbs`・`TnxUsageSheet` の構造変更は設計議論を経てから

## miracle 連動（既存動作、改変禁止）

- style レベル変更 → miracle usageCount 増減 (`preUpdateItem`)
- style 削除 → 連動 miracle 削除 (`preDeleteItem`)
- miracle 削除 → usageCount チェックで抑制 (`preDeleteItem`)

## 参照

- DataModel: `scripts/data/item/common/usage.mjs`
- 編集シート: `scripts/module/tnx-usage-sheet.mjs`
- 判定起動: `scripts/actor/tnx-cast-sheet.mjs` `_onStartSkillCheck`
- 解決済み: KI-002（usage edit ハンドラ不在） → [Resolved_Issues.md](../Issues/Resolved_Issues.md)
