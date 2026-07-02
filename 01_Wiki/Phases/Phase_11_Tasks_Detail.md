# Phase 11: guest / troop / extra シート

> ▶ **進行中（2026-07-03 計画承認・着手）**。11-0 完了。次は 11-1（設計）。
>
> 未整備の Actor シート（guest / troop / extra）を、キャスト共通部品化のうえで実装する。トループ／エニグマ／分身の本体・作成ルールもここで確定・実装する（Roadmap: 残シートを1フェーズに統合）。依存は判定システム（完了済み）とキャスト共通部品化のみ。神業（フェーズ20）への依存はない。

## 目的

全 Actor type にカスタムシートを行き渡らせ、「シートを持たないアクター」を解消する。guest はキャストの縮退形（共通部品の再利用）、troop は heads＝HP的リソースを持つ集団、extra は名前中心の最軽量アクターとして、それぞれのルール上の性格に合ったシートを作る。

## スコープ

**含む**（フェーズ11 宛て申し送りの全件・2026-07-03 洗い出し）:

| # | 内容 | 出所 |
|---|---|---|
| 1 | guest / troop / extra カスタムシート | フェーズ6-7/6-8/6-9 移管 |
| 2 | キャスト共通部品化（guest シートの前提） | [Sheets.md](../UI/Sheets.md)・フェーズ6 移管注記 |
| 3 | troop heads のトークンリソースバー連携 | フェーズ6-8 注記・[Actor_Types.md](../Documents/Actor_Types.md) |
| 4 | トループ／エニグマ／分身の本体・作成ルール | フェーズ10 申し送り・[Character_Creation.md](../Game_Rules/Character_Creation.md) |
| 5 | トループの heads 減少・トループ級/エキストラのダメージ例外の器 | [Damage_Rules.md](../Game_Rules/Damage_Rules.md) 申し送り |
| 6 | guest / troop の能力値実効値（`computeAttributeFinal` を各 `prepareDerivedData` から） | フェーズ9-1 監査申し送り |
| 7 | `OUTFIT_TYPES` 重複リファクタ | 設計レビュー 6-6 推奨アクション |
| 8 | extra の initiative 不可（combatSpeed 非保持）の扱い | KI-010 解消記録の残項目（フェーズ11/13） |

**含まない（申し送り）**:

- 攻撃フローからのダメージ自動適用（heads 自動減算の配線を含む）→ **フェーズ12**
- CSカレントのカット中自動管理 → **フェーズ13**
- シーンプレイヤー明示 → **フェーズ14 以降**（シーン/登場概念の実装後・2026-07-03 確定）
- 登場判定 → **フェーズ18**

## 設計の前提（ルール正本）

- [Actor_Types.md](../Documents/Actor_Types.md) — 各 type の DataModel 現況（guest=固有フィールドなし / troop=memo+heads / extra=biography のみ）
- [Terminology.md](../Game_Rules/Terminology.md) — トループ定義（スタイル1つ・神業不可・人数がHP・個性なし）
- [Troops.md](../Game_Rules/Troops.md) — **トループ・トループ級キャラクターの正本**（2026-07-03 言語化: heads・エニグマポイント・分身・共通制約）
- [Damage_Rules.md](../Game_Rules/Damage_Rules.md) — トループ級・エキストラのダメージ例外
- [Character_Creation.md](../Game_Rules/Character_Creation.md) — トループ取得技能（〈分心〉→エニグマ、〈分身〉→分身。フェーズ10 で ID 関連付けまで実装済み）
- [Check_System.md](../Game_Mechanics/Check_System.md) / [Check_Rules.md](../Game_Rules/Check_Rules.md) — 判定（判定フローはアクター型非依存であることをコードで確認済み）
- **guest 確定方針**: キャストからライフパス・報酬点・セッション履歴（EXP 含む）を除いたもの（Actor_Types.md）

---

## サブフェーズ

### 11-0: 準備リファクタ ✅ 完了（2026-07-03）

- [x] `OUTFIT_TYPES`（cast シートの static・重複定義）を `scripts/data/helpers.mjs` の `OUTFIT_ITEM_TYPES` に一本化（本体コミット `5b07a9f`）。両集合は同一の10種で挙動不変。テスト980件 green・lint 新規警告なし。設計レビュー 6-6 の推奨アクションを解消。

### 11-1: 設計（ルール言語化＋データ構造確定 → 一括承認）

CLAUDE.md §2.2 に従い、ルールの言語化とデータ構造（追加フィールド）をここで確定し、まとめて承認を得る。

