# アウトフィット付与AEの既存属性と拡張候補（2026-09-20）

状態: 現行コードの調査と拡張案。実装未変更。ユーザー要望は数値以外も含め、アウトフィットの項目をAEで上書き・追加できるようにすること。
確定した除外事項: 用途の追加は不要。付与AEによる新規用途の生成、および用途一式を追加するキーは設計対象から除外する。
基準: helpers.mjs の computeItemEffectiveValues / AE_FLAG_PARAMS / resolveItemTotalPath、character-base.mjs の _applyEffectBuffs、各Item DataModel。
「パーサが文字列を受け付ける」ことと「実効値ができ、読み手で使われる」ことを区別する。以下の既存対応はコード上の経路があるという意味で、全項目のFoundry実機再検証済みを意味しない。

## 既存の数値属性

| 対象 | 現行の正規キー（代表形） |
|---|---|
| 購入値 | `system.buy` |
| 隠匿値 | `system.hide` |
| 危険値 | `system.appearancePenalty` |
| 電脳制御値 | `system.hack` |
| 常備化経験点 | `system.preserveExp` |
| 攻撃力 | `system.attack.value` |
| 受け値 | `system.guardValue` |
| フルオート値 | `system.FAValue` |
| 防御力 | `system.defence.S` / `.P` / `.I` |
| 制御値修正 | `system.controlMod` |
| 移動力（SF） | `system.speedFactor` |
| 乗員数 | `system.passenger` |
| サイクル | `system.cycle` |
| CS修正 | `system.combatSpeedMod` |
| 住宅の登場判定目標値 | `system.appearanceTarget` |
| 住宅の電脳・アナログセキュリティ | `system.cyberSecurity` / `system.analogSecurity` |
| 使用回数最大値 | `system.uses.max` |

対象型にある属性だけが有効。数値は実効値へ適用し、元データは維持する。value/totalなどを受け付ける互換形があるが、新しい正準表記は公開キー設計に従って統一する。mode=none等を数値補正だけで変更する機能ではない。
スロットはslots[].count.totalの算出自体は存在するが、種別から解決する `system.slot.capacity` / `system.slot.software.capacity` 等は未実装。配列位置を手書きする可能性と、安定した公開キーの対応を混同しない。

## 既存の数値以外の属性

| 対象 | 現行キー | 現行動作 |
|---|---|---|
| 攻撃のダメージ種別 | `system.attack.damageType` | S/P/I/X等を文字列上書き |
| 名称 | `name` | 文字列上書き、`{}`で元名を含める装飾。新規約では `system.name` へ移行 |
| フルオート・レーザー・生身変更 | `system.isFullAuto` / `system.isLaser` / `system.isFleshChange` | 真偽の上書き |
| 変異器官・式神装備 | `system.isMutantOrgan` / `system.isShiki` | 真偽の上書き。式神タイプは別で未対応 |
| 準備不要・消費アイテム | `system.noPrepareRequired` / `system.isConsumption` | 真偽の上書き |
| 故障・破壊 | `system.isMalfunction` / `system.isDestroyed` | 真偽の上書き。サービスの免疫は維持 |
| 装備部位の追加 | `system.part.<部位キー>` | `and[:数]`で追加占有、`or[:数]`で既存部位一式の代替候補追加 |

故障・破壊はアウトフィットの使用可否フラグで、Actorのstatusesではない。付与AEのステータス機能撤去と混同しない。
装備部位は追加だけで、全体置換、既存行の個別変更、ホスト条件の変更等は未対応。
旧isCyberは副分類へ移行済みで現在のレジストリにない。技能用のusesBounty/suits等は通常AE側であり、この付与AE一覧には含めない。

## 追加が必要な属性案

以下はすべて新機能または適用経路の拡張が必要。既存項目名を優先した候補で、全項目の最終命名は別途統一する。

