# Phase 6: 未実装シート追加

## 目的

シート未実装のドキュメント型（outfit 系 Item 11 種 + lifePath、guest / troop / extra Actor 3 種）に
ApplicationV2 シートを新規作成する。フェーズ3 の CSS 構造（tnx2.css 変数体系）とフェーズ4 のデザインを踏襲する。
あわせてキャストシート「アウトフィット」タブを整備する。

現状、`tnx.mjs` でコアの ActorSheet / ItemSheet を unregister しているため、
これらの型は**シートを開くことすらできない**状態にある。

## スコープ外

- 判定システムの設計・実装（フェーズ7-8）。用途タブの完成は判定システム依存
- 神業システム（フェーズ9-10）
- ActiveEffect パイプライン整備（ActiveEffect フェーズ。本フェーズで対象フィールドが固まる）
- KI-010（combatSpeed UI）の配置先確定（フェーズ11 までに決める）
- シナリオ作成ウィザード再設計（フェーズ11）

## 設計決定事項（2026-06-12 ユーザー確認済み）

- **cast の `player_name` は削除**（B-4 申し送りのデッドフィールド判断に決着）
- **`identificationKey` は全アイテム型に追加**（フェーズ5-5 では実装済み 5 種のみだった）
- **ライフパスは出自・経験・邂逅の 3 種のみ**。life-path.mjs 旧コメントの「運命」は誤り
- **キャスト詳細タブのライフパススロットは型と対応させる**（出自スロットには lifePathType=origin のみ）
- **ライフパスの概要は参照先アイテムの description を表示**（アイテム側に summary フィールドは追加しない）
- **参照表示は毎回 fromUuid で現物解決**。キャッシュ名はアイテム削除時のフォールバック表示のみ。
  ドロップ時の名前を保持し続ける方式は今後一切採用しない（全アイテム共通の恒久方針）
- **ゲスト = キャストからライフパス・報酬点・セッション履歴（EXP 含む）を除いたもの**
- **トループ**: スタイル・能力値・技能・アウトフィット・状態を持ち、神業なし。
  `heads`（人数）が HP のように機能し、トークンのリソースバーに割り当てる
- **エキストラ**: 基本は名前のみだが、スタイル・神業・技能・アウトフィットを持てる
- **装備品共通フィールドのルール解釈**は [Outfits.md](../Game_Rules/Outfits.md) を正本とする
- **majorCategory / minorCategory は選択リスト化**。シートでは大分類→小分類の連動ドロップダウンとし、
  その Item type に有効な小分類のみを提示する

## サブフェーズ

### 6-0: DataModel 変更・設計記録 ✅ 完了（2026-06-12）

- [x] cast: `player_name` 削除（テスト更新含む）（`0bdb47a`）
- [x] 未実装 12 種（weapon / armor / ianus / cyborg / tron / tap / vehicle / residence / housingArea / combiner / general / lifePath）に `identificationKey` 追加（`2fc7128`）
- [x] outfitBase: `isCarrying`（Boolean）追加（`7d314e7`）
- [x] outfitBase: `buy` / `hide` を `{mode, value}` SchemaField 化（mode = none / value / reference）（`7d314e7`）
- [x] outfitBase: `preserveExp` / `appearancePenalty` を NumberField 化（`7d314e7`）
- [x] outfitBase: `majorCategory` / `minorCategory` を choices 付き StringField 化（`outfit-categories.mjs` 新設）（`7d314e7`）
- [x] troop: `heads`（`{value, max}` SchemaField）追加（`9a0df1a`）
- [x] lifePath: `lifePathType` を choices 付き（origin / experience / encounter）に変更 + 「運命」誤コメント修正（`50b331a`）
- [x] wiki: Game_Rules/Outfits.md 新設、監査・設計レビューエントリ追加

全テスト 654 件グリーン / ESLint 0 errors（警告は既存分のみ）。装備品データは実運用前のためマイグレーション不要（ユーザー確認済み）。

### 6-1: 装備品シート共通基盤 + general ▶ 実装完了・実機確認待ち（2026-06-12）

- [x] DataModel 微調整（`331881e`）: hack → nullable NumberField（なし=null）/
  timing → スタイル技能と共通の配列構造 / part・exclusive → 自由記述（初期値 ""）
- [x] `TokyoNovaOutfitSheet` + `outfit-sheet.hbs` 新規実装、general に登録（`74423ee`）
  - 説明 / データ / 設定 / エフェクトの 4 タブ（usage を持たないため用途タブなし）
  - データタブ閲覧モード: ルルブ表記（購：購入値／常備化経験点、隠：隠匿値／危険値、解説参照分岐込み）
  - ヘッダーに大分類→小分類の連動カテゴリドロップダウン（type で小分類フィルタ）
  - タイミング行 UI はスタイル技能と同形（選択肢共通、`TnxSkillUtils.formatTimingLabel` に整形処理を共通化）
  - 真偽値の配置: isPrepared / isCarrying = データタブ「運用状態」、isOption / isPre-play / isCyber = 設定タブ「アウトフィット性質」
