# フェーズ12 残作業 実装計画（2026-07-20 策定）

設計正本 → [Focus_System_Mechanics.md](../Game_Mechanics/Focus_System_Mechanics.md)、ルール正本 → [Focus_System.md](../Game_Rules/Focus_System.md) / [Damage_Rules.md](../Game_Rules/Damage_Rules.md) / [Bounty_Points.md](../Game_Rules/Bounty_Points.md) / [Active_Effects.md](../Game_Mechanics/Active_Effects.md)。

**ゴール**: フェーズ12 を完了し v0.12.0 に到達する。残りは「RL 任意付与（3件）」「FS判定基幹（6件）」「仕上げ（2件）」＋実機確認2件。

## 全体制約

- DataModel ベース。`template.json` を復活させない。
- 技能起動は唯一の関数 `TnxCharacterSheetBase._activateItemCheck(actor, item, extraOpen)` に集約。**新しい起動関数を作らない**。
- 手札からカードを出す操作は必ず HUD 直接クリック。
- チャットカードのボタンは全幅・縦積み。選択肢ダイアログも縦積み。
- 数値入力欄は `number-input-spinner`（例外＝使用回数）。
- 技能名の表示は名前欄・アクター技能リスト以外では `formatSkillName` / `itemDisplayName` を通す（〈〉囲い）。識別キーの生値は表示しない。
- アイテム参照表示は `fromUuid` で live 解決。名前キャッシュを持たない。
- ユーザー可視文字列にフェーズ番号・内部機構名・内部キー生値を入れない。説明系 UI 文言（？バッジ・解説 title・notes 段落）を足さない。
- 表・リスト系 UI は既定で手動並び替え可能にする。
- CSS 編集後は `npm run lint:css` も回す。共通 CSS の汎用セレクタを場当たりで広げない。
- 各作業単位ごとに `npm test` / `npm run lint` を通してからコミットする。
- コミットは Conventional Commits・description は日本語。

## 接続する既存関数（調査済み）

| 関数 | 位置 | 用途 |
|---|---|---|
| `applyDamageToTarget(target, category, final, stage, {persuade})` | `damage-flow.mjs:1019` | ダメージ適用の入口 |
| `openDamageRollDialog(attackMessage)` | `damage-flow.mjs:100` | ダメージカード算出ダイアログ（攻撃文脈つき） |
| `renderDamageCard(message, html)` / `applyDamagePatch` | `damage-flow.mjs:320` / `:988` | ダメージカードの描画・ライブ書き換え |
| `buildInflictedEffectsData(kind, {hidden})` / `CONDITION_KINDS` | `conditions.mjs:275` / `:75` | BS・負傷・戦闘不能の AE データ生成 |
| `resolveUsageEffectData` / `prepareUsageEffectPayload` | `usage-effects.mjs:40` / `:311` | 効果の複製ペイロード生成 |
| `evaluateFormula(formula, data)` / `buildFormulaData(actor, result, bearer, target)` | `tnx-formula.mjs:175` / `:49` | 式評価 |
| `TnxRlRequestApp` / `check-request.hbs` | `tnx-rl-request-app.mjs` | 判定要求（踏襲元・プリセット読み込みの追加先） |
| `TnxScenarioSheet` | `journal/tnx-scenario-sheet.mjs` | アクトシート（判定要求タブの追加先・flags 保存） |
| `game.settings.register(SCOPE, ...)` | `tnx.mjs:1108` 他 | ワールド設定の登録先 |
| `getSceneControlButtons` フック | `tnx.mjs:1540` | シーンコントロールへのボタン追加 |

---

## 作業順序と依存

```
T1 RL任意ダメージ ──┐
T2 RL任意状態     ─┼→ T7（敗北確定の導線）
T3 報酬点配布     ──┘        ↑
T4 focusSystem スキーマ → T5 シート → T6 パネル+起動 → T7 敗北確定 → T8 判定要求2種
                                              T9 プリセット（T3・T6 の後）
                                                        ↓
                                          T10 宣言の棚卸し → T11 仕上げ
```

T1〜T3 は FS と独立。T4 以降は順に積む。T9 は T3（報酬点）と T6（読み込みプルダウン）の後。

---

## T1: RL 任意ダメージ付与

**ファイル**
- 作成: `scripts/module/rl-grant.mjs`（RL 付与の共通入口・シーンコントロールのボタンから開く）
- 作成: `templates/app/rl-grant-damage.hbs`
- 修正: `scripts/module/damage-flow.mjs`（攻撃文脈なしでダメージカードを作る経路を追加）
- 修正: `scripts/tnx.mjs`（シーンコントロールに「ダメージ付与」を追加・GM 限定）
- テスト: `tests/module/rl-grant.test.mjs`

