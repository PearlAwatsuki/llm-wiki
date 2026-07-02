# 解消済み既知課題

フェーズ0〜2 で解消された項目の要約。

## KI-001: active-effects-list action 名不一致

- **解消**: フェーズ0 (P0-02)
- hbs の data-action を EffectsSheetMixin の switch 文と一致

## KI-003: styleSkill 正規表現パスミス

- **解消**: フェーズ0 (P0-03)
- `system.styleSkill.` プレフィックスを除去

## KI-004: initiative 式パスミス

- **解消**: フェーズ0 (P0-04)
- `@system.CS.value` → `@system.combatSpeed.value`

## KI-007: tap.conbatSpeedMod タイポ

- **解消**: フェーズB-6a (2026-05-23)
- DataModel で正しい綴り `combatSpeedMod` を定義

## KI-011: ジャーナルアクトシート作成導線

- **解消**: フェーズ1-1 (2026-05-24)
- `[data-action="createEntry"]` 直後にボタン挿入方式

## KI-012: ホットバー自動折りたたみ

- **解消**: フェーズ1-0 (2026-05-24)
- v13 で機能自体が削除 → 死コード削除

## KI-013: TnxHud と標準 UI 衝突

- **解消**: フェーズ1-2 (2026-05-24)
- CSS 座標調整 + 死にルール削除

## KI-014: switch case 内 lexical declaration

- **解消**: フェーズA-3 (2026-05-21)
- case ブロックを `{}` で囲む

## KI-015: 等価比較演算子

- **解消**: フェーズA-3 (2026-05-21)
- `==` → `===`

## KI-016: outfitBase フィールド型

- **解消**: フェーズB-9 (2026-05-24)
- exclusive/hide は StringField 維持で確定

## KI-005: handMaxSize 所有モデル確定 + ゲーム設定連携 (ベース連携)

- **解消**: フェーズ2 締め v0.3.1 (2026-06-05)
- `ActorBaseTemplate.handMaxSize` 削除 → 権威を User flag に一本化
- `resolveEffectiveHandMaxSize(user)` 新設 → User flag → ゲーム設定フォールバック
- 全消費箇所(drawCard / takeFromDiscard / dealInitialHands / drawMultipleCardsFromDeck)をリゾルバ経由に統一
- **残存 → KI-020**: ActiveEffect 修正層(層③)は ActiveEffect フェーズへ

## フェーズ3 で解消

### KI-009: EffectsSheetMixin 適用方式の二系統

- **解消**: フェーズ3-1/3-3 (2026-06-06/07)
- V1 の二系統（CastSheet: インスタンス代入、ItemSheet: 静的呼び出し）を `ACTIONS` オブジェクトに統一
- 両シートで `...EffectsSheetMixin.ACTIONS` スプレッドを `DEFAULT_OPTIONS.actions` に展開
- V1 互換メソッド (`activateEffectListListeners` / `_onXxx` 群) は `@deprecated` 付きで維持し、実コードから参照ゼロを確認

### KI-017: cast シートのデッドコード

- `_getHistoryData` / `_saveHistoryData` / `_updateExpTotal` を 3-3 にて完全削除
- **解消コミット**: a3110e5 (2026-06-07)

### KI-018: RewritedTarget タイポ → RewrittenTarget

- `StyleSkillDataModel` フィールド名を正規化。テンプレート・ロジック参照ゼロのため移行コスト不要
- **解消コミット**: 83f8600 (2026-06-07)

### KI-019: RewritingMiracle_ID 命名規約違反 → rewritingMiracleId

- `StyleSkillDataModel` フィールド名を camelCase に正規化
- **解消コミット**: 83f8600 (2026-06-07)

## KI-002: usage-list edit ハンドラ不在

- **解消**: フェーズ8-6 (2026-06-15)
- `_onUsageEdit` ハンドラ追加 + `UsageCreationDialog.editAction()` で type / name / description / skillRefs の全フィールド編集 UI を実装

## KI-006: "socket": true の未実装

- **解消**: フェーズ8-1 (2026-06-15)
- **解消コミット**: 22a0c51（`TnxSocketHandler` 新設・ソケット集約）
- `scripts/module/tnx-socket-handler.mjs` で emit/on を集約。`ready` フックで `game.socket.on("system.tokyo-nova-axleration", ...)` を登録。RL 要求判定の結果反映（`emitCheckResult` → `ChatMessage.update`）・アクセスカード提示で稼働。