- [x] 実機確認 1 回目のフィードバック反映（`0dde383`）:
  データタブ廃止（概要 = 購/隠/電制/部位 のみを説明タブのエディタ上部に常時表示、編集 UI は
  すべて設定タブでスタイル技能レイアウト踏襲）/ 電脳制御値は略号「電制」表記 /
  数値入力欄は number-input-spinner 必須（**恒久指示**。例外は使用回数のみ）/
  フラグの意味確定（準備済み・携帯中・サイバーウェア・プリプレイ購入 → Outfits.md 参照。isOption は未言語化）
- [x] 実機確認 2 回目のフィードバック反映（`4b581ca` / `e9965bb`）:
  isOption の意味確定（別アウトフィットに装備するオプション → Outfits.md）/
  part を部位名 + スロット数の `{value, slots}` 構造に変更（表記: slots 1 は部位名のみ、0・2 以上は「武器2」形式）/
  hack を `{mode, value}` のドロップダウン化（なし/数値）/ uses に種別（type）追加 + スタイル技能と同形の使用回数 UI /
  form-group ラベルの横並び共通 CSS 化（スタイル技能側にも適用、フィールドセット内ラベル幅 110px）
- [x] ラベル横並び CSS の修正やり直し（`676da0e`）: 前回の form-group 全体への :has 拡張 + 固定ラベル幅は
  既存レイアウトを広範に崩したため**全面撤回**。`.tnx-form-field > .form-group` にスコープを絞った
  130px ラベル列 + 入力列のグリッドで実現（チェックボックス・スピナー行は既存 :has ルールが詳細度で優先され不変）。
  **教訓: 共通 CSS の汎用セレクタを場当たりで広げない**（メモリにも記録済み）
- [x] 追加対応 3 件（`1408be5`、2026-06-12）:
  名称マーカー表示（★/†/※/＠/末尾※ → ルールは Game_Rules/Terminology.md「名称表記マーカー」、
  実装は `tnxDecoratedName` ヘルパーに集約。シートヘッダー閲覧モード + キャストシートのリスト表示に適用）/
  アウトフィット 10 種に usage テンプレート合成 +「用途」タブ追加 /
  旧アウトフィットリストの部位表示を `tnxPartLabel` で新構造に追従
- [ ] ユーザー実機確認（general アイテムの作成・編集・閲覧表示）
- [→] extensible のスロット追加/削除 UI → **6-2（最初の extensible 型 = weapon）で実装**

確認済み（2026-06-12）: hack はなし/数値の 2 状態、timing はスタイル技能と選択肢共通、exclusive は自由記述（自動化なし）。

### 6-2: 戦闘系装備シート（weapon / armor / cyborg） ▶ 実装完了・実機確認待ち（2026-06-12）

- [x] 固有フィールドのルール解釈を確認（攻/受/射/FA/ダメージ種別 S/P/I/X/攻撃範囲/防/制/義体の宣言ルール
  → Outfits.md に記録）
- [x] DataModel（`c5385cf` / `7aa7dfc`）: isthrow → `isConsumption` + `quantity {value, max}` に一般化 /
  `isBiological` も outfitBase へ一般化 / range → `{min, max}`（5 択）/ damageType choices（S/P/I/X）/
  `attackArea` 追加（-/範囲/範囲(選択)/シーン/シーン(選択)）/
  extensible → `slots[] {kind, count}` プール方式（kind 6 種、装備側はスロット数のみ）
- [x] シート実装（`405cc36`）: 共通シートを weapon / armor / cyborg に登録し type 判定で差分吸収。
  型別サマリ（weapon = 購/隠/攻/受/射/ス/電制/部位、armor = 購/隠/防/制/電制/部位、
  cyborg = 購/隠/防/攻/受/電制/部位）、防御/攻撃データのフィールドセット、
  スロット数スピナー、消費アイテムの「×個数」ヘッダー表示
- [x] 実機確認フィードバック反映（`589a8b7`、2026-06-13）:
  状態・性質フラグ 7 種を設定タブ**最下部**のフィールドセットに集約し 2 カラムのチェックリスト化
  （アウトフィットシートの**フットノートは全廃**）/「攻撃値」廃止 →「攻撃力」1 行（種別ドロップダウン + 値）、
  damageType は単一選択化 / isCarrying デフォルト true /
  カテゴリ「生体装備：武器/防具」を「生体装備」に統合（型対応を配列化）/
  防表記は「防(S／P／I)：n／n／n」（(S／P／I) を省略しない）
- [x] 状態フラグのアイコントグル化 + フットノート全シート撤去（`fdac3ba` ほか、2026-06-13）:
  準備済み（fa-shield-halved）/ 携帯中（fa-suitcase）/ プレアクト購入（fa-cart-shopping）を
  D&D 5e 装備済みトグル踏襲の**ウィンドウヘッダー**アイコンに（閲覧モードでも切替可。
  用語は「プリプレイ購入」ではなく TNX 用語の「プレアクト購入」）。
  フットノートは一般技能・スタイル技能・神業シートからも撤去（**以後シートにフットノートを置かない**。
  アクトシートの注釈 5 件のみ存続）
