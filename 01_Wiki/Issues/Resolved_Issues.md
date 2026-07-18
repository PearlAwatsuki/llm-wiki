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

## KI-024: スタン/説得の 10 上限位置が 2026-07-16 裁定と不一致（先キャップ→恒久軽減）

- **発見**: 2026-07-16（受けるダメージ軽減 AE の設計中・ユーザー裁定）
- **解消**: フェーズ12 途中（2026-07-16・本体 `ee5443b`）
- **症状**: `computeDamage` が攻撃側合計を先に 10 上限にしてから恒久軽減（防御力・受け値）を引いていた。裁定は「恒久軽減を引いた後＝ダメージ算出の一番最後に 10 上限」＝スタン/説得宣言時に最終ダメージが過小（例: 攻撃側合計18・防御力5 → 旧 5／正 10）。
- **解消内容**: `computeDamage` を恒久軽減（`mitigation`・10上限の前）と適用時軽減（`applyMitigation`・事後修正の後）の2段に分離し、10上限を恒久軽減後＝算出の一番最後へ移動。事後修正（modifyDamage・共有 `f.mods`＋防御側 per-target `t.mods`）はキャップ後に合流。上限は対象ごと（防御力が対象で違う）のため、共有台帳の10上限行を廃し各対象行の内訳に「スタン／説得（10上限）」を表示・台帳の「攻撃側合計」と修正クリックの基準は上限・軽減に依存しない攻撃側の数字（`attackerTotal`）に統一。テスト追従（KI-024 の正例を固定）。※起票時の「受けるダメージ軽減 AE（damage.taken）実装と同一作業で」は予定の同梱であり依存ではない＝AE は未実装のまま（フェーズ12 の別作業）。

## KI-026: 辞典（コンペンディウム）アイテムのシートが2回目以降の更新を映さない（用途削除が画面に残る）

- **発見/解消**: 2026-07-17（フェーズ12 途中・本体 `ed8d8f3`。前段の是正: `be86937`=委譲ガードの要素単位化 → `64471e3`=用途一覧ボタンを自前リスナー全廃・コア標準 data-action ディスパッチへ換装＋失敗の可視化＋配線回帰テスト）
- **症状**: 辞典アイテム（システム同梱パック内）のシートで用途を追加→削除すると、**データは削除されるのに一覧の行が画面に残る**（シートを開き直すと消えている）。リロード直後の「辞典アイテムを開く→用途追加→削除」で 100% 再現。辞典読み込みが先に済んだセッションでは起きない＝「条件不明の間欠バグ」に見えていた。アクター所持/ワールド直下では発生しない。
- **機構**: `pack.getDocuments()` はパック内の**全キャッシュ文書を新インスタンスへ差し替える**。開いているシートは旧インスタンスに登録されたまま孤児化し、以後の更新は正準（新）インスタンスに適用されて `_onUpdate`→再描画がシートへ届かない。トリガーは用途追加直後に自動で開く用途シートの `_prepareContext`→`loadSkillChoices`→`loadSkillEntries`→`getDocuments`（スタックトレースで実証）。実ワールド複製＋ヘッドレス実機（Playwright）で機構を特定し、修正後の解消も同環境で実証。
- **解消内容**: ① `loadSkillEntries`（skill-dictionary）と `TnxRlRequestApp._loadCompendiumSkills` を `getIndex(fields)` へ変更（文書インスタンスを作らない＝差し替えが起きない）② `setupDefaultSkills` をインデックス絞り込み＋`getDocument`（キャッシュ優先・差し替えなし）へ ③ `updateUsageActions` は書き込み直前に `fromUuid` で正準文書へ解決（孤児シート経由の stale 全配列上書き＝並行編集の巻き戻しも封止）。
- **教訓**: コンペンディウム文書を「選択肢の列挙」のために読むときは `getIndex` を使う。`getDocuments` の一括再取得は開いているシートを黙って殺す。

## KI-027: 攻撃の適用効果が命中・最終ダメージと突き合わされていない

- **発見/解消**: 2026-07-18（適用タイミング2種の設計調査中に発見・同日実装 `1bca435` →実機報告で是正 `0125fa1`）
- **症状**: 適用効果の付与がダメージ適用の一括処理のみで、対象リストは用途使用時のターゲットのまま攻撃結果と突き合わされない——①複数対象の部分命中で回避した対象にも付与②最終適用値 0 でも付与③ダメージカードを出さない限り付与経路が無い。
- **解消内容（最終形 `1ac4a08`・設計3転の経緯は [Combat_Flow.md](../Game_Rules/Combat_Flow.md)「攻撃の適用効果のタイミング2種」）**: タイミングは**攻撃用途の効果リスト所属**で決まる——「適用される効果」（一般＝命中時）＋「適用される効果（ダメージ時）」（`damageEffects`）。適用は全て手動の「効果を適用」ボタン: 一般＝攻撃カード（全対象解決後・命中1体以上・適用先=押下時点の命中対象で**カバー宣言済みはカバー側へ付け替え** `hitEffectTargetRefs`）／ダメージ時＝ダメージカード（適用先=カードの対象・ダメージ適用処理と独立）。「1点でも」等の条件はコード判定せず、0 の対象はボタンを押さない/除外ダイアログで外す＝卓判断。複数対象は除外チェックボックスダイアログ（全カード共通）。
- **経緯（同日）**: ① `1bca435`＝AE 設定フラグ＋hit 遷移時自動付与＋ダメージ適用連動の ≥1 ゲート → ② `0125fa1`＝実機報告（timing 伝搬漏れバグ・自動付与だとカバー宣言前に元対象へ乗る）で自動付与撤回・手動ボタン化・`usageEffectGrant` ソケット等撤去 → ③ `1ac4a08`＝「命中時の特別視は逆」「AE 側設定は普通の付与効果に攻撃概念を背負わせる」の指摘で用途の2リスト方式へ確定・≥1 のコード判定も撤廃。実機確認待ち。

