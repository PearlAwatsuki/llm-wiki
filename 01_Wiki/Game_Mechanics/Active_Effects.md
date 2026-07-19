# ActiveEffect 適用モデル（確定設計 v3・実装済み）

フェーズ9 の中核。**標準の ActiveEffect 設定 UI（変更タブ＝キー＋モード＋値）だけで完結**させ、マクロや非存在フィールドに依存しない。2026-06-21 に effectMod 方式から**「total 直接適用」方式**へ再設計し、同日に実装完了（ユーザー確定）。

> **v3 再設計（2026-07-13 ユーザー確定・フェーズ12）**: 「恒久のデータ変更」と「能力の使用による効果の転送」の区別が付かない問題への対処。
> 1. **ネイティブ transfer チェック＝「効果を対象に自動適用」ゲート**に昇格（§0.4）。オン＝キーの示す対象へ常時自動適用。オフ＝**使用時付与用ペイロード**（値バフ・実行時系統・物理転送のどれにも自動では乗らず、用途の「適用される効果」でのみ付与）。既存ワールドの移行はしない（ユーザー裁定）。
> 2. **キー綴りの整理**: `self`/`parent`/`system.skill` 綴りを廃止。アイテム自身＝**素のパラメータキー**（`system.attack.value` 等）、識別キー狙い＝`item.<識別キー>.system.*` に一本化、準備先＝AE 設定の「準備先（親アイテム）に適用」チェック。**キーは「どのパラメータ＋絞り込み」だけを語り、乗り先の関係は AE 自身の設定が語る**。
> 3. **使用時付与のアイテム着地**: ペイロードがアイテム狙いキーを持てば、付与時に対象アクターの所持アイテムから**選択して付与**（正本は [Usage_System.md](Usage_System.md)「適用される効果」）。
> 4. **付与先（対象/自分）**: ペイロードの付与相手を AE 設定で宣言（「効果Aを相手に・代償デバフを自分に」が表現可能）。

ルール前提: [Character_Creation.md](../Game_Rules/Character_Creation.md)（0clamp）・[Damage_Rules.md](../Game_Rules/Damage_Rules.md)・[Time_Management.md](../Game_Rules/Time_Management.md)。フェーズ計画: [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md)。

> **用語**: 「判定」は英語で **check**（TRPG 標準。judge/judgment ではない）。キーも `check.` / `controlCheck.` を用いる。コード全体の英名も Check に統一済み（2026-07-03、旧 Judgment 系を改名）。

---

> **正本の分離（2026-07-13 ユーザー指摘で整理）**: **AE の change キー（着地キー・実行時キー）の唯一の正本はこのページ**。式（`@` 参照＝AE の「値」・判定ボーナス・ダメージ修正・目標値の式で使うデータ）の正本は [Formula_Reference.md](Formula_Reference.md)。キーの表を他ページに複製しない。

## 0. 大原則

1. **base（UI 編集値）と total（実効値）を分離**。UI が編集するのは base（value / growth / mod 等）、表示・判定・ダメージが読むのは total。
2. **エフェクトは total に直接、ネイティブのモードで効く**（ADD / OVERRIDE / MULTIPLY / UPGRADE / DOWNGRADE）。effectMod のような加算専用の中間枠は持たない（OVERRIDE/MULTIPLY が表現できないため**全廃済み**）。
3. **パラメータと絞り込みは「変更（changes）のキー」で、乗り先の関係は AE 自身の設定で表す**（v3）。標準 UI に存在するのはキー（テキスト）＋モード（選択）＋値（テキスト）のみで、これに詳細タブへの自前注入（§5＝自動適用ラベル差し替え・準備先・付与先・重複可）を加える。
4. **ネイティブ transfer チェック＝「効果を対象に自動適用」ゲート**（v3・2026-07-13 ユーザー確定）。アイテム上の効果は **オンのときだけ**自動適用（値バフ・判定/ダメージの実行時系統・物理転送のすべて）に乗る。**オフ＝使用時付与用ペイロード**（自動では一切効かない）。アクター上の効果と実体化済みインスタンス（転送コピー transferredFrom／付与コピー grantedFrom）は常に生きる（無効化手段は disabled のみ）。ゲートの実装は `effectAutoApplies()`（helpers.mjs）で、`collectActorEffectBuffs`（実行時系統）と `_applyEffectBuffs`／状態ペナルティ収集（恒常）が共用する。適用自体は従来どおり**アクターの単一適用パス**（`CONFIG.ActiveEffect.legacyTransferral=false`、ネイティブ applyActiveEffects には依存しない）。※旧 v2 の「transfer OFF でも反映される」は**廃止**。
5. **パラメータと解説の境界（2026-07-02 ユーザー確定）**: アイテムの**パラメータ（データ欄の値。ユーザー呼称「データ」）によるキャラクターデータへの修正は AE で表現しない**——派生パイプラインが**直接読み取って適用**する（例: アウトフィットの制御値修正・CS 修正・危険値は `computeOutfitAggregates` が携帯/準備ゲート込みで直接集計。防御力・受け値をダメージから直接引くのも同方針＝[Damage_Rules.md](../Game_Rules/Damage_Rules.md)）。**パラメータの特殊挙動**（適用条件・例外。例: ゴースト登場時の CS 修正無視）**も AE を使わず、直接読み取りロジック側の条件で処理**する。AE で表現するのは (a) **完全に解説由来の挙動**、(b) 『CS：0（+1）』のように**機械化がどうしても難しく同様の例がほぼ存在しない個別事象**（仕方なく AE）——の2つに限る。
6. **CS への AE 着地点は必要（2026-07-02 ユーザー確定）**: 準備・使用することで CS に修正を与える**解説由来の効果を持つアウトフィットが存在する**ため、v2 キー文法に CS の着地点を設ける。ただし **CSベース／CS／CSカレントのどれへの修正かを区別**できるキー設計とする → **実装済み**（フェーズ10-5＝KI-010 の CS パイプラインで確定。仮想名前空間 `system.cs.base` / `system.cs.value` / `system.cs.current`・§2.1）。

## 1. 注入点（恒常・判定時・ダメージ時・無視ゲート）

