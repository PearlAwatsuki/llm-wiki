# 判定システム

TNX の判定ルールの正本。フェーズ7（設計）で言語化・確定。

## 判定の種類

### 能力判定

1. RL が能力値・目標値を決定
2. 指定された能力値に対応するスートのカードを出す
3. 達成値 = カード値 + 指定能力値（実効値）
4. 目標値と比較。場合によっては差分値（達成値 − 目標値）を算出

### 技能判定

1. RL が技能・目標値を決定
2. 指定技能で使用できるスートのうち好きなスートのカードを出す
3. 判定成立（カードを出した時点）
4. 報酬点の使用を決定（使う場合：1点 = 達成値 +1、上限は所持点数のみ）
5. 達成値 = カード値 + 出したスート対応の能力値実効値 + 使用報酬点
6. 目標値と比較。場合によっては差分値を算出

**重要**: 技能の「初期スート」に関係なく、出したカードのスートに対応する能力値が使われる。
例: ♠♥の両方が使える技能で♥を出した場合 → 生命値で計算。

技能レベルは達成値に**加算しない**。レベルが寄与するのは:
- 使用可能なスートの数（レベルが高いほど選択肢が増える）
- 一部スタイル技能・アウトフィットの発動条件・装備条件

## スート ↔ 能力値対応表

| スート | 記号 | 能力値キー | 表示名 |
|---|---|---|---|
| スペード | ♠ | `reason` | 理性 |
| クラブ | ♣ | `passion` | 感情 |
| ハート | ♥ | `life` | 生命 |
| ダイヤ | ♦ | `mundane` | 外界 |

## カード値の対応

| カード | 判定値 | 備考 |
|---|---|---|
| 2〜10 | 数字通り | |
| J / Q / K（絵札） | 10 | 山札判定時のみファンブル（→下記） |
| A（エース） | 11 **または** 達成値21固定 | 判定時にプレイヤーが選択 |
| Joker | スート・値ともに自由宣言 | ワイルドカード |
| 切り札（Trump） | — | 消費することで手札1枚を Joker として扱える。消費後はシーンカードへ |

### Aの「達成値21固定」

- 選択した場合、達成値はちょうど21で確定する
- 能力値・報酬点・修正値は一切加算されない（完全固定）
- 自動成功ではない（目標値が21を超えることもある）

### Joker の使い方

- スートと数値の両方を自由に宣言して使用
- 例:「♠9 として使う」「♥A として使う」など

### 切り札（Trump）の使い方

1. 切り札を消費することを宣言する
2. 手札から好きな1枚を選び、Joker として扱う（スート・値を自由宣言）
3. 消費した切り札はそのシーンのシーンカードになる（シーンの雰囲気を塗り替える）

## 自動失敗の条件

| 条件 | 判定 | 備考 |
|---|---|---|
| 使用できないスートのカードを出した | 自動失敗 | プレイヤーの選択は妨げない（UI で拒否しない） |
| 山札判定で絵札（J/Q/K）が出た | ファンブル（致命的失敗） | RL によりペナルティが追加される場合もあるが、ルール上は失敗のみ |
| 手札判定で絵札 | ファンブルにならない（通常の判定値10） | |

## 能力値の実効値

判定で加算する能力値は、キャストシート閲覧モードに表示される全修正込みの実効値：

```
totalValue = growth + Σ(style.value × styleLevel) + mod + effectMod + abilityOutfitMod
```

コード参照: `scripts/actor/tnx-cast-sheet.mjs` の `_prepareAbilities()`（`totalValue` フィールド）

## UIフロー

### HUD中心フロー（確定 2026-06-15、フェーズ8-3で変更）

**カード選択はダイアログではなく HUD で行う。** ダイアログは判定コンテキスト（種別・スート・目標値）の表示のみ。

#### プレイヤー主導（自発判定）

1. キャストシートの技能行/能力値/制御値クリック、またはアクトシートのハンマーボタン
2. `TnxCheckFlow.open(context)` → `TnxCheckDialog` 起動（コンテキスト表示のみ）+ HUD がペンディング状態に
3. プレイヤーが HUD 上の手札カードをクリック（有効スートのカードを強調表示）
   - カードホバー時: 達成値プレビューをバッジ表示（**報酬点含まない暫定値**）
   - スート不一致カードをクリックした場合: 達成値0の失敗として処理（プレイを拒否しない）
