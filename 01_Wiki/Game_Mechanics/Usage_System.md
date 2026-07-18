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
    rangeMax:     StringField,          // 最長射程（2026-07-16）: "none"=単点。range が物理射程のとき「近～遠」等の幅の上限
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

> **一本化（2026-07-13）**: タイプは `check`/`declaration` のみ（→ [Check_Rules](../Game_Rules/Check_Rules.md)「用途タイプ」）。改造は廃止（check へ移行・機能未実装）・NPC取得はフラグ化（設定 UI は効果タブの〈NPC取得〉fieldset へ移動。設定できるのはトループ取得技能とアウトフィット＝旧作成ゲートの移管。OFF で設定リセット）。

| type | 説明 |
|---|---|
| `check` | 判定（攻撃・回復・NPC取得等はフラグ/設定として載る） |
| `declaration` | 宣言（神業を含む。修正系・スート変更・回復・NPC取得を設定可） |

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
  - **ダメージを修正（`modifyDamage`・2026-07-11＝旧 `boostDamage` を置換）**: 〈攻撃〉fieldset の攻撃モード**ラジオ**（なし／攻撃に使う／ダメージを修正＝`attackMode`）で選ぶ（排他はラジオが保証。当初のチェックボックス2つ＋後勝ち判定はフラグとUI表示のずれを生みユーザー指摘で廃止）。選択で〈攻撃〉内に**ダメージ修正（式）**（`damageBonusSelf` 流用・増加=正/軽減=負・空なら適用時に手入力）を表示。使用はアイテムロール＝技能クリックで「ダメージクリック待ち」モード（`TnxCheckFlow.startAchievementAction("modifyDamage")`・再使用でキャンセル）→ダメージ・チャットカードの攻撃側合計クリック→`handleDamageModifyClick`（damage-flow）が評価・消費適用・フラグ `mods` へ「事後修正」行を追記（有効=算出後〜適用前）。旧 `useDamageBoost`（待ち受け登録方式）は廃止。正本 [Damage_Rules.md](../Game_Rules/Damage_Rules.md)「ダメージ修正技能」。
  - **宣言（declaration）の判定/ダメージ修正（2026-07-12 ユーザー確定）**: 宣言ビューの効果タブに、判定用途とレイアウトを揃えた**〈判定〉〈ダメージ〉fieldset** を置き、それぞれ**チェックボックス**「判定を修正」（`modifyCheck`）／「ダメージを修正」（`modifyDamage`）＋ ON で修正値（式）欄（`checkBonusSelf`/`damageBonusSelf`）を表示する。**チェックボックス2つは独立**（ラジオ排他にしない＝ユーザー裁定「排他は意味が無い」。両方 ON の運用は想定せず、使用時は「判定を修正」が先に振られる＝決定的な優先順のみで専用機構は作らない）。スキーマ変更なし（フィールドは全用途タイプ共通で既存）。導線＝アクターシートの技能クリック（修正フラグ付き宣言を実行対象に追加・フラグ無し宣言は従来どおり解説カード）とアイテムシートの使用ボタン（クリック待ち系分岐を check/declaration のフラグゲートに統一・攻撃より先＝アクターシートと同順）。
  - **スート変更（2026-07-12 ユーザー確定）**: 〈判定〉の判定モードラジオに「**スートを変更**」（`grantSuitChange`＝スート変更待ち・次の自分の判定）、通常モードに「**スート変更可能**」チェックボックス（`allowSuitChange`＝この用途の判定でスート不一致を置換）、宣言の〈判定〉に「スートを変更（次の判定）」チェックボックス。他者付与は適用効果に AE キー `check.suitChange` を設定して行う。正本 [Check_Rules.md](../Game_Rules/Check_Rules.md)「スートの変更」。
  - **回復（2026-07-13 ユーザー確定）**: 効果タブの〈回復〉fieldset（check/declaration 両対応）。「回復を行う」チェック→範囲の行（大分類→小分類・部位エディタの前例踏襲）＋除外タグ（チップ＋ドロップダウン追加＝無視する指定技能と同型）＋「該当すべて」/回復数スピナー＋目標値の式（check のみ・`@condition.magnitude`/`@condition.woundValue`）。正本 [Conditions.md](Conditions.md)「回復・治療系スタイル技能」。
  - **カバーリング（2026-07-16 ユーザー指摘で分離）**: `covering` トグルは当初〈リアクション〉fieldset に同居していたが、**カバーリングはリアクションではない**ため独立〈カバーリング〉fieldset（トグル「カバーを行う」）へ移動。テンプレートのみの変更でロジック（attack-flow / character-sheet-base の `covering` 読み）は不変。
  - **射程の幅（2026-07-16）**: 発動タブの射程は物理射程（至近〜超遠）を選んだとき **min～max の2セレクト**（武器エディタの min～max と同形・最長「なし」=単点）。「近～遠」等の幅を持つ武器の射程が単一値（最長）に潰れて読めない問題への対応。物理射程以外へ変更したら `rangeMax` はリセット・セレクトの出し入れは明示再描画（submitOnChange は再描画しないため）。
  - **使用回数の消費の行追加ボタン（2026-07-16 是正）**: fieldset 先頭の独立ヘッダー行（左寄せ＋）は誤り。**fieldset 全体が行集合のセクションは legend 内に＋**が規約（部位・スタイル技能の技能/対決と同型）のため legend 内へ移動（`usage-consume-header` 廃止）。※fieldset 内サブフィールドの行集合（回復対象の範囲・判定ボーナス行）は構造が別で対象外。
  - ベース技能＋組み合わせ技能の UI は `usage-sheet-combo.hbs` パートに切り出し（check / attack で共用。`tnx.mjs` の `loadTemplates` に登録）
