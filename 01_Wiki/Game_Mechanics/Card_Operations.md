# カード操作

TNX の判定はすべて **Foundry Cards API** 経由。ダイスロール (`Roll`) は未使用。

## TnxActionHandler

`scripts/module/tnx-action-handler.mjs`

| メソッド | 操作 | 処理概要 |
|---|---|---|
| `checkFromDeck()` | 山札から判定 | 山札→捨て札へ1枚表向き移動 + チャット通知 |
| `drawCard()` | ドロー | 山札→手札へ1枚 |
| `playCard()` | プレイ | 手札→捨て札 |
| `useTrump()` | 切り札使用 | 切り札置き場→シーン/RL捨て場 + チャット |
| `drawNeuroCard()` | ニューロドロー | ニューロデッキ→シーン置き場 + チャット |

## カード置き場 (Piles)

**`TnxCardSetupApp`**（ゲーム設定メニュー、フェーズ5。`scripts/module/tnx-card-setup-app.mjs`）で Cards ドキュメントを設定する:

- 山札 (deck), 捨て札 (discard)
- 手札 (hand), 切り札 (trump) — **ユーザーごとに User flag で保持**
- ニューロデッキ, シーンカード置き場, RL 用切り札捨て場 (`gmTrumpDiscardId`)

`TnxActionHandler` の `getActiveDeck()` / `getActiveDiscardPile()` / `getActiveNeuroDeck()` / `getActiveScenePile()` 等が現在の置き場を解決する。

## カード生成

| 種類 | モジュール | 枚数 |
|---|---|---|
| 標準トランプ | `tnx-playing-cards.mjs` | 52×N + ジョーカー2 |
| ニューロ | `tnx-neuro-cards.mjs` | 31 (value: -18〜21) |
| アクセス | `tnx-access-cards.mjs` | 8 (シーン, 切り札, YES/NO 等) |

## フック連動

- `preCreateCard`: 切り札置き場上限 1枚
- `createCard` / `deleteCard`: `game.tnx.refreshSheets()` で UI 更新

## 手札上限

手札上限の権威は **User flag**（`handMaxSize`、既定 4）。実効上限は `resolveEffectiveHandMaxSize(user)`（`user-flag-schema.mjs`）が **①既定 ②User flag 明示 ③所有 cast の AE 合算（`cast.system.handMaxSizeMod`）** を 0clamp で解決する。

`handMaxSizeMod` は cast の AE 着地点として新設済み（KI-005 → KI-020 解消・フェーズ9-3）。User は DataModel/AE を持てないため所有 cast の AE から合流する。

## 所有モデル

手札・切り札の所有は **User flag**（`handPileId` / `trumpCardPileId`）に一本化済み（フェーズ2 完了・player Actor 廃止）。HUD は `game.user` の flag から手札置き場を取得する。配布・受け渡しは User 単位（`TnxCardSetupApp` / `dealTrumpFromNeuroDeck` / `selectAndPassToUser` 等）。

## チャット

`ChatMessage.create()` で HTML 直接投稿。カスタム ChatMessage クラス・`renderChatMessage` フックなし。

テンプレート: `templates/chat/scene-card.hbs`

## 判定ロジックとの関係

本ドキュメントは**物理的なカード移動**を扱う。達成値計算・技能判定ロジックはフェーズ7（設計）/8（実装）で実装済み（→ [Check_System.md](Check_System.md)）。usage テンプレートが判定のデータ基盤（→ [Usage_System.md](Usage_System.md)）。
