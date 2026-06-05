# フェーズ2: User 移行 ✅ (完了)

**目標**: player Actor の全役割を User flag へ移行し、player Actor を廃止する。

**完了日**: 2026-06-03

詳細原本: `01_Raw/PHASE_2_INVESTIGATION.md`

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
- 設計レビュー: `01_Raw/DESIGN_REVIEW.md` (2-0〜2-2 エントリ)
