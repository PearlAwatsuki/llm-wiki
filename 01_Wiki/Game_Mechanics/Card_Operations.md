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

システム設定 (`renderSettingsConfig`) で Cards ドキュメントを UUID 指定:

- 山札 (deck), 捨て札 (discard)
- 手札 (hand), 切り札 (trump)
- ニューロデッキ, シーンカード置き場

設定 UI は v13 DOM API 対応済 (ドロップダウン選択)。

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

`ready` フックで全 cast の `handMaxSize` を設定値 (+ ActiveEffect `handMaxSizeMod`) で更新。

`handMaxSizeMod` は DataModel 未定義 (KI-005) — フェーズ2 で User flag 設計と合わせて決着。

## 所有モデル (移行中)

| 現状 | フェーズ2 移行先 |
|---|---|
| player Actor の handPileId / trumpCardPileId | User flag |
| cast シートの手札表示 | 削除 (2-3) |
| HUD 手札取得 | `game.user` flag 経由 (2-3) |

手札作成: アクトシートの RL 手札作成を全ユーザー分に拡張 (GM 一括 + ownership 設定)。

## チャット

`ChatMessage.create()` で HTML 直接投稿。カスタム ChatMessage クラス・`renderChatMessage` フックなし。

テンプレート: `templates/chat/scene-card.hbs`

## 今後 (フェーズ7/8)

本ドキュメントは**物理的なカード移動**のみ。達成値計算・技能判定ロジックはフェーズ7 (設計) / フェーズ8 (実装) で追加。

usage テンプレートが判定のデータ基盤 → [Usage_System.md](Usage_System.md)
