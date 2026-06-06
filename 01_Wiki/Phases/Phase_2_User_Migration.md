# フェーズ2: User 移行 ✅ (完了)

**目標**: player Actor の全役割を User flag へ移行し、player Actor を廃止する。

**完了日**: 2026-06-03

詳細原本: [設計過程・検討記録](#設計過程検討記録)

## 設計過程・検討記録

*調査期間: 2026-05-24〜25*

### User が DataModel を持てないことの一次確認

フェーズ2 着手前の調査で、Foundry VTT v13 でカスタム DataModel を持てるドキュメントは
ActiveEffect / Actor / Card / Cards / ChatMessage / Combat / Combatant / Item /
JournalEntryPage のみであることを確認。**User はこのリストに含まれない。**

User にシステム固有データを保持する手段は **flag のみ**（型検証・スキーマ定義・
TypeDataModel の prepare* ライフサイクルなし）。
`CONFIG.User.dataModels` への登録は実現不可能であることも一次情報で確定。

### 所有モデルの選択肢検討と棄却理由

| 選択肢 | 概要 | 判断 |
|---|---|---|
| **A: User flag 移行** | history / exp / 手札を User flags に格納。型検証はコード側責任 | **採用** |
| B: player Actor 維持 | DataModel を維持し cast との関係を別アプローチで整理 | 棄却 |
| C: 別ドキュメント型 | カスタム Item(playerRecord 等)に格納。型付きを維持できる | 棄却 |

選択肢 B は DataModel の型安全性を維持できるが、「余分な Actor が UI に残る」「所有モデルの
確立が後回しになる」問題を解消しない。  
選択肢 C は Item が DataModel を持てるため型付きを維持できるが、「プレイヤーに紐付く Item」
という設計は Foundry 慣行から外れるとして棄却。

### トレードオフの記録

User 移行は **型付き Actor → 型なし flag への移行**という側面を持つ。
フェーズB で確立した「DataModel で型を持つ」流儀から、User だけ外れることになる。

| 観点 | 旧: player Actor | 新: User flags |
|---|---|---|
| 型保証 | ObjectField / SchemaField で型付き | 型なし(任意 JSON) |
| スキーマ定義 | DataModel で一元管理 | 実装者の自己管理 |
| Foundry 慣行 | DataModel が v13 推奨 | flags は補助的な用途が本来の意図 |
| 意味的正しさ | Actor が余分に存在 | FEAR 系「EXP はプレイヤーに帰属」をデータ構造で表現 |

### 選択肢 A を採用した判断根拠

1. **所有モデルを先に据える**: cast シートより先に所有モデルを確立することで二度手間を避ける
2. **player Actor の廃止**: 余分な Actor が UI に残ることの解消
3. **意味的正しさ**: FEAR 系「EXP はプレイヤーに付与」をデータ構造で表現するなら User が自然
4. **型安全性の妥協**: flag 運用の型なしは受け入れ、型検証はコード側責任で管理する設計とした

---

## 設計確定 (2026-05-25)

### 基本方針

- User は **DataModel 不可** → **flag** でデータ保持
- player Actor **廃止**
- cast は手札に**非関与**
- `ownerUserId` = EXP・履歴同期専用 (手札には使わない)

### User flag に移行するデータ

| データ | 旧所在 |
|---|---|
| history (オブジェクトマップ) | player Actor |
| exp.total | player Actor |
| handPileId, trumpCardPileId | player Actor |
| handMaxSize | player Actor / 設定 |

### UI

- **レコードシート** (ApplicationV2) — プレイヤーリスト右クリックから開く
- cast シートの手札表示 → **削除** (2-3)
- HUD 手札 → `game.user` flag 経由 (2-3)

### EXP 同期

二フック構成 + `{ syncing: true }` 無限ループ防止:

```
updateActor (cast) → cast → User
updateUser         → User → cast (全紐づき cast)
```

## サブフェーズ進捗

| サブ | 内容 | 状態 |
|---|---|---|
| **2-0** | User flag スキーマ + レコードシート (表示・編集) | ✅ 2026-05-25 |
| **2-1** | cast → User 紐付け (`ownerUserId`) | ✅ 2026-05-25 |
| **2-2** | EXP・履歴二フック同期 | ✅ 2026-05-25 |
| **2-3** | 手札・切り札 User flag + HUD 張替 + cast 手札削除 | ✅ 2026-05-25 |
| **2-4** | 既存 player データ → User flag 移行 (可逆) | スキップ (pre-production) |
| **2-5** | player Actor 廃止 (不可逆・最後) | ✅ 2026-05-25 |

### 2-3 成果物

- HUD テンプレートから player Actor 参照を削除
- 手札管理ロジックの User flag への移行
- cast シートからの手札管理インターフェースの削除

### 2-4 成果物

- スキップ(実運用前のためデータ移行不要と確認済み)

### 2-5 成果物

- system.json からの player Actor の削除
- 関連する DataModel とテストの削除
- 既存の player Actor エンティティの削除処理

### 2-0 成果物

- `scripts/module/user-flag-schema.mjs`
- `scripts/module/tnx-record-sheet.mjs`
- `templates/user/record-sheet.hbs`
- テスト +35 (計 579)

### 2-1 成果物

- `scripts/module/cast-ownership.mjs`
- `cast.system.ownerUserId` 新規フィールド
- `playerId` は並存 (2-5 まで)

### 2-2 成果物

- `scripts/module/exp-sync.mjs`
- `syncPlayerExpFromCasts` 削除 → `syncCastExpToUser` に置換
- `updateUser` フック新設
- cast シート history → User flag 翻訳

## 着手時の原則

- **可逆を先に、不可逆 (2-5) を最後**
- 各段でテスト/実機確認
- 2-5 直前でユーザー報告

## 関連 KI

- **KI-005 [解消済み]**: handMaxSize ベース連携(所有モデル確定 + ゲーム設定連携)→ v0.3.1 で決着
- **KI-020**: handMaxSize ActiveEffect 修正(層③)→ ActiveEffect フェーズへ
- **KI-017**: cast デッドコード → 2-2 で部分解消、3 で完全整理

## 参照

- [Experience_Points.md](../Game_Mechanics/Experience_Points.md)
- [HUD_and_Dialogs.md](../UI/HUD_and_Dialogs.md)
- 設計レビュー: [Design_Review_Entries.md](../../02_System/Design_Review_Entries.md) (2-0〜2-2 エントリ)