| 注入点 | 対象 | タイミング | キー接頭 |
|---|---|---|---|
| **恒常（適用パス）** | キャラ値・アイテム値 | prepareDerivedData で base→total を出した後、アクターの適用パスが total を改変 | `system.` |
| **判定時（check）** | 判定の達成値・制御値への加算 | 判定実行時にエフェクト単位で評価 | `check.` / `controlCheck.` |
| **ダメージ時（damage.*）** | 与えるダメージへの加算（攻撃側・無条件 `damage.dealt`・対象依存 `damage.vs*`）＋**受けるダメージへの加算（対象側・負=軽減・`damage.taken`/`damage.from*`）** | ダメージ算出時に攻撃の系統/攻撃対象/攻撃者で照合し評価（2026-07-10〜11・taken 系=2026-07-17） | `damage.dealt` / `damage.vsStyle.` / `damage.vsWorks.` / `damage.taken` / `damage.fromStyle.` / `damage.fromWorks.` |
| **無視ゲート（ignore.*・対象自己）** | **自分が受けているコンディションの効果**（数値ペナルティ・行動制限の両方）を消費段階で**抑止**（加算でなく除外） | 各効果の消費時（判定/派生値/将来の戦闘・時間）に評価。タグ・BS・負傷は残す | `ignore.` |

- 後三者（判定時・ダメージ時・無視ゲート）は**実行時評価の別系統**で、恒常の適用パス（`_applyEffectBuffs`）からは除外する。恒常はシート値へ焼き込むため「攻撃対象」を知らない＝対象依存を表せないのに対し、実行時系統はアクションが対象を確定した瞬間に評価するため対象を見られる。無視ゲートは「対象自己」＝自分が受けている状態の効果を、その効果が読まれる各消費点で除外する（§2.11）。

## 2. 変更キーの文法と全キー一覧（標準 UI で打つ・実装済み）

`parseEffectTargetKey(key)`（`scripts/data/item/helpers.mjs`）が解析する。**キーの増減はコードが権威**＝迷ったら `parseEffectTargetKey`（キー文法）と `resolveItemTotalPath`（着地パスの正規化）を見る。本節は [Formula_Reference.md](Formula_Reference.md) と同じ粒度の実務用索引で、**書けるキーを全数**載せる。

文法は `<セレクタ>[<条件>].<パス>`（条件は省略可・§2.6）。系統は4つ＝**値バフ**（恒常・§2.1〜2.5）・**判定バフ**（`check.`／`controlCheck.`・§2.7）・**ダメージバフ**（`damage.`・§2.8）・**無視ゲート**（`ignore.`・§2.11＝加算でなく効果の除外）。

### 2.1 値バフ① キャラクターへの着地（この表が全数）

| キー | 着地 total | 備考 |
|---|---|---|
| `system.ability.reason` / `.passion` / `.life` / `.mundane` | `<能力>.total` | 能力値（理性／感情／生命／外界） |
| `system.control.reason` / `.passion` / `.life` / `.mundane` | `<能力>.totalControl` | 制御値 |
| `system.cs.base` | `combatSpeed.baseTotal` | **CSベース**への常時修正。**準備ゲートあり**（保持アイテムが未準備なら読み飛ばし。`noPrepareRequired` 品・styleSkill・アクター自身は対象外＝2026-07-02 裁定/2026-07-09） |
| `system.cs.value` | `combatSpeed.valueTotal` | **CS**への修正（準備ゲートなし） |
| `system.cs.current` | `combatSpeed.currentTotal` | **CSカレント**への修正（準備ゲートなし） |
| `system.ar.max` | `actionRank.maxTotal` | **付与AR**への常時修正（フェーズ11）。`cs.base` と同じ準備ゲートあり |
| `system.baseAttack.damageType` | `baseAttack.damageTypeTotal` | **生身のダメージ種別上書き**（2026-07-13。値=S/P/I/X の選択式・モード不問で常に上書き・素値は不変） |
| `system.partSlot.<部位キー>` | `partSlotsEffective`（非スキーマ・`partSlotDeltas.<キー>` 経由） | **アクター部位スロットの増減**（フェーズ12・値=増減数・式可）。仮想名前空間 `partSlot`。既存部位はキー照合で count に加算（0クランプ）・**未知キーは正のデルタのときだけ新規実効スロットを作る**（負は既存のみ＝幽霊部位ガード）。`cs.base` と同じ準備ゲートあり。負傷 `partSlotMod` と合成して `partSlotsEffective` に着地（占有計算・装備先候補が読む。base の `partSlots` は編集用に不変） |
| `system.handMaxSizeMod` | ネイティブ着地（base） | 手札上限修正の特例（KI-020）。所有 cast に着地（ADD）→ `resolveEffectiveHandMaxSize` が User の実効手札上限へ合算 |

- `cs.*`／`ar.*`／`partSlot.*` は**仮想名前空間**——実スキーマパス（`system.combatSpeed.*`・`system.partSlots` 等）と分けることで、アクター自身の効果がネイティブ適用と二重に効くのを防ぐ。
- **部位キー（フェーズ12）**: 部位スロットは `key`（安定参照）と `value`（表示ラベル）を持つ。AE・アイテム part 行・エイリアスの参照は**キー**で持ち、表示は**逆引きした現在ラベル**（識別キー・分類コードキーと同じ原則。ラベルのリネームで参照が壊れない）。既定ラベルは英語キー（`one-hand`＝片手持ち・`occiput`＝後頭部・「〜部」は `-part` 等・2026-07-15 ユーザー確定）。既存データは ready 一回の `migratePartSlotKeys` でキーを永続付与。正本は [Outfits.md](../Game_Rules/Outfits.md)「部位管理」。
- **予約名前空間（ability / control / cs / ar / baseAttack / category / handMaxSizeMod）以外の `system.<パス>` はすべて素のパラメータキー＝「乗っているアイテム自身」の意味**（§2.2）。したがって**アクター上の効果に素の `system.<パス>` を書いてもキャラ値には効かない**（self スコープは保持元が Item のときだけ着地する）。アイテムのトップレベルフィールドと予約語の衝突は無いことを確認済み（style の `control`・uses の `type` 等は入れ子で無関係）。
- ネイティブ着地（base を直接書き換える枠外の特例）は `system.handMaxSizeMod` と、`system.baseAttack` の damageType 以外（`.value`／`.mod`）のみ。他の実スキーマパスの直書きは規範外。

### 2.2 値バフ② アイテムへの着地——乗り先の選び方（この表が全数）

