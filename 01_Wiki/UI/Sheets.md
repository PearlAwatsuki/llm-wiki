# シート一覧

## Application API 状況

| API | 使用状況 |
|---|---|
| ApplicationV1 | フェーズ3 で**全廃済み**。コード内に一切残存しない |
| ApplicationV2 | 全シート・HUD・ダイアログが V2 に移行完了（フェーズ3）|

## 実装済みシート

| クラス | 継承 | 対象 type | ファイル |
|---|---|---|---|
| TokyoNovaCastSheet | ActorSheetV2 | cast | `scripts/actor/tnx-cast-sheet.mjs` |
| TokyoNovaItemSheet | ItemSheetV2 | 基底（直接登録なし） | `scripts/item/tnx-item-sheet.mjs` |
| TokyoNovaStyleSheet | TokyoNovaItemSheet | style | `scripts/item/tnx-style-sheet.mjs` |
| TokyoNovaMiracleSheet | TokyoNovaItemSheet | miracle | `scripts/item/tnx-miracle-sheet.mjs` |
| TokyoNovaGeneralSkillSheet | TokyoNovaItemSheet | generalSkill | `scripts/item/tnx-general-skill-sheet.mjs` |
| TokyoNovaStyleSkillSheet | TokyoNovaItemSheet | styleSkill | `scripts/item/tnx-style-skill-sheet.mjs` |
| TokyoNovaOrganizationSheet | TokyoNovaItemSheet | organization | `scripts/item/tnx-organization-sheet.mjs` |
| TokyoNovaOutfitSheet | TokyoNovaItemSheet | weapon/armor/ianus/cyborg/tron/tap/vehicle/residence/combiner/general | `scripts/item/tnx-outfit-sheet.mjs` |
| TokyoNovaHousingAreaSheet | TokyoNovaItemSheet | housingArea | `scripts/item/tnx-housing-area-sheet.mjs` |
| TokyoNovaLifePathSheet | TokyoNovaItemSheet | lifePath | `scripts/item/tnx-life-path-sheet.mjs` |
| TnxScenarioSheet | DocumentSheetV2 | JournalEntry（アクト） | `scripts/journal/tnx-scenario-sheet.mjs` |
| TnxRecordSheet | ApplicationV2 | User（レコード） | `scripts/module/tnx-record-sheet.mjs` |

## コアデフォルト（カスタム未実装）

**Actor**: guest, troop, extra — **フェーズ11** でカスタムシートを追加予定（判定・神業システム完成後）

## 共通 UI パターン

- **view/edit モード**: `_isEditMode` + `_render()` オーバーライド
- **UUID コピー**: 全シートヘッダー
- **テキスト自動スケール**: CastSheet（`_applyTextSqueezing()`）
- **フラグトグル**: アウトフィットシートのヘッダーアイコン（準備済み・携帯中・プレアクト購入）
- **ContextMenu**: 全シートの閲覧/編集/削除（type 依存で複製・リンク解除等を追加）

## キャストシートタブ構成

| タブ | 内容 |
|---|---|
| 能力値 | reason/passion/life/mundane 各能力値・combatSpeed |
| スタイル | スタイル一覧・紐付き技能・神業 |
| 一般技能 | generalSkill 一覧・報酬点フラグ |
| アウトフィット | 大分類別グループ（武器/防具/サイバーウェア/トロン/ヴィークル/住居/その他） |
| 詳細 | ライフパス 3 スロット・キャラクター説明エディタ |
| 戦闘 | ★内容未実装（フェーズ11 で整備） |
| エフェクト | ActiveEffect 管理 |

## アウトフィットシートタブ構成

| タブ | 内容 |
|---|---|
| 説明 | 概要サマリ（型別）+ ProseMirror エディタ |
| 設定 | 全フィールド編集（カテゴリ・各数値・フラグ群） |
| 用途 | usage テンプレート一覧（usage を持つ 10 種のみ） |
| エフェクト | ActiveEffect 管理 |

housingArea は専用シート（説明 + 修正値 7 種）。lifePath は説明 + 設定（種別・習得技能・識別キー）の 2 タブ。

## アクトシート（TnxScenarioSheet）

- `JournalEntry` はサブタイプを持たない → 作成ダイアログでの型選択不可
- 導線: ジャーナルディレクトリの「アクトシートを作成」ボタン（フェーズ1-1）
- 構造: flag + 独自タブ UI（ハンドアウト・情報項目・シーン・テキスト）
- **JournalEntryPage カスタムサブタイプ化は不採用**（通常ジャーナルに混入するため）

## 参照

- テンプレート: [Templates.md](Templates.md)
- v13 移行: [Foundry_v13.md](../Migration/Foundry_v13.md)
