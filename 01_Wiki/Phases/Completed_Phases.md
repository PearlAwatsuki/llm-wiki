# 完了フェーズ概要

## フェーズ0: 土台を v13 で動かす ✅

**目標**: v13 起動・シート表示・致命的エラー解消

| タスク | 内容 |
|---|---|
| P0-02 | active-effects-list action 名修正 |
| P0-03 | styleSkill 正規表現修正 |
| P0-04 | initiative 式修正 |
| P0-08 | renderSettingsConfig v13 DOM 対応 |
| P0-09 | renderHotbar v13 対応 (→ 後に削除) |
| P0-01 | verified → 13.351 |

**保留**: P0-07 → フェーズ1-1 で本格対応

詳細: [Phase_0_Tasks_Detail.md](Phase_0_Tasks_Detail.md)

---

## フェーズA: 開発基盤整備 ✅ (v0.1.0)

| サブ | 内容 |
|---|---|
| A-1 | ライセンス・著作権・コントリビューション文書 |
| A-2 | MECHANICS_AUDIT / DESIGN_REVIEW 基盤 |
| A-3 | Vitest テスト基盤 + ESLint |
| A-4 | GitHub Actions CI |
| A-5 | バージョニング・リリース方針 |

---

## フェーズB: DataModel 完全移行 ✅ (v0.2.0)

**目標**: template.json 廃止、全 25 type を TypeDataModel 化

| サブ | 内容 | 完了日 |
|---|---|---|
| B-0 | 設計方針確定 | 2026-05-21 |
| B-1 | Actor 共通 template | 2026-05-21 |
| B-2 | Item 共通 template | 2026-05-21 |
| B-3 | 単純 Actor (guest/troop/extra) | 2026-05-21 |
| B-4 | 複雑 Actor (cast/player) | 2026-05-22 |
| B-5 | 単純 Item 10 type | 2026-05-22 |
| B-6 | 中程度 Item 5 type | 2026-05-23 |
| B-7 | 複雑 Item (style/styleSkill) | 2026-05-23 |
| B-8 | Card 3 type | 2026-05-23 |
| B-9 | template.json 廃止 + HTMLField | 2026-05-24 |
| B-10 | 移行後検証 | 2026-05-24 |

**テスト**: 544 件グリーン (B-10 時点)

詳細: [Phase_B_Tasks_Detail.md](Phase_B_Tasks_Detail.md), [Design_Review_Entries.md](../../02_System/Design_Review_Entries.md)

---

## フェーズ1: v13 UI 整合 ✅ (2026-05-24)

**目標**: KI-011/012/013 解消。**ApplicationV2 本体移行は含まない**。

| サブ | 内容 |
|---|---|
| 1-0 | ホットバー折りたたみ死コード削除 (KI-012) |
| 1-1 | アクトシート作成導線復活 (KI-011) |
| 1-2 | HUD CSS 座標調整 (KI-013) |

**設計判断**: JournalEntryPage サブタイプ化は不採用 (FUTURE_CONSIDERATIONS)

---

## フェーズ2: Player Actor → User flag 移行 ✅ (v0.3.1)

**目標**: player Actor の全役割を User flag へ移行し、player Actor を廃止する。

| サブ | 内容 | 状態 |
|---|---|---|
| 2-0 | User flag スキーマ + レコードシート | ✅ 完了 |
| 2-1 | cast → User 紐付け (`ownerUserId`) | ✅ 完了 |
| 2-2 | EXP・履歴二フック同期 | ✅ 完了 |
| 2-3 | 手札・切り札 User flag + HUD 張替 + cast 手札削除 | ✅ 完了 |
| 2-4 | 既存 player データ → User flag 移行 | スキップ (pre-production) |
| 2-5 | player Actor 廃止 + handMaxSize 連携締め | ✅ 完了 |

詳細: [Phase_2_User_Migration.md](Phase_2_User_Migration.md)

---

---

## フェーズ3: シート ApplicationV2 移行 + CSS リファクタリング ✅ (v0.4.0, 2026-06-07)

**目標**: ApplicationV1 完全廃止。全シート・HUD・ダイアログを V2 に移行し、CSS を構造整理する。