- [ ] **guest の DataModel 追加フィールドの取捨**: cast 固有フィールドのうち guest にも要るもの（戦闘タブ・アウトフィットタブを持つ場合: `partSlots` / `baseAttack`・`baseDefence`・`baseGuard` / `weaponRefs` / `isGhost` / `appearanceModifier`・`outfitMod` 等）。ownerUserId・exp・history・bounty・lifePath は持たない（確定方針）。
- [x] **troop の本体・作成ルールの言語化**（ユーザー・2026-07-03 受領）: [Troops.md](../Game_Rules/Troops.md) に記録。heads＝純粋な人数（敵対トループは RL 任意・取得技能由来は判定達成値）・**トループ側で自動化すべき点はない**。技能・アウトフィット・能力値の持ち方は残確認。
- [x] **トループ級キャラクター（エニグマ・分身）の類型の言語化**（同上・Troops.md）: エニグマ＝エニグマポイントが HP 相当（達成値適用）、分身＝1点で消滅（ダメージ管理不要）・本体とほぼ同一データ・スタイル複数・AR=1 固定・CS/CSカレント=0 固定。全類型で神業不可・トループ取得技能使用不可。
- [ ] **heads／エニグマポイント／分身固定値のデータ構造**への落とし込み（フィールド設計・一括承認対象）。
- [ ] **extra のシート表現**: スタイル・神業・技能・アウトフィットを持てる（表示）。ダメージ概念なし。initiative 不可（combatSpeed 非保持）の扱い。
- [ ] **ダメージ例外の 11/12 境界確定**（案: 11 は heads リソース＋手動増減 UI＋例外の器まで。攻撃フローからの自動適用はフェーズ12 で配線）。
- [ ] **キャスト共通部品化の設計**: cast-sheet.hbs（848行）の partial 分割と、共有基底クラスの範囲（tnx-cast-sheet.mjs は約2,500行）。
- [ ] **トループ取得技能の本体生成フロー**（フェーズ10 の ID 関連付けの続き。※トループ側の自動化は不要とユーザー明言＝人数/ポイントの自動転記は必須でない）。
- [x] **戦闘タブの AR 表示**（追加タスク・ユーザー指示 2026-07-03。フェーズ10 戦闘タブの積み残し追補）→ **実装完了（2026-07-03・本体 `fb9c08d`・実機確認待ち）**。言語化は [Combat_Flow.md](../Game_Rules/Combat_Flow.md) AR 節（基準は付与型＝「カット進行のシーン開始時に 1 付与」／クリンナップは付与値まで全回復／シーン進行中は消費不可。**付与型・シーン中消費不可はルール明記でなくユーザー解釈**＝原文との矛盾が判明したら見直す）。実装構造: 基準値の入力欄なし・ルール定数 1（`ACTION_RANK_GRANT`）＋freeMod＋AE 着地（`system.ar.max`）の派生 `maxTotal`、保存は `value`（現在AR）と freeMod のみ、表示は CS と同じ自動制御、カット開始時に付与自動セット。消費/回復の自動管理はフェーズ13。分身 AR=1 固定は修正を付けないデータ運用。**要確認: ar.max への準備状態ゲート適用（cs.base の特例の2例目・Code 判断）**。記録: Audit_and_Review / Design_Review_Entries（2026-07-03）。

### 11-2: キャスト共通部品化（挙動不変リファクタ）

11-1 で確定した設計に沿って cast シートを部品化し、cast 側の回帰確認をこの段で締める。詳細タスクは 11-1 確定後に記載。

### 11-3: guest シート

DataModel 追加フィールド＋`prepareDerivedData`（`computeAttributeFinal` 等、実効値を cast と同経路に）＋シート実装。判定起動の実機確認。

### 11-4: troop シート

実効値派生、シート（memo・heads・能力値・CS）、heads のトークンリソースバー連携、heads 手動増減（ダメージ例外の器）。エニグマ/分身の類型対応。

### 11-5: extra シート

biography＋所持アイテム表示（スタイル・神業・技能・アウトフィット）。

### 11-6: トループ取得技能の本体連携

〈分心〉→エニグマ、〈分身〉→分身の本体生成・作成ルール実装。

### 11-7: 仕上げ

- [ ] テスト・記録（メカニクス監査／設計レビュー 各エントリ）・wiki 整合化
- [ ] CHANGELOG 更新・v0.11.0 バージョンアップ（package.json / system.json / タグの3箇所同期。タグ付け・push はユーザー）

## 完了条件

- guest / troop / extra がカスタムシートを持ち、フェーズ4 デザインで表示・編集できる
- troop の heads がトークンリソースバーに連携している
- トループ／エニグマ／分身の作成ルールが wiki に言語化され、実装と整合している
- v0.11.0 タグ

## 関連

- [Roadmap.md](../../02_System/Roadmap.md)
- [Phase_10_Tasks_Detail.md](Phase_10_Tasks_Detail.md)（前フェーズ・申し送り元）