- **レイアウト（fieldset グルーピング＋式表示・2026-07-10 ユーザー指摘→改修）**: 各タブ内の項目を **`<fieldset>`＋`<legend>`（既存 `.tnx-fieldset`）でグループ化**（グリッド/縦積みの平坦配置をやめる。D&D アクティビティの意匠の要点＝legend グルーピングを参考）。発動＝〈取得〉〈タイミング〉〈対象・射程〉〈目標値・対決〉〈対決（参加技能より）〉〈使用回数の消費〉／効果＝〈参加技能〉〈判定〉〈攻撃〉〈効果量〉〈改造可能パラメータ〉〈適用される効果〉。
  - **式の表示＋干渉できる入力**: 「判定やダメージの式が全く無い」というユーザー指摘に対応。〈判定〉に **判定ボーナス（式）欄＋達成値の構成**（達成値＝カード＋能力値＋報酬点＋判定バフ＋判定ボーナス）、〈攻撃〉に **ダメージ修正（式）欄＋ダメージの構成**（ダメージ＝ダメージカード＋攻撃力＋修正−軽減）を `.usage-formula-line` で表示（式には**それに干渉する入力を必ず添える**＝判定ボーナス／ダメージ修正）。〈攻撃〉の内側は 2 カラムグリッドでなく**単一カラムの縦積み**（`usage-form-section`。武器の複数選択リストや式が全幅で並ぶべきため）。
  - **判定ボーナス/ダメージ修正（行＋供給元・2026-07-10 ユーザー確定）**: `usage.checkBonuses`/`usage.damageBonuses` ＝ **`{formula, source}` の行の配列**。UI はコンボ/消費と同型の複数行（`usage-sheet-bonus-rows.hbs` パート・`checkBonusFormula-N`/`checkBonusSource-N` を submit で再構成＝consumeTargets 同型・`checkBonusAdd`/`Delete` 等で行操作）。評価は `evaluateBonusRows`（各行を評価し合計＋供給元名つき内訳）。
  - **用途自身の修正値（専用欄・2026-07-10 ユーザー確定）**: 供給元つきの追加行とは**別枠の専用フィールド** `usage.checkBonusSelf`/`usage.damageBonusSelf`（単一の式）＝「その用途の親アイテムが持つ修正値」の入力欄。判定 fieldset／攻撃 fieldset の**先頭に常設**（増減・供給元選択のない固定欄）。式で **`@item.self`＝用途の親アイテム**を参照でき（識別キー不要）、実行時に親アイテムを式コンテキストへ渡す（`sourceItemId` を ctx／攻撃ペイロードに載せ、`bearer` として `buildFormulaData` へ）。評価は `evaluateSelfBonus`（値＋**親アイテム名で帰属**）。台帳/内訳は「判定ボーナス（親名）」「ダメージ修正（親名）」。
  - **式で参照できる値**: 「AE で参照できる値は式でも参照できるべき」＝`buildFormulaData(actor, result, bearer, target)`（`tnx-formula.mjs`）＝`getRollData()` の **`@system.*`（AE の `system.*` と対応）** ＋ **`@item.<識別キー>.system.*`（アクターの任意アイテムを識別キーで参照）** ＋ **`@item.self`（用途の親アイテム・bearer）** ＋ **`@target.*`（攻撃対象・ダメージ側のみ）** ＋ 判定結果 `@diff`/`@achievement`。判定ボーナスは判定前のため `@diff`/`@target` 不可（`@card` は可）。
  - **供給元（source）＝チャットの帰属表示専用**（式の参照とは独立。**識別キーを保存し表示は逆引きした現在名**・`identification.mjs` の `resolveItemNameByKey`/`findItemByIdentificationKey`）。供給元プルダウン＝組み合わせスタイル技能（一般技能除外）＋使用武器をグループ表示・値＝識別キー。
