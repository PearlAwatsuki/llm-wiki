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
        itemId:   StringField,          // 効果が乗るアイテム（空=親／組み合わせ技能／使用武器・2026-07-10）
        effectId: StringField,          // この用途使用時に付与する ActiveEffect の id
    })),

    // 判定/攻撃の修正値（2026-07-10）
    checkBonuses:    ArrayField(SchemaField({ formula, source })),  // 供給元つきの追加行
    checkBonusSelf:  StringField,       // 用途自身の判定修正値（専用欄・@item.self=親アイテム）
    damageBonuses:   ArrayField(SchemaField({ formula, source })),  // attack のみ
    damageBonusSelf: StringField,       // 用途自身のダメージ修正値（専用欄・attack のみ）

    baseSkillRef: SchemaField({
        itemId: StringField,            // ベース技能の Item ID。generalSkill 新規作成時に親 ID で自動設定
    }),

    skillRefs: ArrayField(SchemaField({
        itemId: StringField,            // 組み合わせ技能（ベース技能は含まない）の Item ID
    })),

    // attack タイプのみ
    weaponRef: SchemaField({ itemId: StringField }),
    damageType: StringField,            // "S" | "P" | "I"

    // check(攻撃)のみ
    damageCategory: StringField,        // 攻撃系統 "physical" | "mental" | "social"

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
  - **効果**: 種別固有設定（check=コンボ / attack=武器・ダメージ種別・コンボ / modification=改造パラメータ）＋「適用される効果」（ActiveEffect）。旧「種別固有」タブと「エフェクト」タブを統合。※damageBoost/damageReduce の効果量欄は廃止（2026-07-11）
  - **ダメージを増加（`boostDamage`・2026-07-11）**: 〈攻撃〉fieldset の攻撃モード**ラジオ**（なし／攻撃に使う／ダメージ増加＝`attackMode`）で選ぶ（排他はラジオが保証。当初のチェックボックス2つ＋後勝ち判定はフラグとUI表示のずれを生みユーザー指摘で廃止）。ON で〈攻撃〉内に**ダメージ増加（式）**（`damageBonusSelf` 流用）を表示。使用はアイテムロール＝ダメージカード待ち受け中に技能クリック→`useDamageBoost`（damage-flow）が pending へ登録（トグル）→カードプレイと同時に評価・合算・消費適用。正本 [Damage_Rules.md](../Game_Rules/Damage_Rules.md)「ダメージ増加技能」。
  - ベース技能＋組み合わせ技能の UI は `usage-sheet-combo.hbs` パートに切り出し（check / attack で共用。`tnx.mjs` の `loadTemplates` に登録）