| サブ | 内容 | 完了日 |
|---|---|---|
| 3-0 | V2 移行設計・基底クラス選定・API 調査 | 2026-06-06 |
| 3-1 | TokyoNovaItemSheet + EffectsSheetMixin V2 化 (KI-009 解消) | 2026-06-06 |
| 3-2 | Item 派生シート 5 本 V2 化 | 2026-06-06 |
| 3-2b | アイテムシート CSS 回帰バグ修正 | 2026-06-06 |
| 3-CSS-0 | CSS デザインシステム基盤確立・tnx.css 全体再監査 | 2026-06-07 |
| 3-3 | TokyoNovaCastSheet + TnxHistoryMixin V2 化 (KI-017/018/019 解消) | 2026-06-07 |
| 3-4 | TnxHud ApplicationV2 移行 | 2026-06-07 |
| 3-5 | TnxScenarioSheet ApplicationV2 移行 | 2026-06-07 |
| 3-6 | tnx-dialog.mjs 8 クラス DialogV2 化 + TnxScenarioSettingWizard V2 化 | 2026-06-07 |
| 3-7 | 最終検証・タブバグ修正事後検討・KI 解消記録 | 2026-06-07 |

**テスト**: 625 件グリーン (3-7 時点)
**解消 KI**: KI-009, KI-017, KI-018, KI-019

詳細: [Phase_3_Tasks_Detail.md](Phase_3_Tasks_Detail.md)

---

## フェーズ4: デザイン刷新（視覚デザイン）✅ (v0.5.0, 2026-06-08)

**目標**: tnx2.css を新設し、カラーパレット・タイポグラフィ・装飾要素を確立。旧 tnx.css を廃止。

| サブ | 内容 | 完了日 |
|---|---|---|
| 4-1 | tnx2.css 新設 — Phase 4 サイバーパンク・デザイン基盤 | 2026-06-07 |
| 4-2 | バッドステータスグリッド実装・キャストシート閲覧モード整備 | 2026-06-08 |
| 4-3 | i18n localize/format 呼び出しをハードコード日本語に一括置換 | 2026-06-08 |
| 4-4 | ニューロカードチャット UI 刷新・HUD ラベルデザイン更新（グローチップ）| 2026-06-08 |
| 4-5 | tnx.css 廃止・削除 | 2026-06-08 |

**主要成果**:
- `tnx2.css` を新設し、CSS 変数によるデザインシステム基盤を確立
- サイバーパンク調のカラーパレット（アクセント: `#00c8e0`）・タイポグラフィを定義
- HUD ラベルをグローチップスタイルに刷新
- ニューロカードのチャットメッセージ UI をリデザイン
- バッドステータス表示グリッドを実装
- `game.i18n.localize/format` および `{{localize}}` を全ファイルでハードコード日本語に置換（フェーズ12 まで i18n 凍結）
- 旧 `tnx.css` を削除（`git rm`）

詳細: [Phase_4_Tasks_Detail.md](Phase_4_Tasks_Detail.md)

---

## フェーズ5: 修正・軽微な機能追加 ✅ (v0.6.0, 2026-06-12)

**目標**: 既存シートのバグ修正と、判定システム以前に実施可能な軽微な機能追加。

| サブ | 内容 | 状態 |
|---|---|---|
| 5-0 | 既存シートの表示バグ修正一式 | ✅ |
| 5-1 | v13 非推奨警告の全解消 | ✅ |
| 5-2 | カード渡し・切り札配布の User 化 | ✅ |
| 5-3 | プレイヤーの手札エリア表示 | ✅ |
| 5-4 | カードセットアップを設定メニューへ移行（TnxCardSetupApp） | ✅ |
| 5-5 | 識別キー（identificationKey）全アイテム型追加 | ✅ |
| 5-6 | キャストシート詳細タブ（ライフパス3スロット + 説明エディタ）・プレイヤーシート詳細タブ | ✅ |
| 5-7 | 報酬点（bounty）+ ドロー表（RollTable カードドロー拡張） | ✅ |
| 5-8 | HUD カラム折りたたみ・アクセスカードエリア/使用システム・プレイヤー手札表示 | ✅ |
| 5-9 | 技能セクション再設計・コンテキストメニュー統一・チャット通知デフォルト変更 | ✅ |

**主要成果**:
- キャストシート詳細タブ: `lifePath` を SchemaField 構造に拡張、説明エディタはシート編集モード連動
- コンテキストメニューを「閲覧/編集/削除」（技能は + 複製）に統一
- HUD 両カラムの flex 統一と折りたたみ、アクセスカード提示システム、プレイヤー手札開示
- ドロー表（コア RollTable 保持のフック注入方式。documentTypes に RollTable を追加してはならない教訓を記録）

