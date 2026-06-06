# リリース手順

[Semantic Versioning](https://semver.org/lang/ja/) に従う。

## バージョン同期箇所

| ファイル | フィールド |
|---|---|
| `package.json` | `"version"` |
| `system.json` | `"version"` |
| git | タグ `vX.Y.Z` |

## 進行ルール

| イベント | バージョン |
|---|---|
| フェーズ完了 | MINOR up (例: 0.1.0 → 0.2.0) |
| フェーズ間バグ修正 | PATCH up (例: 0.2.0 → 0.2.1) |
| 公開 (フェーズ12) | 1.0.0 |

## リリースチェックリスト

フェーズ完了時 (MINOR)。PATCH は 1〜5, 7 のみ。

### 1. テスト・リント確認

```bash
npm test    # エラー 0
npm run lint  # エラー 0
```

### 2. CHANGELOG.md 更新

- `[Unreleased]` → `[X.Y.Z] - YYYY-MM-DD` に切り出し
- 新しい空 `[Unreleased]` セクションを追加

### 3. バージョン番号更新

`package.json`, `system.json` (README があれば同様)

### 4. コミット

```
chore: vX.Y.Z をリリース
```

### 5. git タグ

```bash
git tag vX.Y.Z
git push origin main --tags
```

### 6. GitHub Releases (MINOR 以上)

- タグ `vX.Y.Z`
- タイトル: `vX.Y.Z - <フェーズ名>`
- 本文: CHANGELOG 該当部分

### 7. CLAUDE.md 更新

- §2.2 現在フェーズ → 次フェーズ
- ロードマップ進行マーク更新

## 完了済みリリース

| バージョン | フェーズ | 日付 |
|---|---|---|
| v0.1.0 | フェーズA | — |
| v0.2.0 | フェーズB | 2026-05-24 |

## 関連

- [Roadmap.md](Roadmap.md)
- [Phase_Operations.md](Phase_Operations.md)
