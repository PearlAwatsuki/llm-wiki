# Phase 9: ActiveEffect / 派生値計算パイプライン

> ▶ **設計中（9-0）**。本ページは設計フェーズの叩き台。サブフェーズ 9-1 以降の実装着手は、§「9-0 で裁定する論点」をユーザーと確定し承認を得てから行う（CLAUDE.md §2.2 設計→実装分割）。タグ付け・push はユーザー。

## 目的

`prepareDerivedData` を確立し、**実効値（能力値・制御値）計算を一本化**する。あわせて ActiveEffect が実効値に正しく流れ込む計算経路を作る。

N◎VA のダメージ適用＝チャート式効果累積（＝ActiveEffect）であり、派生値計算と AE 土台は後続フェーズ（10〜20）すべての前提になる。AE がこのシステムの心臓であると判明したため、判定・戦闘の前に本フェーズを繰り上げた（Roadmap 2026-06-20 再編）。

## スコープ

- **含む**: 派生値パイプライン確立、実効値一本化、最終値0clamp、outfitMod/appearanceModifier の B-2 化、AE 適用モデル確立、KI-020 解消、バッドステータスの AE 表現、効果文ユーザー入力機能、AE 持続時間の「進行単位」表現の**基盤**。
- **含まない（申し送り）**:
  - ダメージ算出（カード数字＋攻撃力＋修正−軽減）→ フェーズ12
  - チャート参照→効果付与の**実行**（damage apply のトリガ）→ フェーズ12
  - AE 持続時間の**境界発火**（カット/シーン/アクト境界でのリセット・失効の発火）→ フェーズ13/14/15
  - EXP 同期機構の書き換え（§4.2 禁止。KI-008 もこの領域で対象外）

## 設計の前提（ルール正本）

- [Character_Creation.md](../Game_Rules/Character_Creation.md) — 能力値/制御値の最終値0clamp、社会技能の区分タグ論点
- [Damage_Rules.md](../Game_Rules/Damage_Rules.md) — ダメージ＝チャート式効果累積、バッドステータス＝AE
- [Time_Management.md](../Game_Rules/Time_Management.md) — 効果の持続時間＝進行単位

---

## 現状把握（コード実態 / 2026-06-20）

### 実効値計算が4箇所に重複

能力値・制御値の実効値式 `growth + Σ(style.value × level) + mod + effectMod + outfitMod`（制御は control 系）が、以下に重複している：

| 箇所 | 用途 |
|---|---|
| `tnx-cast-sheet.mjs` `_prepareAbilitiesContext`（~L1383） | シート表示 |
| `tnx-cast-sheet.mjs` `_computeMundaneTotalValue`（~L2021） | mundane 単独（bountyBase 算出） |
| `tnx-judgment-flow.mjs` `_buildAbilitiesCtx`（~L367） | 判定の達成値 |
| `tnx-cast-sheet.mjs` `updateCastExp`（~L2042） | EXP コスト（**触れない**。形だけ同型） |

### 派生値パイプライン未確立

- `prepareData` / `prepareDerivedData` のオーバーライドは**現状ゼロ**（CLAUDE.md §3）。
- `effectMod` / `controlEffectMod` フィールドは AE の着地点として用意済みだが**参照0件**＝AE が実効値に流れ込んでいない（KI-020 と同根の構造問題）。
- `outfitMod` / `appearanceModifier` は**フックで計算し `{syncing:true}` で DB に書き戻す**方式（`tnx.mjs` `recalcOutfitAggregates`／起動時スキャン／isGhost 変更ハンドラ）。派生値を DB に焼く Foundry アンチパターン。

### 既存の AE 土台（部分的に存在）

- `effects-sheet-mixin.mjs`（AE リスト UI）
- `CONFIG.statusEffects`（バッドステータス11種、`tnx.mjs` L395）
- usage → `effects[]` 参照（用途使用時に付与する AE の参照、`usage.mjs`）
- **欠けているもの**: AE 変更が実効値に反映される計算経路。

---

## 決定事項

### B-2: 派生値はフック書き戻しでなく prepareDerivedData で算出

「派生値（合計）をどこで算出するか」の軸で、**フック書き戻し（B-1）→ prepareDerivedData での都度算出（B-2）**へ移行する（2026-06-20 ユーザー確定）。