- **発動パラメータの自動入力**（`_onAutoFill`）: 参加技能（ベース＋コンボ）の**固有値**（スタイル技能の skill レベル `target`/`range`/`targetValue`/`timing`/`confrontation`。一般技能は固有値を持たず制約なし＝寄与しない）を読み、優先度で合成して用途へ書き込む。対象は対象優先度、射程は射程優先度（※複数は最短）、目標値は数値最大、タイミングはベース技能の先頭、対決不可は参加技能に「対決不可」があれば true。**射程「武器」の実体解決（2026-07-12→13 再設計）**: 優先度はそのまま（武器＝至近※に次ぐ）で、「武器」が勝った場合は**使用武器（一本目＝シートの「攻撃で使用」＋用途の追加分）**の射程を参照して具体値を設定する（武器の実効射程＝最長 max・max=なし の単一射程は min。複数武器は最短を採用＝※複数は最短の既定を踏襲。**武器なし（生身）・射程なしは至近＝生身の射程**＝ユーザー確定。「武器」のまま残す挙動は廃止）。**使用武器の追加・削除時にもライブで再解決する**（2026-07-13 ユーザー指摘＝自動入力の押し直し不要。「ライブ追従はしない」原則の例外——射程「武器」は値の実体が使用武器に委譲されており追従しないと値が成立しない。優先度の勝者が「武器」でない用途には触らない）。**幅の保持（2026-07-16）**: 「武器」への解決は幅を潰さず `{range, rangeMax}` で保存する（`resolveAttackRangeSpan`）。複数武器は幅の**交差**（下限=各下限の最長・上限=各実効射程の最短=従来の「複数は最短」と同値）・交差が成立しない組み合わせは従来の単点（最短実効）へフォールバック。単一値解決（`resolveAttackRangeValue`）は幅の上限と常に一致＝挙動不変（テストで固定）。技能由来の射程（勝者が武器以外）は単点＝`rangeMax` リセット。常時上書きせずボタン式（誤更新防止）。手編集で上書き可能
- **対決（情報表示）**: 参加技能の `confrontation`（値=技能名/技能名※）を読み取り専用で表示。相手側リアクションの自動解決・対決値統合はフェーズ13。書き込み用の対決フィールドは usage に持たせない（`isUnopposable` フラグのみ保持）
- `submitOnChange: true` — フォーム変更で即時 `_patchUsage()` を呼び自動保存
- `_patchUsage()` は `foundry.utils.setProperty` でドット記法キーを処理（`mergeObject` は配列でバグあり）
- **skillRefs 追加**: ドロップダウン選択で即時追加（「追加」ボタン廃止）
- **effects 管理（エフェクトタブ）**: D&D の「適用される効果」方式。セレクトボックスで ActiveEffect（未追加のもの）を選ぶと `usage.effects` に **`{itemId, effectId}`** を追加。追加済みは行表示＋「解除」ボタンでデタッチ（効果自体は削除しない＝アイテムシートのエフェクトタブの責務）。`addedEffects` / `hasAnyEffect` をコンテキストに供給（旧チェックボックス羅列方式を 2026-06-20 に置換）。
  - **供給元＝親アイテム＋組み合わせ技能＋使用武器（2026-07-10 ユーザー確定）**: 従来は親アイテム（`this._item`）の効果のみ選択可だったのを、**参加技能（ベース＋組み合わせ）・使用武器**の効果からも選べるよう拡張。`availableEffectGroups` を**供給元アイテムごとにグループ化**（`<optgroup>`・選択値＝`itemId|effectId`）。保存の `itemId` 空＝親アイテム（旧データ互換）。`effectId` だけでは供給元アイテムを一意に解決できないため `itemId` を明示保持。解除は `itemId`＋`effectId` で1件特定。
  - **対象への付与（実装済み 2026-07-10・2026-07-13 v3 再設計・`usage-effects.mjs`）**: **用途を解決した際に AE を付与**する（攻撃に限らず全用途）。正本ロジックは [Combat_Flow.md](../Game_Rules/Combat_Flow.md)「用途の適用効果」、キー文法・自動適用ゲートは [Active_Effects.md](Active_Effects.md)。
    - **ペイロード＝「効果を対象に自動適用」オフの効果**（v3）: 自動では効かず、ここで参照されたときだけ付与される。自動適用オンの効果も参照可（恒常効果の複製付与）。
    - **付与先（AE 設定 `flags.grantTarget`・v3 ユーザー確定）**: **「対象」**（既定）＝ターゲットしたキャラクターへ従来どおり。**「自分」**＝使用者へ**用途解決時に即時自動付与**（「効果Aを相手に与える代わりに自分がデバフを負う」の代償表現。後回しにしない・選択キャンセルは用途中止）。効果自身が付与相手を宣言するので AE を見れば分かる（用途エントリ側には持たせない＝Code 当初案をユーザー指摘で修正）。用途シートの適用効果リストに「（自分に付与）」タグを表示。
    - **着地（changes のキーで判断・v3 ユーザー確定）**: アイテム狙いキー（素の `system.<パス>`／分類・疑似分類／識別キー）が1つでもあれば**アイテム着地**＝付与先アクターの所持アイテムから**選択して付与**。候補はキーで絞る（素のキーのみ＝**全アウトフィット**のうち対象パラメータを持つもの・分類/疑似分類/識別キーがあればその該当）。候補1件は無確認、0件は通知してその効果だけスキップ（用途続行）。付与時に changes を素のキーへ正規化した実体コピーを選択アイテム上に生成（`grantedFrom` フラグ・**転送同期の対象外＝一回性インスタンス**）。アクター向けの変更は落ちる（**混在非対応**＝ユーザー裁定。両方欲しければ効果を分けるか、攻撃力+ をダメージ+（アクター着地）で代用）。それ以外は**アクター着地**（従来どおり）。
    - **多重付与の置き換えリフレッシュ（v3・Code 提案をユーザー承認）**: 付与コピーに同一性 `flags.effectId`（供給元に無ければ供給元 uuid）を刻み、付与先に同一 identity のコピーが既に居れば**新規作成でなく上書き**する（「明記なき同一効果は重複しない」原則の付与形。stackable のみ並ぶ）。
    - **ターゲット**: 使用時の `game.user.targets`（数・種別は検証せず「居るか居ないか」だけで判断）。**居なければ確認ダイアログ**（[自分を対象に続行]/[キャンセル]。原則ターゲット必須で、未ターゲット事故を防ぐ。続行＝自分対象＝自己バフ。キャンセルで用途中止）。**付与先「対象」の効果が無ければターゲット確認自体を行わない**（純粋な自己バフは付与先「自分」で書けばターゲット操作不要＝v3 の UX 改善）。
    - **付与**: 結果カード（判定結果／攻撃／用途使用）に **「効果を適用」ボタン**を出し、**対象の所有者か RL(GM)** が押して付与（ダメージ適用と同じ権限・**自動付与はしない**＝命中/成功や誤爆の判断は卓に委ねる。アイテム着地の選択ダイアログもこの押下者が選ぶ。将来ボタンを自動化した場合は解決タイミングでダイアログが出る＝ユーザー確定）。付与＝供給元 AE を対象アクター（またはそのアイテム）へ**複製生成**（`disabled=false`）。選択キャンセル時は適用済みにしない（ボタンが残りやり直せる。既付与分はリフレッシュで二重にならない）。適用済みフラグは author/GM が更新、非作者の対象所有者は GM へソケット委譲（`applyMessagePatch`・2026-07-16 に旧 usageEffectApplied を messagePatch へ一本化）。**適用の可視化は Foundry 標準のトークン演出**（2026-07-11 ユーザー指摘・2度の修正）＝+効果名の浮遊テキストとトークン上のアイコン。コアの演出条件（浮遊テキスト=statuses/changes を持つ効果のみ・アイコン=temporary=statuses か持続時間あり）を満たすよう、**statuses が空の効果には付与マーカー status `tnx-applied` を注入**する（CONFIG.statusEffects 未登録 id=HUD パレットに出ないバッジ表示専用）。独自の通知・チャットカードは出さない（Code が一度勝手に追加し「勝手な通知を出せとは言っていない」と却下。アイテム着地のみ ui.notifications で付与先を通知＝トークン演出が無いため）。
    - **適用タイミング2種（2026-07-18 ユーザー提供・裁定済み・同日実装済み）**: 攻撃で対象に与える効果は「**命中時点で適用**」と「**1点でもダメージを与えたら適用**」の2種類（ルール正本 [Combat_Flow.md](../Game_Rules/Combat_Flow.md)「攻撃の適用効果のタイミング2種」・裁定4点と実装挙動もそちら）。AE 設定 TNX 欄の効果ごとフラグ **`grantTiming`**（「適用タイミング」セレクト・`hit`/`damage`・既定=ダメージ時＝既存データ無移行・付与先「自分」では非表示）。ペイロードのエントリに `timing` を載せ `splitEffectsByTiming` で二股: 命中時＝hit への**遷移**時に対象ごと自動付与（`grantHitTimedEffects`＝postAttackCard/completeReactionFromCheck/再判定 `newlyHitTargets`。権限が無ければ `usageEffectGrant` ソケットで GM 委譲・攻撃カードに付与済みノート・`usageEffects.hitGranted` に記録）／ダメージ時＝ダメージカードへ引き継ぎ、適用ループで**最終適用値≥1 の対象のみ** `grantDamageTimedEffects`（`usageEffects.appliedTargets` に実付与対象を記録・0件は「付与されませんでした」表示）。旧一括付与の穴（回避対象へも付与・最終0でも付与・ダメージカード未使用で到達不能）＝**KI-027 是正**。対決（非攻撃）カードの効果ボタン到達不能＝**KI-028 是正**（`attackCardEffectMode`＝非攻撃対決は解決後に手動ボタン）。対象なし攻撃（RL 手動運用）は命中解決が無いため命中時効果も手動ボタンで拾える。付与先「自分」（解決時即時付与）とは直交・非攻撃用途は挙動不変。用途シートの適用効果リストに「（命中時）」タグ。
    - **適用ボタンはフローの一番最後に出す（2026-07-11 ユーザー確定・同日再修正）**: **攻撃（ダメージフローを持つ用途）ではダメージ・チャットカードへの表示に一本化**——攻撃カード側には一切出さない（`finalizeDamageRoll` が攻撃カードのペイロードを適用済み状態ごとダメージカードへ引き継ぐ。当初の「命中後・ダメージ未算出の間は攻撃カードに表示」はユーザー指摘で撤去）。判定結果カード/用途使用カード（非攻撃）は投稿＝フロー終端のため従来どおり。
    - 見出しの様式（2026-07-11 修正）: 用途使用カードは `cr-head`（暗背景の見出し）を持つ既存カード様式を踏襲（素の `<p>` 見出しは文字が見えづらく不可）。
    - `prepareUsageEffectPayload`（効果データ解決＋付与先分岐＋自分向け即時付与＋ターゲット確定）→ ctx/攻撃ペイロード経由でカードのフラグ `usageEffects{effects,targets,applied,selfApplied?}` に載せ、`renderUsageEffectButton` フックがボタン（と「自分へ付与済み」注記）を描画。自分向けのみの用途は `applied:true` で運ぶ（ボタンなしの適用済み表示）。
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

