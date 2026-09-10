# シート一覧

## Application API 状況

| API | 使用状況 |
|---|---|
| ApplicationV1 | フェーズ3 で**全廃済み**。コード内に一切残存しない |
| ApplicationV2 | 全シート・HUD・ダイアログが V2 に移行完了（フェーズ3）|

## 実装済みシート

| クラス | 継承 | 対象 type | ファイル |
|---|---|---|---|
| TnxCharacterSheetBase | ActorSheetV2 | 基底（直接登録なし・フェーズ11-2 新設） | `scripts/actor/tnx-character-sheet-base.mjs` |
| TokyoNovaCastSheet | TnxCharacterSheetBase | cast | `scripts/actor/tnx-cast-sheet.mjs`（EXP 系＋セッション履歴のみ） |
| TokyoNovaGuestSheet | TnxCharacterSheetBase | guest | `scripts/actor/tnx-guest-sheet.mjs`（宣言のみの薄い派生・フェーズ11-3） |
| TokyoNovaTroopSheet | TnxCharacterSheetBase | troop | `scripts/actor/tnx-troop-sheet.mjs`（宣言のみの薄い派生・フェーズ11-4） |
| TokyoNovaExtraSheet | TnxCharacterSheetBase | extra | `scripts/actor/tnx-extra-sheet.mjs`（名前＋説明のみ・全ドロップ拒否・フェーズ11-5） |
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

**なし** — 全 Actor / Item type がカスタムシートを持つ（フェーズ11-5 で完了）

## 共通 UI パターン

- **view/edit モード**: `_isEditMode` + `_render()` オーバーライド
- **ヘッダーの配置規約**（2026-08-08 ユーザー明示）: シートヘッダーのボタン（トグル操作・UUID をコピー・シートを閉じる、およびアウトフィットのフラグトグル）は**すべて右寄せ**。**左寄せは編集/閲覧切り替えトグルのみ**。Foundry 既定では `flex:1` の `.window-title` が余白を吸ってコントロールを右へ押し出すが、本システムはシートのタイトルを非表示にする（本文に名前を出す方針）ため、余白を吸う要素が無いとコントロールが左端へ寄る。編集モードトグルの `margin-right:auto` に依存すると、トグルが出ない場合（権限なし・閲覧のみ）に左寄せへ崩れるので、`.tokyo-nova.sheet .window-header { justify-content: flex-end; }` でヘッダー側を右寄せ既定にする。タイトルを再表示するシート（アクト・FS 判定）は `flex:1` のタイトルが余白を吸うため影響を受けない
- **UUID コピー**: 全シートヘッダー
- **テキスト自動スケール**: CastSheet（`_applyTextSqueezing()`）
- **フラグトグル**: アウトフィットシートのヘッダーアイコン（準備済み・携帯中・プレアクト購入）
- **ContextMenu**: 全シートの閲覧/編集/削除（type 依存で複製・リンク解除等を追加）
- **コンボボックス（自由入力＋候補）**: `scripts/module/combobox.mjs` の `enhanceComboboxes(root)`（2026-07-23・KI-036）。標準の `<input list="x">` ＋ `<datalist id="x">` を、スタイル可能な独自コンボボックスへ「昇格」させる横断機構（number-input-spinner と同じ明示配線方式）。ネイティブ `<datalist>` はドロップダウンの見た目・スクロール・▼位置がブラウザ任せで制御不能なため置き換える。元 `<input>` はそのまま値の器として使い（name/value/フォーム送信を壊さない）、候補は datalist から昇格時に読んで `list` 属性を外す（ネイティブUI無効化・冪等マーカ `data-tnx-combo`）。ドロップダウンは `document.body` 直下に `position:fixed`（親 overflow でクリップされない・`max-height`＋`overflow-y:auto` で必ずスクロール可）、視覚はアンカー入力の computed style を継承しテーマ非依存。自由入力は常に許容（select 化しない）・↑↓/Enter/Esc 対応。CSS は `.tnx-combobox*`（`.tokyo-nova` 非依存の自立スコープ）。配線先=AE設定シートの変更キー入力＋ partAdd 値入力（`tnx.mjs` の `renderActiveEffectConfig`）、部位スロットプリセットの占有部位入力（`part-slot-preset-app.mjs`）。今後の同型入力は `<input list>`＋`<datalist>` を書いて `enhanceComboboxes` を1行呼ぶだけで対応。