- これは「修正値を AE で持つか直接書き換えるか」の軸とは**無関係**。outfitMod/appearanceModifier は依然として「アイテムから直接計算される数値」であり AE 化はしない。変わるのは「DB に保存するか／その場で出すか」だけ。
- 利点: 派生値を DB に焼かない本来形。`{syncing:true}`・無限ループ回避の小細工が消え、KI-008（一括インポート時の競合）のリスク源も減る。

**B-2 化の対象（2つに確定）**:

| 対象 | 現状 | 移行後 |
|---|---|---|
| `outfitMod`（携帯アウトフィットの制御値/CS修正の合算） | `updateCastOutfitMods` がフックで DB 書き戻し | `prepareDerivedData` で items から都度算出 |
| `appearanceModifier`（アウトフィットの外見ペナルティ合算） | `updateCastAppearanceModifier` が同様 | 同上 |

**B-2 化しないもの**:

- **EXP 同期機構**（`updateCastExp` / `syncCastExpToUser` / history・ownerUserId 同期）— §4.2 で書き換え禁止。EXP は User flag が権威でドキュメント跨ぎの実保存が必要なため、そもそも B-2 対象外。
- **`bountyBase`**（ボタン操作で mundane 実効値をスナップショット保存）— 継続派生値でないため自動 B-2 化しない（論点5 で要確認）。

### AE 適用順序

Foundry は AE の changes を `prepareBaseData()` と `prepareDerivedData()` の**間**に `system.*` へ適用する。これを前提に：

```
prepareBaseData       … スキーマ初期値・style/growth など素の値
  ↓
applyActiveEffects    … AE changes を effectMod 等へ加算（ADD モード想定。論点1）
  ↓
prepareDerivedData    … 実効値 = growth + Σ(style×level) + mod + effectMod + outfitMod を算出
  ↓                      outfitMod / appearanceModifier もここで items から算出
最終値0clamp           … total = max(0, …)。全修正合算後の最終値に適用（順序非依存）
```

---

## 9-0 論点と裁定（2026-06-20 ユーザー確定）

§4.2 ルール解釈・§4.3 DataModel 変更に関わる5論点を確定した。

1. **AE の着地モデル** — ✅ **確定**: 技能/アイテム効果の AE は `effectMod` / `controlEffectMod` に **ADD モード**で着地 → 派生で合算 → 最終値0clamp。`.total` への直接着地は採らない。
2. **バッドステータスのデータ表現** — ✅ **確定**: status flag だけのマーカーでなく **changes を持つフル AE** とし、**全 BS の自動化を目指す**。ただし AR 減少・タイミング制限・カット終了回復など戦闘/判定/時間管理機構が前提の効果は、その**実発火**が 13/14/15 の領分。9-4 では「全 BS を changes ベース AE として表現する器」＋「現機構で表現できる効果」を実装し、残りは各機構が載った時点で changes を積み増す。効果文はルール本文でなく**ユーザー入力テキスト**として AE に持たせる（§4.1 焼き込み回避）。
3. **社会技能の区分タグ** — ✅ **確定（タグを設けない）**: 区分まるごとバフは存在するが、区分内の**公式社会技能は有限**であり、その identificationKey を**列挙**して指定する（公式コンテンツはキー付与済みで堅牢）。タグ新設はそれを正当化する強い理由がない（入力負担論点は wiki で決着済み）。ユーザーオリジナル社会技能が区分バフに自動マッチしない点は**受容するトレードオフ**。Character_Creation の「区分タグの要否を再判断」申し送りは**クローズ**。
   - 技能は system フィールドでなく Item のため、このバフ照合は素の Foundry AE changes でなく**判定/派生層のカスタム照合**で行う（9-3 実装要素）。