## 判定起動の共通前段（2026-07-16 一本化）

用途から通常判定へ渡すコンテキストの組み立ては **`buildUsageCheckContext`（`scripts/module/usage-check-context.mjs`）一箇所**に集約されている（エキストラ制限→用途不備検知→参加技能解決 `resolveUsageSkillSet`→報酬点 usesBounty→消費確認→適用効果ペイロード→目標値解決）。**攻撃（useAttack）・NPC取得（useCheckAcquire）・回復（useRecovery）・技能クリックの通常判定は全てこの前段を通り**、各フローは固有の前処理（武器・対象・回復対象の解決）と固有ペイロード（attack/npcAcquire/recovery）だけを重ねる。

- **経緯**: 従来は 4 箇所が同じ前処理をコピーしており、後から足した仕様が専用フローに反映されない取りこぼし（報酬点ブロック・NPC取得の判定ボーナス/適用効果/目標値欄/再判定可能・回復の適用効果/再判定可能）が繰り返されていた（2026-07-16 監査で確定→一本化）。
- **NPC取得の目標値**: 分身＝10（Troops.md のルール固定・ハードコード維持）。トループ/エニグマは用途の目標値欄に従う（通常は「なし」＝達成値分を取得）。
- **アイテムシートの使用ボタンも統合**: `_onUsageUse` は用途 ID を直接指定して唯一の起動関数 `_activateItemCheck(actor, item, { usageId })` へ委譲する薄い入口（第2ディスパッチャの複製を廃止＝カバー・固定値分岐が使用ボタン側に無かったドリフトを解消）。実行フラグの無い宣言の「使用」（消費＋適用効果＋使用カード）は `_useDeclarationUsage` として `_activateItemCheck` 側へ移設（用途直接指定時のみ到達＝技能クリックの解説カード挙動は不変）。check 用途を使用ボタンから使うと技能クリックと同様に判定が起動する（旧: 使用カードのみ）。
- **報酬点使用不可（口座凍結/信用失墜）**: 前段では見ない——消費時点の状態で判定するため `TnxCheckFlow._execute` の報酬点ダイアログ直前で一元ゲート（→ [Bounty_Points.md](../Game_Rules/Bounty_Points.md)）。

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

