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

## フェーズ2 で部分的解消

### KI-017 (部分)

- `_getHistoryData` / `_saveHistoryData` に ownerUserId 分岐追加 (2-2)
- 完全整理はフェーズ3

詳細は `01_Raw/KNOWN_ISSUES.md` を参照。
