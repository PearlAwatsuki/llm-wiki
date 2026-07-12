# ActiveEffect 適用モデル（確定設計 v2・実装済み）

フェーズ9 の中核。**標準の ActiveEffect 設定 UI（変更タブ＝キー＋モード＋値）だけで完結**させ、マクロや非存在フィールドに依存しない。2026-06-21 に effectMod 方式から**「total 直接適用」方式**へ再設計し、同日に実装完了（ユーザー確定）。

ルール前提: [Character_Creation.md](../Game_Rules/Character_Creation.md)（0clamp）・[Damage_Rules.md](../Game_Rules/Damage_Rules.md)・[Time_Management.md](../Game_Rules/Time_Management.md)。フェーズ計画: [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md)。

> **用語**: 「判定」は英語で **check**（TRPG 標準。judge/judgment ではない）。キーも `check.` / `controlCheck.` を用いる。コード全体の英名も Check に統一済み（2026-07-03、旧 Judgment 系を改名）。

---

> **正本の分離（2026-07-13 ユーザー指摘で整理）**: **AE の change キー（着地キー・実行時キー）の唯一の正本はこのページ**。式（`@` 参照＝AE の「値」・判定ボーナス・ダメージ修正・目標値の式で使うデータ）の正本は [Formula_Reference.md](Formula_Reference.md)。キーの表を他ページに複製しない。

## 0. 大原則

1. **base（UI 編集値）と total（実効値）を分離**。UI が編集するのは base（value / growth / mod 等）、表示・判定・ダメージが読むのは total。
2. **エフェクトは total に直接、ネイティブのモードで効く**（ADD / OVERRIDE / MULTIPLY / UPGRADE / DOWNGRADE）。effectMod のような加算専用の中間枠は持たない（OVERRIDE/MULTIPLY が表現できないため**全廃済み**）。
3. **対象・パラメータは「変更（changes）のキー」で表す**。標準 UI に存在するのはキー（テキスト）＋モード（選択）＋値（テキスト）のみ。これ以外のフィールドは前提にしない（例外: 重複可チェックボックスは詳細タブに自前注入＝§5）。
4. Foundry の Item は自己効果を適用せず、change.key は他アイテムへ解決できない。よって**アクターの単一適用パスが全 buff を集めて total へ適用**する（`CONFIG.ActiveEffect.legacyTransferral=false`、ネイティブ applyActiveEffects には依存しない）。横断バフは **transfer（効果をキャラクターに適用）OFF でも反映**される。
5. **パラメータと解説の境界（2026-07-02 ユーザー確定）**: アイテムの**パラメータ（データ欄の値。ユーザー呼称「データ」）によるキャラクターデータへの修正は AE で表現しない**——派生パイプラインが**直接読み取って適用**する（例: アウトフィットの制御値修正・CS 修正・危険値は `computeOutfitAggregates` が携帯/準備ゲート込みで直接集計。防御力・受け値をダメージから直接引くのも同方針＝[Damage_Rules.md](../Game_Rules/Damage_Rules.md)）。**パラメータの特殊挙動**（適用条件・例外。例: ゴースト登場時の CS 修正無視）**も AE を使わず、直接読み取りロジック側の条件で処理**する。AE で表現するのは (a) **完全に解説由来の挙動**、(b) 『CS：0（+1）』のように**機械化がどうしても難しく同様の例がほぼ存在しない個別事象**（仕方なく AE）——の2つに限る。
6. **CS への AE 着地点は必要（2026-07-02 ユーザー確定）**: 準備・使用することで CS に修正を与える**解説由来の効果を持つアウトフィットが存在する**ため、v2 キー文法に CS の着地点を設ける。ただし **CSベース／CS／CSカレントのどれへの修正かを区別**できるキー設計とする（3層の派生関係・具体キー文法はフェーズ10-5＝KI-010 の CS パイプライン設計で確定）。

## 1. 注入点は3つ