4. A クリック: 「11として使う / 達成値21固定」ダイアログ
5. Joker クリック: スート・値宣言ダイアログ（`_promptJokerDeclaration`）
6. 切り札モード中にカードクリック: そのカードを Joker として使い切り札消費
7. カードがプレイされる（捨て札へ移動）
8. 報酬点入力ダイアログ（`_promptBountyUsage`）— スート不一致・制御判定・`bountyAvailable=0` の場合はスキップ
9. 達成値計算 → チャットに結果カード投稿
10. ダイアログを閉じると判定キャンセル（`_onClose` → `TnxCheckFlow.cancel()`）

#### RL 主導（判定要求、フェーズ8-5で実装）

1. 左サイドバー（GM のみ表示）のボタン → `TnxRlRequestApp` 起動
2. 対象アクター・判定種別（能力/技能）・能力値 or 技能・目標値を設定
3. 「要求を送信」→ チャットに判定要求カードを投稿（全員に見える）
4. 対象アクターの**所有者権限を持つユーザー**がカードの「判定する」ボタンをクリック（2026-07-19: ユーザー選択方式を廃止しアクター登録方式へ）
5. 上記プレイヤー主導フロー(2〜9)が実行される
6. 達成値計算 → 要求カードを更新（結果を反映） — socket 送信経由

チャットカードの更新（ステップ6）にはソケット通信が必要（KI-006 解決込みでフェーズ8-5 実装）。

## チャット表示内容（判定結果カード）（確定 2026-06-15）

- 判定者名
- 技能名: 組み合わせ技能を「+」でつなぐ（例: 〈射撃〉+〈知覚〉）。能力値判定は能力値名
- 使用カード（スート・値）
- 通常判定: 達成値 / 目標値 / 差分値 / 成否マーク
- 制御判定: 「達成値 ≤ 制御値 ▶ 成功」形式（成否の表記に ▶ 使用）
- 報酬点使用量（0 の場合は非表示）
- 結果: 成功 / 失敗 / ファンブル

## RL 判定要求の対象

複数対象への同時送信が基本（Monk's Token Bar 方式）。全員とは限らず選択制。
対象は**アクター（キャスト）を登録**する（2026-07-19 ユーザー指示: ユーザー選択方式を廃止。
「判定する」はそのアクターの所有者権限を持つユーザーが押せる＝接続状況・キャラクター割り当てに
依存しない）。制御判定要求（controlNegate）も同じ targets 形式（actorId/actorName のみ）に統一。

## チャット結果の公開範囲

- 秘匿ルールは存在しない
- ただし RL が個人を対象に行った判定の結果は秘匿される傾向にある
- **捨て札は全員が平等に見ることができる**（カードの内容は公開情報）
- → 実装方針: デフォルトは全員公開。RL 主導要求ダイアログに roll mode（公開 / GM+本人のみ）の選択肢を設ける

## 情報収集項目（アクトシート連動）

アクトシート（JournalEntry の flags）で情報収集項目を管理する。

> **実装状況（2026-07-20 コードとの突き合わせで訂正）**: 実装されているのは **RL 側の項目編集 UI（データ構造の CRUD）と、技能行のボタンから TN 付きで判定ダイアログを開く配線**までである。**「情報収集判定」というメカニクスは未実装**。以下の「判定トリガーの設計方針」は**方針であって実装ではない**。従前この節と下表が「実装済み／✅」とだけ書かれていたため、以降の作業で実装済みメカニクスとして参照され続けていた。

### データ構造（実装済み）

