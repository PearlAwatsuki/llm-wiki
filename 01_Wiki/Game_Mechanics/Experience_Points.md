# 経験点 (EXP) システム

FEAR 系の特性: **経験点はプレイヤーに付与**され、キャスト (cast) は消費・計算の単位。

## データ構造

### cast.exp

| フィールド | 管理 | 説明 |
|---|---|---|
| additional | 独立 | 追加経験点 (シート直接入力) |
| total | 派生 | additional + historyTotal |
| spent | 派生 | realSpent - 170 |
| value | 派生 | (total + 170) - realSpent |

### history (オブジェクトマップ)

```javascript
{
  [randomId]: { id, date, title, exp, rl, players }
}
```

- 追加: `system.history.${newId}`
- 削除: `system.history.-=${targetId}`
- Foundry EmbeddedDocument ではなく plain object

### 権威モデル (移行中)

| データ | 旧権威 | 新権威 (フェーズ2) |
|---|---|---|
| history | player Actor | User flag |
| exp.total (履歴合計) | player Actor | User flag |
| exp.spent/value (cast 側) | cast (計算結果) | cast (変更なし) |

## 計算: updateCastExp

`TokyoNovaCastSheet.updateCastExp(actor)` — static, async

**呼び出し元**: createItem/deleteItem/updateItem (cast 親), updateActor (cast), 履歴変更後

```
historyTotal = ownerUserId あり → User flag exp.total
             = なし           → Σ cast.system.history[*].exp

abilityExpCost = Σ (growth × 段階コスト)  // value系閾値11, control系閾値17
itemExpCost    = generalSkill/styleSkill の level × カテゴリコスト

newTotal = additional + historyTotal
newSpent = (abilityExpCost + itemExpCost) - 170
newValue = (newTotal + 170) - (abilityExpCost + itemExpCost)
```

更新時 `{ calcExp: false }` で再帰防止。

## 同期 (フェーズ2-2 完了)

### 二フック構成

| フック | 方向 | 処理 |
|---|---|---|
| updateActor (cast) | cast → User | overflow 合計を User flag に書き込み |
| updateUser | User → cast | flag 変更を紐づく全 cast へ配布 + updateCastExp |

### 無限ループ防止

`{ syncing: true }` フラグ — 各フック先頭でガード。

### cast ↔ User 紐付け

- `cast.system.ownerUserId` — ownership 変更時に最初の非 GM User を記録 (2-1)
- `cast.system.playerId` — 旧 player Actor 参照 (2-5 廃止予定)

## 履歴 UI

| UI | 実装 |
|---|---|
| cast シート | TnxHistoryMixin + history-list.hbs |
| レコードシート | User flag 直接編集 (ApplicationV2) |
| player シート | TnxHistoryMixin (player 廃止で消滅予定) |

履歴変更時: cast シート → User flag へキー変換 (`system.history.X` → `flags.TNX.history.X`)

## 能力値の実効値（表示用）

実効値は `CastDataModel.prepareDerivedData`（フェーズ9）で `system.<key>.total` / `.totalControl` に一本化。シートは `_prepareContext()`（ApplicationV2）→ `_getAbilitiesData()` で集約済みの total を読む（`getData()` は ApplicationV1 の廃止メソッド）。

```
total        = growth + Σ(styleValue × level) + mod + outfitMod   // effectMod 項は v2 で全廃
totalControl = controlGrowth + Σ(styleControl × level) + controlMod + outfitControlMod
```

値バフ（AE）は適用パス `_applyEffectBuffs` が total へ直接適用し、最後に 0clamp する（`computeAttributeFinal`、`scripts/data/helpers.mjs`）。

## 既知の設計課題

| ID | 内容 | 担当 |
|---|---|---|
| KI-008 | 一括更新時の updateCastExp 競合リスク | **解消（2026-07-19・本体 `659a2df`）** |

KI-008 の是正: `updateCastExp` は同一アクターへの並行呼び出しを 1 本に合流する（実行中に来た呼び出しは「完了後にもう 1 回」へ畳む＝毎回の全量再計算なので最後の 1 回が最新を反映）。計算・書き込みロジック（`_recalcCastExp`）は不変＝EXP 同期機構の設計（権威は User flag・ドキュメント跨ぎの同期）はそのまま。ユーザー言明（2026-07-19）: 自動インポート（基本技能セットアップ）では経験点は消費されないため、当該タイミングでの値ずれは元々起こり得ない——合流は冗長書き込みの排除としての整理。詳細は [Resolved_Issues.md](../Issues/Resolved_Issues.md) の KI-008。

## 参照

- 実装: `scripts/actor/tnx-cast-sheet.mjs`, `scripts/module/exp-sync.mjs`
- 調査: [B4_Investigation_Detail.md](../Phases/B4_Investigation_Detail.md), [Phase_2_User_Migration.md#設計過程検討記録](../Phases/Phase_2_User_Migration.md#設計過程検討記録)
