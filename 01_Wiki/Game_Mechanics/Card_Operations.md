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
| `passCardToUser()` | 受け渡し(D&D) | 手札→指定ユーザーの手札へ1枚（参加者パネルの行へのドロップから起動） |

## HUD の D&D 移動（2026-08-22 拡充）

HUD のカードは別エリアへのドラッグ&ドロップで移動できる。種類の異なる置き場へは対応する処理が無く移動しない。

| ドラッグ元 | ドロップ先 | 処理 |
|---|---|---|
| 山札 | 手札 | `drawCard()` |
| 山札 | 捨て札 | `flipFromDeck()` |
| 手札のカード | 捨て札 | `playCard()` |
| 手札のカード | 参加者パネルの行 | `passCardToUser()`（そのユーザーへ渡す） |
| 捨て札（一番上） | 山札 | `returnTopDiscardToDeck()`（**RL のみ**＝右クリックメニューと同権限） |
| 捨て札（一番上） | 手札 | `takeFromDiscard()` |
| ニューロデッキ | シーン置き場 | `drawNeuroCard()` |
| 切り札 | シーン置き場 | `useTrump()`（PL のみ） |

手札→山札の D&D は対応しない（対応する既存操作がどこにも無く、ゲーム操作の新設＝裁定を要するため見送り。必要になったら裁定を仰ぐ）。

## 手札の受け渡し（2026-08-22 一本化）

- 導線は2つ: 手札右クリック**「指定枚数を渡す」**（カード選択→相手選択）と、**参加者パネルの行へのカード D&D**（1枚・相手固定）。旧「手札を渡す」メニュー（`passSingleCard`）と行クリック起動（`selectAndPassToUser`）は廃止。
- 手札の ownership は**本人+GM のみ OWNER**（`TnxCardSetupApp._makeOwnership`）のため、PL は他ユーザーの手札にカードを作成できない。相手の手札を所有していない場合は **`passHandCard` ソケットで activeGM が移動を代行**する（要求者の手札=移動元であることを検証）。実体は `TnxActionHandler._passHandCards` に一本化（D&D と「指定枚数を渡す」が共用）。

## デッキの前提状態（2026-08-27 ユーザー裁定）

**アクトを開始する時点で、ニューロデッキはリセット済み（全カードが山に戻っている）であることが前提**。「ニューロカードはリセットされている状態しか想定されていません」（verbatim）。一部だけドローされた状態は**卓の運用としては起こらない**——開発のテスト環境で毎回リセットしていないだけである。

- リセットは RL が HUD のニューロデッキを右クリック →**「リセットする」**（`recall()`＝全カードを山札へ回収）で行う。山札（判定用のカード）側にも同じ操作がある。
- この前提の実装上の含意: アクト開始の**切り札の自動配布**（`_dealTrumpsForCasts`）は、キースタイルに対応するカードが山にあることを当てにしてよい。ドロー済みのカードを `pass` しようとするとコアが例外を投げるが、**それは前提が崩れた状態であり、ガードを設けるべき欠陥ではない**（→ Issues KI-043 は同日撤回）。
- 同様に、シーンカードはニューロデッキから引かれてシーンカード置き場に溜まる。アクト終了はこれを自動回収しない（回収は上記の RL 操作）。

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

**ゲスト → RL（GM ユーザー）への合算（2026-07-03 確定・実装はフェーズ11）**: RL の手札上限はゲストの使用するスタイル技能でも修正されうる。guest も `handMaxSizeMod` を持ち、**GM ユーザーの実効上限にはワールド内全ゲストの修正（有効な AE 由来）を合算**する。ゲストは全員 RL の持ち物のため ownerUserId 相当の絞り込みは行わない——**手札上限修正は基本的に効果時間「メインプロセス中」なので重複の懸念は実質ない**（ユーザー確認）。登場概念の導入（フェーズ14・2026-08-06 再編で旧14/16/18 を統合）後に「登場中のゲストのみ」へ絞り込みを差し替え可能。

## 所有モデル

手札・切り札の所有は **User flag**（`handPileId` / `trumpCardPileId`）に一本化済み（フェーズ2 完了・player Actor 廃止）。HUD は `game.user` の flag から手札置き場を取得する。配布・受け渡しは User 単位（`TnxCardSetupApp` / `dealTrumpFromNeuroDeck` / `selectAndPassToUser` 等）。

## チャット

`ChatMessage.create()` で HTML 直接投稿。カスタム ChatMessage クラス・`renderChatMessage` フックなし。

テンプレート: `templates/chat/scene-card.hbs`

## 判定ロジックとの関係

本ドキュメントは**物理的なカード移動**を扱う。達成値計算・技能判定ロジックはフェーズ7（設計）/8（実装）で実装済み（→ [Check_System.md](Check_System.md)）。usage テンプレートが判定のデータ基盤（→ [Usage_System.md](Usage_System.md)）。
