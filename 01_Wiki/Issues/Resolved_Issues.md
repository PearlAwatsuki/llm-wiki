# 解消済み既知課題

フェーズ0〜2 で解消された項目の要約。

## KI-001: active-effects-list action 名不一致

- **解消**: フェーズ0 (P0-02)
- hbs の data-action を EffectsSheetMixin の switch 文と一致

## KI-003: styleSkill 正規表現パスミス

- **解消**: フェーズ0 (P0-03)
- `system.styleSkill.` プレフィックスを除去

## KI-004: initiative 式パスミス

- **解消**: フェーズ0 (P0-04)
- `@system.CS.value` → `@system.combatSpeed.value`

## KI-007: tap.conbatSpeedMod タイポ

- **解消**: フェーズB-6a (2026-05-23)
- DataModel で正しい綴り `combatSpeedMod` を定義

## KI-011: ジャーナルアクトシート作成導線

- **解消**: フェーズ1-1 (2026-05-24)
- `[data-action="createEntry"]` 直後にボタン挿入方式

## KI-012: ホットバー自動折りたたみ

- **解消**: フェーズ1-0 (2026-05-24)
- v13 で機能自体が削除 → 死コード削除

## KI-013: TnxHud と標準 UI 衝突

- **解消**: フェーズ1-2 (2026-05-24)
- CSS 座標調整 + 死にルール削除

## KI-014: switch case 内 lexical declaration

- **解消**: フェーズA-3 (2026-05-21)
- case ブロックを `{}` で囲む

## KI-015: 等価比較演算子

- **解消**: フェーズA-3 (2026-05-21)
- `==` → `===`

## KI-016: outfitBase フィールド型

- **解消**: フェーズB-9 (2026-05-24)
- exclusive/hide は StringField 維持で確定

## KI-005: handMaxSize 所有モデル確定 + ゲーム設定連携 (ベース連携)

- **解消**: フェーズ2 締め v0.3.1 (2026-06-05)
- `ActorBaseTemplate.handMaxSize` 削除 → 権威を User flag に一本化
- `resolveEffectiveHandMaxSize(user)` 新設 → User flag → ゲーム設定フォールバック
- 全消費箇所(drawCard / takeFromDiscard / dealInitialHands / drawMultipleCardsFromDeck)をリゾルバ経由に統一
- **残存 → KI-020**: ActiveEffect 修正層(層③)は ActiveEffect フェーズへ

## フェーズ3 で解消

### KI-009: EffectsSheetMixin 適用方式の二系統

- **解消**: フェーズ3-1/3-3 (2026-06-06/07)
- V1 の二系統（CastSheet: インスタンス代入、ItemSheet: 静的呼び出し）を `ACTIONS` オブジェクトに統一
- 両シートで `...EffectsSheetMixin.ACTIONS` スプレッドを `DEFAULT_OPTIONS.actions` に展開
- V1 互換メソッド (`activateEffectListListeners` / `_onXxx` 群) は `@deprecated` 付きで維持し、実コードから参照ゼロを確認

### KI-017: cast シートのデッドコード

- `_getHistoryData` / `_saveHistoryData` / `_updateExpTotal` を 3-3 にて完全削除
- **解消コミット**: a3110e5 (2026-06-07)

### KI-018: RewritedTarget タイポ → RewrittenTarget

- `StyleSkillDataModel` フィールド名を正規化。テンプレート・ロジック参照ゼロのため移行コスト不要
- **解消コミット**: 83f8600 (2026-06-07)

### KI-019: RewritingMiracle_ID 命名規約違反 → rewritingMiracleId

- `StyleSkillDataModel` フィールド名を camelCase に正規化
- **解消コミット**: 83f8600 (2026-06-07)

---

## フェーズ2 で部分的解消

### KI-017 (部分 → 3-3 で完全解消)

- `_getHistoryData` / `_saveHistoryData` に ownerUserId 分岐追加 (2-2) → 3-3 で削除完了

詳細は [Known_Issues_Detail.md](Known_Issues_Detail.md) を参照。