- **レイアウト（fieldset グルーピング＋式表示・2026-07-10 ユーザー指摘→改修）**: 各タブ内の項目を **`<fieldset>`＋`<legend>`（既存 `.tnx-fieldset`）でグループ化**（グリッド/縦積みの平坦配置をやめる。D&D アクティビティの意匠の要点＝legend グルーピングを参考）。発動＝〈取得〉〈タイミング〉〈対象・射程〉〈目標値・対決〉〈対決（参加技能より）〉〈使用回数の消費〉／効果＝〈参加技能〉〈判定〉〈攻撃〉〈効果量〉〈改造可能パラメータ〉〈適用される効果〉。
  - **式の表示＋干渉できる入力**: 「判定やダメージの式が全く無い」というユーザー指摘に対応。〈判定〉に **判定ボーナス（式）欄＋達成値の構成**（達成値＝カード＋能力値＋報酬点＋判定バフ＋判定ボーナス）、〈攻撃〉に **ダメージ修正（式）欄＋ダメージの構成**（ダメージ＝ダメージカード＋攻撃力＋修正−軽減）を `.usage-formula-line` で表示（式には**それに干渉する入力を必ず添える**＝判定ボーナス／ダメージ修正）。〈攻撃〉の内側は 2 カラムグリッドでなく**単一カラムの縦積み**（`usage-form-section`。武器の複数選択リストや式が全幅で並ぶべきため）。
  - **判定ボーナス/ダメージ修正（行＋供給元・2026-07-10 ユーザー確定）**: `usage.checkBonuses`/`usage.damageBonuses` ＝ **`{formula, source}` の行の配列**。UI はコンボ/消費と同型の複数行（`usage-sheet-bonus-rows.hbs` パート・`checkBonusFormula-N`/`checkBonusSource-N` を submit で再構成＝consumeTargets 同型・`checkBonusAdd`/`Delete` 等で行操作）。評価は `evaluateBonusRows`（各行を評価し合計＋供給元名つき内訳）。
  - **用途自身の修正値（専用欄・2026-07-10 ユーザー確定）**: 供給元つきの追加行とは**別枠の専用フィールド** `usage.checkBonusSelf`/`usage.damageBonusSelf`（単一の式）＝「その用途の親アイテムが持つ修正値」の入力欄。判定 fieldset／攻撃 fieldset の**先頭に常設**（増減・供給元選択のない固定欄）。式で **`@item.self`＝用途の親アイテム**を参照でき（識別キー不要）、実行時に親アイテムを式コンテキストへ渡す（`sourceItemId` を ctx／攻撃ペイロードに載せ、`bearer` として `buildFormulaData` へ）。評価は `evaluateSelfBonus`（値＋**親アイテム名で帰属**）。台帳/内訳は「判定ボーナス（親名）」「ダメージ修正（親名）」。
  - **式で参照できる値**: 「AE で参照できる値は式でも参照できるべき」＝`buildFormulaData(actor, result, bearer, target)`（`tnx-formula.mjs`）＝`getRollData()` の **`@system.*`（AE の `system.*` と対応）** ＋ **`@item.<識別キー>.system.*`（アクターの任意アイテムを識別キーで参照）** ＋ **`@item.self`（用途の親アイテム・bearer）** ＋ **`@target.*`（攻撃対象・ダメージ側のみ）** ＋ 判定結果 `@diff`/`@achievement`。判定ボーナスは判定前のため `@diff`/`@target` 不可（`@card` は可）。
  - **供給元（source）＝チャットの帰属表示専用**（式の参照とは独立。**識別キーを保存し表示は逆引きした現在名**・`identification.mjs` の `resolveItemNameByKey`/`findItemByIdentificationKey`）。供給元プルダウン＝組み合わせスタイル技能（一般技能除外）＋使用武器をグループ表示・値＝識別キー。
