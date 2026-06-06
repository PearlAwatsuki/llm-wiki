# プロジェクト概要

## 名称

**トーキョーN◎VA THE AXLERATION (TNX)** — Foundry VTT システム (`tokyo-nova-axleration`)

## ゲーム特性 (実装に影響)

| 特性 | 実装への影響 |
|---|---|
| カード判定 | Cards API 中心。Roll 未使用 |
| シーン制 | 1アクト = シーン連なり。サブシーン仕組みはフェーズ11 |
| 神業 | 強力なルール書き換え。フェーズ9/10 で独立設計 |
| プレイヤー EXP | cast ではなく User に帰属。フェーズ2 で User flag へ移行完了 |

## 運用方針

- **長期戦**: カレンダー 1.5〜2.5 年、累積 400〜800h
- **完走・公開志向**: Foundry 公式公開申請予定
- **堅く作る**: 技術的負債回避
- **監査分離**: ルール正しさ (MECHANICS_AUDIT) ≠ 技術妥当性 (DESIGN_REVIEW)

## 著作権・ライセンス

| 項目 | 内容 |
|---|---|
| コード | MIT |
| Foundry | Foundry VTT Limited License Agreement |
| TNX 原作 | 鈴吹太郎 / F.E.A.R. / KADOKAWA の著作物 (二次創作) |
| ルールテキスト | リポジトリに同梱しない。GM 手入力前提 |
| コード内データ | enum 等の構造的最小限のみ可 |

デザイン (フェーズ5): 公式アート模倣禁止。一般的サイバーパンク表現に留める。

## 現状スナップショット

| 項目 | 値 |
|---|---|
| FVTT | v13.351 検証済 |
| バージョン | v0.3.1 (フェーズ2 締め完了) |
| 現在フェーズ | **フェーズ3** — 既存シート ApplicationV2 移行 |
| テスト | 625 件 (v0.3.1 時点) |
| DataModel | 25 type 移行済 |

## ドキュメント体系

```
llm-wiki/
  01_Wiki/      # FVTT 開発向け Wiki (正本)
  02_System/    # 本フォルダ — 全体方針・運用 (正本)
CLAUDE.md       # Claude Code 向け指針 (ルート)
```

## 関連

- [Development_Rules.md](Development_Rules.md)
- [Roadmap.md](Roadmap.md)
- Wiki 入口: [../01_Wiki/00_Index.md](../01_Wiki/00_Index.md)