**移管**: シーンプレイヤー明示 → フェーズ11 / KI-010（combatSpeed UI）配置先未定で継続

詳細: [Phase_5_Tasks_Detail.md](Phase_5_Tasks_Detail.md)

---

## フェーズ6: 未実装シート追加 ✅ (v0.7.0, 2026-06-14)

**目標**: outfit 系 Item 11 種（weapon/armor/cyborg/ianus/tron/tap/vehicle/residence/housingArea/combiner/general）+ lifePath のシート整備、キャストシート「アウトフィット」タブ。

**主要成果**:
- `TokyoNovaOutfitSheet`（共通シート）を 10 種に登録、`HousingArea` / `LifePath` シートを新設
- キャストシート「アウトフィット」タブを全面実装（大分類別グループ・フラグトグル・グループ内 D&D 並び替え）
- アウトフィット集計の自動更新（フック + ready 初期化）、識別キー追加
- guest/troop/extra・キャストシート戦闘タブの実質内容は判定/神業後（フェーズ11）に先送り

詳細: [Phase_6_Tasks_Detail.md](Phase_6_Tasks_Detail.md)

---

## フェーズ7-8: 判定システム 設計→実装 ✅ (v0.8.0, 2026-06-20)

**目標**: TNX のカードベース判定を設計・実装。能力/技能/制御判定・コンボ・RL 要求・使用回数消費まで。攻撃判定＋ダメージ算出は判定システムの未完部分としてフェーズ12 に分離。

**主要成果**:
- 判定エンジン/フロー/ダイアログ（カード値計算・報酬点・切り札・山札判定）
- RL 判定要求 + ソケット基盤（KI-006 解決）
- usage を pseudo-document（単一 UsageTemplate ＋ type 6種）に再設計、`TnxUsageSheet`（基本/発動/効果の3タブ）（KI-002 解決）
- 発動パラメータ（対象/射程/目標値/変更不可※/対決不可）・参加技能からの自動入力・対決の読み取り表示
- 使用回数の消費（D&D 方式の消費ダイアログ・組み合わせ技能の遠隔消費・spent 管理）
- mixin の静的 migrateData 合成修正（潜在的に死んでいた template マイグレーションを修復）

**テスト**: 757 件グリーン
**解消 KI**: KI-002, KI-006

詳細: [Phase_7_Tasks_Detail.md](Phase_7_Tasks_Detail.md) / [Phase_8_Tasks_Detail.md](Phase_8_Tasks_Detail.md)、[Usage_System.md](../Game_Mechanics/Usage_System.md)、[Judgment_Rules.md](../Game_Rules/Judgment_Rules.md)

### フェーズ9: ActiveEffect / 派生値計算パイプライン（完了 v0.9.0, 2026-06-24）

`prepareDerivedData` 確立・実効値一本化（能力値/制御値 total・0clamp）・outfitMod/appearanceModifier の B-2 派生化。**v2 AE 適用モデル**（base/total 分離・total 直接適用・`system.<名前空間>` 値バフ・`check.`/`controlCheck.` 判定バフ・同一効果の重複適用不可・重複可 UI・条件付き）。**コンディション**（BS・戦闘不能・負傷を CONDITION_KINDS へ統合、効果値の固定/可変区別、詳細タブ UI、判定/派生への適用、電子妨害計算、重圧遮断）。**ダメージチャート**（肉体/精神/社会×0-21 同梱、§4.1 境界＝効果文のみユーザー入力、効果文設定アプリ）。**状態カスケード**（inflicts・状態のみ付与・ダメージ由来 AE のリスト非表示）・**カードドロー**（衰弱/重圧）・**controlNegate**・KI-020 解消。発火（block 系/萎縮憎悪/邪毒/回復）は 12/13/15 申し送り。

詳細: [Phase_9_Tasks_Detail.md](Phase_9_Tasks_Detail.md)、[Active_Effects.md](../Game_Mechanics/Active_Effects.md) / [Conditions.md](../Game_Mechanics/Conditions.md)

---

## 今後のフェーズ

現在は **フェーズ10（戦闘タブ）**。以降の一覧・順序・詳細は [Roadmap.md](../../02_System/Roadmap.md) を正本とする（本ファイルは完了フェーズの概要に徹し、未来の番号は重複保持しない）。