```javascript
// JournalEntry.getFlag("tokyo-nova-axleration", "infoItems")
infoItems = [
  {
    id: "randomId",
    title: "情報名",          // 例: "黒幕の素性"
    isPublic: boolean,        // 公開中かどうか
    contents: [
      {
        id: "contentId",
        isDisclosed: boolean, // 調査済み（入手済み）かどうか
        skills: [
          { id: "skillId", name: "技能名（テキスト）", tn: 12 }
          // tn = 目標値（Target Number）
          // name は自由記述テキスト、Itemへの参照ではない
        ],
        text: "情報の内容テキスト（公開時に表示）"
      }
    ]
  }
]
```

### 判定トリガー（設計方針・**未実装**）

- プレイヤーが HUD または公開済み情報カード上のスキル行をクリック
- `name`（技能名テキスト）と `tn`（目標値）が判定ダイアログに自動入力される
- どのキャストの技能 Item と対応するかは自動解決しない（テキスト表示のみ）
- プレイヤーはダイアログ内でカード選択と報酬点を指定して判定を実行

#### 現状の実装（`_onStartInfoSkillCheck`・`scripts/journal/tnx-scenario-sheet.mjs`）

アクトシート「情報」タブの技能行の ◆ ボタンから `TnxCheckFlow.open({ type: "skillCheck", actorId: game.user.character.id, skillIds: [], skillLabel: 技能名テキスト, targetValue: tn, validSuits: 全スート })` を直接呼ぶ。

#### 未実装の項目（2026-07-20 に列挙）

- **技能が解決されない**: `skillIds` が空のため `TnxCheckFlow._computeCheckBonus` の分岐で criteria が `ability` に落ちる（技能レベル・技能バフ・組み合わせのいずれも乗らない）。実質は**能力値判定に技能名の文字列ラベルを貼ったもの**。[Check_Rules](../Game_Rules/Check_Rules.md)「技能指定判定の代用」も効かない。
- **PL 側の起動導線が無い**: ボタンは RL のアクトシート内にのみ存在し、`game.user.character` で走る（RL が押せば RL のキャラクターで判定される）。上記「設計方針」の HUD／公開情報カードからの起動は未着手。
- **結果が情報項目に還らない**: 成功しても `isDisclosed` は立たず情報テキストも開示されない（調査済みは RL の手動トグルのみ）。判定 → 開示のループが存在しない。
- **ルール上の枠組みが無い**: 舞台裏・基本1シーン1回（→ [Scenario_Progress](../Game_Rules/Scenario_Progress.md)）は未実装。前提となるシーン管理がフェーズ14以降。

なお**報酬点の無条件付与は仕様どおり**（`usesBounty` フラグに関わらず消費可能・2026-07-16 ユーザー裁定 → [Bounty_Points](../Game_Rules/Bounty_Points.md)）。

#### 着手時期（2026-07-20 ユーザー裁定）

**セッション進行の土台（シーン進行管理＝フェーズ14／時間単位管理＝フェーズ15）が揃ってから着手する**（「実セッションに大きく関わる部分なので、実セッションを行う土台がそろってから」）。[Scenario_Progress](../Game_Rules/Scenario_Progress.md) の申し送り「舞台裏判定 → フェーズ16-17」に沿う。**技能解決や PL 導線だけを先に是正することはしない**——舞台裏・基本1シーン1回というシーン単位の制約と不可分で、土台なしに起動経路だけ整えても作り直しになるため。

## フェーズ8 スコープ（判定起動の対象）

| 種別 | スコープ | 理由 |
|---|---|---|
| 情報収集判定（技能） | △ フェーズ8（**ダイアログ起動の配線のみ**） | TN がアクトシートデータにあるため配線した。**メカニクスとしては未完**＝上の「未実装の項目」を参照（この行の旧記述「✅」は 2026-07-20 に訂正） |
| RL 要求判定（能力・技能） | ✅ フェーズ8 | TN は要求カードに含む。socket 実装と同時 |
| 購入判定（〈信用〉技能） | ❌ 辞典ブラウザ後 | 辞典ブラウザ（D&D 辞典ブラウザ相当）から起動。ブラウザ完成後に自然接続 |
| 登場判定 | ❌ フェーズ18 | アクトシートのシーン設定に TN・使用技能フィールドが必要。サブシーンシステムに依存（旧記述「フェーズ11 候補」は再編前の旧番号・2026-07-03 訂正） |

