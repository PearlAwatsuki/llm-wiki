# ActiveEffect 適用モデル（確定設計 v2）

フェーズ9 の中核。**標準の ActiveEffect 設定 UI（変更タブ＝キー＋モード＋値）だけで完結**させ、マクロや非存在フィールドに依存しない。2026-06-21 に effectMod 方式から**「total 直接適用」方式**へ再設計（ユーザー確定）。

ルール前提: [Character_Creation.md](../Game_Rules/Character_Creation.md)（0clamp）・[Damage_Rules.md](../Game_Rules/Damage_Rules.md)・[Time_Management.md](../Game_Rules/Time_Management.md)。フェーズ計画: [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md)。

---

## 0. 大原則

1. **base（UI 編集値）と total（実効値）を分離**。UI が編集するのは base（value / growth / mod 等）、表示・判定・ダメージが読むのは total。
2. **エフェクトは total に直接、ネイティブのモードで効く**（ADD / OVERRIDE / MULTIPLY / UPGRADE / DOWNGRADE）。effectMod のような加算専用の中間枠は持たない（OVERRIDE/MULTIPLY が表現できないため廃止）。
3. **対象・パラメータは「変更（changes）のキー」で表す**。標準 UI に存在するのはキー（テキスト）＋モード（選択）＋値（テキスト）のみ。これ以外のフィールドは前提にしない。
4. Foundry の Item は自己効果を適用せず、change.key は他アイテムへ解決できない。よって**アクターの単一適用パスが全 buff を集めて total へ適用**する（ネイティブ applyActiveEffects には依存しない）。

## 1. 注入点は2つ

| 注入点 | 対象 | タイミング |
|---|---|---|
| **恒常（適用パス）** | キャラ値・アイテム値 | prepareDerivedData で base→total を出した後、アクターの適用パスが total を改変 |
| **判定時** | 技能判定への加算 | 判定実行時にエフェクト単位で評価（非スタック） |

## 2. 変更キーの文法（標準 UI で打つ）

`<対象セレクタ>.<system パス>`。対象セレクタで「誰の」を、パスで「どのパラメータの total」を指す。

| セレクタ | 対象 | 例（キー） |
|---|---|---|
| `system.` | **そのエフェクトが乗る/転送されるドキュメント**（キャラ＝アクター値） | `system.reason` |
| `self.` | **効果が乗っているアイテム自身**（装備中の親アウトフィット等） | `self.attack` |
| `parent.` | **効果が乗るアイテムの親アウトフィット**（オプション→親） | `parent.attack` |
| `cat:<小分類キー>.` | その小分類のアイテム群 | `cat:melee.attack` |
| `<識別キー>.` | その識別キーを持つ技能（技能は一意） | `melee.judgment` |
| `<プレフィックス>*.` | 識別キーが前方一致する技能群 | `society_*.judgment` |

- パスは total を出すパラメータ。実装上 `attack` 等は内部で `attack.total` に解決する（base は不変）。
- 大分類は `cat:weapon.` のように同じ `cat:` で（小分類キーは全体一意なので主に小分類を使う）。
- 識別キー（技能）と `cat:`（アウトフィット）でアイテム種を住み分ける（アウトフィットは識別キーで個体指定しない方針＝Point 1）。

### 条件付き（標準キーに畳み込む）

`<セレクタ>[<条件>].<パス>`。角括弧内に `パス 比較 値`。複数条件は `;` 区切り。

- 例: `cat:melee[hack>=3].attack` = 電制 total が 3 以上の白兵武器の攻撃力。
- 比較演算: `>= <= > < == !=`。左辺は対象アイテムの system パス（`hack`→`hack.total` 等に解決）。
- 条件は angle/flag でなく**キー文字列内**で完結（標準 UI のみで書ける）。

### モード・値

- モード = 標準の Add / Override / Multiply / Upgrade / Downgrade。
- 値 = 数値（標準の値欄）。
- 複数効果は change.priority で順序づけ（Foundry 既定に準拠）。