## KI-021: 判定の達成値からスタイル基本値が欠落

- **発見**: フェーズ9-1 着手時のコード精査（2026-06-20）
- **解消**: フェーズ9-1 (2026-06-20)
- **症状**: `TnxCheckFlow._buildAbilitiesCtx` が `actor.items.filter(i => i.system.equipped)` でスタイルを抽出していたが、`style` DataModel に `equipped` フィールドは存在せず常に空配列。結果、技能・能力判定の達成値が `growth + mod + effectMod + outfitMod` のみとなり、能力値の大半を占める**スタイル基本値（×レベル）が丸ごと欠落**していた（達成値が大幅に過小）。シート表示・bountyBase は全スタイルで正しく算出されていたため表面化しにくかった。
- **原因**: 能力値の最終実効値が `prepareDerivedData` で保持されず、4箇所が同式を各自再計算。判定の再計算だけが実在しない `equipped` フィルタを残していた（派生値パイプライン未整備に起因する分岐バグ）。
- **解消内容**: `CastDataModel.prepareDerivedData` で実効値を `system.<key>.total` / `.totalControl` に一本化（`computeAttributeFinal` 純粋関数＋0clamp）。シート表示・mundane 算出・判定の3箇所を集約値参照に置換。判定はシート表示と同一値になった。

## KI-020: handMaxSize への ActiveEffect 修正が未実装

- **解消**: フェーズ9-3 (2026-06-20、commit e4fc70c)
- **症状**: KI-005 の AE 修正層（層③）。`handMaxSizeMod` が未定義・参照0件で手札上限バフが機能しなかった（既定上限・個別設定は正常）。
- **解消内容**: `CastDataModel` に AE 着地点 `handMaxSizeMod` を新設（User は DataModel/AE を持てないため、所有 cast の AE がここに ADD で着地）。`resolveEffectiveHandMaxSize` を3層化（①既定 ②User flag 明示 ③所有 cast の handMaxSizeMod 合算、0clamp）。`computeEffectiveHandMaxSize` / `gatherHandMaxSizeMod` を分離しテスト可能化。派生値計算パイプライン（9-1〜9-3）の確立が前提だった。

## KI-022: キャストシートで辞典選択した住宅エリアが解決されない

- **発見**: フェーズ10-4 実装中（住宅エリアランクのバッジが出ない、2026-07-01）
- **解消**: フェーズ10-4 (2026-07-01)
- **症状**: 住宅施設に住宅エリアを**辞典ドロップダウンで紐づける**と、キャストのアウトフィットタブで**エリア供給値（修正後の登/セ）も住宅エリアランクのバッジ（10-4）も表示されない**。ドロップで紐づけた場合のみ表示されていた。住宅施設シート側は常に正常表示で乖離していた。
- **原因**: `system.housingArea` は辞典選択（`_housingAreaChoices` の `entry.uuid`）・ドロップ（`dropped.uuid`）のいずれも **UUID** を格納するが、`TokyoNovaCastSheet._resolveHousingAreaMods` が `useHousingAreaDrop` で分岐し、非ドロップ時に `pack.getDocument(ref)` を呼んでいた。`getDocument` は素の `_id` を要求するため UUID では解決できず null。供給値とバッジが同経路のため両方欠落。`useHousingAreaDrop` は入力 UI のモード切替であり保存形式は区別しない、という誤解に起因。
- **解消内容**: `_resolveHousingAreaMods` を住宅施設シートと同様 **`fromUuid(ref)` で無条件解決**（`type === "housingArea"` ガード付き）に統一。死んだ `HOUSING_AREA_PACK` 静的・`useHousingAreaDrop` 分岐を撤去。辞典選択・ドロップ双方で供給値・バッジが表示される。**既存不具合（供給値表示は 6-6 以来潜在）を 10-4 が顕在化**。

## KI-023: V2 シートで DEFAULT_OPTIONS.dragDrop が効かずアイテムのドロップ不可