PLが目標値を手入力するフローは採用しない（Foundry VTT のデータ駆動設計に反する）。

## 辞典ブラウザ（将来機能）

将来フェーズで実装予定。D&D 5e の Compendium Browser 相当機能：
- カテゴリ別タブ切り替えで全コンペンディアムドキュメントを閲覧
- ドロップエリアクリックでも起動
- アイテム選択 → 決定でドロップエリアに自動インポート
- **購入判定との接続**: アウトフィットタブで目的アイテムを選択 → 購入ボタンで `buy.value` を TN として〈信用〉技能判定を起動

## 能力判定のプレイヤー側起動経路（確定 2026-06-14）

- キャストシートの能力値（理性・感情・生命・外界）の数値をクリック → 判定ダイアログ起動
- 能力値判定はRL主導が基本で自発はほぼないが、トリガーとして一応残す
- TN はこの経路では稀なケースとして入力を許容（主流フローはRL要求カード経由）

## ボーナス修正の供給元（確定 2026-06-15）

| 修正の種類 | 供給元 |
|---|---|
| 技能判定ボーナス（大半） | ActiveEffect（アウトフィット提供含む） |
| 登場判定ペナルティのみ | `outfitMod`（AE 経由でなく直接集計） |

組み合わせ判定のボーナス集計は ActiveEffect Phase 完了後に実装。
Phase 8 では基本判定メカニクス（カード選択・達成値計算・チャット投稿）を先に整備する。
→ **フェーズ9-3 v2 の判定バフ（`check.*`/`controlCheck.*` 着地）で実装済み**（[Active_Effects.md](Active_Effects.md)）。なお組み合わせの**解決**（ベース/コンボの確定・個数制限）はフェーズ10-2 で用途設定時に確定する方式になった（判定起動時は保存済みの `baseSkillRef`/`skillRefs` を読むのみ・[Usage_System.md](Usage_System.md)）。

## usage `actions[]` DataModel（フェーズ8 実装確定 2026-06-17）

→ 詳細は [Usage_System.md](Usage_System.md) を参照。

### 確定フィールド一覧

```javascript
// usage.mjs の actions[] SchemaField（確定実装）
{
  _id:          StringField,          // randomID() で自動付与（インデックス参照を廃止）
  type:         StringField,          // "check"|"attack"|"declaration"|"damageBoost"|"damageReduce"|"modification"
  name:         StringField,
  description:  StringField,

  timing: SchemaField({
      value:       StringField,       // "blank"=自由 | "action" | "process" | "reaction" | "other" | ...
      actionName:  StringField,
      processName: StringField,
      timingOther: StringField,
  }),
  target: StringField,                // "blank"=自由 | "self" | "single" | ...

  effects: ArrayField(SchemaField({ effectId: StringField })),

  baseSkillRef: SchemaField({ itemId: StringField }),  // ベース技能（check/attack）
  skillRefs:    ArrayField(SchemaField({ itemId: StringField })),  // コンボ技能

  // attack 固有
  weaponRef: SchemaField({ itemId: StringField }),
  damageType: StringField,            // "S" | "P" | "I"

  // damageBoost / damageReduce 固有
  formula: StringField,
  damageCategory: StringField,        // "physical" | "mental"

  // modification 固有
  modifiableParams: ArrayField(StringField),
}
```

### 設計根拠

- `controlCheck` は用途タイプではなく独立した判定種別（timing で区別）
- `baseSkillRef` 導入: 「判定の主体は用途である」。ベース技能を用途が明示的に保持することで用途起動の連鎖バグを構造的に防ぐ
- `_id` 導入: インデックスによる参照を廃止し、配列操作でエントリを追加・削除しても ID が変わらない
- `timing.value = "blank"` / `target = "blank"` は「自由（指定なし）」の意味
- `migrateData` で既存データに `_id` と `baseSkillRef` を自動付与

## 実装詳細（フェーズ8-3/8-4 完了時点）

### モジュール構成