## KI-028: 対決（非攻撃）カードで「効果を適用」ボタンが出ない

- **発見/解消**: 2026-07-18（同上・本体 `1bca435`）
- **症状**: `renderUsageEffectButton` の attackCheck 一律抑止（2026-07-11 ダメージカード一本化・当時は攻撃カード=攻撃のみ）が、対決判定カードの攻撃カード一般化（`238d775`）後の非攻撃対決（isAttack=false）にも効き、非攻撃対決はダメージフローを持たないため付与先「対象」の適用効果が到達不能だった（付与先「自分」は解決時即時付与のため影響なし）。
- **解消内容**: `attackCardEffectMode`（usage-effects・純関数/テスト付き）で出し分け——攻撃="attack"（抑止＋付与済みノート）・非攻撃対決=解決後（全対象解決/オープンリアクション解決/終端状態 fumble・miss・failed）に "button"＝結果カードと同じ手動ボタン（適用判断は卓＝v3 原則）・未解決は "hide"。実機確認待ち。

## KI-029: 辞典インポートで「判定」用途が正準名リタイプされ別タイプへ化ける

- **発見/解消**: 2026-07-18（ユーザー報告・同日修正・本体 `a5aa574`）
- **症状**: アクター作成時の基本技能自動インポートで、正準名の技能（回避/白兵/自我/信用/医療）が持つ素の「判定」用途が技能名に基づき別タイプへ付け替えられる（医療→治療・回避→ドッジ・白兵→パリー・自我/信用→各リアクション）。辞典ドロップ・アクター間コピー・該当名での手動作成でも同経路。付け替えは既存の同タイプ用途の有無を見ないため、辞典側が該当タイプの用途を併せ持つ場合は重複も生む。
- **機構**: 正準名の一回限り移行（本体 `238d775`・2026-07-17）の付け替え表 `canonicalizeSkillActions` が `preCreateItem` に**常設**で配線されており、ready 側の移行（`usageTypeCanonicalMigrated` でゲート）と違い移行済みデータかを区別するゲートが無い。移行後の辞典が正当に持つ「判定」用途を、全作成経路で旧データ扱いのまま正規化し続けていた。
- **解消内容**: preCreateItem からの適用を撤去（リタイプ・操縦への移動用途追加とも）。旧データ救済は ready の一回限り移行が完了フラグ付きで担っており、常設経路に置く必然性が無い。隔離実機（Foundry+Playwright・KI-026 の手法・W: subst＋port 30099）でアクター作成→インポート後の用途タイプ列が辞典の記載と完全一致（医療: check/treatment/treatment・回避: check/dodge・白兵: check/physicalAttack/parry・自我: check/mentalReaction・信用: check）することを検証済み。
- **教訓**: 一回限り移行の変換表を常設経路（preCreateItem 等）へ配線しない——現行データと旧データを区別できない正規化は、ユーザーが編集した正本データを黙って書き換え続ける。ユーザー正本（辞典・アイテム）へ副作用する機構は実装時に明確に報告し承認を得る。

## KI-030: 移動・離脱用途で対象選択ダイアログが出る

- **発見/解消**: 2026-07-18（ユーザー報告）／2026-07-19（本体 `fc95cc8`）
- **症状**: 移動・離脱タイプの用途は対象が存在しない行動（かつ非攻撃）なのに、対決欄に有効行があるため攻撃と共通の対象解決が走り、未ターゲット時はトークン選択ダイアログが出ていた。
- **解消内容**: タイプ分岐でなく**対象解決とカード形式の一般化**（用途の「対象」×「対決」の決定表・`usage-target-plan.mjs`）で是正。対象「-」「解説参照」「その他」は対象という概念がない＝対象要求せず（レティクルも読まない）、対決ありならオープンリアクションへ。移動・離脱の既定は対象「-」のためダイアログは出ない。正本は [Combat_Flow.md](../Game_Rules/Combat_Flow.md)「対象解決とカード形式の一般化」。実機確認待ち。

## KI-031: 用途ピッカーで長い用途名がアイコンに被る

- **発見/解消**: 2026-07-18（ユーザー報告・スクリーンショット）／2026-07-19（本体 `1baa983`）
- **症状**: 用途選択ダイアログのボタンはアイコンを `position: absolute; left: 12px` で置きラベルを中央寄せしており、長い用途名（例:「リアクション（移動妨害）（操縦：地上車両）」）が折り返すと行箱がボタン全幅に広がりアイコンと重なっていた。
- **解消内容**: ボタンの左右パディングをアイコン幅ぶん対称に確保（`padding: 8px 36px`・`.tnx-usage-picker` スコープ内のみ・中央寄せの見た目は維持）。実機確認待ち。

---

## フェーズ2 で部分的解消

### KI-017 (部分 → 3-3 で完全解消)

- `_getHistoryData` / `_saveHistoryData` に ownerUserId 分岐追加 (2-2) → 3-3 で削除完了

詳細は [Known_Issues_Detail.md](Known_Issues_Detail.md) を参照。
