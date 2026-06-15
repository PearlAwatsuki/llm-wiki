# Phase 8: 判定システム 実装

## 目的

フェーズ7 で確定した設計に基づき、TNX の判定システムをコードとして実装する。
KI-002（usage edit ハンドラ不在）・KI-006（socket 未実装）を同時解決する。

## スコープ外

- AE ボーナス集計（ActiveEffect Phase 後）
- 購入判定（辞典ブラウザ後）
- 登場判定（フェーズ11）
- 神業システム（フェーズ9-10）

## 設計の前提

- [Judgment_Rules.md](../Game_Rules/Judgment_Rules.md) がルール正本
- [Judgment_System.md](../Game_Mechanics/Judgment_System.md) が実装設計の正本

---

## サブフェーズ

### 8-0: DataModel 変更

**`scripts/data/item/common/usage.mjs`**

```javascript
actions: ArrayField(SchemaField({
  type:        StringField,  // "check"|"controlCheck"|"attack"|"declaration"|"miracle"
  name:        StringField,
  description: StringField,
  skillRefs:   ArrayField(SchemaField({ itemId: StringField })),  // 新設
}))
```

- [x] `skillRefs` 追加・`type` 値域コメント更新
- [x] テスト更新（既存テストへの影響確認）

---

### 8-1: ソケット基盤（KI-006 解決）

**新規: `scripts/module/tnx-socket-handler.mjs`**

```javascript
// メッセージ種別
const SOCKET_MSG = {
  JUDGMENT_RESULT: "judgmentResult",   // PL → GM: 判定結果を送信
};

// 登録（tnx.mjs の ready フック内）
game.socket.on("system.tokyo-nova-axleration", TnxSocketHandler.onMessage);
```

- [x] ソケットハンドラクラス新設
- [x] `tnx.mjs` への登録（ready フック）
- [x] メッセージ種別: `judgmentResult`（PL → GM、ChatMessage 更新用）

---

### 8-2: 判定エンジン

**新規: `scripts/module/tnx-judgment-engine.mjs`**

核心的な計算ロジックを UI から分離して保持する。

```javascript
class TnxJudgmentEngine {
  /**
   * カード値を算出する
   * @param {Card} card
   * @param {boolean} fixedAt21  A を21固定として使うか
   * @returns {number|"FUMBLE"}
   */
  static getCardValue(card, { fixedAt21 = false, isFromDeck = false } = {}) {}

  /**
   * 技能判定の達成値を算出する
   * @param {number} cardValue
   * @param {string} suit  "spade"|"heart"|"diamond"|"club"
   * @param {ActorData} actorSystem
   * @param {number} bountyUsed
   * @returns {{ achievement, abilityVal, abilityKey }}
   */
  static calcSkillCheck(cardValue, suit, actorSystem, bountyUsed) {}

  /**
   * 制御判定の成否を判定する
   * @param {number} cardValue
   * @param {string} suit
   * @param {ActorData} actorSystem
   * @returns {{ success, cardValue, controlVal }}
   */
  static calcControlCheck(cardValue, suit, actorSystem) {}

  /**
   * 組み合わせ技能の使用可能スートを算出する（積集合）
   * @param {Item[]} skills
   * @returns {string[]}  使用可能スートの配列
   */
  static getComboSuits(skills) {}
}
```

- [x] カード値計算（2〜10・絵札=10・A=11/21固定・Joker 宣言値・切り札）
- [x] 技能判定達成値計算
- [x] 制御判定成否計算
- [x] 組み合わせスート積集合計算
- [x] スート → 能力値キー変換テーブル（`SUIT_TO_ABILITY`定数）

---

### 8-3: 判定ダイアログ

**新規: `scripts/module/tnx-judgment-dialog.mjs`**
**新規: `templates/dialog/judgment-dialog.hbs`**

プレイヤーがカードを選択・報酬点を入力して判定を実行するダイアログ。

```
┌────────────────────────────────────┐
│ 〈射撃〉+〈知覚〉  目標値: 12     │
│ 使用可能スート: ♠ ♥               │
│                                    │
│ 手札:  [♠7] [♥K] [♦A] [♣3]       │
│                                    │
│ 報酬点使用: [0]  (所持: 5)         │
│                                    │
│ [山札から判定]     [決定]           │
└────────────────────────────────────┘
```

- [x] 手札カード一覧表示（HUD で実装。有効スートを強調表示）
- [x] カード選択（HUD カードクリック。`TnxJudgmentFlow.executeFromHand`）
- [x] Joker 選択時: スート・値の宣言フォーム表示（`_promptJokerDeclaration`）
- [x] 切り札使用: HUD の切り札モードボタン → 手札から追加で1枚選択
- [x] A 選択時: 「11 として使う」/「達成値を21に固定する」ダイアログ（`_handleAceChoice`）
- [x] 山札から判定ボタン（絵札 → ファンブル自動判定。`executeFromDeck`）
- [x] 使用可能スート表示（ダイアログに記号バッジで表示。スート不一致は達成値 0 の失敗）
- [x] 報酬点入力（カード選択後に別ダイアログ。制御判定・スート不一致はスキップ）

**設計変更 (2026-06-15)**: 当初の「ダイアログ内カード選択」案から「HUD 中心フロー」案に変更。
ダイアログは判定コンテキスト（種別・スート・目標値）のみ表示。カード選択は HUD で行い、
クリック時に `TnxJudgmentFlow.executeFromHand` を呼び出す。
ダイアログを×ボタンで閉じると判定キャンセル。報酬点はカード選択後に別プロンプトで入力（ルール順序に準拠）。
達成値プレビューをカードホバー時にバッジ表示。