| ファイル | 役割 |
|---|---|
| `scripts/module/tnx-check-engine.mjs` | 計算ロジック（UI なし）。`getCardCheckValue`, `calcSkillCheck`, `calcControlCheck`, `getComboSuits`, `getAbilityBySuit`, `ALL_SUITS`, `SUIT_TO_ABILITY` をエクスポート |
| `scripts/module/tnx-check-flow.mjs` | フロー管理シングルトン（`TnxCheckFlow` static class）|
| `scripts/module/tnx-check-dialog.mjs` | `TnxCheckDialog`（ApplicationV2）— コンテキスト表示のみ |
| `templates/dialog/check-dialog.hbs` | ダイアログ本体（スート表示・山札から判定ボタン）|
| `templates/chat/check-result.hbs` | 判定結果チャットカード |

### `CheckContext` typedef

```javascript
/**
 * @typedef {object} CheckContext
 * @property {"skillCheck"|"controlCheck"|"abilityCheck"} type
 * @property {string}        actorId          - 判定を行うキャスト Actor ID
 * @property {string[]}      skillIds         - 使用技能 Item ID（能力値判定は空 []）
 * @property {string}        skillLabel       - 表示用技能名 "〈電脳〉+〈ハッキング〉" 等
 * @property {string[]}      validSuits       - 使用可能スート ["spade","heart"] 等
 * @property {number|null}   targetValue      - 目標値（未決定は null）
 * @property {number}        bountyAvailable  - 使用可能報酬点（0 のとき報酬点ダイアログをスキップ）
 * @property {string|null}   requestMessageId - RL 要求 ChatMessage ID（自発判定は null）
 */
```

### `TnxCheckFlow` 公開 API

```javascript
// 判定状態の確認
TnxCheckFlow.isPending     // boolean: 判定進行中か
TnxCheckFlow.context       // CheckContext | null
TnxCheckFlow.trumpMode     // boolean: 切り札モード中か

// 起動・キャンセル
await TnxCheckFlow.open(context)   // 判定開始。ダイアログ + HUD を更新
TnxCheckFlow.cancel()             // 判定中止。ダイアログ閉じ、HUD リフレッシュ
TnxCheckFlow.toggleTrumpMode()    // 切り札モード切替

// 実行（HUD から呼ぶ）
await TnxCheckFlow.executeFromHand(cardId)  // 手札カードをクリック
await TnxCheckFlow.executeFromDeck()        // 山札から判定

// 内部注入（tnx.mjs の init で設定）
TnxCheckFlow.dialogClass = TnxCheckDialog
```

### 判定起動のエントリーポイント一覧

| 起動元 | 呼出し方 | 引数の注意点 |
|---|---|---|
| キャストシート 技能行クリック | `_onStartSkillCheck` → `open()` | 用途（check タイプ）を解決し、`baseSkillRef` からベース技能を決定。`skillRefs` + 親技能自動追加でコンボを構成。`bountyAvailable` は **ベース技能** の `usesBounty` で判定 |
| キャストシート 能力値クリック | `_onStartAbilityCheck` → `open()` | `abilityCheck` 型。`validSuits` は1スートのみ |
| キャストシート 制御値クリック | `_onStartControlCheck` → `open()` | `controlCheck` 型。`bountyAvailable: 0` 固定 |
| アクトシート ハンマーボタン | `_onStartInfoSkillCheck` → `open()` | `game.user.character` をアクターとして使用。`validSuits: ALL_SUITS`（全スート） |
| RL 要求チャット「判定する」 | `TnxRlRequestApp.onDoCheck()` → `open()` | identificationKey でキャラ上の技能を解決。技能未所持なら早期 return |

### `_onStartSkillCheck` のコンボ解決フロー（フェーズ8 改訂）

1. 技能行クリック → 技能アイテムの `check` タイプ用途を取得（複数なら選択ダイアログ）
2. `selectedUsage.baseSkillRef.itemId` からベース技能を解決（空の場合は親技能 ID にフォールバック）
3. `selectedUsage.skillRefs` からコンボ技能を収集
4. 用途所持技能（親）がベース技能と異なる場合はコンボに自動追加（非アクション技能からの起動）
5. 全技能スートの積集合で `validSuits` を計算 → `TnxCheckFlow.open()` へ渡す