> **11-6 で消費先設定（consumeTargets）に一本化**: 本節の `planUsesConsumption`/`_consumeUses`（参加技能の自動スキャン）は**全廃済み**。全ての消費は用途の消費先設定からのみ発生する（`usage-consumption.mjs`＝`resolveConsumeRowsForActor`→`promptConsumption`→`applyConsumptionPlan`。判定用途は判定実行時・他の用途は使用時・クリック待ち系は発動確定時に適用）。
> **消費先の種別（2026-07-18 再編・本体 `f5292d8`）**: `{type:"item"|"actionRank", itemId, resource:"uses"|"ammo", amount}`。先頭は**「アイテム」「AR」の2択**。`type="item"` は `itemId` 空=**このアイテム自身**（用途の親）・値=同アクター内 Item ID、`resource`=「使用回数」（uses.spent。**神業も同じ uses に一本化**＝特例 `miracleUses` 廃止）/「残弾数」（武器の残弾）。UI は二段選択のブロック（tnx-combo-card 踏襲・資源は選択アイテムの能力で絞る）。旧 type（parent/itemUses/miracleUses/ammo/actionRank）は migrateData で item+resource へ移行。「親アイテム」表記は「このアイテム自身」へ。
> **残弾（旧記述の訂正・2026-07-18）**: 「任意」モードは廃止（具体的残弾数の無い武器＝FA武器等は**残弾1**）。**残弾の消費・回復は自動では行わず用途の消費（resource="ammo"・負値=回復=リロード）からのみ**。FA ダメージ自動化は廃止＝FA 値は用途のダメージボーナス式で手動参照（本体 `221cc13`）。
> **既定行の全廃（2026-07-17 ユーザー指示「そもそもいらない」）**: 旧・無条件の「親×1」既定行（11-6 の互換明示化）は全廃。新規用途・自動挿入・migrate のいずれも**既定は空**。「参加技能から自動入力」だけが、使用回数制限（isLimit）を持つ参加技能の行を敷く（親が制限を持つ場合のみ parent 行）。
> **AR（actionRank）**: 実行アクターの AR（`actionRank.value`）を消費する。対象アイテム選択なし・消費量可変。**カット進行外は AR を消費できないため原則使用不可**（消費要求は進行外でも生きる・支払えない＝残量 0 扱いの原則ブロック。チェックを外せば卓裁定で実行可。**2026-07-12 ユーザー訂正**: Code 当初の「進行外は no-op」は旧パリー自動化の挙動の誤った一般化）。AR 不足も同じ原則ブロック。分身でも本体へ差し替えない（AR は実行アクター自身の戦闘リソース）。**パリー専用の AR>0 検証＋AR−1 自動化（startReaction 内）は廃止**——パリー技能の用途に「AR を消費」を設定して表す（AR 消費の一本化）。

