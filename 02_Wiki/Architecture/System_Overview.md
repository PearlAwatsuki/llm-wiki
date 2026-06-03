# システム概要

## プロジェクト

**トーキョーN◎VA THE AXLERATION (TNX)** 第5版の Foundry VTT システム実装。

### ゲームシステム上の特性

- **判定**: カード (Foundry Cards API) ベース。`Roll` クラス・ダイスロールは未使用
- **進行**: 1アクト = シーンの連なり (シーン制)
- **神業 (miracle)**: ゲーム進行を書き換える強力な効果。数値計算だけでは表現困難
- **経験点**: キャストではなく**プレイヤー**に付与 (FEAR 系)。フェーズ2 で User flag へ移行中

## エントリポイント

| ファイル | 役割 |
|---|---|
| `system.json` | システム定義・`documentTypes`・互換性・Compendium |
| `scripts/tnx.mjs` | 初期化・フック登録・シート登録・設定 UI |
| `lang/ja.json` | 日本語ローカライズ |
| `css/tnx.css` | 全 UI スタイル |

## ディレクトリ構成

```
scripts/
  tnx.mjs                 # メインエントリ
  data/
    abstract.mjs          # SystemDataModel + mixin()
    helpers.mjs           # Actor 側フィールドヘルパー
    actor/                # Actor DataModel (5 type + common/)
    item/                 # Item DataModel (17 type + common/)
    card/                 # Card DataModel (3 type + common/)
  actor/                  # Actor シート (ApplicationV1)
  item/                   # Item シート
  journal/                # アクトシート (JournalSheet)
  module/                 # HUD・ダイアログ・同期ロジック等
templates/                # Handlebars テンプレート
tests/                    # Vitest ユニットテスト
```

## 技術スタック

| 項目 | 現状 |
|---|---|
| FVTT | v13.351 検証済 |
| モジュール | ESM (`.mjs`) |
| シート API | **ApplicationV1** 主体。レコードシートのみ ApplicationV2 |
| データ | **TypeDataModel** (25 type)。`template.json` 廃止 |
| テスト | Vitest |
| リント | ESLint |
| CI | GitHub Actions |

## 派生値の計算方式

`prepareData()` / `prepareDerivedData()` のオーバーライドは**なし**。

派生値はすべてイベント駆動の手動計算:

- **能力値表示**: `TokyoNovaCastSheet.getData()` → `_getAbilitiesData()` (永続化しない)
- **EXP**: `TokyoNovaCastSheet.updateCastExp()` — `ready` フックの CRUD 連動
- **手札上限**: `ready` フックで一括更新

## Compendium Packs

`system.json` で 7 パック定義 (すべて Item):

`styles`, `miracles`, `general-skills`, `style-skills`, `works-skills`, `outfits`, `works-outfits`

デフォルト所有権: PLAYER=OBSERVER, ASSISTANT=OWNER

## ソケット

`system.json` で `"socket": true` を宣言しているが、`game.socket.emit/on` の実装は**未実装** (KI-006)。`game.tnx.refreshSheets()` は自クライアントのみ更新。