- [ ] ユーザー実機確認（weapon / armor / cyborg の作成・編集・サマリ表記）
- [→] オプションの装備連携（スロットへの装備・解除）→ アイテム側はスロット数のみ確定のため、
  キャストシートのアウトフィットタブ（6-6）で設計
- [→] 消費アイテムのセッション終了時復活・経験点連動 → EXP 連携の設計が必要なため 6-6 以降で検討

### 6-3: 電脳系装備シート（ianus / tron / tap） ▶ 実装完了・実機確認待ち（2026-06-13）

- [x] 固有フィールドのルール解釈を確認（制御値修正は既存同様 / CS = combatSpeedMod、準備で CS に修正 /
  サ = cycle、タップの処理能力でハッキング FS 判定に使用 → Outfits.md に記録）
- [x] 大修正（`4a7c121`）: **part を配列化**（複数部位の占有に対応。行追加/削除 UI、表記は「、」連結）/
  **timing を単一 SchemaField 化**（アウトフィットのタイミングは一つのみ。行追加 UI 撤去）
- [x] ianus / tron / tap をアウトフィットシートに登録（`4a7c121`）。
  サマリ: ianus = 購/隠/ス/表/深/無/制/部位（**電制なし**・編集欄も非表示）、
  tron = 購/隠/ス/電制/部位、tap = 購/隠/ソ/ハ/CS/電制/部位（サは概要に含めない）。
  IANUSデータ（制御値修正）/ タップデータ（サイクル数・CS修正）のフィールドセット追加
- [x] IANUSオプションのカテゴリ対応を ianus → general に修正（装備対象とカテゴリが違う唯一のオプション）
- [ ] ユーザー実機確認（ianus / tron / tap + 部位複数行・タイミング単一化の回帰確認）

### 6-4: 生活系装備シート（vehicle / residence / housingArea / combiner） ▶ vehicle/residence/housingArea 完了・combiner 一部のみ（2026-06-13）

- [x] 固有フィールドのルール解釈を確認（SF/乗員・住宅施設/エリアの分離・コンバイナーの合成ルール → Outfits.md）
- [x] DataModel（`a8c0725`）: vehicle に attack/defence 追加 / residence の重複 mod 除去（住宅エリアの役割） /
  housingArea に hideMod/slotMod 追加（修正値 7 種） / カテゴリ修正（住宅オプション・アクセサリ → general、IANUSオプション → general）
- [x] vehicle/residence/combiner をアウトフィットシートに登録（`8c3f532`）。
  `_prepareView` を型別 switch に整理。vehicle = 購/隠/攻/SF/防/制/乗員/ス/電制/部位、
  residence = 購/隠(隠匿値のみ)/登場/セ(電／ア)/ス/部位（危険値・電制なし）
- [x] housingArea 専用シート（`984681a`）: 説明 + 修正値 7 種。number-input-spinner を基底へ汎用化
- [ ] **combiner のコンバイン連携（二つの元の指定・見た目選択・合成表示）は設計確認後に実装**
- [→] residence + housingArea の実効値算出、住宅アクセサリの準備可能数 → フェーズ6-6（キャストシート）
- [ ] ユーザー実機確認（vehicle / residence / housingArea）

### 6-5: lifePath シート + キャスト詳細タブ連携

- [ ] lifePath シート（lifePathType ドロップダウン / skillName / description）
- [ ] キャスト詳細タブ: 型対応ドロップ検証・fromUuid live 解決・description の概要表示

### 6-6: キャストシート「アウトフィット」タブ整備

### 6-7: guest シート

- [ ] キャストシートの共通部品化（キャストの挙動は変えない範囲）

### 6-8: troop シート

- [ ] heads のトークンリソースバー連携

### 6-9: extra シート

### 6-10: 仕上げ

- [ ] wiki 整合化（Sheets.md / Actor_Types.md / Item_Types.md の V1 時代記述の更新）
- [ ] 監査・レビュー全エントリ通読・最新化
- [ ] CHANGELOG 更新・v0.7.0 バージョンアップ

## 実装サブフェーズ着手時に確認する残課題

- ~~`hack` / `timing` / `exclusive` の形式~~ → 6-1 で確定（なし/数値の 2 状態 / スタイル技能と共通 / 自由記述・自動化なし）
- 各種別の固有フィールドの意味と UI 形式（各サブフェーズ冒頭で確認）

## 完了条件

- 全ドキュメント型がカスタムシートを持ち、フェーズ4 デザインで表示・編集できる
- キャストシートにアウトフィットタブがあり、装備品の管理ができる
- v0.7.0 タグ

## 参照

- [Roadmap.md](../../02_System/Roadmap.md)
- [Outfits.md](../Game_Rules/Outfits.md)
- [Item_Types.md](../Documents/Item_Types.md) / [Actor_Types.md](../Documents/Actor_Types.md)
- フェーズ5 申し送り → [Phase_5_Tasks_Detail.md](Phase_5_Tasks_Detail.md)、Design_Review_Entries の 5-6 エントリ