判定起動時に参加技能の使用回数を消費する（ルール: [Check_Rules.md](../Game_Rules/Check_Rules.md#使用回数の消費確定-2026-06-20)）。

- 使用回数は D&D 方式の **`uses.spent`（消費済み）** で保持（残り = `max − spent`）。旧 `uses.value`（残り回数）から `migrateUsesValueToSpent()` で移行（`helpers.mjs`、style-skill / outfit-base 共用）。
- `TnxCheckFlow.planUsesConsumption(actor, skillIds)`: `_onStartSkillCheck` が `open()` の前に呼ぶ。`uses.isLimit` を持つ参加技能があれば D&D 方式の消費ダイアログを表示。チェック済みで残り（`max − spent`）0なら起動不可（null 返却）。確定した消費対象を `open()` の `consumeUses` として渡す
- `TnxCheckFlow._consumeUses(actor, skillIds)`: `_execute` 内（カードプレイ直後）で `updateEmbeddedDocuments` により各技能の `uses.spent` を +1（max 上限）。組み合わせ技能は別アイテムのため遠隔消費
- 消費の確定は起動時、減算は実行時。カード選択キャンセル時は `_execute` 未到達＝非消費

## 技能の役割(skillRoles)と用途モデルの再設計（2026-07-09 ユーザー方針）

> **本節は 2026-07-17 の行動種別タイプ再編で全面的に置き換えられた（正本: [Check_Rules.md](../Game_Rules/Check_Rules.md)「用途タイプ＞行動種別への再編」「対決の解釈（是正 2026-07-17）」）**。skillRoles・正準名ブリッジ・既定技能・搭乗ドッジのハード上書き＋フォールバックは**全廃**——資格・候補は**用途タイプの所持**で判定し、搭乗中は対決が「〈操縦〉※」になる。攻撃は damageCategory フラグでなく**攻撃タイプ**（isAttackUsage はタイプ判定）。以下は経緯の記録として残す。

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

## miracle 連動（2026-07-18 uses 一本化・本体 `f5292d8`）

> **神業の使用回数を汎用 uses へ一本化**（ユーザー提案・合意のうえ「改変禁止」を更新）。旧 `usageCount {value(母数)/total(残り)/mod(バフ)}` を廃し、他アイテムと同じ `uses {isLimit, max, spent}` を持つ（残り = max − spent）。migrateData で `max=value+mod`・`spent=max−total`。**この一本化で、旧実装にあった「連動フックは value=母数/total=残り、消費経路は value=残り」という value/total の解釈矛盾（＝用途経由の神業消費が母数を削る潜在バグ）も解消**。
>
> - **母数 = uses.max = 連動スタイルの合計レベル（上限3。「母数=スタイルレベルと同一」ユーザー確定）**。万能神業（ファイト！等）による母数増加は **AE で uses.max を増やす**（専用の加算欄は設けない）。
> - style レベル変更 → 連動 miracle の `uses.max` をスタイル合計レベル（上限3）へ設定・spent はクランプ (`preUpdateItem`)。
> - style 削除（レベル1）→ 連動 miracle 削除 (`preDeleteItem`)。
> - miracle 削除 → `uses.max>1` なら削除でなく max−1（多重取得の1つを外す・`preDeleteItem`／コンテキストメニュー）。
> - 「使用済み(isUsed)」true→false → `spent=0`（残りを満タンへ・`preUpdateItem`）。
> - 消費は他アイテムと同じ汎用 uses 経路（特例 kind `miracleUses` は撤去）。神業の取得(+1母数)・使用(spent+1)・シート(残り=max−spent の +/-)も uses へ更新。

## 参照

- DataModel: `scripts/data/item/common/usage.mjs`
- 編集シート: `scripts/module/tnx-usage-sheet.mjs`
- 判定起動: `scripts/actor/tnx-cast-sheet.mjs` `_onStartSkillCheck`
- 解決済み: KI-002（usage edit ハンドラ不在） → [Resolved_Issues.md](../Issues/Resolved_Issues.md)


## 用途シートのセレクト競合の修正（2026-07-13）

用途シートのカスタムセレクト（武器・組み合わせ技能・効果・無視する指定技能・回復の除外タグ）は change リスナーで `_patchUsage`（actions 全配列の書き込み）を行うが、同じ change イベントがフォームの submitOnChange も発火させ、**二つの全配列書き込みが競合して後勝ちで片方の更新が巻き戻る**ことがあった（攻撃セクションの武器表示が古い内容に見える不整合の有力因＝ユーザー報告）。各リスナーで `stopPropagation` して submitOnChange への伝播を止めることで解消。


## 攻撃の使用武器の解決（2026-07-13 ユーザー確定）

**一本目の武器＝アクターシート戦闘タブの「攻撃で使用」**（`system.weaponRefs.attackItemId`・空欄＝生身）。**用途の weaponRefs（「使用武器の追加」）＝2本目以降の追加分**（複数武器合算能力の表現）。旧設計の「シート設定は表示のみ・通常武器は用途に移管済み」（フェーズ10-6 スキーマコメント）は本裁定で上書き。

- 攻撃フロー（攻撃力合算・FA 候補・通常射撃の残弾消費）・用途シートの使用武器表示・射程「武器」の解決は、すべて `attack-weapons.mjs`（`resolveAttackWeapons`＝シート武器→追加分・ID 重複除去）の解決結果を使う。
- 生身書き換え（全身義体・生身変更装備）が「攻撃で使用」に選ばれている場合は一本目として扱い、表記は戦闘タブと同じ「生身（名前）」。
- **経緯**: 従来は攻撃フロー・表示・射程のどれもシート設定を読んでおらず（用途に追加しない限り常に生身）、Code が旧コメントを鵜呑みにして誤ったモデルのまま「修正した」と報告した（ユーザー指摘で全面配線）。
- 申し送り: 戦闘タブで「攻撃で使用」を変更しても、既存用途の**保存済み射程**は自動更新されない（自動入力ボタンか用途の武器操作で再解決される。表示・攻撃はライブ）。