- **発動パラメータの自動入力**（`_onAutoFill`）: 参加技能（ベース＋コンボ）の**固有値**（スタイル技能の skill レベル `target`/`range`/`targetValue`/`timing`/`confrontation`。一般技能は固有値を持たず制約なし＝寄与しない）を読み、優先度で合成して用途へ書き込む。対象は対象優先度、射程は射程優先度（※複数は最短）、目標値は数値最大、タイミングはベース技能の先頭、対決不可は参加技能に「対決不可」があれば true。常時上書きせずボタン式（誤更新防止）。手編集で上書き可能
- **対決（情報表示）**: 参加技能の `confrontation`（値=技能名/技能名※）を読み取り専用で表示。相手側リアクションの自動解決・対決値統合はフェーズ13。書き込み用の対決フィールドは usage に持たせない（`isUnopposable` フラグのみ保持）
- `submitOnChange: true` — フォーム変更で即時 `_patchUsage()` を呼び自動保存
- `_patchUsage()` は `foundry.utils.setProperty` でドット記法キーを処理（`mergeObject` は配列でバグあり）
- **skillRefs 追加**: ドロップダウン選択で即時追加（「追加」ボタン廃止）
- **effects 管理（エフェクトタブ）**: D&D の「適用される効果」方式。セレクトボックスで ActiveEffect（未追加のもの）を選ぶと `usage.effects` に **`{itemId, effectId}`** を追加。追加済みは行表示＋「解除」ボタンでデタッチ（効果自体は削除しない＝アイテムシートのエフェクトタブの責務）。`addedEffects` / `hasAnyEffect` をコンテキストに供給（旧チェックボックス羅列方式を 2026-06-20 に置換）。
  - **供給元＝親アイテム＋組み合わせ技能＋使用武器（2026-07-10 ユーザー確定）**: 従来は親アイテム（`this._item`）の効果のみ選択可だったのを、**参加技能（ベース＋組み合わせ）・使用武器**の効果からも選べるよう拡張。`availableEffectGroups` を**供給元アイテムごとにグループ化**（`<optgroup>`・選択値＝`itemId|effectId`）。保存の `itemId` 空＝親アイテム（旧データ互換）。`effectId` だけでは供給元アイテムを一意に解決できないため `itemId` を明示保持。解除は `itemId`＋`effectId` で1件特定。
  - **対象への付与（実装済み 2026-07-10・`usage-effects.mjs`）**: **用途を解決した際に、ターゲットしたキャラクターへ AE を付与**する（攻撃に限らず全用途。自己バフは自分をターゲット）。正本ロジックは [Combat_Flow.md](../Game_Rules/Combat_Flow.md)「用途の適用効果」。
    - **ターゲット**: 使用時の `game.user.targets`（数・種別は検証せず「居るか居ないか」だけで判断）。**居なければ確認ダイアログ**（[自分を対象に続行]/[キャンセル]。原則ターゲット必須で、未ターゲット事故を防ぐ。続行＝自分対象＝自己バフ。キャンセルで用途中止）。
    - **付与**: 結果カード（判定結果／攻撃／用途使用）に **「効果を適用」ボタン**を出し、**対象の所有者か RL(GM)** が押して付与（ダメージ適用と同じ権限・**自動付与はしない**＝命中/成功や誤爆の判断は卓に委ねる）。付与＝供給元 AE を対象アクターへ**複製生成**（`disabled=false`）。適用済みフラグは author/GM が更新、非作者の対象所有者は `usageEffectApplied` ソケットで GM へ委譲。**適用はサイレントにしない**（2026-07-11 ユーザー指摘で修正）＝適用時に「効果の適用」チャットカード（効果名・対象名）を全員へ投稿し、適用者には通知を出す。
    - **適用ボタンはフローの一番最後に出す（2026-07-11 ユーザー確定・同日再修正）**: **攻撃（ダメージフローを持つ用途）ではダメージ・チャットカードへの表示に一本化**——攻撃カード側には一切出さない（`finalizeDamageRoll` が攻撃カードのペイロードを適用済み状態ごとダメージカードへ引き継ぐ。当初の「命中後・ダメージ未算出の間は攻撃カードに表示」はユーザー指摘で撤去）。判定結果カード/用途使用カード（非攻撃）は投稿＝フロー終端のため従来どおり。
    - 見出しの様式（2026-07-11 修正）: 用途使用カードは `cr-head`（暗背景の見出し）を持つ既存カード様式を踏襲（素の `<p>` 見出しは文字が見えづらく不可）。
    - `prepareUsageEffectPayload`（効果データ解決＋ターゲット確定）→ ctx/攻撃ペイロード経由でカードのフラグ `usageEffects{effects,targets,applied}` に載せ、`renderUsageEffectButton` フックがボタンを描画。
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
- **無視する指定技能（`ignoreComboSkills`・2026-07-10 ユーザー確定）**: **用途側の設定**として辞典識別キーのリスト（`ArrayField(StringField)`）を持つ。ここに設定した技能（例〈射撃〉）を指定「技能」とするスタイル技能は、**その指定技能の自動追加をスキップして単体で組み合わせに入る**（＝連鎖の必須を引き込まない・指定技能がアクションでも「アクション同士」で弾かれない）。`resolveUsageSkills(root, skillItems, seedComboIds, ignoreComboKeys)` の第4引数（識別キー集合）で seed 連鎖の必須から除外（seed 自身は常に参加・**起点＝用途の親自身の連鎖には適用しない**＝ベース決定の根幹）。
  - **表現するルール**: 〈技能AⅡ〉系の効果——「技能：〈心理〉で射撃攻撃を行う〈技能A〉」の攻撃に、本来は組めない**技能：〈射撃〉のスタイル技能**を、**〈射撃〉を連れてこずに**組み込めるようにする。**組み込んだ技能の「組み合わせた攻撃への」効果（判定＋/ダメージ＋等）は乗るが、〈射撃〉自体は判定に入らないため〈射撃〉へのボーナスは乗らない。** ※技能一致でのブロックは元々存在せず、実在するブロックは「指定技能がアクション技能同士」のみ（既存）。
  - UI: 参加技能 fieldset に「無視する指定技能」リスト（辞典技能の即時追加セレクト＋行削除。保存＝識別キー・表示＝逆引き名）。**一度設定すれば、以後この用途への技能追加は何も聞かれず単体参加になる。** 削除時は指定技能（アクション）が引き込まれてアクション重複になる場合は拒否（先に該当の組み合わせ技能を外す）。auto-fill（`deriveUsageAutoFill`）も同設定を反映し必須補完で再追加しない。
  - **却下された設計（再発防止の記録）**: ①参加技能の行ごとの「指定技能を無視」トグル——**追加自体が「アクション同士」で弾かれるため、必要な場面でトグルに到達できない**（ユーザー指摘で発覚）。②追加時の確認ダイアログ——組み合わせのたびに毎回聞くのは誤り。**正: 用途側の永続設定**。
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
- **ヴィークルの操縦（回避ブロック・実装済み 2026-07-10）**: 搭乗中（＝準備済みヴィークルが在る。部位「操縦」は1枠のため常に1つ）はドッジ＝**対応する〈操縦〉のみ**で、〈回避〉・他技能は**ブロック**する。`startReaction`（`attack-flow.mjs`）の dodge 分岐で、準備済み＋`operateSkillKey` 設定のヴィークルを検出→`resolveOperateSkill` で操縦技能を解決し、**ドッジ候補をその操縦技能1つに差し替える**（役割の加算宣言でなく、文脈による差し替え）。対応する操縦技能を**持たない**場合はドッジ不可＝警告して**パリー／制御値受けをダイアログで選ばせフォールバック**（回避へは戻さない・ユーザー確定 2026-07-10）。パリー/精神・社会リアクションは対象外（ドッジのみ）。※かつて Outfits.md に「自由選択で対応済み・自動上書きは後日」とあったが、これは**ヒント文だけの状態**でありハード上書きは未実装だった（本実装で解消）。
- **用途モデルの一本化(実装済み `7b035ad`)**: 「他者を攻撃する白兵判定」も「別用途の白兵判定」もルール上同じ判定であり、*attack* は判定の種類でなく**帰結(攻撃として使うか)**。attack 用途タイプを廃止(USAGE_TYPES から除去)。**攻撃は判定(check)の一種で damageCategory 付きのもの**＝`isAttackUsage(u) = u.type==="check" && !!u.damageCategory`。migrateData で `type="attack" → "check"`(damageCategory 保持・未設定は物理)。起動/検出(item-sheet・_onStartSkillCheck・用途ピッカー)を isAttackUsage 判定に置換。**declaration(宣言)は判定なしで使える能力を表すため残す**(ユーザー明言)。
- **用途エディタ(実装済み `7b035ad`)**: 攻撃は「判定＋『この判定を攻撃に使う』トグル→系統/武器(物理のみ)/種別」に。発動タブを**2カラムグリッド**化(縦積みの見辛さ改善)・ウィンドウを 640×560 に拡幅。以後の見た目は実機で試行錯誤。
- **skillRoles 編集UI(実装済み `7b035ad`)**: 一般/スタイル技能シートの設定タブに役割チェック(`item/parts/skill-roles.hbs`)。`toggleSkillRole` で system.skillRoles を更新。表示は実効役割(フィールド or 正準名の既定)。

段階: ① skillRoles＋役割ベース検出(治療・リアクション)=`ad34b3a` → ② attack→check 一本化＋移行=`7b035ad` → ③ 用途エディタ刷新＋skillRoles 編集UI=`7b035ad`。**3段階とも実装済み**(見た目は実機フィードバックで調整)。技能の散在フラグ(アクション技能・報酬点・組み合わせ不可)と役割は「特性」セクションに集約(`0bc0e30`・アウトフィットの特性に倣う)。残: ヴィークル搭乗時の回避ブロック(準備済みヴィークルから判定・13 とは無関係=上記訂正)、既定技能へのコンペンディウム役割設定(現状は正準名の既定でブリッジ)。

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