| 注入点 | 対象 | タイミング | キー接頭 |
|---|---|---|---|
| **恒常（適用パス）** | キャラ値・アイテム値 | prepareDerivedData で base→total を出した後、アクターの適用パスが total を改変 | `system.` |
| **判定時（check）** | 判定の達成値・制御値への加算 | 判定実行時にエフェクト単位で評価 | `check.` / `controlCheck.` |
| **ダメージ時（damage.*）** | 与えるダメージへの加算（無条件 `damage.dealt`・対象依存 `damage.vs*`） | ダメージ算出時に攻撃の系統/攻撃対象で照合し評価（2026-07-10〜11） | `damage.dealt` / `damage.vsStyle.` / `damage.vsWorks.` |

- 後二者（判定時・ダメージ時）は**実行時評価の別系統**で、恒常の適用パス（`_applyEffectBuffs`）からは除外する。恒常はシート値へ焼き込むため「攻撃対象」を知らない＝対象依存を表せないのに対し、実行時系統はアクションが対象を確定した瞬間に評価するため対象を見られる。

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
| `item.<識別キー>.system.<パラメータ>` | **アイテム着地の統一記法（2026-07-13・推奨）**: 式（`@item.<識別キー>.system.*`）と同じ綴りで書ける。`system.skill.<識別キー>.<パラメータ>` と同義（前方一致 `*`・`item.self.system.*`／`item.parent.system.*` も同様に self/parent と同義） |
| `system.handMaxSizeMod` | ネイティブキー（KI-020 特例。所有 cast に着地→User へ合算） |

- パスは内部で total 系へ解決（`resolveItemTotalPath`）: `attack`→`attack.total`、`defence.S`→`defence.S_total`、`level`→`levelTotal`、`FAValue`→`FAValueTotal`、その他→`<param>.total`。base は不変。
- 識別キー（技能）と `category`（アウトフィット）で住み分け（アウトフィットは識別キーで個体指定しない方針）。

### 判定バフ（判定時・`check.` / `controlCheck.`）

| キー | 対象 |
|---|---|
| `check.all` | **判定全般**（無印「判定」＝能力値判定＋技能判定のすべて・2026-07-13）。制御判定には掛からない（用語規約）。※`all` は識別キーの予約語 |
| `check.melee` | 技能判定（識別キー完全一致） |
| `check.society_*` | 技能判定（プレフィックス前方一致＝固有名詞技能の大分類はこれで束ねる） |
| `check.style.<スタイル識別キー>` | **そのスタイルのスタイル技能すべて**（技能の `system.style` で照合。2026-07-10） |
| `check.works.<組織識別キー>` | **そのワークスのワークス技能すべて**（技能の `system.special.works.organization` で照合。2026-07-10） |
| `check.reason` | 能力値判定（reason/passion/life/mundane） |
| `controlCheck.reason` | 制御判定（能力値のみ。グループ参照不可） |
| `check.cardValue` | **カード数字の上書き**（2026-07-13）: 判定に使用したカードの数字を **A〜K で上書き**する（値は AE 設定シートで選択式・自由入力なし・先勝ち）。以降の数字規約は上書き後の数字に従う（手札の絵札=10・A の21固定選択・山札の絵札=FUMBLE）。ジョーカー/切り札の宣言には適用しない。制御判定は対象外（用語規約）。結果カードに「数字上書き（元 → 後）」を明示 |
| `check.suitChange` | **スート変更マーカー**（値不要・2026-07-12）: 判定で使用できないスートのカードを使用可能なスートに変更できる（[Check_Rules](../Game_Rules/Check_Rules.md)「スートの変更」）。他者バフの付与形＝用途の適用効果で対象へ付与。失効は当面手動（「1回の判定」Duration は時間管理フェーズで持続時間側に追加＝**持続時間と AE キーは独立**）。制御判定は対象外 |

- 末尾にパラメータを付けない技能キー＝その技能の判定への加算（`.judgment` 等の接尾は不要）。
- **ダメージ種別の上書き（2026-07-13）**: 値は S/P/I/X の選択式・**モードは何でもよい（常に上書きとして適用）**。着地は実効フィールド（設定欄の素値は書き換えない）。**そのまま書けば動くキー**:

