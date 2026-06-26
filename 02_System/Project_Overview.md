# プロジェクト概要

## 名称

**トーキョーN◎VA THE AXLERATION (TNX)** — Foundry VTT システム (`tokyo-nova-axleration`)

## ゲーム特性 (実装に影響)

| 特性 | 実装への影響 |
|---|---|
| カード判定 | Cards API 中心。Roll 未使用 |
| シーン制 | 1アクト = シーン連なり。サブシーン仕組みはフェーズ16 |
| 神業 | 強力なルール書き換え。フェーズ20 で設計・実装（AE・全システムの上） |
| プレイヤー EXP | cast ではなく User に帰属。フェーズ2 で User flag へ移行完了 |

## 運用方針

- **長期戦**: カレンダー 1.5〜2.5 年、累積 400〜800h
- **完走・公開志向**: Foundry 公式公開申請予定
- **堅く作る**: 技術的負債回避
- **監査分離**: ルール正しさ（メカニクス監査）≠ 技術妥当性（設計レビュー）

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

時点依存の現状値（現在フェーズ・バージョン）は [Roadmap.md](Roadmap.md) を正本とし、本ファイルは固定値を持たない。

| 項目 | 値 |
|---|---|
| FVTT | v13.351 検証済 |
| 現在フェーズ・バージョン | [Roadmap.md](Roadmap.md) を参照 |
| 型構成 | `scripts/data/`・system.json を正とする（数値は直書きしない） |
| テスト件数 | 各フェーズ記録（[../01_Wiki/Phases/](../01_Wiki/Phases/) / Audit_and_Review）を参照 |

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
