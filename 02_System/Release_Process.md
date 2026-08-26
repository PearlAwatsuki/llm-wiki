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
| 公開 (フェーズ19) | 1.0.0 |

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

### 7. wiki 更新（フェーズ記録の閉め）

- [Roadmap.md](Roadmap.md) の進行状況マーク更新（完了フェーズ ✅・次フェーズ ▶）
- 現フェーズの Phases 詳細ページを完了状態に更新
- [Completed_Phases.md](../01_Wiki/Phases/Completed_Phases.md) に完了フェーズの概要を追加
- Audit_and_Review / Design_Review_Entries / Issues の全エントリ通読・最新化
- 本ファイルの「完了済みリリース」表を更新

## 完了済みリリース

| バージョン | フェーズ | 日付 |
|---|---|---|
| v0.1.0 | フェーズA | — |
| v0.2.0 | フェーズB | 2026-05-24 |
| v0.3.0 / v0.3.1 | フェーズ1・2 | 2026-06-05 |
| v0.4.0 | フェーズ3 | 2026-06-07 |
| v0.5.0 | フェーズ4 | 2026-06-08 |
| v0.6.0 | フェーズ5 | 2026-06-12 |
| v0.7.0 | フェーズ6 | 2026-06-14 |
| v0.8.0 | フェーズ7・8（判定システム） | 2026-06-20 |
| v0.9.0 | フェーズ9（ActiveEffect / 派生値計算パイプライン） | 2026-06-24 |
| v0.10.0 | フェーズ10（既存シート関連のやり残し総仕上げ） | 2026-07-02 |
| v0.11.0 | フェーズ11（guest / troop / extra シート＋トループ級・NPC取得用途） | 2026-07-08 |
| v0.12.0 | フェーズ12（攻撃判定 + ダメージ算出 + FS判定基幹） | 2026-07-21 |
| v0.13.0 | フェーズ13（カット進行＝戦闘トラッカー） | 2026-08-06 |
| v0.14.0 | フェーズ14（セッション進行＝アクト/シーン・登場判定・情報収集判定） | 2026-08-27 |

## 関連

- [Roadmap.md](Roadmap.md)
- [Phase_Operations.md](Phase_Operations.md)
