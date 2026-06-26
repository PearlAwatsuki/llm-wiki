# システム概要

## プロジェクト

**トーキョーN◎VA THE AXLERATION (TNX)** 第5版の Foundry VTT システム実装。

### ゲームシステム上の特性

- **判定**: カード (Foundry Cards API) ベース。`Roll` クラス・ダイスロールは未使用
- **進行**: 1アクト = シーンの連なり (シーン制)
- **神業 (miracle)**: ゲーム進行を書き換える強力な効果。数値計算だけでは表現困難
- **経験点**: キャストではなく**プレイヤー**に付与 (FEAR 系)。フェーズ2 で User flag へ移行完了

## エントリポイント

| ファイル | 役割 |
|---|---|
| `system.json` | システム定義・`documentTypes`・互換性・Compendium |
| `scripts/tnx.mjs` | 初期化・フック登録・シート登録・設定 UI |
| `lang/ja.json` | 日本語ローカライズ |
| `css/tnx2.css` | 全 UI スタイル（旧 `tnx.css` は廃止） |

## ディレクトリ構成

```
scripts/
  tnx.mjs                 # メインエントリ
  data/
    abstract.mjs          # SystemDataModel + mixin()
    helpers.mjs           # Actor 側フィールドヘルパー
    actor/                # Actor DataModel (cast/guest/troop/extra + common/)
    item/                 # Item DataModel (+ common/)
    card/                 # Card DataModel (+ common/)
  actor/                  # Actor シート (ApplicationV2)
  item/                   # Item シート (ApplicationV2)
  journal/                # アクトシート (DocumentSheetV2)
  module/                 # HUD・ダイアログ・判定・コンディション・同期ロジック等
templates/                # Handlebars テンプレート
tests/                    # Vitest ユニットテスト
```

## 技術スタック

| 項目 | 現状 |
|---|---|
| FVTT | v13.351 検証済 |
| モジュール | ESM (`.mjs`) |
| シート API | **ApplicationV2**（フェーズ3 で V1 全廃。HUD・ダイアログ含む全シート V2） |
| データ | **TypeDataModel**（型構成は `scripts/data`・`system.json` を正とする）。`template.json` 廃止 |
| テスト | Vitest |
| リント | ESLint |
| CI | GitHub Actions |

## 派生値の計算方式

**`CastDataModel.prepareDerivedData` を確立**（フェーズ9、`scripts/data/actor/cast.mjs`）。能力値・制御値の実効値を `system.<key>.total` / `.totalControl` に一本化し、ActiveEffect は適用パス `_applyEffectBuffs` で total へ直接適用、最後に 0clamp する（→ [Active_Effects.md](../Game_Mechanics/Active_Effects.md)）。outfitMod / appearanceModifier も prepareDerivedData で都度算出（B-2、`computeOutfitAggregates`）。

- **シート表示**: `TokyoNovaCastSheet._prepareContext()`（ApplicationV2）→ `_getAbilitiesData()` が集約済み total を読む
- **EXP**: フック駆動の `recalcActorExp()`（CRUD 連動・cast↔User 同期）。EXP は派生化せず User flag が権威（B-2 対象外）
- **手札上限**: `resolveEffectiveHandMaxSize(user)` — ①既定 ②User flag ③所有 cast の AE 合算（`handMaxSizeMod`、KI-020）を 0clamp で解決

## Compendium Packs

`system.json` で 11 パック定義（最新は system.json を正とする）。Item パック（styles / miracles / general-skills / style-skills / works-skills / outfits / works-outfits / housing-areas / life-paths）に加え、`npc`（Actor）・`draw-tables`（RollTable）を含む。

デフォルト所有権: PLAYER=OBSERVER, ASSISTANT=OWNER

## ソケット

`system.json` で `"socket": true` を宣言し、**`TnxSocketHandler` で集約実装済み**（KI-006 解消・フェーズ8-1、`scripts/module/tnx-socket-handler.mjs`）。`ready` フックで `game.socket.on("system.tokyo-nova-axleration", TnxSocketHandler.onMessage)` を登録し、RL 要求判定の結果反映・アクセスカード提示などで稼働。`game.tnx.refreshSheets()` は自クライアントの再描画。