**インターフェース**
- 産出: `buildRlDamageCardData({ targets, category, value, note })` → ダメージカードのフラグ用データ（純ロジック）
- 消費: `applyDamageToTarget`（既存・変更しない）

**方針**: RL が対象・系統・値・自由記述を指定 → ダメージカードを投稿 → 以降は**既存の適用/軽減フローに合流**。RL 専用の分岐・軽減有無のトグルは**置かない**（防護点の可否は種別 X で表現・軽減技能の可否は自由記述と口頭）。値は数値直接入力（カードプレイはしない）。

**テスト**: 系統ごとのカードデータ生成／自由記述が空でも成立／複数対象で対象行が対象数ぶん出る／値 0 でもカードは出る（適用側で判断）。

**コミット**: `feat: RL が任意にダメージを与えられる機能（ギミック・敗北時処理用）`

---

## T2: RL 任意状態・効果付与

**ファイル**
- 修正: `scripts/module/rl-grant.mjs`
- 作成: `templates/app/rl-grant-effect.hbs`
- 修正: `scripts/tnx.mjs`（シーンコントロール）
- テスト: `tests/module/rl-grant-effect.test.mjs`

**インターフェース**
- 産出: `buildRlEffectPayload(source)` — `{kind}`（BS 選択）または `{itemUuid, effectId}`（効果複製）から、対象に作る AE データを返す。**供給元との同期情報を持たない**（切り離したコピー）

**方針**: 入口は2つ——BS・負傷・戦闘不能は `CONDITION_KINDS` から選び `buildInflictedEffectsData` に流す／任意の効果は既存アイテムの効果を選んで複製する（`usage-effects.mjs` の複製処理を再利用）。**ゼロからの自作 UI は作らない**（Foundry 標準のアクター効果タブで既に可能）。付与は対象アクターに直接（親＝アクター）。所有権が無い場合は既存の委譲パターンに倣う。

**テスト**: BS 選択が `buildInflictedEffectsData` と同じデータを返す／効果複製が供給元同期のフラグを**持たない**こと（切り離しの検証）／存在しない効果 ID で null を返す。

**コミット**: `feat: RL が任意に状態・効果を付与できる機能（供給元とは切り離す）`

---

## T3: 報酬点の配布カード

**ファイル**
- 修正: `scripts/module/rl-grant.mjs`
- 作成: `templates/chat/bounty-grant.hbs`
- 作成: `scripts/module/bounty-grant.mjs`（カードの描画・受け取り処理）
- 修正: `scripts/tnx.mjs`（シーンコントロール・`renderChatMessageHTML` への配線）
- テスト: `tests/module/bounty-grant.test.mjs`

**インターフェース**
- 産出: `buildBountyGrantData({ targets, amount, note })`／`applyBountyGrant(actor, amount)` → `system.bounty` に加算（`bountyBase` は非変更）

**方針**: 判定要求と同じアクター登録方式。カードの対象行のボタンを**そのアクターの所有者**（または RL）がクリックして加算＝権限委譲不要。**負数を受け付ける**（没収も同機構）。二重受け取りを防ぐため、受け取り済みはフラグに記録しカードをライブ書き換えする。

**テスト**: 正数で加算・負数で減算／`bountyBase` が変わらない／受け取り済みの対象は再加算されない／複数対象で個別に受け取れる。

**コミット**: `feat: 報酬点の配布カード（前金・負数で没収も表現）`

---

## T4: `focusSystem` ページのスキーマ

**ファイル**
- 作成: `scripts/data/journal/focus-system.mjs`
- 修正: `system.json`（`documentTypes.JournalEntryPage.focusSystem`）
- 修正: `scripts/tnx.mjs`（DataModel の登録）
- テスト: `tests/data/journal/focus-system.test.mjs`

**インターフェース**
- 産出: `FocusSystemDataModel` — `restriction` / `defeatCondition{type: "cut"|"other", text, cutLimit}` / `defeatEffect` / `targetProgress` / `supportSkillKey` / `rows[]{id, threshold, skillKey, targetValue, progressMod{source, param, formula}, note}` / `memo`

**テスト**: スキーマ取得／既定値（`defeatCondition.type` の初期値・`rows` 空配列）／`rows` の各フィールドが存在する／FS 名は DataModel に持たない（ページ名を使う）。

**コミット**: `feat: FS判定シートのデータ構造（JournalEntryPage 型 focusSystem）`

---

## T5: `focusSystem` シート＋進行修正の参照パラメータ

**ファイル**
- 作成: `scripts/journal/tnx-focus-system-sheet.mjs`
- 作成: `templates/journal/focus-system-sheet.hbs`
- 作成: `scripts/module/progress-mod.mjs`（参照パラメータの候補表と解決）
- 修正: `css/tnx2.css`
- テスト: `tests/module/progress-mod.test.mjs`

