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

## 今後のフェーズ (概要)

次フェーズ詳細: [Phase_4_Tasks_Detail.md](Phase_4_Tasks_Detail.md)

| フェーズ | 内容 | 状態 |
|---|---|---|
| 4 | デザイン刷新（視覚デザイン）| **現在** |
| 5 | シート内容追加 (combatSpeed 等) | 未着手 |
| 6 | 未実装シート追加 | 未着手 |
| (新設) | ActiveEffect フェーズ(KI-020 含む) | 未着手・位置未確定 |
| 7-8 | 判定システム 設計→実装 | 未着手 |
| 9-10 | 神業システム 設計→実装 | 未着手 |
| 11 | 戦闘進行/サブシーン | 未着手 |
| 12 | 公開準備 → v1.0.0 | 未着手 |

ロードマップ詳細: `../02_System/Roadmap.md`
