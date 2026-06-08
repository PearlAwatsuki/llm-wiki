# 開発ルール

`CLAUDE.md` §4 および運用方針の要約。

## 命名規約

| 対象 | 規則 | 例 |
|---|---|---|
| ファイル | kebab-case | `tnx-cast-sheet.mjs` |
| クラス | PascalCase | `TokyoNovaCastSheet` |
| DataModel | プレフィックスなし | `CastDataModel` |
| パッケージID・flag | system.json 踏襲 | — |

## 実装方針

1. **大きな変更前**: プラン提示 → 承認後に実装
2. **1 PR/コミット = 1 機能** — 複数バグ修正を混ぜない
3. **DataModel フィールド追加・改名**: 事前承認
4. **動作中機能の「ついでリファクタ」禁止**
5. **現フェーズ外の作業は提案しない**
6. **先回り拡張禁止** — 必要になった時に追加
7. **新規機能にはテスト併設** (`tests/`)

## やってはいけないこと

| 禁止事項 | 理由 |
|---|---|
| ルールテキスト・ゲームデータの埋め込み | 著作権 |
| ゲームルール解釈の推測実装 | ユーザー確認必須 |
| 神業の無合意自動化 | 設計合意必須 (usageCount 等既存範囲は可) |
| usage 構造の無議論改変 | 判定基盤 |
| システム全体書き換え提案 | スコープ爆発 |
| 動作確認なし大規模移行 | 品質 |
| EXP 計算の「効率化」書き換え | 複雑なルール上の事情 |

## 設計指針

### usage ベースの判定

D&D 5e 思想を踏襲。Item の usage (actions[]) が判定・効果のデータ基盤。
「用途を起動する」共通インタフェースを上位に置く。

### UI フェーズ分離

「構造 (V2)」「所有モデル (User)」「見た目 (デザイン)」を分離し、
シートを二度蒸し返さない一気通貫を狙う。

### DataModel とシート

DataModel 移行とシート書き換えは**別フェーズ**で実施 (フェーズB vs フェーズ3)。

## 開発コマンド

```bash
npm install          # 依存インストール
npm test             # Vitest (1回)
npm run test:watch   # ウォッチモード
npm run lint         # ESLint
npm run lint:fix     # 自動修正
```

Foundry 起動・リロードは手動。

## コミット規約

[Conventional Commits](https://www.conventionalcommits.org/ja/) — description は日本語。

```
feat: 新機能
fix: バグ修正
docs: ドキュメント
chore: ビルド・設定
refactor: リファクタ
test: テスト
```

フェーズ作業: **1タスク = 1コミット** (PHASE_*_TASKS 参照)

## ご無沙汰期間

| 期間 | 対応 |
|---|---|
| 2週間+ | CLAUDE.md + 現フェーズ TASKS を見直し |
| 1ヶ月+ | CLAUDE.md + wiki 全体から現状再構築、新規セッション |
| 3ヶ月+ | wiki から現状再構築、新規セッション |

## 関連

- [Audit_and_Review.md](Audit_and_Review.md)
- [Phase_Operations.md](Phase_Operations.md)