| 対象 | 追加キー案 | 許可する操作案 |
|---|---|---|
| 大分類・小分類 | `system.majorCategory` / `system.minorCategory` | 選択値の上書き |
| 副分類 | `system.additionalCategories` | 分類行の追加／集合全体の上書き |
| 装備部位一式 | `system.equip.parts` | 構造化した部位行の追加／全体上書き |
| 部位の結合 | `system.equip.relation` | and/or選択の上書き |
| 任意部位 | `system.equip.optional` | 真偽上書き |
| 必須・代替部位の簡易追加 | `system.equip.required.<部位キー>` / `system.equip.alternative.<部位キー>` | 既存part追加の新キー化。値は消費数 |
| スロット容量 | `system.slot.capacity` と名前付き種別 | 数値補正。種別解決と実効読取の接続 |
| 射程 | `system.range.min` / `system.range.max` | 列挙値の上書き |
| 攻撃範囲 | `system.attackArea` | 列挙値の上書き |
| 白兵／射撃武器の区分 | `system.isMeleeWeapon` / `system.isRangedWeapon` | 真偽上書き |
| スタン可能 | `system.canStun` | 真偽上書き |
| 式神タイプ | `system.shikiType` | 列挙値の上書き |
| ゴースト時のCS修正無視 | `system.combatSpeedModGhostIgnore` | 真偽上書き |
| 「なし／数値／参照」等の方式 | `system.buy.mode` / `system.hide.mode` 等、`system.defence.mode` | 対応項目の選択値上書き |
| 使用回数の制限有無・回復区分 | `system.uses.isLimit` / `system.uses.type` | 真偽・選択値上書き |
| 装弾数・弾数制限 | `system.ammo.max` / `system.ammo.isLimit` | 数値補正・真偽上書き |
| 専用条件 | `system.exclusive` | style/organizationの条件行追加／上書き |
| 説明・画像 | `system.description` / `system.img` | 文の追加／置換、画像の置換。imgはDocument側へ解決 |
| 対応する操縦技能 | `system.operateSkillKey` | 技能参照の上書き。候補はActorの能力として表示 |
| 住宅エリアの参照・利用有無 | `system.housingArea` / `system.useHousingAreaDrop` | 参照・真偽上書き。住宅補正の再計算を含む |

部位行には種別・部位キー・消費数・ホストの大分類／小分類／除外条件／特徴／識別キー・解説参照種別がある。system.equip.partsの行編集でこれらを表現する。ここでのhostKeyは装備条件の参照であり、廃止した「AEの変更対象をアウトフィット識別キーで指定するキー」ではない。
既存用途の属性変更は新規用途の追加とは別の機能であり、今回の指示だけでは採否を確定しない。上表の追加対象には含めず、用途システムの構造変更は行わない。

## 項目は存在するが別の扱いが必要なもの

属性変更を分類・性能・説明へ広げることは可能。ただし次は数値や文字列という理由だけで通常のAE変更に混ぜない。

- **資源の現在状態**: quantity.value/max、ammo.spent、uses.spent。容量補正と実消費は異なる。AE失効で消費済み弾数等が戻る処理にはしない。対応するなら実効容量と消費台帳の規約を先に定める。
- **実際の準備状態**: isPrepared/isCarrying、parentItemId/parentSlotKind、partOrChoice/partAltChoice、combineGroupId。部位条件の変更と装備先の付け替えは異なる。関係更新・準備制約・解除を含む処理が必要。
- **生成・取得・履歴情報**: derivedDataRefs、hasDerivedData/isDerivedData、fromStyleSkillKey、isPre-play、isCheckAcquired、modifications、extraActorRef。派生データ生成や所有Actor更新を伴うため、再計算のたびに実体を生成・削除しない設計が必要。
- **同一性・管理情報**: _id、UUID、Item.type、ownership、identificationKey、転送／移行用flags。特に小分類とItem.typeは別物。小分類変更でDocumentの型まで変更する設計にはしない。
- **既存計算の出力**: total、isOption、占有数・空き数等。必要な入力属性へAEを適用し、出力を再計算する。
- 補助Item型のhousingAreaはOUTFIT_TYPESに含まれないため、そのエリア補正値等は付与対象に追加しない。アウトフィットであるresidenceの住宅エリア参照属性とは区別する。

上記は機能を無条件に禁止する決定ではなく、通常の性能上書きとは別の操作仕様が必要という区分。

## 「上書き」「追加」の意味

| 値型 | 上書き | 追加 |
|---|---|---|
| 数値 | 指定値へ | 加算 |
| 単一選択 | 選択を置換 | 原則なし。小分類を複数扱いしたい場合は副分類へ追加 |
| 真偽 | オン／オフ | 原則なし |
| 文字列 | 文字列を置換 | 意図した追記。名称は既存の{}装飾も維持 |
| 集合 | 全体置換 | 要素の追加、重複ルールを定義 |
| 部位・副分類等の行配列 | 構造化データ全体を置換 | 安定IDまたは定義した一致条件で行追加 |

部位・分類をJSONベタ打ちだけにせず、対応する行編集UIをAEの値欄から開く案。値の型に合わないモードは提示しない。

## 実装上の要点

- 元データは保存したまま、適用中の実効属性を生成する。無効化・解除・失効で元へ戻る。
- 分類・部位・性能の実効値を、シート・準備候補・占有計算・判定・ダメージ・用途実行が共通して読む。
- 分類変更時の武器区分初期化など、通常編集時の永続更新フックをAE再計算で発火させない。AEで小分類だけを変えた時に武器区分も変えるかは既存の明示フラグ設計と照合して決める。
- 構造属性→性能の実効値→アクター集計という依存順を設計する。親子関係やカテゴリによる同期の循環・元データへの書き戻しを防ぐ。
- 同一項目への複数AEは既存priority・重複制御に合わせる。配列の追加／上書きの前後関係も試験する。
- 小分類変更、部位全体置換、副分類追加、効果解除のそれぞれで「表示だけ変わり動作が変わらない」状態がないことを実機検証する。