4. **AE 持続時間モデル** — ✅ **確定**: 各 AE が自分の進行単位（アクト/シーン/カット/メインプロセス）を**フラグで保持**する。各単位の「開始」でカウンタを進め、AE は生成時のカウンタ値を記録、カウンタが進めば失効、という Foundry duration 同様の形を想定。**フェーズ9 はこのフラグ（スキーマ）の確定まで**。境界の実発火は 13/14/15 へ申し送り。
5. **bountyBase** — ✅ **確定（スナップショット維持）**: 報酬点はプレアクトで確定し、アクト中の外界値変動を反映しないルールのため、prepareDerivedData での派生化（＝外界変動に追従）は誤り。ボタン操作で値を焼くスナップショット方式を維持する。
   - これにより **B-2（派生化）の対象は「常に最新であるべき真の派生値」に限定**され、outfitMod / appearanceModifier の2つのみ。**意図的なスナップショット**（bountyBase、および同方式で表現予定の CS ベース・CS）は B-2 対象外。outfitMod 内の combatSpeed 修正は引き続きライブな派生入力であり、将来 CS ベースのスナップショット取得時にその時点値を読む（B-2 と CS スナップショット計画は非競合）。

---

## サブフェーズ

### 9-0: 設計確定 ▶ 進行中

- [x] 現状把握（実効値4箇所重複・派生値パイプライン未確立・既存 AE 土台の棚卸し）
- [x] B-2 方針確定・対象確定（outfitMod / appearanceModifier）
- [x] 本ページ起草
- [x] 上記5論点をユーザー裁定（2026-06-20 確定）
- [ ] Audit_and_Review にメカニクス監査・設計レビュー起票
- [ ] 9-1 実装着手の承認取得

### 9-1: 派生値パイプライン確立 ✅ 完了 (2026-06-20)

- [x] `CastDataModel.prepareDerivedData` 新設、実効値を `system.<key>.total` / `.totalControl` に集約（システム初の prepareDerivedData オーバーライド）
- [x] **最終値0clamp** をここで適用（能力値・制御値とも。`computeAttributeFinal` で `max(0, …)`、順序非依存）
- [x] 3箇所の重複（シート表示 `_getAbilitiesData`・mundane `_computeMundaneTotalValue`・判定 `_buildAbilitiesCtx`）を集約値参照に置換（EXP コストは対象外）
- [x] **KI-021 解消**: 判定の `equipped` フィルタ欠落バグ（達成値からスタイル基本値が脱落）を一本化で修正
- [x] `computeAttributeFinal` 単体テスト追加（tests/data/helpers.test.mjs。全766テスト green・lint 新規警告ゼロ）
- [ ] 実機回帰確認（シート表示値が不変・判定達成値にスタイル基本値が載る）→ ユーザー確認待ち

### 9-2: outfitMod / appearanceModifier の B-2 化 ✅ 完了 (2026-06-20)

- [x] `outfitMod` / `appearanceModifier` の算出を `CastDataModel.prepareDerivedData`（`_prepareOutfitAggregates`）へ移動。能力値 total が outfitMod を読むため先に計算
- [x] 集計を純粋関数 `computeOutfitAggregates` に切り出し単体テスト追加（8件）
- [x] フック書き戻し（`recalcOutfitAggregates`＋3フック / 起動時スキャンの集計呼び出し / isGhost 変更ハンドラ）と関連 `{syncing}` 書き込みを撤去。起動時スキャンは EXP（実保存・User flag 同期）のみ残置
- [x] schema フィールド（outfitMod / appearanceModifier）は不変（派生で上書き＝§4.3 非該当）。各クライアントがローカル算出する本来形に（GM 限定書き戻しを解消）
- [ ] 実機回帰確認（アウトフィット着脱・isGhost 切替で制御値修正・CS・危険値が即時反映）→ ユーザー確認待ち
- [ ] isGhost 時の CS 修正無効化ロジックを派生計算側へ移設
- [ ] 回帰確認（EXP 同期機構に波及していないこと）

### 9-3: AE 適用モデル確立

> 詳細設計は [Active_Effects.md](../Game_Mechanics/Active_Effects.md) が正本（バフの2分類・モードA/B・prepare 順序・クロスターゲット注入・棚卸し・DataModel 変更提案）。アイテム横断バフは素の Foundry AE で扱えないため、本サブフェーズはカスタム照合層の設計が核。**アイテムの実効値（AE 込み）の表現は全パラメータをフェーズ9 で確定**する（表示=10・ダメージ=12 はそれを消費するだけ。旧「歩調を合わせる」案は撤回、Active_Effects §11）。