- **発見**: スタイルシートへの神業ドロップが効かない（ユーザー報告 2026-07-02）。調査中に同根の scenario(アクト)シートも発覚。
- **解消**: フェーズ10 途中 (2026-07-02)
- **症状**: スタイルシートに神業アイテムをドロップしてもリンクされない（「作成してリンク」ボタンは機能）。同様に scenario シートへのカード(デッキ/山)ドロップも不可（UI は「設定タブでドロップしてください」と案内するのに効かない）。
- **原因**: ApplicationV2（`DocumentSheetV2`/`ItemSheetV2`）は V1 の `options.dragDrop` を自動処理しない。両シートは V1 流の `dragDrop:[{…}]` オプション＋`_onDrop` オーバーライドに依存し、`foundry.applications.ux.DragDrop` を手動で束ねていなかったため `_onDrop` が一度も呼ばれなかった（死んだ設定＋死んだハンドラ）。アウトフィット/キャストシートは `_onRender` で手動束ねしていて動作＝乖離。フェーズ3 の V2 移行時の取りこぼし。
- **解消内容**: 両シートの `_onRender` で `new foundry.applications.ux.DragDrop.implementation({ dropSelector:".tnx-import-box--dropzone", permissions:{drop:()=>this.isEditable}, callbacks:{drop:this._onDrop.bind(this)} }).bind(this.element)` を追加（アウトフィットシートと同方式）。死んだ `dragDrop` オプションを撤去。`_onDrop` は既存のまま。**教訓: V2 シートのドロップは `_onRender` で DragDrop を手動束ねする（options.dragDrop は不活性）。**

## KI-010: combatSpeed (CS) がシートに表示・入力できない

- **解消**: フェーズ10-5 (2026-07-02)
- **症状**: `attributes.combatSpeed` はフィールド定義のみで派生計算・シート表示・入力 UI が存在せず、initiative 式（P0-04 で修正済み）が常に 0 を返しコンバットトラッカーが機能しなかった。
- **解消内容**: CS を3層モデル（正本 [Combat_Flow.md](../Game_Rules/Combat_Flow.md)「コンバットスピード」＋「実装表現の原則」）で実装。
  - **決定値（保存・再計算不可）**: `base`（CSベースの能力値項＝プレアクト初期化で `floor((理性+感情+生命)÷2)` を実効値から焼き込み）／`value`（CS＝プレアクトに決定）／`current`（CSカレント）。旧 `mod` フィールドは役割がなく削除（§4.3 承認済み）。
  - **修正値（ライブ）**: `freeMod`（手動）＋タップ CS 修正（フラグ駆動＝OFF:準備起点/ON:携帯起点・ゴースト登場中読み飛ばし）＋AE（`system.cs.base|value|current`・cs.base のみ保持アイテムの準備ゲート）。実効値 `baseTotal`/`valueTotal`/`currentTotal` を派生算出（0clamp なし＝CS は負になりうる）。
  - **UI**: シート上は常に**単一の「CS」表示**（3層は内部データ・ユーザー裁定 2026-07-02）。表示層は**自動制御**＝カット（開始済み戦闘）進行中はカレント・それ以外は CS（派生 `displayTotal`）。カット開始/参加時に**カレントへ CS 実効値を自動セット**（戦闘フック・GM 書込）し、開始/終了/参加/離脱で該当アクターを再準備・再描画。戦闘タブ先頭の**リードアウトパネル**（左アクセントバー＋大型モノスペース数字・カット進行中は状態チップ・読み飛ばし分「X（Y）」併記・編集モードで内部データ行＋プレアクト初期化）。詳細タブには置かない（5-6 確定）どおり戦闘タブ軸。
  - **initiative**: `@system.combatSpeed.displayTotal`（表示中の CS を読む。カット開始時はカレント＝CS なのでセットアップ順も正しく、以後のカレント修正がメイン順に反映される）。guest/troop でも式が解決できるよう `AttributesTemplate.prepareDerivedData` に素の実効値フォールバックを設置。
- **残**: CSカレントのカット中自動管理（セットアップ末決定・メジャー後0・待機1）＝フェーズ13。extra は combatSpeed 自体を持たないため initiative 不可（従前どおり・フェーズ11/13 で扱う）。

---

## フェーズ2 で部分的解消

### KI-017 (部分 → 3-3 で完全解消)

- `_getHistoryData` / `_saveHistoryData` に ownerUserId 分岐追加 (2-2) → 3-3 で削除完了

詳細は [Known_Issues_Detail.md](Known_Issues_Detail.md) を参照。