**インターフェース**
- 産出: `PROGRESS_MOD_SOURCES`（`none` / `outfit` / `actor`）／`buildProgressModChoices(source)` → グループ形式プルダウンの選択肢（`{group, value, label}[]`・**表示は日本語ラベル・内部キーの生値を出さない**）／`resolveProgressMod({source, param, formula}, actor, {paramValue})` → 数値

**方針**: 判定行は手動並び替え可。参照元プルダウン → パラメータのグループ形式プルダウン1本 → 式欄。式には `@param` が入る。`source: "outfit"` の実解決（判定者の準備済みアイテムから探す）は**13 送り**——ここでは保存と、`paramValue` を外から渡された場合の評価までを実装する。敗北条件は `type` で表示を切り替え（`cut` のときだけ `cutLimit` 欄を出す）。

**テスト**: `source: "none"` で式の数値がそのまま返る／`@param` に `paramValue` が入る／`@param * 2` などの係数式が評価される／候補表が内部キーの生値をラベルに出さない／不正な式で例外を投げず null を返す。

**コミット**: `feat: FS判定シートと進行修正の参照パラメータ`

---

## T6: 進行状態の正本＋FS判定パネル＋起動

**ファイル**
- 作成: `scripts/module/focus-system-state.mjs`（ワールド設定の読み書き）
- 作成: `scripts/module/tnx-focus-system-panel.mjs`（パネル＋起動フォーム）
- 作成: `templates/app/focus-system-panel.hbs` / `templates/app/focus-system-start.hbs`
- 作成: `templates/chat/focus-system-start.hbs`
- 修正: `scripts/tnx.mjs`（設定登録・シーンコントロールに「FS判定」を**全員に**表示）
- 修正: `css/tnx2.css`（バー式ゲージ部品を新設・`outfit-slot-gauge` のセレクタは広げない）
- テスト: `tests/module/focus-system-state.test.mjs`

**インターフェース**
- 産出: `listActiveFocusSystems()` / `startFocusSystem(snapshot)` / `updateFocusSystem(id, patch)` / `endFocusSystem(id, outcome)`
- 産出: `activeProgressRow(rows, progress)` → 閾値 ≦ progress のうち閾値最大の1行（該当なし null）
- 産出: `clampGauge(value, max)` / `gaugeMarkers(rows, max)` → 閾値マーカーの位置（%）

**方針**: ワールド設定 `activeFocusSystems` が**唯一の正本**。起動時にページの内容をスナップショットとして取り込む（以後ページ編集の影響を受けない）。パネルは RL 編集可・PL 閲覧のみ、**進行中が無いときは「進行中のFS判定はありません」**。進行値ゲージに閾値マーカーを刻む。カットゲージは `defeatCondition.type === "cut"` のときのみ。起動フォームの先頭に読み込み元プルダウン（`focusSystem` ページをジャーナル名の optgroup で列挙／「手動設定」）。FS開始カードは**通知のみで状態を持たない**。

**テスト**: `activeProgressRow` — 初期値行が progress 0 で有効／閾値ちょうどで切り替わる／飛ばされた行が無視される（3が飛ばされ5が有効）／行が空なら null／閾値未満しかなければ null。`clampGauge` — 下限0・上限max。`gaugeMarkers` — 最大値を超える閾値を含まない。状態 CRUD — 追加・更新・除去。

**コミット**: `feat: FS判定パネルと進行状態の正本（ワールド設定）・起動フロー`

---

## T7: 達成・敗北の確定

**ファイル**
- 修正: `scripts/module/tnx-focus-system-panel.mjs`
- 作成: `templates/chat/focus-system-result.hbs`
- テスト: `tests/module/focus-system-state.test.mjs`（追記）

**方針**: パネルに「達成」「敗北」の確定ボタン。押すと `activeFocusSystems` から除去し結果カードを投稿する。**敗北時の処理は自動化しない**（FS ごとに異なるため）。結果カードに `defeatEffect` の記載を表示し、**T1 の RL 任意ダメージ付与への導線**を置く。

**テスト**: 確定で配列から除去される／結果カードのデータに outcome と `defeatEffect` が入る／既に終了した FS の再確定が no-op。

**コミット**: `feat: FS判定の達成・敗北の確定とダメージ付与への導線`

---

## T8: 進行判定・支援判定の要求

**ファイル**
- 修正: `scripts/module/tnx-focus-system-panel.mjs`
- 作成: `scripts/module/focus-system-request.mjs`
- 修正: `scripts/module/tnx-rl-request-app.mjs`（FS 文脈をフラグに載せる）
- テスト: `tests/module/focus-system-request.test.mjs`