**9-3 スコープ確定（2026-06-20）**: アイテム着地フィールドは**全 buff 対象パラメータ**（ヘルパー単位で網羅）。除外＝住宅エリア供給値・スタイル能力値（それ自体がエフェクト的）。FA値・技能レベルは含む。

- [x] **着地フィールド新設（§4.3 承認済・substrate 完了 7c6bef0）**: `modeValueField` に effectMod（隠匿値/電制/危険値/購入値/常備化EXP/受け値/制御修正/SF/乗員/サイクル/CS修正/スロット数を一括）、`defenceField` に S/P/I_effectMod、attack.effectMod（既済）、weapon.FAValueEffectMod、skill levelEffectMod、residence の3stat effectMod。漏れ防止のためヘルパー単位で追加
- [x] 能力値の AE 着地（モードA・actor）は 9-1 で成立済（Foundry が `system.<key>.effectMod` を適用→9-1 が total に合算）
- [x] アイテム実効値 total の派生算出（`computeItemEffectiveValues`＋OutfitBase/SkillBase の prepareDerivedData。mode=value 時のみ value+effectMod、clampなし）`cc8bcc7`
- [x] 表示切替（アウトフィット概要・キャスト装備列・_attackLabel を total 参照。編集入力は base 維持）`81ae6d7`。コンバイン表示は合成元 base のため不変。residence の登場/セキュリティ列（effectiveValues 経由）への AE 反映は申し送り
- [x] モードB（アクター→アイテム）のカスタム照合・注入エンジン `b14f12e`。AE flag `aeTarget`（byItemType/byMajorCategory/byMinorCategory「白兵武器」等/byIdentificationKey、指定次元AND）を `CastDataModel._applyCrossTargetEffects` が照合し一致アイテムの total に加算。flag 設定 UI は改造判定(17)/スタイル技能(20)の領分
- [x] **KI-020 解消** `e4fc70c`: cast に AE 着地点 `handMaxSizeMod` を新設し、`resolveEffectiveHandMaxSize` を3層化（①既定 ②User flag ③所有 cast の AE 合算、0clamp）。User は AE を持てないため所有 cast の AE から合流
- [x] **AE 配線（必須・抜けていた）** `ef274fb`: `CONFIG.ActiveEffect.legacyTransferral=false`（これが無いとモードA 自己適用・転送が機能せず着地点に流れ込まない）。効果一覧を allApplicableEffects 列挙へ、転送効果の編集/削除を resolveEffect で解決
- [ ] AE 適用の実機確認（モードA：アイテム自己 effectMod／能力値 effectMod、転送 true、モードB：横断バフ、handMaxSizeMod）→ ユーザー確認待ち

#### 9-3 v2 再設計（2026-06-21・実装完了）

上記の effectMod 着地＋モードB（aeTarget flag 照合）方式は、OVERRIDE/MULTIPLY を表現できない・非存在フィールド前提など限界が判明し、**「total 直接適用」方式へ全面再設計**（ユーザー確定）。詳細は [Active_Effects.md](../Game_Mechanics/Active_Effects.md)（v2 正本）。

- [x] **effectMod 着地フィールド全廃**（`e02b920`）: modeValue/defence/attack/weapon.FAValue/skill.level/residence/能力値の effectMod 系を削除。`computeItemEffectiveValues` は total=base のみ（形状・base名で検出）。`computeAttributeFinal` から effectMod 項除去。能力値内訳の「効果」行撤去
- [x] **v2 キー文法**（`parseEffectTargetKey`）: 値バフ `system.<名前空間>`（ability/control/self/parent/category.<小or大キー>/skill.<識別キー[*]>/handMaxSizeMod）、条件 `[path op value;...]`。判定バフ `check.` / `controlCheck.`（技能識別キー/プレフィックス・能力値・制御）
- [x] **恒常適用パス**（`CastDataModel._applyEffectBuffs`）: アクター＋全所有アイテムの effects を集約→解決→`effect.apply` で total へモード適用（priority 順）。transfer 非依存（legacyTransferral=false）
- [x] **判定時バフ**（`computeCheckBonus` / 判定フロー `_computeCheckBonus`）: 判定種別ごとの criteria 照合で達成値・制御値に加算。`calcSkillCheck`/`calcControlCheck` に checkBonus 配線。報酬点プレビューにも反映
- [x] **同一効果の重複適用不可（一般原則）**: 値・判定の両方で、安定 ID（`flags.effectId` ?? document `_id`）単位に重複排除・最大採用。stackable のみスタック
- [x] **重複可チェックボックス UI**: ActiveEffectConfig 詳細タブに注入（`renderActiveEffectConfig`、`flags.stackable` へ setFlag）
- [x] **死蔵ヘルパー撤去**（`cac11f6`）: matchesAeTarget / addToItemEffectMod / parseCrossTargetKey / migrateAttackModToEffectMod 削除
- [x] 全テスト green（814）・lint 0 error（既存 15 warning）
- [ ] 実機確認（値バフ・判定バフ・重複可・条件付き category）→ ユーザー確認待ち
- [ ] 申し送り: スタイル/ワークス述語の構造化（辞典ドロップダウン・専用欄）。手札上限の横断パス統一（現状 native）