---

### 8-4: プレイヤー主導フロー

**`scripts/actor/tnx-cast-sheet.mjs`** に click ハンドラ追加

| トリガー | 起動する判定種別 | TN の出どころ |
|---|---|---|
| 技能行クリック | 技能判定 | infoItem.tn または稀にプレイヤー入力 |
| 能力値数値クリック | 能力判定 | プレイヤー入力（稀なケース） |
| 制御値数値クリック | 制御判定 | なし（カードのスートで決まる） |

**`scripts/journal/tnx-scenario-sheet.mjs`** に infoItem skill クリックハンドラ追加

- [x] キャストシート: 技能行クリック（一般・スタイル両方）→ `TnxJudgmentFlow.open`（スートは技能 Item の suits から算出）
- [x] キャストシート: 能力値クリック → `TnxJudgmentFlow.open`（abilityCheck モード。対応スート1択）
- [x] キャストシート: 制御値クリック → `TnxJudgmentFlow.open`（controlCheck モード。対応スート1択）
- [x] アクトシート: infoItem skill 行のハンマーボタン → `TnxJudgmentFlow.open`（TN=tn、技能名=name テキスト、validSuits=全スート）
- [x] 判定結果 → `templates/chat/judgment-result.hbs` でチャット投稿（8-3 で実装済み）

---

### 8-5: RL 要求フロー ✅ 完了 (2026-06-15)

**新規: `scripts/module/tnx-rl-request-app.mjs`**
**新規: `templates/app/rl-request-app.hbs`**
**新規: `templates/chat/judgment-request.hbs`**

> 設計は当初案から一部変更。rollMode 廃止（全公開のみ）、identificationKey による技能照合、「その他」自由入力、description フィールド追加。詳細は [Judgment_System.md](../Game_Mechanics/Judgment_System.md#rl-判定要求フローフェーズ-8-5) を参照。

#### RL 側（GM）

- [x] シーンコントロール（tokens グループ）に「判定要求」ボタン追加（GM のみ表示）
- [x] `TnxRlRequestApp` ダイアログ:
  - 対象プレイヤー（複数選択可、チェックボックス）
  - 判定種別ドロップダウン (skillCheck / abilityCheck / controlCheck)
  - 技能: 一般技能コンペンディウムの identificationKey から選択 or「その他」自由入力 + スート手動指定
  - 能力値: abilityCheck/controlCheck 時は理性/感情/生命/外界を選択
  - 目標値（number-input-spinner + targetValueHidden 非公開フラグ）
  - 説明テキストエリア（省略可）
  - rollMode 廃止（全公開のみ）
- [x] 送信 → ChatMessage 作成（`judgmentRequest` フラグ付き）

#### PL 側

- [x] `renderChatMessage` フック: 要求カードの「判定する」ボタンをバインド（自分のキャラ or GM のみ表示）
- [x] ボタンクリック → `TnxRlRequestApp.onDoJudgment()` → identificationKey でキャラ上技能を解決 → `TnxJudgmentFlow.open()`
- [x] 判定完了 → socket emit（`judgmentResult`）→ GM が ChatMessage.update() で results フラグ更新 → renderChatMessage 再発火で結果表示

#### ChatMessage フラグ構造（実装後確定版）

→ [Judgment_System.md#chatmessage-フラグ構造](../Game_Mechanics/Judgment_System.md) を参照。

---

### 8-6: usage edit 接続（KI-002 解決）✅ 完了 (2026-06-15)

**`templates/parts/usage-list.hbs`** の edit ボタンに `_onUsageEdit` ハンドラを接続。
**`UsageCreationDialog`** を拡張して `skillRefs` の編集 UI を追加。

- [x] `_onUsageEdit` ハンドラ追加（対象 Item・action index を渡す）
- [x] `UsageCreationDialog.editAction()`: type セレクト / name / description / skillRefs multi-select
  - skillRefs: アクターが所持する generalSkill / styleSkill アイテムを列挙（自分自身は除外）
  - 新規テンプレート `templates/dialog/usage-edit-dialog.hbs` を追加
  - スタンドアロンアイテム（actor なし）の場合は skillRefs 欄を非表示

---

### 8-7: テスト・仕上げ

- [ ] `TnxJudgmentEngine` の全計算パターンに単体テストを追加
- [ ] wiki: メカニクス監査エントリ・設計レビューエントリ追加
- [ ] wiki: Judgment_Rules.md / Judgment_System.md の最終整合確認
- [ ] CHANGELOG 更新・v0.8.0 バージョンアップ

---

## 完了条件

- 情報収集判定・RL 要求判定が実機で動作する
- 技能組み合わせ設定が usage タブから行える（KI-002 解決）
- socket 経由で RL 要求カードが判定結果で更新される（KI-006 解決）
- 全既存テスト + 新規判定エンジンテストがグリーン

## 参照

- [Roadmap.md](../../02_System/Roadmap.md)
- [Judgment_Rules.md](../Game_Rules/Judgment_Rules.md)
- [Judgment_System.md](../Game_Mechanics/Judgment_System.md)
- [Phase_7_Tasks_Detail.md](Phase_7_Tasks_Detail.md)