| 置き場所と狙い | キー |
|---|---|
| 武器自身に効果を置く（**手作りの武器はこれ**・識別キー不要） | `item.self.system.attack.damageType` |
| 別のアイテム（技能等）から特定の武器を指定 | `item.<武器の識別キー>.system.attack.damageType` |
| カテゴリ一括（小分類/大分類） | `system.category.<カテゴリ>.attack.damageType` |
| 生身 | `system.baseAttack.damageType` |

  - **識別キー指定は、識別キーを持つアイテムにしか届かない**（辞典由来のアイテムは持つ。手作りアイテムは武器シートで識別キーを設定するか、`item.self`／カテゴリを使う）。ここが「上書きが効かない」ときの最頻の原因。
- **キーの綴りと着地の正規化（2026-07-13）**: 統一記法では式と同じフルパス綴り（`….attack.value`・`….attack.total`・`….levelTotal` 等）で書かれるため、**素値/実効のどちらの綴りでも必ず実効（total 系）へ正規化して着地**する（`attack.value`→`attack.total` 等）。**設定欄（base）は不変**で、追加モードは実効値の上に加算・上書きモードは実効値を置き換える。
- **アイテム狙いの効果は物理転送＋供給元が正の片方向同期（2026-07-13 転送確定・2026-07-12 同期追加）**: `item.<識別キー>`／`system.skill`／`system.category`／`item.parent` の変更は、効果の作成/更新時とアイテム追加時に**対象アイテム上の実体コピー**として転送される（キーは `item.self.system.*` に書き換え・由来は origin／transferredFrom フラグ）。コピーは対象アイテムの通常の効果＝**無条件でそのアイテムのパラメータに効く**（parent 連鎖や識別キーの再照合に依存しない）。遠隔の直接適用は廃止（コピーと二重にならない）。**同期は供給元が正の片方向**（「後から供給元を更新しても反映されない」とのユーザー指摘で無同期から変更）: 供給元の更新（変更行・有効/無効・名前・アイコン）でコピーは現在値に上書きされ、供給元がそのアイテムを狙わなくなったとき／供給元の効果・供給元アイテムが削除されたときはコピーも除去される。**コピー側の手動編集・切替は供給元の次の更新で上書きされる**（コピー自身の削除は独立＝供給元へ波及しない。ただし供給元が狙い続ける限り次の更新で再転送される）。既存の効果は一度保存（または OFF→ON）すると転送される。コピーは対象アイテムのエフェクト一覧の**「転送された効果」セクション**に表示される（名前は斜体＋供給元名・操作は通常の効果と同じ）。
- **グループ参照（`check.style.*` / `check.works.*`）**: 識別キーでなく技能の `system.style` / `system.special.works.organization` で束ねる。`parseEffectTargetKey` は `{scope:"skillCheck", group:"style"|"works", selector:<キー>}` を返し、`checkChangeMatches` は判定に使った技能のうち該当 style/organization を持つものがあれば適用。

### ダメージバフ（ダメージ時・`damage.*`・2026-07-10〜11）

ダメージ算出時に攻撃者の effects から集計してフラット加算する（判定バフと同じ実行時評価の別系統）。※`damage.replaceTag`/`damage.addTag` のみ**対象（ダメージを受ける側）の effects** をチャート適用時に評価する。

| キー | 対象 |
|---|---|
| `damage.dealt` | **与えるダメージ +値**（全系統・2026-07-11） |
| `damage.dealt.<physical\|mental\|social>` | その系統の攻撃の与えるダメージ +値 |
| `damage.vsStyle.<スタイル識別キー>` | 攻撃対象がそのスタイル（`type:"style"` アイテム）を持つとき +値 |
| `damage.vsWorks.<組織識別キー>` | 攻撃対象がそのワークス（ワークス技能の組織）を持つとき +値 |
| `damage.replaceTag.<元タグ>` | **ダメージタグ改変（対象側・2026-07-12）**: チャート適用時、元タグを値のタグへ**置換**（例 `damage.replaceTag.stupor`＝`dominated`＝昏睡→支配）。値=CONDITION_KINDS のタグキー |
| `damage.addTag.<元タグ>` | 同・元タグ付与時に値のタグを**追加**（例 `damage.addTag.erased`＝`dominated`＝抹殺に支配を追加）。追加分は同じ負傷に紐づく |