### 9-4: コンディション（BS・戦闘不能）の AE 表現

> 設計正本: [Conditions.md](../Game_Mechanics/Conditions.md)（4型モデル・5論点裁定済み・承認待ち）。BS（Bad_Status）と戦闘不能（Damage_Rules）を同一モデルで扱う。付与 UI は既存の `CONFIG.statusEffects`。

**5論点 裁定（2026-06-21）**: ①`check.*`=上方判定のみ（制御は明示時のみ）②重圧=該当能力値を使う判定を不可 ③萎縮/憎悪=対象UUID＋`game.user.targets`照合で 9-4 完結（13不要）④重複明記なしは非重複・強度は最大n ⑤専用UIなし（既存ステータスエフェクト）。

- [ ] 文法拡張: `parseEffectTargetKey` にワイルドカード（`check.*`/`controlCheck.*`/`system.ability.*`/`system.control.*`）と block スコープ（`block.*`）
- [ ] 数値修正型①: 酩酊/衰弱を標準 changes で（既存適用パス・判定バフに乗る）
- [ ] 行動制限型②: `block.check.<ability>`（重圧）・`block.attackWith.<識別キー>`（捕縛）を判定/攻撃判定時に遮断。恐慌・メインプロセス不可は 13 申し送り
- [ ] 計算型③: 電子妨害n（category＋電制≤n の所持数→全判定マイナス、上限/特例）
- [ ] 条件付き攻撃判定デバフ: 萎縮/憎悪を対象UUID＋Foundry targets 照合で
- [ ] 11 BS＋戦闘不能状態の `CONFIG.statusEffects` 化・型割り当て（Conditions §3/§4）
- [ ] AE フラグスキーマ（intensity/conditionKind/targetUuid/targetMode/durationUnit。持続は基盤のみ・発火は 13/14/15）
- [ ] 効果文（ルール本文でなくユーザー入力テキスト）の入力機能
- [ ] 著作権配慮確認（チャート本文・効果文を本体へ焼き込まない。§4.1）

### 9-5: テスト・仕上げ

- [ ] 派生値計算・AE 適用・clamp の単体テスト
- [ ] wiki 更新（本ページ・関連ルールページ・Audit_and_Review）
- [ ] CHANGELOG・バージョンアップ（フェーズ完了で MINOR up → v0.9.0）

---

## 完了条件

- 実効値計算が `prepareDerivedData` に一本化され、シート・判定・登場判定が同一の集約値を読む
- 能力値/制御値の最終実効値が0を下回らない（0clamp）
- outfitMod / appearanceModifier が DB に焼かれず派生算出される（B-2）
- 技能/アイテム効果の AE が実効値に反映される（KI-020 解消含む）
- バッドステータスが AE として表現され、効果文をユーザーが入力できる
- 全既存テスト + 新規テストがグリーン

## 関連

- [Roadmap.md](../../02_System/Roadmap.md)
- [Character_Creation.md](../Game_Rules/Character_Creation.md)
- [Damage_Rules.md](../Game_Rules/Damage_Rules.md)
- [Time_Management.md](../Game_Rules/Time_Management.md)
- [DataModel.md](../Architecture/DataModel.md)
- [Active_Issues.md](../Issues/Active_Issues.md) — KI-020
- [Phase_8_Tasks_Detail.md](Phase_8_Tasks_Detail.md)
</content>
</invoke>