## 3. 恒常適用パス（アクター prepareDerivedData）

```
1. prepareBaseData / 各アイテム prepareDerivedData: base→total を算出（total = base、エフェクト抜き）
   - アイテム: total = value（modeValue は mode=value 時のみ）/ defence は X_total = X_defence / 技能 judgmentMod=0 等
   - 能力値: total = growth + Σ(style×level) + mod（effectMod は廃止）
2. アクター prepareDerivedData の末尾＝適用パス:
   a. アクター自身＋全所有アイテムの effects から、有効な変更を集める
   b. 各変更のキーを parseEffectTargetKey で解析（セレクタ/条件/パス）
   c. 対象（キャラ＝アクター / self / parent / cat / 識別キー / prefix）を解決し、条件を満たす対象の
      `<パス>.total` に effect.apply でモード適用（priority 順）
3. 能力値・制御値は最後に 0clamp
```

- `effect.apply(targetDoc, {...change, key: "system.<path>.total"})` でモード処理（ADD/OVERRIDE/MULTIPLY）をネイティブに再利用。
- self/parent はそのエフェクトの親アイテム（と parentItemId）で解決。所持時のみ反映（単体プレビューは非対応）。

## 4. 判定時バフ（技能）

技能判定への加算は文脈依存（コンボ）かつ**非スタック**のため、恒常 total でなく**判定実行時にエフェクト単位**で評価する。

- 各エフェクトは「技能述語（識別キー完全一致 / プレフィックス / 将来: スタイル・ワークス）＋ボーナス」を持つ（キーは `melee.judgment` / `society_*.judgment` 等）。
- 判定フローで、参加技能（ベース＋コンボ）のうち**いずれか1つでも述語に該当すれば、そのエフェクトにつき1回だけ**達成値に加算。
- 単一技能バフ（〈白兵〉+1）は述語が1要素の同枠。
- 「バサラ/マヤカシに+1、組み合わせても+1」は同一エフェクトで複数該当 → 1回。別エフェクト同士は各1回（スタックする）。

## 5. 構造化の前提（別途・後続）

スタイル/ワークス述語（`byStyle:バサラ` 等）には、スタイル技能の「スタイル」欄・ワークス技能の「組織」欄・専用欄の**構造化（辞典読み取りドロップダウン・キー化）**が前提。ワークス装備フラグも要追加。これらは AE 本体の後の作業。当面は識別キー/プレフィックス/カテゴリで賄う。

分類（カテゴリ）の**コードキー化は実装済**（commit 2fc0685、`cat:melee` 等が使える）。

## 6. 実装計画（substrate 撤去＋適用パス）

1. **effectMod 撤去**: modeValueField/defenceField/attackField/skill level/FAValue/residence の effectMod 系フィールドを削除。能力値の式から effectMod 項を外す。
2. **base→total 算出**: `computeItemEffectiveValues` は total=base のみに簡素化。能力値 total も effectMod 抜きで算出。
3. **キー解析**: `parseEffectTargetKey(key)` 新設（セレクタ/条件/パス、テスト可能な純粋関数）。
4. **適用パス**: `CastDataModel._applyEffectBuffs`（旧 `_applyCrossTargetEffects` を一般化）。集約→解決→`effect.apply` で total へモード適用→0clamp。
5. **Item オーバーライド撤去**: `TokyoNovaItem.applyActiveEffects` 削除（self は適用パスで解決）。
6. **判定時バフ**: 判定フローでエフェクト単位の技能加算を収集（非スタック）。
7. **表示**: 既存の total 参照表示はそのまま使える（total の中身が base+buff になる）。

## 参照

- [Phase_9_Tasks_Detail.md](../Phases/Phase_9_Tasks_Detail.md) / [Judgment_System.md](Judgment_System.md)
- [Character_Creation.md](../Game_Rules/Character_Creation.md)（0clamp）
</content>
