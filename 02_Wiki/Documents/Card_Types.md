# Card 型定義

Card 3 type — フェーズB-8 で DataModel 化済。

## Type 一覧

| type | DataModel | 固有 system フィールド |
|---|---|---|
| playingCards | `PlayingCardsDataModel` | なし (description のみ) |
| neuroCards | `NeuroCardsDataModel` | なし |
| other | `OtherDataModel` | なし |

すべて `CardBaseTemplate` (description のみ) を mixin。

## 設計方針

- **Foundry コアフィールドは system 側に持たせない**: suit, value, face 等は Card document のコア属性
- `Item.BaseTemplate` とは独立 (`scripts/data/card/common/base.mjs`)

## カード生成モジュール

| モジュール | 内容 |
|---|---|
| `tnx-playing-cards.mjs` | 標準トランプ 52枚×デッキ数 + ジョーカー 2枚 |
| `tnx-neuro-cards.mjs` | ニューロデッキ 31枚 (value: -18〜21) |
| `tnx-access-cards.mjs` | アクセスカード 8枚 (シーン・切り札・YES/NO 等) |

## Cards API 操作

`TnxActionHandler` 経由:

| 操作 | API |
|---|---|
| 山札から判定 | deck → discard (表向き) |
| ドロー | deck → hand |
| プレイ | hand → discard |
| 切り札 | trump pile → scene/RL discard |
| ニューロ | neuro deck → scene pile |

## v13 互換

Cards API に関する v13 致命的エラーは観測されていない (実機カード操作は要別途確認)。

## 参照

- DataModel: `scripts/data/card/*.mjs`
- 操作実装: `scripts/module/tnx-action-handler.mjs`