**インターフェース**
- 産出: `buildProgressRequest(fs)` → `{identificationKey, targetValue, focusSystemId}`（有効行から導出）
- 産出: `buildSupportRequest(fs)` → 同上（技能＝`supportSkillKey`・目標値＝有効行の `targetValue`）
- 産出: `listParticipantCandidates()` → `game.combat?.combatants` のキャスト（無ければワールドのキャスト）

**方針**: 既存 `checkRequest` にそのまま乗せる。**支援判定**は参加キャスト全員が宛先（複数人可）。**進行判定**は宛先を **RL が手動選択**（手番の自動判別は 13）。カードに `focusSystemId` を持たせ、13 が結果を受け取れるようにする。**AR 残量での絞り込みは 13 送り**。

**テスト**: 進行判定の技能・目標値が有効行から導出される／進行値が進んで行が切り替わると導出結果も変わる／支援判定の技能は `supportSkillKey`・目標値は有効行と同じ／有効行が無いとき null を返す／Combat が無いときワールドのキャストにフォールバックする。

**コミット**: `feat: FS判定の進行判定・支援判定の要求カード`

---

## T9: プリセット（判定要求・報酬点）

**ファイル**
- 修正: `scripts/journal/tnx-scenario-sheet.mjs`（判定要求タブ）
- 修正: `templates/journal/scenario-sheet.hbs`
- 作成: `scripts/module/request-presets.mjs`
- 修正: `scripts/module/tnx-rl-request-app.mjs`（読み込み元プルダウン）
- 修正: `scripts/module/rl-grant.mjs`（報酬点の読み込み元プルダウン）
- 修正: `css/tnx2.css`
- テスト: `tests/module/request-presets.test.mjs`

**インターフェース**
- 産出: `listCheckRequestPresets()` / `listBountyPresets()` → `{journalName, id, label, ...}[]`（optgroup 用にジャーナル名を持つ）
- 産出: `applyPresetToForm(preset)` → フォーム値への写像

**方針**: アクトシートの flags に `checkRequests[]` / `bountyGrants[]` を保存。`label` のプレースホルダーは「判定要求n」。手動並び替え可。**対象アクターはプリセットに含めない**（起動時に選ぶ）。

**テスト**: プリセット → フォーム値の写像（各フィールド）／`label` 未入力時のプレースホルダー生成／対象アクターが写像に含まれない／複数アクトシートがジャーナル名でグループ化される。

**コミット**: `feat: 判定要求・報酬点のプリセット（アクトシートに判定要求タブ）`

---

## T10: 宣言（declaration）の実行時棚卸し

**ファイル**: `scripts/actor/tnx-character-sheet-base.mjs` / `scripts/data/item/common/usage.mjs` ほか（調査の結果に依存）

**方針**: スコープ #6「declaration の実行時棚卸し（薄い配線のみ・本格実装はフェーズ20）」。`declaration` タイプは実装済みのため、**実行時に取りこぼしが無いかの通し確認**を行い、見つかった穴だけを塞ぐ。**先回りの拡張をしない**。

**コミット**: 見つかった不足ごとに `fix:` で分割（無ければコミットなし）。

---

## T11: 仕上げ（記録・v0.12.0）

- Phase_12 ページに 2026-07-19 の判定要求ワークストリーム（`4313515` / `5b5c8ad` / `fa8d40a` / `adcd7cf` / `67694ac` / `659a2df`）を反映する（現在 12-4x までで止まっている）。
- 全 KI・監査・設計レビューのエントリを通読しステータスを最新化する（§6.2 フェーズ完了時）。
- `package.json` / `system.json` を 0.12.0 に手動同期する。**タグ付けと push はユーザー**。

**コミット**: `docs: フェーズ12 の記録を最新化` / `chore: v0.12.0`

---

## 並行して消化する実機確認（2件）

1. **適用効果の2リスト方式**（`1ac4a08`）— 設定 UI・攻撃カード/ダメージカードのボタン一巡・カバー付け替え・除外ダイアログ・非攻撃対決ボタン
2. **リアクションフロー大改修**（`2e25db5` 以降）— 対象行クリック→本人/代理→シークレット→公開の一巡・スキップ・複数併存表示・オープン直接起動と結果カード・全カードの内訳表示・受け値/社会ゲートのダメージ接続・治療/修理/回復の自動セルフ・妥当性警告

隔離 Foundry ＋ Playwright のハーネスで消化する。実機で見つかった表示の破綻（はみ出し・不自然な折り返し・状態と合わない文言）は**自分で見つけて直す**。

---

## 13 への申し送り（この計画では実装しない）

- 進行判定の実行 → `floor(@diff/10) + 進行修正` の自動加算
- 支援判定の AR 消費と、対象の次の進行判定への +1
- 支援判定要求の宛先を AR 残量で絞り込む／進行判定要求の宛先を手番から自動判別する
- `progressMod.source === "outfit"` の実解決
- コンバットトラッカーのカット進行と `cut` の連動