## キャストシートタブ構成（2026-07-03 実態に整合化）

サイドバー: ポートレート・基本情報・**報酬点**（基準＋現在・再計算ボタン）・**EXP**（cast 専用）。

| タブ | 内容 |
|---|---|
| 能力 | 能力値ブレークダウン・スタイル（役割トグル・神業）・一般/スタイル技能 |
| 戦闘 | **フェーズ10 で実装**。CS ボックス（表示は単一「CS」・カット進行中はカレント参照＋グロー・編集モードで決定の連鎖＝ベース/修正/CS/カレント）＋ **AR ボックス**（フェーズ11 追補・付与型＝カット進行中のみ現在AR・それ以外「なし」）→ 武器 2 行（攻撃/パリー・weaponRefs による使用武器選択・空欄は生身・「生身（名前）」表記）→ 防御力（全義体込み種別合計 1 行）→ タイミングごとの使用技能・アウトフィット（**標準 7 種常設＋リアクションは非空のみ**・差し込みタイミングは非空のみ・その他＋解説参照は末尾）。一覧・効果系（負傷/BS）は置かない（他タブの領分）。<br>**2026-07-20 是正（ユーザー指摘）**: ①**リアクションを「該当する用途があるときのみ表示」へ**——常設 8 種のうちリアクションだけ表示条件を差し込みタイミングと同じにする（当初「チャットカード起動に一本化したので戦闘タブから除去」としたが同日ユーザーが撤回・**表示はする**が条件付き）。**位置はアクションの並び（メジャー→リアクション→オート）を保つ**——表示条件が同じでも末尾送りにするとルール上の行動順から外れて読めなくなるため（`fixedBuckets` の `onlyWhenFilled`）。②**差し込みタイミングの取りこぼしを修正**——「ダメージ算出の直前」等は自由記述ではなく `getSkillOptions().timing` の**名前つき選択肢**（`preCalcDamge` 等）だが、`classify()` が `action`/`process`/`other` の 3 値しか見ておらず、該当する用途があっても**丸ごと落ちていた**。名前つき選択肢も差し込み群として扱い、群の並びは選択肢の定義順→自由記述の順。**群はラベルで束ねる**（自由記述に同名を手入力した用途と選択肢を選んだ用途は同じ群に入る）。③用途の**「戦闘タブに表示しない」**（`actions[].hideInCombatTab`）が立つ行は全群から除外する。詳細は [Combat_Flow.md](../Game_Rules/Combat_Flow.md) |
| アウトフィット | 大分類別グループ（武器/防具/サイバーウェア/トロン/ヴィークル/住居/その他）・部位占有パネル |
| 状態 | ActiveEffect 一覧・バッドステータス一覧（`templates/parts/` の partial） |
| セッション履歴 | 履歴一覧（TnxHistoryMixin）・レコードシート同期トグル（cast 専用） |
| 詳細 | ライフパス 3 スロット・キャラクター説明エディタ |

> 旧表（能力値/スタイル/一般技能/…/エフェクトの 7 分割）はフェーズ5 以前の構成で実態と乖離していたため更新（2026-07-03）。

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

### アクター画像（2026-09-11）

共通サイドバーの単一画像枠に立ち絵（actor.img）またはコマ画像を表示する。編集モードのみ画像上部のスイッチを表示し、クリックで切り替える。編集モードで表示画像をクリックすると立ち絵・コマのどちらも画像選択を直接開き、対象の画像フィールドだけを保存する。閲覧モードは選択中の画像だけを表示する。表示選択はシートインスタンス内で保持し、アクターデータには保存しない。通常アクターのコマ設定はプロトタイプ（以後配置するコマ）、非リンクトークンの合成アクターではそのトークンが対象。

### アクトシートの追加目標値（2026-09-11）

情報項目で「目標値を追加」を押すと、本文を持つ追加情報を作成する。各追加情報の「元の技能＋目標値」で1段目の技能行を選び、その条件の目標値を入力する。「目標値だけを追加」は同じ本文へ別の条件を足す。各条件の×はその条件だけを削除し、追加情報の×は本文を含むブロックを削除する。旧データは「すべての使用技能（共通）」として表示される。