| 書き方 | 乗り先 | 適用経路 |
|---|---|---|
| `system.<パラメータ>`（**素のパラメータキー**） | 効果（またはその転送/付与コピー）が**乗っているアイテム自身**。旧 `system.self.*`／`item.self.system.*` の後継（§2.10） | 適用パスが直接（所持時のみ） |
| `item.<識別キー>.system.<パラメータ>` | 識別キー**完全一致**のアイテム。**前方一致 `*` 可**（`item.<プレフィックス>*.system.<パラメータ>`）。式（`@item.<識別キー>.system.*`）と同じ綴り・**唯一の綴り** | **物理転送**（§2.5） |
| `system.category.<分類キー>.<パラメータ>` | 大分類/小分類/**疑似分類**のアイテム群（分類キーの全数は §2.4） | **物理転送**（§2.5） |
| （キーでなく）AE 設定「**準備先（親アイテム）に適用**」チェック（flags.applyToParent） | **素のパラメータキー**の変更が**準備先ホスト**（`parentItemId`）へ（準備で転送・解除で除去。素のキー以外の変更は落ちる＝混在非対応） | **物理転送**（§2.5） |

- **キーは「どのパラメータ＋絞り込み」だけを語り、乗り先の関係は AE 自身の設定が語る**（v3）。
- 識別キー（技能）と `category`（アウトフィット）で住み分け（アウトフィットは識別キーで個体指定しない方針）。
- **識別キー指定は、識別キーを持つアイテムにしか届かない**（辞典由来のアイテムは持つ。手作りアイテムは武器シートで識別キーを設定するか、素のキー／カテゴリを使う）。ここが「効かない」ときの最頻の原因。

### 2.3 値バフ③ 着地できる `<パラメータ>`（型別・全数）

**キーの綴りと着地の正規化（2026-07-13）**: キーは式と同じフルパス綴り（`….attack.value`・`….attack.total`・`….levelTotal` 等）で書かれるため、**素値/実効のどちらの綴りでも必ず実効（total 系）へ正規化して着地**する（`attack.value`→`attack.total` 等。`resolveItemTotalPath`）。**設定欄（base）は不変**で、追加モードは実効値の上に加算・上書きモードは実効値を置き換える。

| 型 | パラメータ（キーに書く綴り → 着地 total） |
|---|---|
| **全アウトフィット共通** | `buy`（購入値）・`hide`（隠匿値）・`appearancePenalty`（危険値）・`hack`（電脳制御値）・`preserveExp`（常備化経験点）→ 各 `.total` |
| **武器 weapon** | `attack`（攻撃力）・`guardValue`（受け値）→ `.total` ／ `FAValue`（フルオート値）→ `FAValueTotal` ／ `attack.damageType`（ダメージ種別上書き）→ `attack.damageTypeTotal` |
| **防具 armor** | `defence.S`／`defence.P`／`defence.I`（防御力）→ `defence.{S,P,I}_total` ・ `controlMod`（制御値修正）→ `.total` |
| **義体 cyborg** | `attack`・`guardValue`・`defence.S/P/I`（＋共通） |
| **ヴィークル vehicle** | `attack`・`defence.S/P/I`・`speedFactor`（移動力）・`passenger`（乗員）・`controlMod`（＋共通） |
| **IANUS ianus** | `controlMod`（＋共通） |
| **タップ tap** | `cycle`（サイクル）・`combatSpeedMod`（CS 修正）（＋共通） |
| **住宅施設 residence** | `appearanceTarget`（外見目標値）・`cyberSecurity`・`analogSecurity`（セキュリティ）→ `<名>Total` |
| **技能 generalSkill / styleSkill** | `level`（技能レベル）→ `levelTotal` |

- **数値フィールドで total を持たないもの**（`quantity`・`uses`・`ammo`・`range` 等）には効かない。文字列・真偽の上書きは §2.3b（ダメージ種別）・§2.3c（特性フラグ）・§2.3d（名前装飾）で個別に扱う。
- ※アイテムのパラメータが「キャラクターへ」与える修正（controlMod・combatSpeedMod・危険値など）は AE でなく派生パイプラインが直接読む（大原則5）。本表は「アイテム自身の実効値」を AE で動かす話。

#### 2.3b ダメージ種別の上書き（2026-07-13）

値は S/P/I/X の選択式・**モードは何でもよい（常に上書きとして適用）**。着地は実効フィールド（設定欄の素値は書き換えない）。**そのまま書けば動くキー**:

| 置き場所と狙い | キー |
|---|---|
| 武器自身に効果を置く（**手作りの武器はこれ**・識別キー不要） | `system.attack.damageType`（素のパラメータキー・v3） |
| 別のアイテム（技能等）から特定の武器を指定 | `item.<武器の識別キー>.system.attack.damageType` |
| カテゴリ一括（小分類/大分類） | `system.category.<カテゴリ>.attack.damageType` |
| 生身 | `system.baseAttack.damageType` |

#### 2.3c 特性フラグのオン/オフ（フェーズ12・この表が全数）

真偽の**特性フラグ**を AE でオン/オフする。値＝オン（`true`/`1`）／オフ（`false`/`0`）——AE 設定シートでは**オン/オフの選択式**。**モード不問で常に上書き**。着地は実効フィールド `<フラグ>Total`（設定欄=素値は不変）。乗り先の選び方は値バフと同じ（素のキー＝乗っているアイテム自身・`item.<識別キー>.<フラグ>`・`system.category.<分類>.<フラグ>`）。

| 型 | フラグ（キーに書く綴り） | 主な読み手 |
|---|---|---|
| 武器 | `isFullAuto`（フルオート）・`isLaser`（レーザー武器）・`isFleshChange`（生身変更装備） | 攻撃フロー・部位ホスト照合・使用武器表示 |
| アウトフィット共通 | `isCyber`（サイバーウェア）・`isMutantOrgan`（変異器官）・`isShiki`（式神装備）・`noPrepareRequired`（準備不要）・`isConsumption`（消費アイテム） | ホスト照合・「※」表示・準備ゲート・常備化経験点 |
| アウトフィット共通（使用不可状態・2026-07-18） | `isMalfunction`（故障）・`isDestroyed`（破壊） | 使用不可ゲート（アイテムロール禁止・使用武器の攻撃不可）・修理対象の列挙・アクターシートの行表示（故障=赤/破壊=グレー） |
| 技能共通 | `usesBounty`（報酬点使用可）・`suits.spade`／`.heart`／`.diamond`／`.club`（使用スート） | 判定エンジン・判定/攻撃フロー・用途シート |

- **読み手は必ず `readFlag(system, "<フラグ>")`** を経由して実効値を読む（`<フラグ>Total ?? base`）。素の直読みは AE が効かない。
- **故障/破壊の読み手は `isOutfitMalfunctioning`／`isOutfitDestroyed`／`isOutfitUnusable`（helpers.mjs）を経由**する（readFlag＋**サービス大分類の免疫**が唯一の権威）。サービスのアウトフィットは AE でフラグを立てても実効は常に false（故障も破壊もしない・2026-07-18 ユーザー確定）。付与手段はアイテムシートのヘッダートグル（素値）と本 AE キー（実効上書き）の2系統（[Outfits.md](../Game_Rules/Outfits.md)「故障・破壊」）。
- **`noPrepareRequired` の順序境界**: アウトフィット集計（`computeOutfitAggregates`＝制御値/CS/危険値）は `_applyEffectBuffs` より前に走るため**base を読む**（AE 反転は乗らない）。実行時ゲート（ダメージ/判定/戦闘表示）は AE 後に走るため実効を読む。
- **除外＝`isAction`／`noCombo`**（フェーズ12・意図的な非対応）: どちらも「特性」だが**構造的読み手**（用途チェーン設定・辞典構築）がフラグを読んで**永続設定を書き込む**。実行時 AE 上書きと両立させるには「config 書き手は base・実行時解決は実効」の作り分け＝チェーン再解決の設計が要るため、実効読みへの一括掃引が構造的に安全でない。よってレジストリ（`AE_FLAG_PARAMS`）から外す（将来別設計で扱う）。レジストリの正本はコード（`AE_FLAG_PARAMS`）。

#### 2.3d 名前装飾（フェーズ12・アイテムのみ）

**キー `name`**（system 外・ドキュメント直下）。効果（またはその転送/付与コピー）が乗っている**アイテム自身の名前**を実行時に飾る（in-memory・`_source.name` は不変）。

| 値 | 挙動 |
|---|---|
| `{}` を含む（例 `{}（マジック）`） | `{}` を**現在の名前に置換**（D&D 方式。「ロングソード（マジック）」） |
| `{}` を含まない（ベタ打ち） | 名前を**そのまま上書き** |

- 複数効果は priority 順に**重ね掛け**（非 stackable は identity ごとに1回・先勝ち）。識別キー狙いは `item.<識別キー>.name`（物理転送で対象アイテム上の素の `name` コピーになる）。
- **ネイティブ適用の一点抑止**: 自動適用オン（transfer:true）のアイテム効果はネイティブ適用パスで**アクターに**適用されるため、`name` を放置すると `actor.name` を破壊する。`TokyoNovaActiveEffect.apply` が **Actor への `name` 変更だけ**無効化する（アイテム名の装飾は適用パスが bearer へ直接行う）。アクターの名前は装飾対象外（アイテムのみ）。
- **表示/編集の分離**: 一覧・チャット・シート見出しは装飾後（`item.name`／`tnxDecoratedName`）を表示。アイテムシートの名前入力欄は**素値**（`sourceName`＝`_source.name`）を編集する（装飾後を素値として保存しないため）。

#### 2.3e アイテム部位行の追加（フェーズ12）

**キー `system.part.<部位キー>`**。値＝`and`（追加占有）／`or`（択一候補）、消費数は `:N` 接尾（省略=1。例 `and:2`）。着地はアイテム派生の実効 part（`partAdded`。base の部位編集欄は不変）。乗り先は値バフと同じ（素のキー＝乗っているアイテム自身・`item.<識別キー>.system.part.<部位キー>`・分類）。

- **and 追加**＝既存部位に加えて常に占有（表記「(既存)+C」）。base の relation（and/or）と独立に無条件加算。
- **or 追加**＝base 部位一式との**択一候補**（表記「(既存)、もしくはC」）。装備先に選んだとき（`partAltChoice`＝部位キー・空=base を占有）base の代わりに占有する。base の part モデル自体は書き換えない。

### 2.4 分類キー（`system.category.<分類キー>`・全数）

正本は `outfit-categories.mjs`（`{majorKey:{minors:{minorKey}}}`）。大分類・小分類は同じ名前空間だが**衝突しない**（小分類キーは全体で一意）。疑似分類は同じ名前空間の予約語（v3）。

| 大分類キー | 小分類キー |
|---|---|
| `weapon`（武器） | `melee` 白兵武器 ・ `ranged` 射撃武器 ・ `mounted` 搭載兵器 ・ `weaponOption` 武器オプション ・ `specialAmmo` 特殊弾 |
| `armor`（防具） | `bodyArmor` ボディアーマー ・ `armorGear` アーマーギア ・ `fashion` ファッション |
| `cyberware`（サイバーウェア） | `ianus` IANUS ・ `ianusOption` IANUSオプション ・ `neuralware` ニューラルウェア ・ `artificialBody` アーティフィシャルボディ ・ `organicware` オーガニックウェア ・ `psychoApp` サイコアプリ ・ `cosmetic` コスメティック ・ `fullCyborg` 全身義体 |
| `tron`（トロン） | `pocketron` ポケットロン ・ `ptOption` PTオプション ・ `tap` タップ ・ `software` ソフトウェア ・ `hardware` ハードウェア |
| `vehicle`（ヴィークル） | `groundVehicle` 地上車両 ・ `aircraft` 航空機 ・ `ship` 船舶 ・ `walker` ウォーカー ・ `drone` ドローン ・ `vehicleOption` ヴィークルオプション |
| `housing`（住宅） | `residence` 住宅施設 ・ `housingOption` 住宅オプション ・ `housingAccessory` 住宅アクセサリ |
| `item`（アイテム） | `tool` ツール ・ `magicItem` マジックアイテム ・ `biotech` 生体装備 ・ `drug` ドラッグ ・ `food` フーズ |
| `service`（サービス） | `social` ソーシャル ・ `background` バックグラウンド ・ `extra` エキストラ ・ `combiner` コンバイナー |
| **疑似分類**（予約語） | `generalSkill` 一般技能 ・ `styleSkill` スタイル技能（アイテムタイプで束ねる） |

### 2.5 アイテム狙いの適用経路——物理転送＋供給元が正の片方向同期（2026-07-13 転送確定/v3 改定・2026-07-12 同期追加）

**自動適用オン**の効果の `item.<識別キー>`／`system.category`（疑似分類含む）の変更は、効果の作成/更新時・アイテムの追加/装着系更新時（parentItemId・分類・識別キーの変更＝v3 で updateItem フックを追加し「装着変更が反映されない」穴を塞いだ）に**対象アイテム上の実体コピー**として転送される（キーは素の `system.<パス>` に書き換え・由来は origin／transferredFrom フラグ・effectId/stackable は引き継ぐ）。「準備先（親アイテム）に適用」チェックの効果は素のパラメータキーの変更が準備先ホストへ同様に転送される。コピーは対象アイテムの通常の効果＝**無条件でそのアイテムのパラメータに効く**（識別キーの再照合に依存しない）。遠隔の直接適用は廃止（コピーと二重にならない）。

**同期は供給元が正の片方向**（「後から供給元を更新しても反映されない」とのユーザー指摘で無同期から変更）: 供給元の更新（変更行・有効/無効・名前・アイコン・自動適用・フラグ）でコピーは現在値に上書きされ、供給元がそのアイテムを狙わなくなったとき（キー変更・自動適用オフ化・準備解除）／供給元の効果・供給元アイテムが削除されたときはコピーも除去される。**コピー側の手動編集・切替は供給元の次の更新で上書きされる**（コピー自身の削除は独立＝供給元へ波及しない。ただし供給元が狙い続ける限り次の更新で再転送される）。既存の効果は一度保存（または OFF→ON）すると転送される。コピーは対象アイテムのエフェクト一覧の**「転送された効果」セクション**に表示される（名前は斜体＋供給元名・操作は通常の効果と同じ）。**使用時付与のコピー（grantedFrom）は同期対象外の一回性インスタンス**（失効は当面手動＝持続時間は時間管理フェーズ）。

### 2.6 条件付き（標準キーに畳み込む）

`<セレクタ>[<条件>].<パス>`。角括弧内に `パス 比較 値`、複数は `;` 区切り。`parseEffectConditions` / `evalEffectConditions`。

- 例: `system.category.melee[hack>=3].attack` = 電制 total が 3 以上の白兵武器の攻撃力。
- 比較演算: `>= <= > < == !=`。左辺は対象アイテムの system パス（`hack`→`hack.total` 等に解決）。
- 条件はキー文字列内で完結（標準 UI のみで書ける）。

### 2.7 判定バフ（判定時・`check.` / `controlCheck.`・この表が全数）

| キー | 対象 |
|---|---|
| `check.all` | **判定全般**（無印「判定」＝能力値判定＋技能判定のすべて・2026-07-13）。制御判定には掛からない（用語規約）。※`all` は識別キーの予約語 |
| `check.<技能識別キー>` | その技能の技能判定（識別キー完全一致。例 `check.melee`） |
| `check.<プレフィックス>*` | 技能判定（プレフィックス前方一致＝固有名詞技能の大分類はこれで束ねる。例 `check.society_*`） |
| `check.style.<スタイル識別キー>` | **そのスタイルのスタイル技能すべて**（技能の `system.style` で照合。2026-07-10） |
| `check.works.<組織識別キー>` | **そのワークスのワークス技能すべて**（技能の `system.special.works.organization` で照合。2026-07-10） |
| `check.reason` / `.passion` / `.life` / `.mundane` | その能力値の能力値判定 |
| `controlCheck.reason` / `.passion` / `.life` / `.mundane` | その能力値の制御判定（**能力値のみ**。グループ参照不可・`controlCheck.all` は現状なし＝必要時に追加） |
| `check.cardValue` | **カード数字の上書き**（2026-07-13）: 判定に使用したカードの数字を **A〜K で上書き**する（値は AE 設定シートで選択式・自由入力なし・先勝ち）。以降の数字規約は上書き後の数字に従う（手札の絵札=10・A の21固定選択・山札の絵札=FUMBLE）。ジョーカー/切り札の宣言には適用しない。制御判定は対象外（用語規約）。結果カードに「数字上書き（元 → 後）」を明示 |
| `check.suitChange` | **スート変更マーカー**（値不要・2026-07-12）: 判定で使用できないスートのカードを使用可能なスートに変更できる（[Check_Rules](../Game_Rules/Check_Rules.md)「スートの変更」）。他者バフの付与形＝用途の適用効果で対象へ付与。失効は当面手動（「1回の判定」Duration は時間管理フェーズで持続時間側に追加＝**持続時間と AE キーは独立**）。制御判定は対象外 |

- 末尾にパラメータを付けない技能キー＝その技能の判定への加算（`.judgment` 等の接尾は不要）。
- **グループ参照（`check.style.*` / `check.works.*`）**: 識別キーでなく技能の `system.style` / `system.special.works.organization` で束ねる。`parseEffectTargetKey` は `{scope:"skillCheck", group:"style"|"works", selector:<キー>}` を返し、`checkChangeMatches` は判定に使った技能のうち該当 style/organization を持つものがあれば適用。

### 2.8 ダメージバフ（ダメージ時・`damage.*`・この表が全数・2026-07-10〜12）

ダメージ算出時に攻撃者の effects から集計してフラット加算する（判定バフと同じ実行時評価の別系統）。※**対象（ダメージを受ける側）の effects を評価するもの**が2群ある: `damage.replaceTag`/`damage.addTag`（チャート適用時）と、**受けるダメージ軽減の `damage.taken`/`damage.fromStyle`/`damage.fromWorks`（ダメージ算出時・2026-07-17）**。

| キー | 対象 |
|---|---|
| `damage.dealt` | **与えるダメージ +値**（全系統・2026-07-11） |
| `damage.dealt.<physical\|mental\|social>` | その系統の攻撃の与えるダメージ +値 |
| `damage.vsStyle.<スタイル識別キー>` | 攻撃対象がそのスタイル（`type:"style"` アイテム）を持つとき +値 |
| `damage.vsWorks.<組織識別キー>` | 攻撃対象がそのワークス（ワークス技能の組織）を持つとき +値 |
| `damage.taken` | **受けるダメージ +値（対象側・2026-07-17）**: 値は受けるダメージへの加算＝**負で軽減**・正で増加（`dealt` と対称の符号規約）。**恒久軽減**としてダメージ算出時＝防御力・受け値と同段（スタン/説得の 10 上限の**前**）に効く。X 種別にも効く（X が無視するのは対応防御力のみ）。派生ダメージには効かない（軽減不可）。「別のダメージとして適用」でも元系統で照合 |
| `damage.taken.<physical\|mental\|social>` | その系統のダメージを受けるとき +値 |
| `damage.taken.<S\|P\|I\|X>` | その**ダメージ種別**（肉体のみ）のダメージを受けるとき +値（種別セレクタは肉体を含意・精神/社会には合致しない） |
| `damage.fromStyle.<スタイル識別キー>` | **攻撃者**がそのスタイル（`type:"style"` アイテム）を持つとき、受けるダメージ +値（`vsStyle` の対称） |
| `damage.fromWorks.<組織識別キー>` | **攻撃者**がそのワークス（ワークス技能の組織）を持つとき、受けるダメージ +値（`vsWorks` の対称） |
| `damage.replaceTag.<元タグ>` | **ダメージタグ改変（対象側・2026-07-12）**: チャート適用時、元タグを値のタグへ**置換**（例 `damage.replaceTag.stupor`＝`dominated`＝昏睡→支配）。値=CONDITION_KINDS のタグキー（一覧は [Conditions.md](Conditions.md)） |
| `damage.addTag.<元タグ>` | 同・元タグ付与時に値のタグを**追加**（例 `damage.addTag.erased`＝`dominated`＝抹殺に支配を追加）。追加分は同じ負傷に紐づく |

- `parseEffectTargetKey` は `{scope:"damageVs", group:"style"|"works", selector:<キー>}` を返す。**恒常適用パスからは除外**（対象が見えないため焼き込めない）。
- ダメージフロー（`damage-flow.mjs` `openDamageRollDialog` → `collectDamageVsBonuses`）が攻撃対象を解決し、対象の style/works と `gatherDamageVsSources`（`collectActorEffectBuffs` で集めた攻撃者の effects）で照合。合致分を用途のダメージ修正と同じ台帳行（供給元＝効果名）としてダメージへ合算する。
- 値は**リテラル**（判定バフと同じ）。重複規約も判定バフと同一（§4＝行は合算・インスタンス単位で最も効果の大きい1つ・stackable は累積）＝`_gatherBonusSources` を判定バフと共用。
- **taken 系（受けるダメージ軽減・2026-07-17）の固有規約**:
  - `parseEffectTargetKey` は `{scope:"damageTaken", category, damageType}` / `{scope:"damageFrom", group, selector}` を返す。恒常適用パスからは除外（他の実行時系統と同じ）。
  - 重複規約は §4 のとおり（インスタンスの強者採用は**受け手基準＝最小値**＝最も負。`gatherDamageTakenSources` が pick=min で集計）。taken と from\* は1回の走査で照合し、同一効果に併記された行は**内容として合算した1行**になる（同名効果の複数インスタンスも合算値どうしで比較）。旧記述「併記しても1回だけ（最有利1つ）」と「攻撃側 dealt×vs\* は二重加算の疑い」は**誤りで撤回**（2026-07-17 ユーザー裁定＝行はすべて効果の内容として適用）。
  - 集計はダメージフローが**対象ごと**に評価（対象行プレビュー `targetPlannedPreview`・適用ダイアログ→ `collectDamageTakenRows`。攻撃者は `attackerUuid` から解決して from\* を照合）。攻撃側 `vs*` のような「先頭命中対象で近似」は生じない。内訳は対象行・適用ダイアログ「軽減（算出済み）」・適用済み表示に効果名で帰属（符号つき）。
  - AE 設定シートのキー候補（datalist）に `damage.dealt`/`damage.taken` の固定キーを載せる（識別キーを取る `vs*`/`from*` は候補外）。

### 2.9 モード・値

- モード = 標準の Add / Override / Multiply / Upgrade / Downgrade。値 = 数値。
- 複数効果は `change.priority`（既定 mode×10）順に適用。

### 2.10 廃止キー（死にキー・書いても何も起きない）

v3 再設計（2026-07-13）で廃止した旧綴り。パーサが明示的に null を返す（素のキーと紛れて誤解釈しない）。

| 旧キー | 現行の書き方 |
|---|---|
| `system.self.<パス>` ／ `item.self.system.<パス>` | 素のパラメータキー `system.<パス>`（§2.2） |
| `system.parent.<パス>` ／ `item.parent.system.<パス>` | AE 設定「準備先（親アイテム）に適用」チェック＋素のパラメータキー |
| `system.skill.<識別キー>.<パス>` | `item.<識別キー>.system.<パス>` |
| `system.name` | 素のキー `name`（system 外・§2.3d） |
| `system.part`（セレクタ無し） | `system.part.<部位キー>`（§2.3e） |

- ※式の相対参照 `@item.self.system.*`／`@item.parent.system.*` は「値を引いてくる」参照で**別物・存続**（[Formula_Reference.md](Formula_Reference.md)）。

### 2.11 無視ゲート（`ignore.*`・実行時・対象自己・フェーズ12・ユーザー確定）

「特定の（あるいは全ての）ダメージや BS の**効果を受けない**」を表す。**ダメージ・BS を防ぐ／消すのではなく**、自分が受けているコンディションが与える**具体的な効果（数値ペナルティ・行動制限の両方）を消費段階で無視**する（タグ・BS・負傷そのものは残る＝シートにバッヂは出る）。値不要のマーカーキー（`check.suitChange` と同型）。ルール解釈の詳細は [Conditions.md](Conditions.md)「効果の無視ゲート」。

| キー | 無視する対象 |
|---|---|
| `ignore.all` | あらゆるコンディションの効果 |
| `ignore.bs` | 全 BS の効果（group=bs） |
| `ignore.bs.<kind>` | 個別 BS（例 `ignore.bs.poison`＝邪毒） |
| `ignore.damage` | ダメージ由来の効果すべて（全系統） |
| `ignore.damage.<physical\|mental\|social>` | その系統のダメージ由来の効果（負傷自身＋付随 BS＋付随戦闘不能） |

- **個別・グループ・全ての全域**を表現できる（複数キーは OR 合成）。「邪毒と酩酊だけ」＝2キー、「全BS＋肉体由来」＝`ignore.bs`＋`ignore.damage.physical`。
- **ダメージ由来の照合**: 各効果の `woundCategory`（負傷自身）または `woundSource` → 元負傷の `woundCategory` で「物理／精神／社会由来か」を辿る（`getEffectiveConditions`）。
- **消費点への配線（正規アクセサ経由が唯一の正本）**: 実効コンディションを**実効果として読む箇所は必ず `getEffectiveConditions(actor)` を経由**し、`effectIgnored` の行を飛ばす（`readFlag` と同じ「必ず経由」規約）。存在自体（バッヂ表示・回復・治療・カスケード）は別途 `readConditions`／`actor.effects` を使う（無視されても残る）。既存消費点＝達成値ペナルティ（酩酊／萎縮／憎悪）・電子妨害・重圧ブロック（`check-flow` `_gatherConditions`）／制御値ペナルティ・部位スロット（`character-base`）。**将来の消費点**（恐慌のリアクション不可・捕縛の武器不可・狼狽・戦闘不能・邪毒＝フェーズ13）は同アクセサ経由で書けば自動でゲートされる（消費点追加時は grep 全数掃引＝`AE_FLAG_PARAMS` と同じ規約）。
- **表示**: 無視中の BS はバッヂを残しつつ**取り消し線＋淡色**、tooltip に「『〈供給元〉』により無視」（供給元＝ignore を与えた効果名）。トークンのステータスアイコンは対象外（ユーザー確定）。
- **手動オーバーライド（卓ツール・2026-07-16）**: ルールの `ignore.*` AE とは別に、BS バッヂのコンテキストメニュー「効果を無視する／無視を解除する」で、そのインスタンスの `flags.tokyo-nova-axleration.manuallyIgnored` をトグル。**同じ `effectIgnored` に合流**（`getEffectiveConditions` の第2ソース）＝消費側の抑止・取り消し線を共有（tooltip は「手動で無視中」）。可否ゲートなし・いつでも（RL／プレイヤーの手動裁定）。
- 純ロジック: `ignoreRuleMatches`（照合）・`gatherIgnoreRules`（`collectActorEffectBuffs` から規則収集）。キー解析は `parseEffectTargetKey`（`scope:"ignore"`）。**実行時系統＝恒常の値バフ適用（`_applyEffectBuffs`）からは除外**。

## 3. 恒常適用パス（`CastDataModel.prepareDerivedData` → `_applyEffectBuffs`）

```
1. 各アイテム prepareDerivedData: computeItemEffectiveValues で total=base を算出（エフェクト抜き）
   能力値: total = growth + Σ(style×level) + mod + outfitMod（effectMod 項は廃止）
2. _applyEffectBuffs:
   a. アクター自身＋全所有アイテムの effects から有効な変更を集める（v3: 自動適用ゲート
      effectAutoApplies を通過したもののみ。「準備先に適用」の供給元自身も除外=準備先のコピーが担う）
   b. parseEffectTargetKey でセレクタ/条件/パス解析。check 系（abilityCheck/controlCheck/skillCheck）は除外
   c. _resolveBuffApplications で対象（ability/control/self）と total パスへ展開、条件評価
      （category/skill は物理転送担当のため空・素のキー=self は bearer のアイテム）
   d. **値を2フェーズで適用（2026-07-10）**: フェーズ1=リテラル値の change を priority 順に適用（base→total 確定）、フェーズ2=**式値**の change を評価してから適用。各フェーズ内で同一効果の重複排除（§4）。
3. 能力値・制御値は最後に 0clamp
```

- `effect.apply(targetDoc, {...change, key: "system.<path>", value: String(評価値)})` でモード処理をネイティブ再利用。
- 素のパラメータキー（self スコープ）はそのエフェクトが乗るアイテム自身へ解決。所持時のみ反映。
- **AE の「値」の式参照（2026-07-10 ユーザー確定・実装済み）**: change.value がリテラル数値でなく式なら `evaluateFormulaSync`（同期・prepareDerivedData 内のため）で評価する。参照は判定/ダメージの式と同系統＝`@system.*`・`@item.<識別キー>.system.*`、加えて **AE 値専用の相対参照 `@item.self.system.*`（効果が乗るアイテム自身）/`@item.parent.system.*`（その親＝装備先ホスト・`parentItemId`）**（`buildFormulaData(actor, null, bearer)`）。例: オプションの AE 値 `@item.parent.system.attack.total` で装備先の攻撃力を参照。
  - **順序**: 式値はリテラル値適用後の total を参照するため**リテラル先→式後**の2フェーズ。**式値どうしは相互参照しない**（ユーザー確定）のでフェーズ2 内は順不同で確定する。`@diff`/`@achievement` は判定文脈でないため不可。`.total` は AE 込み（リテラル AE の結果を含む）。
  - ※**用途（判定ボーナス/ダメージ修正等）の式**は実行時（prepareData 完了後）評価なので、参照する item の `.total` は最初から AE 完全適用済み（順序問題なし）。

## 4. 同名効果の二重適用の禁止（値・判定・ダメージ 共通の一般原則・2026-07-17 是正）

トーキョーN◎VA で禁止されるのは**「同名効果の二重適用」**——同じ技能から同じ AE を二回付与される、という事態は起きない（使用時付与の実装も置き換えリフレッシュ）。BS も「重複する」と明記されたもの以外は重複せず、**最も効果の大きいものが常に適用**される（conditions 実装は準拠済み＝同 kind 非重複・最も重いペナルティを採用）。**重複可（stackable）は例外**で、同じ技能から与えた AE でも**付与された分だけ累積**する（「この効果は重複する」と明記された、1度に複数回重ねがけできるタイプのバフ技能）。（以上 2026-07-17 ユーザー裁定の言語化＝本節の旧記述を是正）

- **同一性の判定子 = 効果の安定 ID**: `flags["tokyo-nova-axleration"].effectId` があればそれ、無ければ ActiveEffect の document `_id`。名前変更に耐える（辞典が effectId を振れば、コピー同士も同一効果として扱える）。
- **1つの効果が持つ変更行は、すべてその効果の内容として適用（合算）**——旧記述「同一 identity 内で複数の変更が該当しても最も有利な値（最大）を1回」は **Code の過大一般化で誤り**（2026-07-17 ユーザー指摘で是正。禁止はインスタンス単位の二重適用であって、行の畳み込みではない）。例: 1効果に `check.all +1`＋`check.melee +2` → 白兵判定 **+3**／`damage.taken −2`＋`damage.fromStyle.x −6` → 該当攻撃者から受けるダメージ **−8**。
- **重複排除の規則（インスタンス単位）**:
  - 非 stackable: 同一 identity のインスタンスが複数あっても**最も効果の大きい1つ**だけ適用（比較は行の合算値どうし。攻撃側バフ=最大・受け手側の taken 系=最小＝最も負）。
  - **stackable=true（重複可）**: 重複排除しない＝付与された分だけ累積。
- **判定バフ**（`computeCheckBonus`）・**ダメージバフ**（`dealt`/`vs*`/`taken`/`from*`）: `_gatherBonusSources` が上記を実装（**行の合算 → インスタンスの強者採用**）。
  - 「バサラ/マヤカシに+1、組み合わせても+1」= **1つの変更行**が複数技能に該当しても、その行は判定に1回だけ効く（プレフィックスが2技能に当たっても +1）。
- **値バフ**（`_applyEffectBuffs`）: インスタンス間は `(対象ドキュメント, total パス, モード, identity)` ごとに最大値1つ＝インスタンス単位の規則に準拠。※**1効果内で同一パスへ同一モードの複数行**を書いた場合もこのキーで畳まれる（合算されない）エッジが残る——病的入力のため据え置き（要すれば別途是正）。
- ※攻撃側ダメージバフの `dealt` と `vs*` は別走査で集計されるが、行はどのみち全て適用されるため通常は結果が一致する。**同名効果の複数インスタンスが系統違いの行を持つ**場合のみ、インスタンス跨ぎで行が混成しうるエッジが残る（taken 系は1走査の合算で回避済み。要すれば別途統合）。

### 判定フローへの配線

`tnx-check-flow.mjs` `_computeCheckBonus(actor, ctx, abilityKey)` が effects を集め、判定種別で criteria を作る:
- controlCheck → `{type:"control", ability}`、skillCheck → `{type:"skill", skills:[{key:識別キー, style, organization}]}`（グループ参照のため識別キーに加えスタイル/組織も載せる。旧 `skillKeys` も後方互換で受理）、それ以外 → `{type:"ability", ability}`。
- `calcSkillCheck` は達成値に `+checkBonus`、`calcControlCheck` は制御値を `+checkBonus`（成功条件が緩む）。報酬点プレビューにも反映。

## 5. AE 設定シートへの注入 UI（詳細タブ・v3 で拡張）

標準 ActiveEffectConfig の**詳細タブ**に注入する（`renderActiveEffectConfig` フック、`scripts/tnx.mjs`）:

| 注入 | 保存先 | 表示条件 |
|---|---|---|
| **「効果を対象に自動適用」**（ネイティブ transfer のラベル・ヒント差し替え） | ネイティブ transfer | 常時 |
| **「準備先（親アイテム）に適用」**チェック | `flags.applyToParent` | アイテム上の効果 かつ 自動適用オン |
| **「付与先」**セレクタ（対象/自分・既定=対象） | `flags.grantTarget` | **アイテム上の効果で常時**（2026-07-13 ユーザー指摘で是正: 用途の適用効果は自動適用オンの効果も選択できる＝付与先は transfer と**直交**。オフ時のみ表示だと自動適用オンの効果の付与先に到達できない。アクター上の効果は用途の効果になれないため非表示） |
| **「重複可」**チェック | `flags.stackable` | 常時。未チェック＝重複適用不可、チェック＝スタック |

**変更（changes）行の入力補助（`renderActiveEffectConfig`・値をベタ打ちさせない）**:

| 対象キー | 値入力の見た目 |
|---|---|
| `check.cardValue` | A〜K の選択式 |
| `*.attack.damageType`／`system.baseAttack.damageType` | S/P/I/X の選択式 |
| 特性フラグ（§2.3c・キーが `AE_FLAG_PARAMS` を指すとき） | **オン/オフの選択式**（フェーズ12） |
| `system.part.<部位キー>`（§2.3e） | `and`/`or`/`and:2`/`or:2` の datalist（消費数付きも打てるよう自由入力保持・フェーズ12） |

- **キー入力の datalist**（フェーズ12）: 全 change 行のキー欄に候補 datalist（`system.partSlot.<キー>`・`system.part.<キー>`・`name`・`system.<フラグ>`）を付けてオートコンプリート（自由入力は妨げない）。部位キーはワールドの部位スロットプリセットから引く。
- **保存はネイティブ項目と同じフォーム送信**（`name="flags.tokyo-nova-axleration.*"` を与えて「変更を保存」で一括保存・2026-07-13 ユーザー指摘で是正）。旧方式の change 即時 `setFlag` は、ドキュメント更新→シート再描画で**未保存のフォーム状態（transfer のオン等）が巻き戻る**バグの原因だった（「準備先をオンにすると自動適用がオフに戻り欄が消える」＝実際はオフになるのではなく未保存のオンが破棄されていた）。
- **準備先のみ** transfer チェックの切替で出し分ける（常時自動適用の乗り先修飾そのもので、使用時付与ではコピー作成時に applyToParent を落とすためオフ時に意味を持つ経路が無い。純粋な表示制御でデータには触れない）。隠れている間も設定値はドキュメントに残るが、エンジンが transfer 状態でゲートするため実害はない。
- **申し送り**: コンディション（BS）効果値フィールドの注入は従来どおり即時 `setFlag`（独自の再注入ロジックと不可分）。同種の巻き戻りを潜在的に持つが、statuses 変更との連動要件があるため別途扱い。

## 6. 構造化の前提（別途・後続）

**判定バフのスタイル/ワークス述語は実装済み（2026-07-10）**: スタイル技能の `system.style`・ワークス技能の `system.special.works.organization` を照合キーに用い、`check.style.<キー>` / `check.works.<キー>` で束ねる（§2「判定バフ」）。専用欄の辞典読み取りドロップダウンによる入力補助や、値バフ（`system.*`）側のスタイル/ワークスのグループ化・ワークス装備フラグは未着手（当面は識別キー/プレフィックス/分類・疑似分類で賄う）。

分類（カテゴリ）の**コードキー化は実装済**（`category.melee` 等。`outfit-categories.mjs` が `{majorKey:{minors:{minorKey}}}`）。

## 参照

- [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md) / [Check_System.md](Check_System.md)
- [Character_Creation.md](../Game_Rules/Character_Creation.md)（0clamp）