---

## RL 判定要求フロー（フェーズ 8-5）

### 概要

GM がシーンコントロールの「判定要求」ボタンからダイアログを開き、対象 PL に判定を要求する機能。

**ファイル構成:**

| ファイル | 役割 |
|---|---|
| `scripts/module/tnx-rl-request-app.mjs` | `TnxRlRequestApp` — ダイアログ本体・送信処理・判定実行ハンドラ |
| `templates/app/rl-request-app.hbs` | 判定要求ダイアログ UI |
| `templates/chat/check-request.hbs` | 判定要求チャットカード（静的 HTML）|
| `scripts/tnx.mjs` | `getSceneControlButtons` フック・`renderChatMessage` フック |

### フロー詳細

```
1. GM がシーンコントロール → トークングループ → 「判定要求」ボタンをクリック
   → TnxRlRequestApp.render(true)

2. ダイアログで入力:
   - 判定種別: skillCheck / abilityCheck / controlCheck
   - 技能: 一般技能コンペンディウムの identificationKey から選択、または「その他」(自由入力 + スート手動選択)。
     プルダウンは**分類グループ(optgroup)×正規ソート順**＝シートの技能リストと同じ並び
     (無条件取得技能→製作→芸術→操縦→社会→コネ・2026-07-19 ユーザー指示)
   - 能力値: abilityCheck/controlCheck 時は理性/感情/生命/外界を選択
   - 目標値: number-input-spinner + 非公開フラグ(targetValueHidden)
   - 説明文: textarea (省略可)
   - 対象アクター: ワールドのキャストのチェックボックス(2026-07-19 ユーザー指示で
     「接続中ユーザーの選択」から変更。接続状況・キャラクター割り当てに依存しない)

3. 送信 → TnxRlRequestApp._onSubmit():
   - skillLabel, validSuits, targetValue, targets を組み立てる
   - ChatMessage.create({ content, flags.checkRequest: {...} })

4. 全クライアントが ChatMessage を受信 → renderChatMessage フック:
   - targetValueHidden かつ非 GM → 目標値を「（非公開）」表示
   - 対象アクターの**所有者権限を持つユーザー**(isOwner=GM 含む) → 「判定する」ボタンを注入
   - それ以外 → 「待機中…」を注入
   - results[actorId] が存在する場合 → 結果を注入（ボタンを置き換え）

5. PL が「判定する」クリック → TnxRlRequestApp.onDoCheck(flagData, actorId, messageId):
   - skillCheck + identificationKey: 応答ダイアログ(_promptSkillUse)は二段階(2026-07-19 ユーザー指示)
     - 第1段: 「〈指定技能〉で判定」/「代用判定」/「キャンセル」
     - 第2段(「〈指定技能〉で判定」時): **用途名のプルダウン**(_promptUsageChoice)。中身は
       指定技能自身の用途+コンボ候補(KI-025=指定技能を参加技能に含む他アイテムの用途・
       enumerateRequestComboCandidates・識別キー照合)の統合(buildRequestUsageChoices)で、
       **どちらも「判定」タイプ限定**(2026-07-19 ユーザー裁定。判定タイプ内でも固定値・NPC取得・
       バフ宣言フラグは要求カードに結果を返せないため除外)。**1件なら第2段を出さず自動起動**・
       0件は警告して中止。選択で _activateItemCheck へ用途 ID 直接指定
       (消費・判定ボーナス・適用効果は共通前段で解決)
     - 代用判定(2026-07-09): 別技能+手動修正(可否・ペナルティの裁定は卓)。未所持時はこれのみ提示
   - abilityCheck: bountyAvailable = actorBounty（全技能で報酬点使用可）
   - controlCheck: bountyAvailable = 0
   - skillCheck + その他: bountyAvailable = 0, validSuits = flagSuits → TnxCheckFlow.open を直接開く

6. 判定完了 → TnxSocketHandler → message.update(flags.results) → renderChatMessage 再発火 → 結果表示
```

### ChatMessage フラグ構造

