# 報酬点 (Bounty Points)

## 概要

報酬点はキャストの財産を表すポイント。現金・資産の抽象的な表現。

## 入手方法

- **プレアクト自動付与**: 自分の外界点と同値の報酬点が自動付与される。
- **前金（オープニング）**: RL から依頼の対価として付与されることが多い。善意でアクトに関わるハンドアウトのキャストなど、前金を受け取らないケースもある。

## 消費方法

一部の技能判定と同時に消費することで、その判定の達成値を底上げできる。
（具体的な達成値ボーナス量はルールブック参照）

## リセットタイミング

ルール上はポストアクトで一度失われる。次回プレアクトで再び外界点分が付与される。
実態としてキャストは常に外界点分の報酬点を持っているように見えるが、ルール上は毎アクトリセット＋再付与。

## 実装方針

### CastDataModel フィールド（`scripts/data/actor/cast.mjs` L54-55）

```javascript
bounty:     new fields.NumberField({ initial: 0, integer: true }),  // アクト中の増減分
bountyBase: new fields.NumberField({ initial: 0, integer: true }),  // 外界点由来の基礎点
```

有効報酬点 = `bountyBase + bounty`。両フィールドとも初期値は 0。

- **`bountyBase`** はプレアクト時にキャストの外界点実効値で書き込む（`_computeMundaneTotalValue`）
- **`bounty`** はアクト中の増減（前金・消費）を反映
- キャストシートの `effectiveBounty` コンテキスト変数でまとめて計算: `(bountyBase ?? 0) + (bounty ?? 0)`
- **表示位置**: キャストシートサイドバー「パーソナルデータ」セクション末尾
- **ボタン `bounty-adjust-btn`**: キャストシートで ±1 調整。`system.bounty` のみ更新（bountyBase は非変更）

### 判定フローとの連携

`CheckContext.bountyAvailable` に使用可能報酬点を入れて `TnxCheckFlow` に渡す。

| 判定種別 | `bountyAvailable` の設定 |
|---|---|
| 技能判定 (`skillCheck`) | `usesBounty === true` なら `bountyBase + bounty`、そうでなければ `0` |
| 能力判定 (`abilityCheck`) | `bountyBase + bounty`（能力判定への報酬点適用は現状許容） |
| 制御判定 (`controlCheck`) | 常に `0`（制御判定に報酬点は使えない） |
| スート不一致 | 常に `0`（`_execute` 内でスキップ） |

`TnxCheckFlow._execute` 内でカードをプレイした後、`bountyAvailable > 0` の場合のみ `_promptBountyUsage` を呼ぶ。

### 報酬点入力ダイアログ（`_promptBountyUsage`）

`DialogV2.wait` + `amount-input-dialog.hbs` テンプレートで実装（`AmountInputDialog` と同パターン）。
`actions: { decrement, increment }` で `stepDown()` / `stepUp()` を呼ぶ。
**キャンセル・閉じた場合は 0 を返す**（判定続行。報酬点なしで達成値計算）。

## 技能との連携

### `usesBounty: BooleanField` on generalSkill

```javascript
// scripts/data/item/general-skill.mjs L42
usesBounty: new fields.BooleanField({ initial: false }),
```

- **`initial: false`** — デフォルトは「報酬点使用不可」。ルールブックで可能と明記されている技能のみ `true` に設定する。
- スタイル技能（`type === "style"`）には `usesBounty` フィールドが存在しない。`item.system.usesBounty` は `undefined` → falsy → 報酬点 0 として扱う。
- キャストシートの `_onStartSkillCheck` でチェック:
  ```javascript
  bountyAvailable: item.system.usesBounty === true ? actorBounty : 0,
  ```

## 関連

- [[Terminology]] — ゲーム用語集（外界点 = `system.mundane`）