- `parseEffectTargetKey` は `{scope:"damageVs", group:"style"|"works", selector:<キー>}` を返す。**恒常適用パスからは除外**（対象が見えないため焼き込めない）。
- ダメージフロー（`damage-flow.mjs` `openDamageRollDialog` → `collectDamageVsBonuses`）が攻撃対象を解決し、対象の style/works と `gatherDamageVsSources`（`collectActorEffectBuffs` で集めた攻撃者の effects）で照合。合致分を用途のダメージ修正と同じ台帳行（供給元＝効果名）としてダメージへ合算する。
- 値は**リテラル**（判定バフと同じ）。重複規約も判定バフと同一（identity 単位で最大1回・stackable はスタック）＝`_gatherBonusSources` を判定バフと共用。

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
   d. **値を2フェーズで適用（2026-07-10）**: フェーズ1=リテラル値の change を priority 順に適用（base→total 確定）、フェーズ2=**式値**の change を評価してから適用。各フェーズ内で同一効果の重複排除（§4）。
3. 能力値・制御値は最後に 0clamp
```

- `effect.apply(targetDoc, {...change, key: "system.<path>", value: String(評価値)})` でモード処理をネイティブ再利用。
- self/parent はそのエフェクトの親アイテム（と parentItemId）で解決。所持時のみ反映。
- **AE の「値」の式参照（2026-07-10 ユーザー確定・実装済み）**: change.value がリテラル数値でなく式なら `evaluateFormulaSync`（同期・prepareDerivedData 内のため）で評価する。参照は判定/ダメージの式と同系統＝`@system.*`・`@item.<識別キー>.system.*`、加えて **AE 値専用の相対参照 `@item.self.system.*`（効果が乗るアイテム自身）/`@item.parent.system.*`（その親＝装備先ホスト・`parentItemId`）**（`buildFormulaData(actor, null, bearer)`）。例: オプションの AE 値 `@item.parent.system.attack.total` で装備先の攻撃力を参照。
  - **順序**: 式値はリテラル値適用後の total を参照するため**リテラル先→式後**の2フェーズ。**式値どうしは相互参照しない**（ユーザー確定）のでフェーズ2 内は順不同で確定する。`@diff`/`@achievement` は判定文脈でないため不可。`.total` は AE 込み（リテラル AE の結果を含む）。
  - ※**用途（判定ボーナス/ダメージ修正等）の式**は実行時（prepareData 完了後）評価なので、参照する item の `.total` は最初から AE 完全適用済み（順序問題なし）。

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

`tnx-check-flow.mjs` `_computeCheckBonus(actor, ctx, abilityKey)` が effects を集め、判定種別で criteria を作る:
- controlCheck → `{type:"control", ability}`、skillCheck → `{type:"skill", skills:[{key:識別キー, style, organization}]}`（グループ参照のため識別キーに加えスタイル/組織も載せる。旧 `skillKeys` も後方互換で受理）、それ以外 → `{type:"ability", ability}`。
- `calcSkillCheck` は達成値に `+checkBonus`、`calcControlCheck` は制御値を `+checkBonus`（成功条件が緩む）。報酬点プレビューにも反映。

## 5. 重複可（stackable）チェックボックス UI

標準 ActiveEffectConfig の**詳細タブ**に「重複可」チェックボックスを注入（`renderActiveEffectConfig` フック、`scripts/tnx.mjs`）。change 時に `flags["tokyo-nova-axleration"].stackable` へ `setFlag`（フォーム送信に依存しない）。未チェック＝重複適用不可、チェック＝スタック。

## 6. 構造化の前提（別途・後続）

**判定バフのスタイル/ワークス述語は実装済み（2026-07-10）**: スタイル技能の `system.style`・ワークス技能の `system.special.works.organization` を照合キーに用い、`check.style.<キー>` / `check.works.<キー>` で束ねる（§2「判定バフ」）。専用欄の辞典読み取りドロップダウンによる入力補助や、値バフ（`system.*`）側の同グループ化・ワークス装備フラグは未着手（当面は識別キー/プレフィックス/カテゴリで賄う）。

分類（カテゴリ）の**コードキー化は実装済**（`category.melee` 等。`outfit-categories.mjs` が `{majorKey:{minors:{minorKey}}}`）。

## 参照

- [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md) / [Check_System.md](Check_System.md)
- [Character_Creation.md](../Game_Rules/Character_Creation.md)（0clamp）