```javascript
// flags["tokyo-nova-axleration"].checkRequest
{
    checkType:      "skillCheck" | "abilityCheck" | "controlCheck",
    identificationKey: string | null,    // 技能識別キー（その他・能力値判定時は null）
    skillLabel:        string,            // 表示用名称
    validSuits:        string[],          // skillCheck(識別キーあり) は []（PL 側で解決）
    targetValue:       number | null,
    targetValueHidden: boolean,
    description:       string,
    targets: [{ actorId, actorName }],  // 2026-07-19: userId/userName を撤去(ボタンは所有者権限で判定)
    results: {
        [actorId]: {
            achievement: number | null,
            success:     boolean | null,  // null = 目標値なし
            fumble:      boolean,
            cardValue:   number,
            controlVal:  number | null,   // controlCheck のみ
        }
    },
    status: "pending" | "closed",
}
```

### 設計上の注意点

- `identificationKey` が空の場合（その他）: PL の技能アイテムを検索しない。RL が指定したスートを使用。
- `validSuits: []`（識別キーあり）の場合: `onDoCheck` 内で `getComboSuits()` により PL 自身の技能から解決する。
- `renderChatMessage` は ChatMessage の更新ごとに再発火するため、results が追記された時点で自動的にボタン→結果に切り替わる。
- 目標値が `null` の場合はチャットカードに「—」表示。成否判定はスキップ（達成値のみ表示）。

### number-input-spinner ダイアログ実装パターン

数値入力を含むダイアログは必ず `number-input-spinner` を使う。`DialogV2.prompt` ではなく `DialogV2.wait` を使い `actions` で +/- を制御する:

```javascript
const spinnerContent = await foundry.applications.handlebars.renderTemplate(
    "systems/tokyo-nova-axleration/templates/dialog/amount-input-dialog.hbs",
    { label: "ラベル", initialValue: 0, min: 0, max: MAX },
);
const result = await foundry.applications.api.DialogV2.wait({
    window:   { title: "タイトル" },
    classes:  ["tokyo-nova", "tnx-amount-dialog"],
    position: { width: 480 },
    content:  `<p>説明文</p>${spinnerContent}`,
    actions: {
        decrement: (_event, target) => target.closest(".number-input-spinner")?.querySelector("input[type='number']")?.stepDown(),
        increment: (_event, target) => target.closest(".number-input-spinner")?.querySelector("input[type='number']")?.stepUp(),
    },
    buttons: [
        {
            action: "ok", icon: "fas fa-check", label: "確定", default: true,
            callback: (_event, _button, dialog) => parseInt(dialog.element.querySelector("[name=amount]")?.value) || 0,
        },
        { action: "cancel", icon: "fas fa-times", label: "キャンセル", callback: () => 0 },
    ],
    close: () => 0,
});
```

`amount-input-dialog.hbs` の input は `name="amount"` 固定。`min: 0` を指定すれば `stepDown()` が自動でクランプする。

### HUD 達成値プレビュー

HUD カードホバー時に `card.preview` バッジを表示（`templates/hud/hud.hbs`、`.card-preview-value` CSS）。

- **スート一致**: `getCardCheckValue(card.value) + getAbilityBySuit(suit, abilitiesCtx)` を表示
- **スート不一致**: `"0"` を赤背景で表示
- **Joker**: `"?"` を表示（宣言前のため不明）
- **A**: `null`（ダイアログ選択後に確定するため表示なし）
- **報酬点は含まない**（カード選択後に入力するため）。プレビューは暫定値であることを前提とした設計。

## 参照

- [Card_Operations.md](Card_Operations.md) — 物理的なカード移動処理
- [Usage_System.md](Usage_System.md) — usage テンプレート・KI-002
- [Resolved_Issues.md](../Issues/Resolved_Issues.md) — KI-002（用途編集ハンドラ）/ KI-006（ソケット）はともに解消済み
- [Phase_7_Tasks_Detail.md](../Phases/Phase_7_Tasks_Detail.md)
- [Check_Rules.md](../Game_Rules/Check_Rules.md) — ゲームルール正本
- [Bounty_Points.md](../Game_Rules/Bounty_Points.md) — 報酬点・usesBounty フィールド詳細
