# ロードマップ

完走志向版。各フェーズはサブフェーズに細分して着実に仕上げる。

**「最小実装」という語は使わない** — 各機能は作り込む。

## 進行状況

```
✅ フェーズ0   土台を v13 で動かす
✅ フェーズA   開発基盤整備 (v0.1.0)
✅ フェーズB   DataModel 完全移行 (v0.2.0)
✅ フェーズ1   v13 UI 整合
✅ フェーズ2   Player → User flag 移行 (v0.3.1) [2026-06-05]
▶  フェーズ3   既存シート ApplicationV2 移行 ← 現在 [2026-06-04]
⬜ フェーズ4   シート内容追加 (combatSpeed 等)
⬜ フェーズ5   CSS リファクタ + デザイン刷新
⬜ フェーズ6   未実装シート追加
⬜ フェーズ7   判定システム 設計
⬜ フェーズ8   判定システム 実装
⬜ フェーズ9   神業システム 設計
⬜ フェーズ10  神業システム 実装
⬜ フェーズ11  戦闘進行/サブシーン
⬜ フェーズ12  公開準備 → v1.0.0
```

## フェーズ詳細

### フェーズ2: User 移行 (完了)

プレイヤーの EXP・履歴・手札を User flag へ移行し、player Actor を廃止した。

- HUD ApplicationV2 化はここでは行わず、参照先の書き換えのみ実施
- KI-005 ベース連携決着 (`resolveEffectiveHandMaxSize` 新設)
- ActiveEffect 修正 (KI-020) は新設 ActiveEffect フェーズへ

→ [Phase_2_User_Migration.md](../01_Wiki/Phases/Phase_2_User_Migration.md)

### フェーズ3: シート ApplicationV2 移行 (現在)

ApplicationV1 を完全廃止。cast を含む全既存シート・HUD・ダイアログ群すべてを V2 に移行する。デザインは含まない。

→ [Phase_3_Tasks_Detail.md](../01_Wiki/Phases/Phase_3_Tasks_Detail.md)

### フェーズ4: シート内容追加

不足機能実装。KI-010 (combatSpeed UI) 等。ルール解釈はユーザー判断。

### フェーズ5: CSS + デザイン

サイバーパンク基調。HUD デザイン含む (構造はフェーズ2 完了前提)。

→ [Phase_5_Tasks_Detail.md](../01_Wiki/Phases/Phase_5_Tasks_Detail.md)

### フェーズ6: 未実装シート

outfit 系 Item 11 種 + guest/troop/extra Actor 3 種。

### ActiveEffect フェーズ (新設・位置未確定)

派生値計算パイプライン設計 + KI-020 (handMaxSize ActiveEffect 修正) を含む。
フェーズ6 で対象フィールドが固まった後、判定システム設計の前を想定。

### フェーズ7-8: 判定システム

**7 = 設計 (Opus チャット)**, **8 = 実装 (Claude Code)**。
usage ベース。KI-002 解決含む。

### フェーズ9-10: 神業システム

同上。設計→実装分割。

### フェーズ11: 戦闘/サブシーン

コンバットトラッカー改修。FVTT Scene 非依存の軽量シーン切替。

### フェーズ12: 公開準備

v1.0.0 リリース。

## UI フェーズ順序の意図

```
2: 所有モデル (User) 確立
 ↓
3: シート構造 V2 化
 ↓
4: シート内容追加
 ↓
5: デザイン一括適用 (HUD 含む)
 ↓
6: 未実装シート新規作成 (5 のデザイン踏襲)
```

## バージョニング

| ルール | 内容 |
|---|---|
| 現在 | 0.x.x (開発中) |
| フェーズ完了 | MINOR up (例: フェーズB → 0.2.0) |
| フェーズ間修正 | PATCH up |
| 公開 | 1.0.0 (フェーズ12) |

## ロードマップ変更

順序は今後変更されうる。**現在フェーズは CLAUDE.md §2.2 を正とする**。

## 関連

- [Release_Process.md](Release_Process.md)
- [Completed_Phases.md](../01_Wiki/Phases/Completed_Phases.md)
