# DESIGN_REVIEW

トーキョーN◎VA THE AXLERATION の実装について、技術的妥当性をレビューする記録。

ルール解釈の正しさは `MECHANICS_AUDIT.md` 側で扱う。本ドキュメントは「ルールが正しく
実装されている前提で、コードとして良い設計か」を見る。

## 運用ルール

- 新規実装時: その実装の範囲で1エントリ追加
- 既存実装に影響する変更時: 影響を受けるエントリのステータスを「リファクタ予定」等に更新
- フェーズ完了時: そのフェーズで増えた全エントリを通読してステータスを最新化

## エントリテンプレート

```
### [機能名 / モジュール名](フェーズX)

**日付**: YYYY-MM-DD
**レビュー対象**: `path/to/file.mjs` または `クラス名`
**ステータス**: レビュー中 | レビュー済(問題なし) | レビュー済(課題あり) | リファクタ予定

#### レビュー観点
- 保守性: コードが読みやすく、後から手を入れやすいか
- 拡張性: フェーズが進んだとき破綻しないか
- パフォーマンス: 実用上の問題はないか
- FVTT 慣行: Foundry VTT の標準的なパターンに準拠しているか
- 既存設計との整合: 既存の TnxActionHandler、Sheet 等と整合しているか

#### 良かった点
- (保守性・拡張性などの観点で評価できる点)

#### 課題
- ⚠️ 課題1: 内容と影響範囲
- ❌ 課題2: 内容と影響範囲

#### 推奨アクション
- 即時対応するか、フェーズXX で対応するか、対応しない(意図的に妥協)か
```

## エントリ一覧

### B-0 設計方針(DataModel 移行)(フェーズB)

**日付**: 2026-05-21
**レビュー対象**: フェーズB 全体の DataModel 実装方針
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: 長期戦(2〜3ヶ月)の DataModel 実装で破綻しないか
- 拡張性: フェーズ1 以降の判定システム実装で破綻しないか
- パフォーマンス: 22 type 分の DataModel 実装で大きな問題が出ないか
- FVTT 慣行: Foundry VTT v13 の推奨パターンに準拠しているか
- 既存設計との整合: 既存の TnxCastSheet、TnxActionHandler 等と整合しているか

#### 決定事項

##### 論点1: クラス命名規約

**決定**: 案C(`CastDataModel` / `StyleSkillDataModel` 等、プレフィックスなし)

検討した他の案:
- 案A: `TnxCastDataModel`(プロジェクトプレフィックス付き)
- 案B: `CastData`(短い)

決定理由: dnd5e の慣行に近く、データモデル単独でクラス名から型がわかる。

見直しの可能性: 既存の `Tnx` プレフィックスクラスとの命名一貫性で違和感が強くなる場合は
案A への切り替えを検討する。

##### 論点2: ファイル配置

**決定**: 案A(`scripts/data/` 配下に集約)

ディレクトリ構造:

```
scripts/
  data/
    abstract.mjs
    actor/
      common/      # biography, attributes, actor-base
      cast.mjs
      guest.mjs
      troop.mjs
      extra.mjs
      player.mjs
    item/
      common/      # base, usage, skill-base, outfit-base, extensible
      style.mjs
      style-skill.mjs
      ...
```

検討した他の案:
- 案B: `scripts/data-models/`(より明示的)
- 案C: 既存の `scripts/actor/` `scripts/item/` の隣に `*.data.mjs`

決定理由: dnd5e の構造を踏襲し、Actor / Item をディレクトリで分離。共通 template を
`common/` に閉じ込めることで依存関係を明示。

見直しの可能性: 実装してみてディレクトリが浅すぎる・深すぎると感じたら再構成する。

##### 論点3: 共通 template の継承戦略

**決定**: 案A(Mixin パターン)

検討した他の案:
- 案B: 単純な継承(1つの親クラスから派生)
- 案C: SchemaField の合成(継承なし)

決定理由: TNX は1つの type が複数の template を使う構造(cast = biography + attributes
+ actorBase など)。多重継承相当の機能が必要。dnd5e の `SystemDataModel.mixin()` パターン
を踏襲する。

参考実装イメージ:

```javascript
// scripts/data/abstract.mjs
export class SystemDataModel extends foundry.abstract.TypeDataModel {
  static mixin(...mixins) {
    const Class = class extends this {};
    for (const mixin of mixins) {
      // mixin の defineSchema、メソッドを Class に追加
    }
    return Class;
  }
}

// scripts/data/actor/cast.mjs
export class CastDataModel extends SystemDataModel.mixin(
  BiographyTemplate, AttributesTemplate, ActorBaseTemplate
) {
  static defineSchema() {
    const fields = foundry.data.fields;
    return {
      ...super.defineSchema(),
      // cast 固有のフィールド
    };
  }
}
```

見直しの可能性: Mixin の実装が複雑すぎる・型エラーが頻発する場合は案C(SchemaField
合成)に切り替える。

##### 論点4: defineSchema の記述スタイル

**決定**: 案A(インライン)+ 一部 案C(ヘルパー関数)

検討した他の案:
- 案B: フィールド定義を別ファイルに切り出してインポート

決定理由: 基本はインラインで可読性を優先。ただし `{ value, min, max }` のような
TNX 固有の繰り返しパターンはヘルパー関数(`scripts/data/helpers.mjs`)に集約する。

ヘルパー関数の例:

```javascript
// scripts/data/helpers.mjs
export function damageField() {
  const fields = foundry.data.fields;
  return new fields.SchemaField({
    value: new fields.NumberField({ initial: 0 }),
    min: new fields.NumberField({ initial: 0 }),
    max: new fields.NumberField({ initial: 0 }),
  });
}
```

見直しの可能性: ヘルパー関数が肥大化したらモジュール分割を検討する。

##### 論点5: 既存シートとの接続方式

**決定**: 案A(既存シートはそのまま、`actor.system.xxx` でアクセスする方式を維持)

検討した他の案:
- 案B: DataModel に getter / setter / メソッドを足して、シートロジックを DataModel に寄せる
- 案C: シートも一緒に書き換える

決定理由: フェーズB のスコープは「DataModel への移行」であってシート書き換えではない。
シートの構造的書き換えはフェーズ6(既存シートの ApplicationV2 完全移行)で扱う。
DataModel に必要最小限のメソッド(`prepareDerivedData` 等)を実装するのは OK だが、
シートのロジックを移すことはしない。

見直しの可能性: 既存シートが DataModel と整合しない箇所が出てきた場合、フェーズB の
範囲で最小限の調整を許容する(ただし大幅な書き換えはしない)。

#### 良かった点

- フェーズB のスコープが明確になった(DataModel 化のみ、シート書き換えは含まない)
- dnd5e という参照例があるため、実装方針に迷ったときに参照できる
- 命名規約・ファイル配置の決定により、Claude Code への実装プロンプトが簡潔になる

#### 課題

なし。実装着手前の方針確定として、必要な決定はすべて行えた。

#### 推奨アクション

B-1(Actor 共通 template の DataModel 化)に進む。実装中に方針見直しが必要と感じた
時点で、本エントリの「見直しの可能性」セクションを更新する。

---

### B-1 Actor 共通 template の DataModel 化(フェーズB)

**日付**: 2026-05-21
**レビュー対象**: `scripts/data/abstract.mjs`, `scripts/data/helpers.mjs`,
`scripts/data/actor/common/biography.mjs`, `scripts/data/actor/common/attributes.mjs`,
`scripts/data/actor/common/actor-base.mjs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: B-0 で確定した方針に沿った実装になっているか
- 拡張性: B-3 / B-4 で実 Actor type が mixin として利用できる形になっているか
- パフォーマンス: 22 type への展開に耐える設計か
- FVTT 慣行: foundry.abstract.TypeDataModel を正しく継承しているか
- 既存設計との整合: template.json の Actor 共通 template 定義に忠実か

#### 良かった点

- **Mixin パターンの実装**: `class extends this` で動的にクラスを生成する dnd5e 流派を採用。
  プロトタイプチェーンが綺麗で、デバッグ性が高い。
- **getter/setter 対応**: `Object.getOwnPropertyDescriptors` を使うことで、template の
  プロトタイプメソッドだけでなく getter/setter も正しく引き継げる。普通の `Object.assign`
  ではここが壊れがちなので、丁寧な実装。
- **defineSchema の空オブジェクト初期値**: `SystemDataModel.defineSchema()` が `{}` を返す
  ことで、mixin を使わない type も SystemDataModel を継承できる。設計の余地を残しつつ
  最小限。
- **テストの段階性**: 段階1 で Mixin パターン自体を 5 件のテストで検証してから、段階2 で
  template 実装に進んだ。心臓部の信頼性を先に確保する進め方は理に適っている。
- **`tests/setup.mjs` の静的インポート方式**: vitest.config.mjs の setupFiles を使わず、
  テストファイル先頭で `import "../../../setup.mjs"` するだけで済む構成。ESM の評価
  順序を正しく利用しており、設定の複雑化を避けている。
- **ダメージフィールドの max 値の扱い**: `damageField()` の汎用化を急がず、TNX 固有の
  max:21 を attributes.mjs 内に明示。「ダメージの max は型ごとの固定値」という TNX の
  ルール構造が読み取れる形を維持している。
- **ヘルパー関数の追加判断**: reason/passion/life/mundane の 14 フィールド構造の繰り返し
  に対して `attributeField()`、combatSpeed の 5 フィールド構造に対して
  `combatSpeedField()` を helpers.mjs に追加。重複削減と可読性向上を両立。
- **テストカバレッジ**: 計 88 件のテスト。defineSchema の構造、ネストフィールドのキー
  存在まで検証している。

#### 課題

なし。B-1 のスコープ内で品質を満たした実装。

#### 推奨アクション

B-2(Item 共通 template の DataModel 化)に進む。Item の共通 template は 5 種
(base / usage / skillBase / outfitBase / extensible)あり、Actor 共通 template の
パターンを踏襲する形で実装可能。

実装中に B-0 の方針見直しが必要と感じた時点で、B-0 エントリの「見直しの可能性」
セクションを更新する。

---

### B-2 Item 共通 template の DataModel 化(フェーズB)

**日付**: 2026-05-21
**レビュー対象**: `scripts/data/item/common/base.mjs`,
`scripts/data/item/common/usage.mjs`, `scripts/data/item/common/skill-base.mjs`,
`scripts/data/item/common/outfit-base.mjs`, `scripts/data/item/common/extensible.mjs`,
`tests/setup.mjs`(MockArrayField 追加)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: B-0 / B-1 の方針を踏襲しているか
- 拡張性: B-5 / B-6 / B-7 で実 Item type が mixin として利用できる形になっているか
- FVTT 慣行: foundry.data.fields の ArrayField / SchemaField を適切に使えているか
- 既存設計との整合: template.json と既存コード(usage-list.hbs 等)に忠実か

#### 良かった点

- **既存コードを情報源とした型推測**: actions の要素構造について、
  `tnx-item-sheet.mjs` の `_onActionCreate()` と `usage-list.hbs` を読んで
  `{type, name, description}` の StringField 3 つと判断。template.json だけでなく
  実装コードまで参照する姿勢は信頼できる。
- **ハイフン含みフィールド名への配慮**: "isPre-play" について、オブジェクトリテラルでは
  キー文字列として記述可能だがアクセス時は `system["isPre-play"]` 記法が必要、と
  JSDoc に明示。将来のメンテナーへの申し送り。
- **template.json への忠実性**: hide / exclusive 等のフィールドが template.json で
  `"-"` で初期化されていることを確認し、StringField で統一。憶測でなく現状を尊重。
- **MockArrayField の最小実装**: setup.mjs に追加する形で対応。element プロパティで
  ラップされた要素スキーマを保持する最小限の実装に留めた。
- **helpers 追加の保留判断**: uses の `{isLimit, max, value}` 構造は outfitBase の
  1 箇所のみのため、ヘルパー化を急がず inline で実装。将来 styleSkill (B-7) で
  同構造が出てくる可能性についても申し送り済み。
- **テストカバレッジ**: 計 138 件(B-1 から 50 件増加)。空配列がデフォルト値である
  ことの検証も含む。

#### 課題

- template.json のフィールド型の一部に、本来あるべき型(Boolean)と異なる型
  (String、`"-"` で初期化)が混在している可能性がある(例: exclusive)。
  これは template.json の設計上の問題であり、B-2 のスコープ
  (「template.json を忠実に再現する」)では対応外。`docs/KNOWN_ISSUES.md` に
  KI-016 として記録し、将来検討対象とする。

#### 推奨アクション

B-3(単純な Actor type: guest / troop / extra)に進む。Item 共通 template の
パターンが揃ったので、Actor type 実装は B-1 のパターンと B-2 で確立した
ArrayField モックを組み合わせる形で進められる。

---

### B-3 単純な Actor type の DataModel 化(フェーズB)

**日付**: 2026-05-21
**レビュー対象**: `scripts/data/actor/guest.mjs`, `scripts/data/actor/troop.mjs`,
`scripts/data/actor/extra.mjs`, `scripts/tnx.mjs`(CONFIG 登録追加)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: B-0 / B-1 の方針(Mixin パターン・プレフィックスなし命名・インライン defineSchema)に沿った実装になっているか
- 拡張性: B-4 の cast / player DataModel 化に向けて、今回のパターンが踏襲できる形か
- FVTT 慣行: `CONFIG.Actor.dataModels` への登録が `init` フックで正しく行われているか
- 既存設計との整合: template.json から削除した範囲が guest / troop / extra に限定されているか

#### 良かった点

- **mixin の合成順が B-1 のパターンを踏襲**: `SystemDataModel.mixin(...)` の引数順は
  template.json の `"templates"` 配列順(biography → attributes → actorBase)に揃えた。
  コードとデータ定義の対応が読み取りやすい。
- **固有フィールドがない場合も `defineSchema()` を明示的に定義**: `{ ...super.defineSchema() }`
  のみでも `defineSchema` をオーバーライドしている。「意図してフィールドを追加しなかった」
  ことが将来のメンテナーに伝わる形。
- **template.json の削除範囲が最小限**: `Actor.types` 配列・`Actor.templates` セクションは
  手を付けず、3 type のエントリのみ削除。二重管理の解消と既存依存(cast / player)の保護を
  両立している。
- **CONFIG 登録が `init` フック内で行われている**: `ready` フックではなく `init` で登録。
  Foundry が Actor インスタンス化より前に DataModel を解決できる正しいタイミング。
- **テストの非存在確認**: 「mixin されていない template のフィールドが含まれない」ことを
  troop(biography なし)・extra(attributes / actorBase なし)で検証。存在確認と非存在確認を
  組み合わせることで mixin の選択を保護している。
- **テストカバレッジ**: 計 201 件(B-2 から 63 件増加)。

#### 課題

なし。B-3 のスコープ内で品質を満たした実装。

#### 推奨アクション

B-4(複雑な Actor type: cast / player の DataModel 化)に進む。以下を申し送る。

- `Actor.templates` セクション(biography / attributes / actorBase)の template.json
  からの削除は、cast / player の DataModel 化が完了する B-4 で行う。
- cast / player の DataModel 化でも今回の「mixin + 固有フィールド追加」パターンが
  そのまま使える。cast は biography + attributes + actorBase の mixin に加え、
  `player_name` / `playerId` / `history` / `exp` / `lifePath` の固有フィールドを追加する形になる。
- B-4 完了後、`CONFIG.Actor.dataModels` に cast / player を追加で登録する。

---

### B-4 複雑な Actor type の DataModel 化(フェーズB)

**日付**: 2026-05-22
**レビュー対象**: `scripts/data/actor/cast.mjs`, `scripts/data/actor/player.mjs`,
`scripts/tnx.mjs`(CONFIG 登録追加), `template.json`(Actor セクション整理),
`tests/setup.mjs`(MockObjectField 追加)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: 既存の経験点同期ロジック(`updateCastExp` / `syncPlayerExpFromCasts`)を
  一切変更せず DataModel 化のみを行えているか
- 拡張性: 将来の `prepareDerivedData` 化への移行余地を残しているか
- FVTT 慣行: `ObjectField` の使い方が適切か
- 既存設計との整合: `history` の実際のデータ構造(オブジェクトマップ)に対して
  DataModel 定義が破壊的でないか
- template.json の整理範囲が正確か(`Actor.types` 配列への player 追加含む)

#### 良かった点

- **既存ロジックへの無介入**: `updateCastExp` / `syncPlayerExpFromCasts` /
  `TnxHistoryMixin` / `tnx.mjs` の updateActor フックをすべて無変更のまま DataModel
  化を完了した。経験点同期という複雑な非同期ロジックを壊さないための最小限の変更に留めた。
- **ObjectField による history の表現**: template.json では `{}` の空オブジェクトとして
  定義されていた `history` フィールドを `ObjectField` で表現。中身スキーマを定義しない
  ことで、`system.history.${randomId}` 形式での動的な追加・削除・マージを既存コードの
  変更なしにサポートした。
- **MockObjectField の最小実装**: setup.mjs への追加は `MockArrayField` と同パターン。
  テスト環境を最小限の変更で拡張できた。
- **cast / player の構造差異をテストで明示**: `exp.additional` の有無、`biography` /
  `attributes` の mixin 有無など、cast と player の差異を「非存在確認」テストで保護した。
- **template.json の完全整理**: B-4 完了により Actor セクションが `types` 配列のみとなり、
  template.json のデータ二重管理が完全に解消された。`player` の `Actor.types` 追加により
  `system.json` の `documentTypes.Actor` との整合性も確保した。
- **テストカバレッジ**: 計 258 件(B-3 から 57 件増加)。

#### 課題

- ⚠️ **player_name がデッドフィールド**: template.json 準拠で `StringField` として定義
  したが、スクリプト・テンプレートのどこからも参照されていない。将来シートを実装する際に
  使用するか削除するかを判断する必要がある。フェーズ6(シート書き換え)で判断予定。
- ⚠️ **lifePath が UI 未実装**: cast の `lifePath.origin` / `experience` / `encounter` は
  DataModel に定義したが、cast-sheet.hbs に表示 UI が存在しない。フェーズ7(未実装シート
  の追加)または将来の cast シート実装時に対応予定。
- ⚠️ **exp 派生計算が非同期フックに依存**: `updateCastExp` / `syncPlayerExpFromCasts` は
  `async` で `Actor.update()` を呼ぶ設計。本来 DataModel の `prepareDerivedData` で
  同期的に計算すべき値だが、非同期処理への移行は大規模な設計変更を伴うため B-4 のスコープ外
  とした。将来フェーズ(B-8 以降の検証フェーズまたは独立したリファクタリングフェーズ)で
  検討する余地がある。

#### 推奨アクション

B-5(単純な Item type の DataModel 化)に進む。以下を申し送る。

- Actor 側の DataModel 化はすべて完了した。`CONFIG.Actor.dataModels` に全5 type が登録済み。
- B-5 / B-6 / B-7 で Item type の DataModel 化を進める際、`ObjectField` は今回の実装例
  (cast / player の `history`)を参照できる。
- `player_name` / `lifePath` のデッドフィールド / UI 未実装問題はフェーズ6 / 7 で対応。

---

### B-5a 単純な Item type の DataModel 化(base のみ 3種)(フェーズB)

**日付**: 2026-05-22
**レビュー対象**: `scripts/data/item/housing-area.mjs`, `scripts/data/item/organization.mjs`,
`scripts/data/item/life-path.mjs`, `scripts/tnx.mjs`(CONFIG.Item.dataModels 登録追加),
`template.json`(housingArea / organization / lifePath エントリ削除)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: B-3 / B-4 の Actor 実装パターンを踏襲しているか
- 拡張性: B-5b / B-5c での mixin 追加に向けて、今回のパターンが流用できる形か
- FVTT 慣行: `CONFIG.Item.dataModels` への登録が `init` フックで正しく行われているか
- 既存設計との整合: template.json から削除した範囲が対象 3 type に限定されているか

#### 良かった点

- **Actor 側パターンの踏襲**: `SystemDataModel.mixin(BaseTemplate)` の記法が
  `SystemDataModel.mixin(BiographyTemplate, ...)` と完全に対称で、Actor / Item 間の
  実装一貫性を保てている。
- **CONFIG.Item.dataModels の初回登録**: B-4 まで Actor 側のみだった `CONFIG.*DataModels`
  登録に Item 側が加わった。`init` フックの正しいタイミングで登録されており、
  Actor 側と同じブロックの直後に配置して視認性を確保している。
- **template.json の整理範囲が最小限**: `Item.types` 配列・`Item.templates` セクション
  は保持し、3 type のエントリのみ削除。B-5 全体完了まで二重管理を最小化しつつ進める
  方針と整合している。
- **lifePathType の型判断を記録**: `@fileoverview` に「実質的に enum だが StringField
  のままとした理由と将来の対応」を明記。設計上の妥協点を将来のメンテナーに伝えている。
- **Cast Actor との混同防止**: `life-path.mjs` の `@fileoverview` に Cast Actor 側の
  `lifePath`(origin / experience / encounter)とは別物であることを明記した。
  2種の同名概念が共存する状況でのドキュメント対応として適切。
- **テストカバレッジ**: 計 283 件(B-4 から 25 件増加)。初期値の一致確認と
  非存在確認(他 type のフィールドが混入していないこと)を含む。

#### 課題

- ⚠️ **lifePathType が実質 enum**: 現状は `StringField` で定義。将来のシート実装時に
  選択肢付きフィールドに変更する必要が生じる可能性がある。`life-path.mjs` の
  `@fileoverview` に申し送り済み。

#### 推奨アクション

B-5b(base + outfitBase の 4種: armor / cyborg / combiner / general)に進む。
以下を申し送る。

- armor / cyborg は `defence: { S_defence, P_defence, I_defence }` 構造を共有する。
  B-5b では `defenceField()` ヘルパーを `helpers.mjs` に追加して両者で流用することを推奨。
- cyborg は `attack: { damageType: [], value, mod }` も持つ。B-6 の weapon でも同構造が
  出るため、`attackField()` ヘルパーの追加も B-5b のタイミングで検討する。
- `CONFIG.Item.dataModels` への B-5b 対象の追加は、既存の B-5a 登録ブロックに追記する形で進める。

---

### B-5b 単純な Item type の DataModel 化(base + outfitBase 4種)(フェーズB)

**日付**: 2026-05-22
**レビュー対象**: `scripts/data/item/helpers.mjs`(新設),
`scripts/data/item/armor.mjs`, `scripts/data/item/cyborg.mjs`,
`scripts/data/item/combiner.mjs`, `scripts/data/item/general.mjs`,
`scripts/tnx.mjs`(CONFIG.Item.dataModels 追記),
`template.json`(armor / cyborg / combiner / general エントリ削除)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: B-5a のパターンを踏襲しつつ、OutfitBaseTemplate mixin が正しく合成されているか
- 拡張性: defenceField() の切り出しタイミングが適切か / attack の inline 判断が妥当か
- FVTT 慣行: ArrayField(StringField) の使い方が正しいか
- 既存設計との整合: template.json から削除した範囲が対象 4 type に限定されているか

#### 良かった点

- **defenceField() の切り出しタイミング**: armor / cyborg の 2 箇所で同構造が出た時点で
  `scripts/data/item/helpers.mjs` として切り出した。B-2 の前例(「再利用が必要になった
  時点で切り出す」)に正確に倣っている。
- **attack の inline 判断**: B-5b 時点では cyborg のみで使用されるため inline に留め、
  `@fileoverview` に「B-7 weapon 着手時に attackField() ヘルパー化を再検討」と申し送り。
  過剰な先回り設計を避けつつ将来の対応点を明記した。
- **Item 専用 helpers.mjs の分離**: `scripts/data/helpers.mjs`(Actor 側)と
  `scripts/data/item/helpers.mjs`(Item 側)を別ファイルに分離。
  責務の分離が明確で混在を防いでいる。
- **combiner.combinedOutfitID の型**: Item ID 文字列の配列として `ArrayField(StringField)`
  で定義。template.json が `[]` のみの定義で要素型が明示されていない状況で、
  ID 文字列という用途から型を推測した判断として適切。
- **general の固有フィールドなし**: B-3 の GuestDataModel・B-5a の OrganizationDataModel
  と同パターン。`{ ...super.defineSchema() }` のみのオーバーライドで「意図してフィールドを
  追加しなかった」ことを示す。
- **テストカバレッジ**: 計 324 件(B-5a から 41 件増加)。defenceField() ヘルパー単体の
  5 件、各 DataModel の mixin 確認と非存在確認を含む。

#### 課題

- ⚠️ **attack フィールドは B-7 までヘルパー化されない**: weapon(B-7)でも同構造が
  出るため、B-7 着手時に `attackField()` ヘルパーを `scripts/data/item/helpers.mjs`
  に追加して cyborg / weapon 共用にすることを推奨。申し送り済み。

#### 推奨アクション

B-5c(base + outfitBase + extensible の 3種: ianus / tron / vehicle)に進む。
以下を申し送る。

- ExtensibleTemplate の mixin は B-5b の OutfitBaseTemplate 追加パターンをそのまま拡張できる。
- ianus は `controlMod` フィールド(NumberField)のみの固有フィールドを持つ。
- vehicle は `speedFactor` / `passenger` / `controlMod` の 3 フィールドを持つ。
- tron は固有フィールドなし(B-3 の guest / B-5a の organization / B-5b の general と同パターン)。
- `CONFIG.Item.dataModels` への B-5c 対象の追加は、既存の B-5a / B-5b 登録ブロックに追記する形で進める。

---

### B-5c 単純な Item type の DataModel 化(base + outfitBase + extensible 3種)(フェーズB)

**日付**: 2026-05-22
**レビュー対象**: `scripts/data/item/ianus.mjs`, `scripts/data/item/tron.mjs`,
`scripts/data/item/vehicle.mjs`, `scripts/tnx.mjs`(CONFIG.Item.dataModels 追記),
`template.json`(ianus / tron / vehicle エントリ削除)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: B-5a / B-5b の mixin パターンに第 3 テンプレート(ExtensibleTemplate)を
  追加した形で一貫しているか
- 拡張性: `controlMod` のヘルパー化見送りが妥当か
- FVTT 慣行: 3 テンプレート合成の mixin 記述が正しいか
- 既存設計との整合: template.json から削除した範囲が対象 3 type に限定されているか

#### 良かった点

- **3 テンプレート mixin の記述一貫性**: B-5b の `mixin(Base, Outfit)` に
  `ExtensibleTemplate` を 3 番目に追加した形。テンプレート順序が
  template.json の `"templates"` 配列順に対応しており読みやすい。
- **controlMod のヘルパー化見送り**: ianus / vehicle / armor(B-5b)の 3 箇所に存在するが、
  単一 `NumberField({ initial: 0 })` のみであり関数化してもコード量が変わらないと判断。
  「ヘルパー化すべき」という判断基準(SchemaField / 再利用頻度 / 保守上の意義)を
  明文化したうえで見送った。
- **tron の固有フィールドなし**: guest / organization / general と同じ「意図してフィールドを
  追加しなかった」パターン。B-5 全体を通して一貫。

#### 課題

なし。B-5c のスコープ内で品質を満たした実装。

#### B-5 全体完了の総括

**日付**: 2026-05-22
**完了した 10 type**:
- B-5a: housingArea / organization / lifePath(base のみ)
- B-5b: armor / cyborg / combiner / general(base + outfitBase)
- B-5c: ianus / tron / vehicle(base + outfitBase + extensible)

**新設ファイル**:
- `scripts/data/item/helpers.mjs`(defenceField())
- `scripts/data/item/{housing-area, organization, life-path, armor, cyborg, combiner, general, ianus, tron, vehicle}.mjs`

**テスト**: B-5 全体で 89 件追加(258 → 347 件)

**`CONFIG.Item.dataModels`**: init フックで 10 type 全登録済み

**申し送り事項**:
- `attackField()` ヘルパー化は B-7 weapon 着手時に再検討
- `lifePathType` の選択肢付き型へのリファクタは将来サブフェーズに送り
- `Item.types` 配列・`Item.templates` セクションは B-7 完了時に整理

#### 推奨アクション

B-6a(中程度の Item type の DataModel 化: outfit 系 3種)に進む。
B-6a の対象 type: weapon / tap / residence。
KI-007(`tap.conbatSpeedMod` タイポ)は B-6a で正しい綴り `combatSpeedMod` で定義することで解消する。
データマイグレーション機構は導入しない(実運用前のため不要。詳細は B-6a エントリ参照)。

---

### B-6a 中程度の Item type の DataModel 化(outfit 系 3種)(フェーズB)

**日付**: 2026-05-23
**レビュー対象**: `scripts/data/item/helpers.mjs`(attackField() 追加),
`scripts/data/item/cyborg.mjs`(attackField() 適用に変更),
`scripts/data/item/weapon.mjs`, `scripts/data/item/tap.mjs`,
`scripts/data/item/residence.mjs`,
`scripts/tnx.mjs`(CONFIG.Item.dataModels 追記),
`template.json`(weapon / tap / residence エントリ削除)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: B-5c の 3 テンプレート mixin パターンを踏襲しているか
- 拡張性: attackField() の切り出しタイミングが適切か
- FVTT 慣行: BooleanField の使い方(weapon の boolean 群)が正しいか
- KI-007 対応: タイポ修正をマイグレーションなしで解消した設計判断の妥当性
- マイグレーション不採用の設計判断: CLAUDE.md §4.2 の方針との整合性

#### 良かった点

- **attackField() の切り出しタイミング**: B-5b の申し送り(「weapon 着手時に再検討」)に
  正確に応答した。cyborg / weapon の 2 箇所で同構造が出た時点でヘルパー化。
  B-5b での先送り → B-6a での実施という一貫した判断軸。
- **cyborg.mjs のリファクタが意味的に無変更**: `attack: attackField()` に置き換えた後も
  生成スキーマは不変。既存の cyborg テストがグリーンのままであることで担保されている。
  回帰リスクのないリファクタとして適切に実施。
- **KI-007 のマイグレーションなし解消**: 実運用前で実データが存在しない状況で、
  DataModel 定義を正しい綴りで書くだけでタイポを解消した。CLAUDE.md §4.2「将来のために
  拡張ポイントを先回りで作らない」の精神に合致する判断。
- **tap の fileoverview に判断根拠を明記**: `combatSpeedMod` を正しい綴りで定義した理由と、
  Actor 側 `combatSpeedField()`(5 フィールド構造)との別物である旨を明記。
  同名の紛らわしい概念が複数存在する状況でのドキュメント対応として適切。
- **residence の *Mod フィールドをヘルパー化しない**: housingArea との 2 箇所共有だが、
  単一 `NumberField({ initial: 0 })` であり関数化のメリットが薄い。B-5c の
  controlMod と同じ判断基準を一貫して適用している。
- **テストカバレッジ**: 計 396 件(B-5 全体 347 件から 49 件増加)。
  attackField() 単体テスト 7 件、KI-007 タイポ確認(conbatSpeedMod が存在しないこと)、
  weapon の Boolean フィールド 4 件の型確認を含む。

#### 課題

なし。B-6a のスコープ内で品質を満たした実装。

#### マイグレーション機構不採用の設計判断記録

B-6 は当初「中程度の Item type + データマイグレーション機構の導入」と定義されていたが、
以下の理由でマイグレーション機構の導入を見送った。

1. **実運用前**: 移行対象の実データが存在しない。マイグレーションを実行しても変換すべきデータが
   ない状態。
2. **CLAUDE.md §4.2**: 「将来のために拡張ポイントを先回りで作らない。必要になったときに追加する」
   という明示的な方針に反する。
3. **KI-007 の性質**: 参照コードのない無害なタイポであり、機構なしで「正しい綴りで定義する」だけで
   解消できる。マイグレーションが本来必要な「既存データの変換」は不要。
4. **本当に必要なタイミング**: 実データがある状態で破壊的変更を行うとき(フェーズ7 以降または公開
   準備フェーズ)。その時点で改めて設計する方が、実際の要件に合った実装になる。

#### 推奨アクション

B-6b(skill 系 2種: miracle / generalSkill の DataModel 化)に進む。
以下を申し送る。

- miracle / generalSkill は `base + skillBase + usage`(または usage なし)の mixin 構成になる見込み。
  usage テンプレートは「判定システムの基盤」(CLAUDE.md §4.3)のため、DataModel 化の範囲と
  既存の `usage-list.hbs` / `UsageCreationDialog` への影響を慎重に確認してから実装すること。
- `Item.types` 配列・`Item.templates` セクションは B-7 完了時に整理する方針を維持。
- `attackField()` は B-6a で weapon / cyborg 共用として確立済み。

---

### B-6b 中程度の Item type の DataModel 化(skill 系 2種)(フェーズB)

**日付**: 2026-05-23
**レビュー対象**: `scripts/data/item/miracle.mjs`, `scripts/data/item/general-skill.mjs`,
`scripts/tnx.mjs`(CONFIG.Item.dataModels 追記),
`template.json`(miracle / generalSkill エントリ削除)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: usage / skillBase テンプレートを初めて実 type に mixin した際のパターン一貫性
- 拡張性: usageCount / initialSkill 等のヘルパー化見送りが妥当か
- 既存設計との整合: 既存シートロジック(general-skill-sheet の連動更新)に触れていないか
- usage テンプレートの不変性: usage.mjs / usage-list.hbs / UsageCreationDialog を
  一切変更していないか

#### 良かった点

- **usage / skillBase の初適用**: B-2 で DataModel 化した UsageTemplate / SkillBaseTemplate
  が、本サブフェーズで初めて実 Item type の mixin として機能した。テンプレート設計の
  正しさがここで初めて実証される形となった。
- **mixin 合成順が templates 配列順に対応**: miracle は `mixin(Base, Usage)`、
  generalSkill は `mixin(Base, Usage, SkillBase)` と template.json の配列順に忠実。
  B-5 以降一貫している規約を正確に踏襲。
- **usageCount の初期値が 1**: template.json に忠実に `value: 1 / total: 1` とした。
  既存フック(preUpdateItem / preDeleteItem)が usageCount を参照しているため、
  初期値ミス(0 にするなど)は実機で重大バグになりえる。テストで明示的に `toBe(1)` を
  検証し、将来の誤修正を防いでいる。
- **既存シートロジックへの無介入**: general-skill-sheet の initialSuit ↔ suits ↔ level
  連動更新は DataModel 化の対象外として明確にスコープ外に置いた。`@fileoverview` に
  その旨を明記。
- **generalSkillCategory を StringField のままとした判断**: B-5a の lifePathType と同じ
  判断基準(実質 enum だが、将来の選択肢付き型リファクタまで StringField を維持)を
  一貫して適用。`@fileoverview` に申し送り済み。
- **ヘルパー化しない判断**: usageCount / initialSkill / onomasticSkill はいずれも当該
  type 固有の 1 箇所のみ。B-5c の controlMod と同じ判断で inline SchemaField に留めた。
  過剰な汎化を避ける一貫した姿勢。
- **テストカバレッジ**: 計 443 件(B-6a の 396 件から 37 件増加)。
  miracle の usageCount.value / total が 1 であることの明示的な検証、
  miracle に skillBase フィールドが含まれないことの非存在確認を含む。

#### 課題

- ⚠️ **generalSkillCategory が実質 enum**: B-5a の lifePathType と同じ状況。
  将来のシート実装時に選択肢付き型へのリファクタが必要になる可能性がある。
  `general-skill.mjs` の `@fileoverview` に申し送り済み。

#### B-6 全体完了の総括

**日付**: 2026-05-23
**完了した 5 type**:
- B-6a: weapon / tap / residence(base + outfitBase + extensible)
- B-6b: miracle / generalSkill(base + usage / base + usage + skillBase)

**新設ファイル**:
- `scripts/data/item/{weapon, tap, residence, miracle, general-skill}.mjs`
- `tests/template-integrity.test.mjs`(B-6 補助タスク: template.json 恒久健全性テスト)

**テスト**: B-6 全体で 96 件追加(347 → 443 件)

**`CONFIG.Item.dataModels`**: init フックで 15 type 登録済み

**設計判断**:
- マイグレーション機構は実運用前のため導入しない(CLAUDE.md §4.2 に準拠)
- KI-007(tap タイポ)は正しい綴りで定義することでマイグレーションなしで解消
- template.json の健全性(JSON 妥当性 + 二重定義なし)を恒久テストで CI 担保

**申し送り事項**:
- `Item.types` 配列・`Item.templates` セクションは B-7 完了時に整理
- generalSkillCategory / lifePathType の選択肢付き型リファクタは将来フェーズに送り
- B-7 対象: style / styleSkill(template.json に残る最後の 2 type 別エントリ)

#### 推奨アクション

B-7a(style の DataModel 化)に進む。以下を申し送る。

- style / styleSkill は Items の中で最もフィールド数が多く設計が複雑。
  特に styleSkill は confrontation / comboSkill / timing 等の配列フィールドと
  SchemaField のネストが複雑。事前に `tnx-style-sheet.mjs` / `tnx-style-skill-sheet.mjs`
  の getData() を読んで、どのフィールドがシートで使われているかを把握してから着手すること。
- B-7 完了時に `Item.templates` セクションと `Item.types` 配列の整理を行う。

---

### B-7a 複雑な Item type の DataModel 化(style)(フェーズB)

**日付**: 2026-05-23
**レビュー対象**: `scripts/data/item/style.mjs`,
`scripts/tnx.mjs`(CONFIG.Item.dataModels 追記),
`template.json`(style エントリ削除)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: base のみの mixin で素直に実装できているか
- 設計の正確性: reason 等が Actor の attributeField()(14 フィールド)と別物であることを
  認識して正しく実装されているか
- 既存ロジックへの無介入: preUpdateItem/preDeleteItem フック・style-sheet に触れていないか
- ローカルヘルパーの適切性: abilityField() を style.mjs 内に閉じた判断が妥当か

#### 良かった点

- **ローカルヘルパー関数 abilityField() の採用**: `{value, control}` の 2 フィールド
  SchemaField が reason/passion/life/mundane の 4 箇所で繰り返されるため、style.mjs 内に
  ローカル関数として切り出した。DRY の観点と「複数 type 共用にするほどではない(style 専用)」
  という判断を両立している。
- **Actor の attributeField() と明確に区別**: style の reason 等(2 フィールド)が
  Actor の reason(14 フィールド、`attributeField()`)と別物であることを `@fileoverview`
  に明記し、混同リスクを文書で排除した。
- **level の初期値が 1**: template.json に忠実。miracle.usageCount.value と同様、
  初期値ミス(0 にするなど)が実機で重大バグになりえる箇所を正確に実装し、
  テストで `toBe(1)` を明示的に検証している。
- **level の範囲制約を入れない判断**: 既存の preUpdateItem フックと cast-sheet が
  レベル制御を担っているため DataModel 側で min/max を設定しない。過剰な制約追加を避け、
  既存ロジックへの依存を変えない。`@fileoverview` に判断根拠を記録済み。
- **既存ロジックへの無介入**: preUpdateItem(style レベル変更時の miracle usageCount 増減
  および isPersona/isKey 強制設定)/ preDeleteItem(miracle 連動削除)/ cast-sheet の
  排他制御すべてを変更しなかった。B-7a のスコープを DataModel 化に限定する方針を徹底。
- **テストの ability ループ**: 4 つの能力値フィールドを `for...of` ループで検証。
  フィールドが増えた場合も一箇所の修正で対応できる形。

#### 課題

なし。B-7a のスコープ内で品質を満たした実装。

#### 推奨アクション

B-7b(styleSkill の DataModel 化)に進む。以下を申し送る。

- styleSkill は template.json 定義と実際のシート動作(getData)の間で乖離が確認されている
  (`tnx-style-skill-sheet.mjs` の getData を要事前確認)。
- confrontation / comboSkill / timing は `["blank"]` 初期値の配列フィールド。要素型の
  判断が必要(StringField か SchemaField か)。
- 4 種類の SchemaField ネスト(special / performance / secret / mystery)の expCost 等の
  初期値を正確に確認すること。
- B-7b 完了時に `Item.templates` セクションと `Item.types` 配列の整理を行う。

---

### B-7b 複雑な Item type の DataModel 化(styleSkill)+ Item 完了(フェーズB)

**日付**: 2026-05-23
**レビュー対象**: `scripts/data/item/style-skill.mjs`,
`scripts/tnx.mjs`(CONFIG.Item.dataModels 全 17 type 登録完了),
`template.json`(Item セクションを空オブジェクト {} に整理),
`tests/template-integrity.test.mjs`(フェーズB 完了形状に対応)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 設計判断の妥当性: 配列要素の型定義方針(SchemaField vs StringField)
- ★ フィールドの網羅: template.json 未定義だがシートが参照する 8 フィールドの明示定義
- mixin 合成の正確性: base + usage + skillBase の順序、skillBase 由来の重複定義なし
- unique の初期値修正: template.json の "" ではなく "none" が正しいことの確認
- タイポ・命名揺れの維持判断: RewritedTarget / RewritingMiracle_ID
- template.json Item セクション整理の適切性

#### 設計判断の汎用記録

**template.json の配列定義は要素スキーマを表現できない**。`template.json` では配列フィールドは
`["blank"]` のような「配列であること」しか表現できない形式で定義される。これは template.json の
制約によるもので、DataModel 化では追従すべき仕様ではない。**DataModel 化においては、シート
実装(getData() の ensureArray 正規化構造・_onSelectChange の保存データ)を真実の源として、
要素の SchemaField を確定する**。この方針は B-7b の styleSkill で初めて適用されたが、
将来の DataModel 化においても同様の状況では同じアプローチをとる。

#### 良かった点

- **★ フィールドの完全網羅**: template.json に未定義だがシートの `_onSelectChange` が
  リセット先として参照する 8 フィールド(maxLevelNumber / maxLevelOther / targetOther /
  rangeOther / targetValueNumber / targetValueOther / comboSkillOther / confrontationOther)
  をすべて明示定義した。未定義のままにすると `_onSelectChange` でバリデーションエラーになる。
- **配列要素を実態の SchemaField で定義**: comboSkill / confrontation / timing について、
  template.json の `["blank"]` 近似ではなく、getData() の正規化後の要素構造
  `{value, name, isMandatory}` / `{value, name}` / `{value, actionName, processName, timingOther}`
  を SchemaField で定義した。既存シートの `ensureArray` は引き続きフォールバックとして機能する。
- **unique の initial 修正**: template.json に `""` と定義されていたが、シートの選択肢定義
  (`TnxSkillUtils.getSkillOptions`) における先頭値が `"none"` であることから、正しい
  デフォルト値は `"none"` と判断。template.json の誤りを修正した。
- **タイポ・命名揺れの意識的な維持**: `RewritedTarget`(→ RewrittenTarget が正)および
  `RewritingMiracle_ID`(→ rewritingMiracleId が正)はタイポ・命名規約違反だが、
  シート・ロジックが同じ名前で参照しているため本フェーズではリネームせずそのまま維持。
  KI-018 / KI-019 として記録した。将来のシート全体整理フェーズで対応。
- **uses の inline 定義**: outfitBase の uses SchemaField と同型だが styleSkill 固有の
  別フィールド。OutfitBaseTemplate を共用せず inline 定義にとどめた。過剰な共用化を避けた。
- **template.json Item セクションの整理**: styleSkill(最後の type エントリ)削除と同時に
  `Item.templates` セクション・`Item.types` 配列も削除し `Item: {}` に整理。v13 では
  type の権威は system.json の documentTypes であり、両者が揃うことで二重管理が解消。
- **template.json ファイル自体の維持**: htmlFields 等のサニタイズ宣言が documentTypes に
  マージされる経路の検証を B-8 に委ねるため、ファイルは削除しなかった。安全な選択。
- **integrity テストの意図保持**: "Item.types が配列として存在する" テストを
  "フェーズB 完了後: Item セクションは空オブジェクト" に更新し、
  「JSON 妥当性 + DataModel と template.json の二重定義禁止」という本来の意図を維持。

#### B-7 全体総括(B-7a + B-7b)

| サブフェーズ | 対象 | mixin | 特記事項 |
|---|---|---|---|
| B-7a | style | base | ローカル abilityField() / level initial=1 |
| B-7b | styleSkill | base+usage+skillBase | ★8フィールド / 配列SchemaField / unique修正 |

#### フェーズB Item DataModel 化 完了総括

B-5〜B-7 の全サブフェーズで Item 全 17 type の DataModel 化が完了した。

| B-5 対象(10 type) | B-6 対象(4 type) | B-7 対象(3 type) |
|---|---|---|
| armor / ianus / cyborg / tron / vehicle | weapon / tap / residence / miracle / generalSkill | style / styleSkill |
| housingArea / combiner / general / organization / lifePath | (generalSkill は B-6b) | |

- `CONFIG.Item.dataModels` に全 17 type 登録完了
- `template.json` の Item セクションは空オブジェクト `{}` に整理
- `template.json` ファイル自体は B-8 での htmlFields 検証後に廃止予定
- 全テスト 530 件グリーン / ESLint 0 errors

#### 推奨アクション(B-8 申し送り)

1. **template.json ファイルの廃止**: Actor セクション(`types` 配列)の削除を含む。
   検証事項: `htmlFields` / `gmOnlyFields` 等のサニタイズ宣言が `system.json` の
   `documentTypes` 側に存在するか(または DataModel の `HTMLField` 等で代替されているか)。
   `description` の ProseMirror サニタイズがサニタイズ宣言を失わないことを確認してから廃止。
2. **v13 実機確認**: template.json 削除後も全 type が正常認識されること。
3. **既存機能の検証**: 既存シート・EXP 計算・カード判定・神業 usageCount 等が
   DataModel 移行後も正常動作することを確認。

---

### 3-0 ApplicationV2 全面移行 設計確定(フェーズ3)

**日付**: 2026-06-06
**レビュー対象**: フェーズ3 全体の ApplicationV2 移行方針
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: V2 移行後のコードが読みやすく、フェーズ4 以降の機能追加に耐えるか
- 拡張性: EffectsSheetMixin の統一が KI-009 を根本解消するか
- FVTT 慣行: v13 で提供される V2 API を正しく使えているか
- 既存設計との整合: TnxRecordSheet(既存 V2 参照実装)のパターンと一致するか

#### 確認した API(v13)

`foundry.applications.api` モジュールに含まれるクラス:
- `ApplicationV2` — 基底
- `DocumentSheetV2` — ドキュメントシート基底
- `DialogV2` — ダイアログ
- `HandlebarsApplicationMixin` — Handlebars テンプレート対応 Mixin

`foundry.applications.sheets` モジュールに含まれるクラス:
- `ActorSheetV2` — DocumentSheetV2 サブクラス。DnD・ポートレート対応組み込み
- `ItemSheetV2` — DocumentSheetV2 サブクラス。`item` / `actor` アクセサ組み込み

`foundry.applications.sheets.journal` モジュールに含まれるクラス:
- `JournalEntrySheet` — `HandlebarsApplicationMixin(DocumentSheetV2)` 継承。ページ管理専用

`TabsApplicationMixin` は v13 に**存在しない**。タブは V2 基底クラスに組み込みのサポートがある。

#### 決定事項

##### 論点1: 各シートの V2 基底クラス

**決定**: 下表の通り確定。すべて `HandlebarsApplicationMixin(...)` でラップする。

| V1 クラス | V2 基底 | 名前空間 |
|---|---|---|
| `ActorSheet` | `ActorSheetV2` | `foundry.applications.sheets` |
| `ItemSheet` | `ItemSheetV2` | `foundry.applications.sheets` |
| `JournalSheet` | `DocumentSheetV2` | `foundry.applications.api` |
| `Application` | `ApplicationV2` | `foundry.applications.api` |
| `FormApplication` | `ApplicationV2` | `foundry.applications.api` |

`Dialog` は `DialogV2` (`foundry.applications.api`) に移行する。

`JournalSheet` の V2 相当である `JournalEntrySheet` はページ管理に特化しており、
`TnxScenarioSheet` の独自 UI と相容れないため、直接 `DocumentSheetV2` を継承する。
アクトシートとしての登録は `DocumentSheetConfig.registerSheet(JournalEntry, ...)` で行う。

決定理由: `ActorSheetV2` / `ItemSheetV2` はそれぞれ Actor/Item 固有の DnD・アクセサが
組み込まれており、直接使うことで手動実装を省ける。`JournalEntrySheet` は逆に機能過多のため回避。

##### 論点2: V1 → V2 主要 API 対応

**決定**: 下表の変換を全シートに適用する。

| V1 | V2 |
|---|---|
| `static get defaultOptions()` | `static DEFAULT_OPTIONS = {...}` |
| `getData(options)` | `async _prepareContext(options)` |
| `activateListeners(html)` | `static DEFAULT_OPTIONS.actions` + `_onRender()` |
| `options.tabs` | V2 組み込みの `static TABS` + `tabGroups` プロパティ |
| jQuery `.find().on(...)` | ネイティブ DOM `addEventListener` |
| `render(true)` | `render({ force: true })` |
| `_getHeaderButtons()` | `static DEFAULT_OPTIONS` の `window.controls` |

##### 論点3: タブ実装方法

**決定**: V2 基底クラス組み込みの `TABS` + `tabGroups` を使う。

`TabsApplicationMixin` が v13 に存在しないため、V2 基底が提供する以下の仕組みを使う:
- `static TABS = { primary: { tabs: [...], initial: 'abilities' } }` でタブグループを定義
- `tabGroups = { primary: 'abilities' }` でアクティブタブの初期値を指定
- `changeTab(tabName, groupName)` でタブ切替
- HTML 側では `data-tab-group` / `data-tab` 属性を使う

既存テンプレートのタブ構造はそのまま維持し、クラス名・属性のみ V2 形式に調整する。

##### 論点4: EffectsSheetMixin の V2 統一(KI-009 解消)

**決定**: `activateEffectListListeners` を削除し、ハンドラを V2 `actions` 形式に統一する。

**現状の問題 (KI-009)**:
- `CastSheet`: インスタンスプロパティ代入 (`_onXxx = EffectsSheetMixin._onXxx`)
- `ItemSheet`: 静的メソッド呼び出し (`EffectsSheetMixin.activateEffectListListeners(...)`)

**V2 統一後の構造**:
```js
export const EffectsSheetMixin = {
    // _prepareContext から呼ぶ (変更なし)
    prepareEffectsContext(document, context) { ... },

    // DEFAULT_OPTIONS.actions に展開する関数群
    ACTIONS: {
        createEffect: async function(event, target) { ... },  // this = sheet instance
        editEffect:   async function(event, target) { ... },
        deleteEffect: async function(event, target) { ... },
        toggleEffect: async function(event, target) { ... },
    }
};
```

各シートの `DEFAULT_OPTIONS`:
```js
static DEFAULT_OPTIONS = {
    actions: {
        ...EffectsSheetMixin.ACTIONS,
        // シート固有の actions
    }
};
```

HTML 側の `data-action` 属性値を対応する名前(`createEffect` 等)に揃える。

この統一により KI-009 を完全解消する。

##### 論点5: TnxHud の V2 化

**決定**: `HandlebarsApplicationMixin(ApplicationV2)` を基底とし、ウィンドウフレームなしで常駐描画する。

`popOut: false` の V2 相当は `window: { frame: false }` で実現。
CSS の `position: fixed` による配置は既存のまま維持。
`_onRender` でイベントリスナーを登録。

`render({ force: true })` が既存の `render(true)` に相当する。

##### 論点6: ダイアログの V2 化

**決定**: 既存 8 ダイアログの内部実装を `DialogV2.wait()` に置き換える。外部 API (`static async prompt(...)`) は変更しない。

`DialogV2.wait()` のシグネチャ:
```js
await foundry.applications.api.DialogV2.wait({
    window: { title: "タイトル" },
    content: "<p>...</p>",   // または renderTemplate() の結果
    buttons: [
        { action: "ok", label: "OK",
          callback: async (event, button, dialog) => { /* 戻り値が wait() の resolve 値 */ } },
        { action: "cancel", label: "キャンセル", default: true }
    ],
    close: () => null  // ×ボタンや Escape での解決値
});
```

各ダイアログの現在の `new Dialog({...}).render(true)` を上記パターンで置き換える。

`TnxScenarioSettingWizard` (FormApplication) は `HandlebarsApplicationMixin(ApplicationV2)` で再実装し、マルチステップ進行はインスタンスプロパティ (`this.step`) で管理する(変更なし)。

#### 良かった点

- `ActorSheetV2` / `ItemSheetV2` が v13 で提供されており、DnD 等の共通機能を手動実装する必要がない
- `TnxRecordSheet` が既存の V2 参照実装として使えるため、移行パターンが明確
- `EffectsSheetMixin` の統一設計が V2 移行を機に KI-009 を根本解消できる

#### 課題

- `TnxHud` の `window: { frame: false }` の挙動は v13 で実機確認が必要 (3-4 サブフェーズで確認)
- V2 の `TABS` / `tabGroups` の具体的な記述形式は実装時に公式 API を都度参照する

#### 推奨アクション

3-1(Item シート基底 + EffectsSheetMixin V2 化)に進む。本エントリの設計方針に従って実装する。

---

### 3-1 TokyoNovaItemSheet + EffectsSheetMixin V2 化(フェーズ3)

**日付**: 2026-06-06
**レビュー対象**:
- `scripts/item/tnx-item-sheet.mjs`
- `scripts/module/effects-sheet-mixin.mjs`
- `templates/parts/active-effects-list.hbs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: V2 パターンが `TnxRecordSheet` の参照実装と一致しているか
- 拡張性: 派生シート(Style/Miracle 等)が基底を継承するだけで動くか
- FVTT 慣行: `_onRender` / `DEFAULT_OPTIONS.actions` の使い方が正しいか
- 既存設計との整合: `usage-list.hbs` 保護制約を守りながらイベント処理を実現できているか

#### 良かった点

- **V1 → V2 直訳**: `getData` → `_prepareContext`、`activateListeners` → `_onRender` + `ACTIONS` の対応が明確。TnxRecordSheet と同パターンで実装できた。
- **EffectsSheetMixin ACTIONS 方式 (KI-009 解消)**: V1 の二系統(インスタンス代入 / 静的呼び出し)を `ACTIONS` オブジェクトの `...EffectsSheetMixin.ACTIONS` で統一。KI-009 を根本解消した。V1 メソッドは `@deprecated` を付けて CastSheet 移行 (3-3) まで維持。
- **usage-list.hbs 保護対応**: `data-action` を変更できないため、`_onRender` で `.action-create` / `.action-delete[data-index]` に `addEventListener` を直接添付する方式を採用。V2 の「毎レンダー後に `_onRender` が呼ばれる」特性を活かしてリスナーを毎回再登録することで、PART 再構築後も正常動作する。
- **cssClass / owner の互換補完**: V2 の `_prepareContext` は `cssClass` を提供しないため、`this.element.className` から補完した。V1 テンプレートとの互換を維持しつつ、CSS 整理に道を開けた。なお 3-2 での回帰バグ対応で `context.cssClass = ""` に変更済み(下記 3-2 エントリ参照)。
- **_isEditMode のスコープ**: インスタンスフィールドとして持ち、`context.editable` と AND してテンプレートに渡す。オーナーでない場合に自動的に無効化される。

#### 課題

- `static PARTS` は基底クラスに定義しない設計のため、基底クラス単体では `render()` できない。これは意図的な制約だが、将来的に `PARTS` 未定義のまま呼ばれた場合にエラーになる。フレームワーク側のガードを信頼する。
- `context.cssClass` タイミング問題 → 3-2 で `context.cssClass = ""` に変更して解消。root 要素のクラスは `_onRender` で直接操作するため PARTS には不要だった。

#### 推奨アクション

3-2(Item 派生シート 5 本 V2 化)に進む。すべて `TokyoNovaItemSheet` を基底とし、`static PARTS` と `_prepareContext` のオーバーライドを追加するだけで移行できる見通し。

---

### 3-2 Item 派生シート 5 本 V2 化 + 回帰バグ修正(フェーズ3)

**日付**: 2026-06-06 〜 2026-06-06
**レビュー対象**:
- `scripts/item/tnx-item-sheet.mjs`
- `scripts/item/tnx-style-sheet.mjs` / `tnx-miracle-sheet.mjs` / `tnx-general-skill-sheet.mjs` / `tnx-style-skill-sheet.mjs` / `tnx-organization-sheet.mjs`
- `templates/item/*.hbs`（5 枚全）
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: 基底 `TokyoNovaItemSheet` パターンを 5 派生シートが一貫して踏襲しているか
- FVTT 慣行: V2 の tab 機構・フォーム自動送信・`_onRender` パターンを正しく使えているか
- 既存設計との整合: `usage-list.hbs` 保護制約を維持できているか

#### 良かった点

- **`changeTab` を `_onRender` で明示呼び出し**: V2 はレンダー後に `active` クラスを自動付与しない。`for...of this.tabGroups` でループして `changeTab(tab, group, {force: true})` を呼ぶパターンで解決。テンプレートが `context.tabs` を使わない静的 HTML 構造であるため毎レンダーで再適用する必要があった。
- **`data-action="tab"` 追記**: V2 の `#onClickAction` は `data-action="tab"` がある要素のみ `_onClickTab` に到達する。5 枚全テンプレートの `<a class="item">` に属性を追加した。
- **`context.cssClass = ""` (stale クラス問題の根本解消)**: 3-1 時点では `this.element.className` から取得していたが、2 回目以降のレンダー時に `edit-mode` / `view-mode` が混入して `.tnx-button` の `pointer-events: none` が常時かかるバグが発覚。PARTS に渡さず root 要素で `classList.toggle` のみで管理する設計に変更。
- **`{{editor button=true}}` → `<prose-mirror toggled>` 置換**: V2 には V1 の `_activateEditor` 相当がない。`customElements.get("prose-mirror")` で `HTMLProseMirrorElement` を取得し、`.editor` div をカスタム要素に置換する方式を採用。保存は `change` イベント → V2 `submitOnChange` が自動処理するため `onSave` コールバック不要。

#### 課題

- `window-header` への `.edit-mode-toggle` ボタン追加は `_onRender` 毎に `remove → prepend` で処理。V2 では header も PART として管理可能だが、現状は JS で動的に挿入している。将来的に window controls として `DEFAULT_OPTIONS.window.controls` に移行できる。
- `<prose-mirror>` カスタム要素の内部スタイルは Foundry が管理するが、外側コンテナ (`tnx-editor-section__content`) との協調はブラウザ確認ベース。3-2b の CSS 作業で対処。

#### 推奨アクション

3-2b (アイテムシート CSS 修正) を実施後、3-3 (CastSheet V2 化) に進む。

---

### 3-2b アイテムシート CSS 修正(フェーズ3)

**日付**: 2026-06-06
**レビュー対象**: `css/tnx.css`（エディタ領域・トグルボタン・suit-selection セレクタ）
**ステータス**: レビュー済(問題なし)

#### 背景

フェーズ3 の V2 移行に伴う視覚的退行が累積したこと、およびフェーズ5 CSS 作業全体をフェーズ3 に前倒しする方針決定により、アイテムシートの CSS 修正サブフェーズを新設。

#### 修正内容と技術的判断

1. **`.edit-mode-toggle` に `display: inline-block; flex-shrink: 0;` 追加**
   V2 の `.window-header` はフレックスコンテナ。`<a>` 要素はデフォルトインライン扱いで `width: 36px; height: 20px` が無効化される場合があるため `inline-block` でブロックサイズを保証した。

2. **`.editor` / `.editor-edit` / `.editor-content` ルール削除 → `prose-mirror` 要素対応**
   3-2 で `{{editor button=true}}` が生成する V1 スタイル `div.editor` を `<prose-mirror>` カスタム要素に置換したため、旧 CSS は dead code となった。`min-width: 513px` もオーバーフロー原因として除去。新たに `prose-mirror { display: block; width: 100%; ... }` および `prose-mirror .ProseMirror { min-height: 200px; ... }` を追加。

3. **`.tnx-editor-section .prosemirror` (小文字) ルール削除**
   ProseMirror の実クラス名は `.ProseMirror`（混在ケース）。小文字 `.prosemirror` は CSS クラスセレクタとして一致しないため実質 dead code だった。

4. **`.suit-selection input:checked + .suit-icon` → `+ i` 修正**
   HTML 構造が `label.suit-icon > input + i`（label が親、input と i は兄弟）のため、親を対象とする `+ .suit-icon` は常に不一致だった。スキルシート固有ブロック（行 2806〜）は元から `+ i` で正しかった。汎用ブロックのみ修正。

#### 追加修正 (2026-06-06 コミット 102ebd7)

初回実装後に以下の不具合を修正:

5. **`prose-mirror` CSS レイアウト修正**
   Foundry V13 の `<prose-mirror>` は `display: flex; position: relative` が前提で、内部の `.editor-content` が `position: absolute; inset: 0` でコンテナを埋め尽くす設計。初回実装では `display: block; padding: 10px; min-height: 120px` で上書きしていたため、120px のコンテナにツールバー + エディタが収まらず編集フィールドにアクセス不能だった。修正: `--min-height: 300px`（Foundry 定義済み変数）を使用し、`display/padding` の上書きを廃止。

6. **エディタ開閉ボタンをヘッダーへ移動**
   Foundry の `button.toggle` は `position: absolute; top: 0; right: 0` でコンテンツ領域内部に表示される。ユーザーの要求はヘッダー行と同高さにアイコン単体で表示すること。対処: Foundry の `button.toggle` を `display: none !important` で非表示にし、各テンプレートの `.tnx-editor-section__header` にアイコン専用の `<a class="tnx-editor-open">` を追加。`_onRender` でクリック時に `pm.open = true` を呼び出すワイヤーアップを実施。

7. **`toggled: !context.isEditMode` 誤適用の取り消し**
   「編集不可」の原因を edit/view モード切替と誤解した修正を revert し、`toggled: true` に戻した。

#### 課題

- `.edit-mode-toggle` のアイコン描画は `::after` 疑似要素 + Font Awesome 依存。FA バージョン変更時はコードポイントの更新が必要。

#### 推奨アクション

実機でアイテムシートを確認後、3-CSS-0 (デザインシステム設計セッション) を実施。その後 3-3 に進む。

---

### B-8 Card type の DataModel 化(フェーズB)

**日付**: 2026-05-23
**レビュー対象**: `scripts/data/card/common/base.mjs`(CardBaseTemplate),
`scripts/data/card/playing-cards.mjs`, `scripts/data/card/neuro-cards.mjs`,
`scripts/data/card/other.mjs`,
`scripts/tnx.mjs`(CONFIG.Card.dataModels 登録追加)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: organization.mjs パターンとの一貫性
- 層依存: data/card → data/item の依存を避けた CardBaseTemplate の独立定義の妥当性
- FVTT 慣行: `CONFIG.Card.dataModels` への登録が `init` フックで正しく行われているか
- スコープの適切性: Foundry Card コアフィールド(suit / value 等)を system 側に持たせないこと

#### 良かった点

- **organization.mjs パターンの踏襲**: `SystemDataModel.mixin(CardBaseTemplate)` と
  `defineSchema()` の記法が Actor / Item 側と完全に対称。フェーズB 全体の実装一貫性を保てている。
- **CardBaseTemplate の独立定義**: `Item.BaseTemplate` は「全 Item 共通」と明記されており、
  data/card → data/item の層またぎを避けるため `scripts/data/card/common/base.mjs`
  として独立定義した。フィールド定義は同一だが概念的に別階層に属する。
- **Foundry コアフィールドの不干渉**: suit / value / face 等は Foundry Card document の
  コアフィールドであり、system 側には持たせない方針を `@fileoverview` に明記した。
- **CONFIG.Card.dataModels への登録**: Actor / Item の登録ブロック直後に同スタイルで配置し、
  3 type のキー名が template.json の types 配列(playingCards / neuroCards / other)と
  完全一致することを確認。
- **テストカバレッジ**: 計 542 件(B-7b の 530 件から 12 件増加)。
  各 type で description の存在、level・suit の非存在を確認。

#### 課題

なし。B-8 のスコープ内で品質を満たした実装。

#### 推奨アクション

B-9(template.json の廃止)に進む。以下を申し送る。

- DataModel 化対象の全 25 type(Actor 5 + Item 17 + Card 3)が揃った。
- B-9 での template.json 廃止前に、description 等の ProseMirror 対象フィールドの
  サニタイズ宣言が system.json の documentTypes 側または DataModel の HTMLField で
  カバーされているかを確認すること(PHASE_B_TASKS.md B-9 の事前確認事項参照)。
- template.json には Actor.types 配列と Card セクションが残置。B-9 でこれらを含む
  ファイル全体を廃止する。

---

### B-9 template.json 廃止 + description の HTMLField 化(フェーズB)

**日付**: 2026-05-24
**レビュー対象**: `scripts/data/item/common/base.mjs`, `scripts/data/card/common/base.mjs`,
`scripts/data/actor/common/biography.mjs`(description HTMLField 化),
`system.json`(documentTypes に htmlFields 宣言追加),
`tests/setup.mjs`(MockHTMLField 追加),
`tests/template-integrity.test.mjs`(documentTypes ベースに全面書き換え),
`template.json`(廃止・削除)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 不可逆作業の順序: HTMLField 化(可逆)→ テストグリーン確認 → template.json 削除(不可逆)
- htmlFields 宣言の type 集合と description 保有 type の完全一致
- template.json 削除後の全 25 type の正常認識可否(system.json documentTypes が権威)
- integrity テストの意図保持: JSON 妥当性 + 型定義整合を documentTypes ベースで継続

#### 良かった点

- **不可逆作業の安全な順序**: HTMLField 化をコミット(可逆)した後に npm test グリーンを
  確認し、その後 template.json を削除(不可逆)した。削除前のグリーン確認が安全弁として
  機能した。
- **htmlFields 宣言の type 集合が description 保有 type と完全一致**: biography mixin を
  持つ Actor(cast / guest / extra)のみに付与し、troop(memo のみ) / player(attributes + actorBase)
  を正しく除外した。Item 全 17 / Card 全 3 はすべて BaseTemplate / CardBaseTemplate 経由で
  description を持つため全件付与。過不足なし。
- **integrity テストの意図を documentTypes ベースで継続**: 旧テストの「JSON 妥当性」「DataModel
  との二重定義禁止」という意図を「system.json 妥当性」「documentTypes への全 25 type 存在確認」
  「DataModel ファイルと documentTypes.Item の過不足なし確認」として正確に置き換えた。
  テスト件数は 10 → 12 件(+2)と増加し、検証密度も向上した。
- **MockHTMLField の最小実装**: setup.mjs への追加は既存の MockStringField と同パターン。
  テスト環境を最小限の変更で拡張した。
- **description フィールドのトップレベル限定**: actions[].description(UsageTemplate 内の
  プレーンテキスト用)は対象外とし、ProseMirror 編集対象のトップレベル description のみを
  HTMLField 化した。スコープの逸脱なし。

#### 課題

なし。B-9 のスコープ内で品質を満たした実装。

#### フェーズB DataModel 化 完了総括

B-1〜B-9 の全サブフェーズが完了した。

| 完了項目 | 内容 |
|---|---|
| DataModel 化 | Actor 5 + Item 17 + Card 3 = 全 25 type |
| CONFIG 登録 | `CONFIG.Actor/Item/Card.dataModels` に全 type 登録済み |
| HTMLField 化 | description(トップレベル)を 3 テンプレートで HTMLField に変更 |
| htmlFields 宣言 | system.json documentTypes に全対象 type へ宣言追加 |
| template.json | 廃止済み。type 権威は system.json documentTypes に一本化 |
| テスト | 544 件グリーン / ESLint 0 errors |

#### 推奨アクション

B-10(移行後の検証)に進む。実機確認はユーザー主役。

- 既存シート(cast / guest 等)の正常表示
- EXP 計算が変わらず動作すること
- カード判定・神業 usageCount 等の既存ロジックが破壊されていないこと
- 確認後に v0.2.0 タグを打つ(PHASE_B_TASKS.md の完了条件を参照)

---

### ホットバー折りたたみ死コードの削除(フェーズ1-0)

**日付**: 2026-05-24
**レビュー対象**: `scripts/tnx.mjs`(renderHotbar フック), `scripts/module/tnx-hud.mjs`(activateListeners 内折りたたみブロック)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

v13 で Foundry コアからホットバー折りたたみ機能が削除されたため、`#bar-toggle` / `#action-bar`
を参照する2箇所のコードが完全な死コード(no-op)となっていた。これを削除する判断の妥当性を確認する。

#### 良かった点

- **削除スコープの明確な限定**: `renderHotbar` フック(tnx.mjs)と `activateListeners` 内の
  ホットバーブロック(tnx-hud.mjs)のみを削除対象とし、サイドバー連動コード(`#sidebar`)は
  意図的に残した。これは別件(V13_MIGRATION_NOTES.md §7)であり、混同しなかったことで
  スコープ逸脱を防いだ。
- **CSS は据え置き**: `.hotbar-is-collapsed` クラス定義は削除しなかった。CSS 整理は
  フェーズ5 の領分であり、ここで手を出すと影響確認が増える。正しい判断。
- **代替実装なし**: v13 ではホットバー展開が標準仕様であり、折りたたみの不在が正しい状態。
  代替実装を足さなかったことは、不要な複雑性の導入を避けた正しい選択。
- **テスト件数維持**: 削除後も 544 件グリーン。ホットバー折りたたみは Foundry DOM に
  依存するためユニットテスト対象外であり、テスト削除は不要だった(期待どおり)。

#### 課題

なし。削除のみで完結し、既存機能への影響もない。

#### 申し送り

`tnx-hud.mjs` のサイドバー連動コード(activateListeners 内の `#sidebar` / `#sidebar-tabs a.collapse`
参照ブロック)は V13_MIGRATION_NOTES.md §7 の未確認事項。フェーズ5 以降で確認・整理する。

---

### アクトシート作成導線の v13 対応(フェーズ1-1)

**日付**: 2026-05-24
**レビュー対象**: `scripts/tnx.mjs`(renderJournalDirectory フックハンドラ)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

v13 で機能停止していたアクトシート(TnxScenarioSheet)の新規作成導線を復活させる際の
設計判断を記録する。

#### 良かった点

- **「資料を作成」のコア委譲**: 旧実装は標準の createEntry ボタンを乗っ取って2択メニューに
  置き換えていたが、これは Foundry コアの UX を破壊する設計だった。今回「資料を作成」を
  コア標準ボタンに委ね、独自ボタンは「アクトシートを作成」1 つに簡素化した。
  機能を分離した結果、コードが 55 行 → 15 行に縮小し保守性が向上した。
- **フォールバック方式の選択根拠の明確化**: `getHeaderControls` 系フックは v13 の
  `ApplicationV2` 標準 API だが、外部コードから onclick 相当を注入する正確なプロパティ名が
  一次情報なしに確定できず、動作保証が困難。`renderJournalDirectory` フックは発火が
  確認済み・DOM ターゲットも KI-011 調査で確定済みであるため、確実に機能する方式を選んだ。
  将来 getHeaderControls 方式に移行しやすいよう、ロジックは1関数(15行)に収めた。
- **JournalEntry サブタイプ不可の確定記録**: Actor/Item のような型ドロップダウン方式が
  JournalEntry では実現不可であることを KNOWN_ISSUES.md KI-011 に記録した。
  将来の再検討(「なぜドロップダウンにしないのか」という問い)に対する参照先を整備した。

#### 課題

- **実機未確認**: `[data-action="createEntry"]` の DOM 位置と `insertAdjacentElement('afterend')`
  の組み合わせが実際の v13 レイアウトでどう見えるかは実機確認が必要。
  ボタンのスタイル(コア標準との見た目の整合)もフェーズ5(CSS リファクタ)で調整する。

#### 申し送り

- JournalEntryPage カスタムサブタイプ化は検討の上**不採用**。詳細は下記エントリを参照。
- TnxScenarioSheet 本体の ApplicationV2 化はフェーズ3 で実施(フェーズ1 では未実施)。

---

### TnxHud レイアウト CSS 調整(フェーズ1-2)

**日付**: 2026-05-24
**レビュー対象**: `css/tnx.css`（HUD セクションの座標定義）
**ステータス**: レビュー済(問題なし)

#### レビュー観点

v13 標準 UI との物理衝突を CSS 座標調整のみで緩和する際の設計判断を記録する。
JS・HTML・レイヤー方式には一切触れず、CSS の値変更と死にルール削除のみを行った。

#### 良かった点

- **影響範囲の最小化**: CSS の値変更のみ。JS クラス・テンプレート・z-index 方式を
  変えないため、フェーズ2(V2 化)での大規模再設計の選択肢を狭めない。
- **死にルールの除去**: `sidebar-collapsed` / `hotbar-is-collapsed` の CSS ルール
  3件を削除。KI-012 でホットバー折りたたみが廃止された時点から死んでいたルールを
  整理した。これにより CSS の意図が明確になり、将来の担当者が誤解しにくくなる。
- **三層構造の維持**: 右側の縦一列構造(上=ニューロ/シーン、下=山札/捨て札)を保ち、
  中段を空けたまま。将来の情報項目エリア追加の余地を確保している。
- **左上の空き維持**: TokenActionHUD の将来的なデフォルト位置として左上を空けたまま。

#### 課題

- ⚠️ **初期値であること**: 今回設定した座標値(right: 360px / top: 50px / bottom: 80px /
  max-width: 680px)は合理的な初期値だが、実機環境によって微調整が必要になりうる。
  実機確認後にユーザーが値を詰める前提。
- ⚠️ **根本的な解決ではない**: `position: fixed` の独立レイヤー方式は変えていない。
  画面解像度・ブラウザ・Zoom 倍率によって挙動が異なる本質的な問題は残る。
  フェーズ2(V2 化)で `#ui-right` 等への組み込みを行うことで初めて解消される。
- ⚠️ **手札枚数の制約**: `max-width: 680px` は 5 枚に相当。6 枚以上は 2 行に折り返す。
  これはデザイン的に意図した挙動か、問題かを実機確認で判断する必要がある。

#### 申し送り(フェーズ2 への)

- **UI 領域組み込みによる理想形**: ApplicationV2 化で TnxHud を Foundry の標準 UI 領域
  (例: `#ui-right` の上部)に組み込めば、座標のハードコードが不要になり、
  画面解像度やサイドバー幅の変動に自動対応できる。
- **将来エリアの予定地**: 右側の中段(ニューロ/シーンと山札/捨て札の間)は
  情報項目エリア・他 PC 手札小型表示の候補地。現在は空きとして保持。
- **左上の確保**: TokenActionHUD は左上を占有する。TnxHud の左上への配置は避ける。

---

### アクトシートのページ化検討と不採用(フェーズ1-1 設計判断)

**日付**: 2026-05-24
**レビュー対象**: TnxScenarioSheet / JournalEntry の構造方針
**ステータス**: レビュー済(問題なし・不採用確定)

#### 検討内容

KI-011 調査の過程で、ハンドアウト・情報項目・シナリオテキストなどアクトシートの各セクションを
`JournalEntryPage` のカスタムサブタイプとして定義する案を検討した。

動機:
- `JournalEntry` はサブタイプを持たないため、作成ダイアログでのドロップダウン選択ができない
- ページレイヤーであればカスタムサブタイプが持てる(`documentTypes` に `JournalEntryPage` を登録)
- ページ追加ダイアログで「ハンドアウト」「情報項目」等の型が選択できるようになる

#### 不採用の理由

`documentTypes` に `JournalEntryPage` サブタイプを登録するとワールドグローバルに有効化される。
その結果、アクトシート専用のページ型が**通常の JournalEntry のページ追加メニューにも混入**する。

「普通のジャーナルにハンドアウトページを追加できてしまう」状態を回避する正規手段が
Foundry v13 の公開 API では確認できない。システム固有のページ型を標準 UI から隔離する
ためには UI 側でフィルタリングを挟む必要があり、その保守コストと UX 上の混乱を
天秤にかけると採用できないと判断した。

シーンは行一覧テーブル UI の必要性から従来の flag + 独自タブ UI 方式が適しており、
同じ理由でアクトシート全体を「ページ化」するメリットが薄い。

#### 結論

現行の「`JournalEntry` を `sheetClass` フラグで TnxScenarioSheet に紐付け、独自タブ UI で
ハンドアウト・情報項目・シーン・テキスト等を管理する」構造を**最終形**として維持する。

アクトシート本体の ApplicationV2 化(フェーズ3)もこの構造のまま素直に行う。

#### 記録の目的

この検討を DESIGN_REVIEW に残す理由は、「なぜページ化しないのか」という問いへの
明示的な回答を用意するため。将来のセッションが同じ検討を繰り返さないよう記録する。

`docs/FUTURE_CONSIDERATIONS.md` の JournalEntryPage カスタムサブタイプ化エントリも
不採用確定として更新済み。

---

### 2-0 レコードシートと User flag スキーマ設計(フェーズ2)

**日付**: 2026-05-25
**レビュー対象**: `scripts/module/user-flag-schema.mjs`、`scripts/module/tnx-record-sheet.mjs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: flag スキーマの変更が一箇所に集約されているか
- 拡張性: 編集機能(2-2)・手札機能(2-3)追加時に既存コードを大きく変えなくて済むか
- FVTT 慣行: ApplicationV2 の使い方が v13 の推奨パターンに準拠しているか
- 既存設計との整合: player Actor の TnxHistoryMixin との共存・再利用

#### 決定事項

##### 論点1: flag 読み出しの集約方式

**決定**: `getUserFlagData(user)` / `getUserFlagHistorySorted(user)` をモジュール関数として公開。
直接 `user.flags[SCOPE]` を散在させない。

決定理由: flag はスキーマがなく、キー名のタイポや初期値の不一致が散在するリスクが高い。
getter を一箇所に集約することで、将来のキー名変更や初期値変更が1ファイルで完結する。
また、純粋な JS 関数としてテスト容易性が高い。

##### 論点2: TnxRecordSheet の基底クラス

**決定**: `HandlebarsApplicationMixin(ApplicationV2)` を直接使用(DocumentSheetV2 ではない)。

決定理由: レコードシートは特定 Foundry Document(Actor/Item 等)のシートではなく、
User を引数に受け取る独自 Application である。DocumentSheetV2 は `this.document` を
前提とした Document 専用 API が多く、User を引数に渡す用途には合わない。
素の ApplicationV2 + HandlebarsApplicationMixin が意図に一致する。

プロジェクト内でフェーズ2-0 が最初の ApplicationV2 実装となる。
以降のシート V2 化(フェーズ3)の先例として参照可能。

##### 論点3: インスタンス重複防止

**決定**: `foundry.applications?.instances?.get(appId)` で既存インスタンスを確認し、
存在すれば `bringToFront()` して早期リターン。id は `tnx-record-sheet-${user.id}`。

決定理由: 同一ユーザーのレコードシートが複数開くのはバグに見える。
ApplicationV2 の instances Map は v13 で利用可能だが optional chaining でフォールバックを保つ。
2-2 で編集機能を追加する際も、同じ id ベースの重複防止方式を踏襲する。

##### 論点4: コンテキストメニューフック

**決定**: `Hooks.on("getUserContextOptions", ...)` を `init` フック内で登録。

確認: v13 のプレイヤーリスト(`PlayerList` ApplicationV2)のコンテキストメニューは
`getUserContextOptions` フックを使用する。`<li>` の `dataset.userId` から対象 User ID を取得。
`condition` コールバックで「GM は全員分 / 一般ユーザーは自分の分のみ」を制御。

#### 課題

特になし。2-2(編集機能追加)で `_prepareContext` の `isEditMode: false` を `true` に切り替え、
FormHandlerMixin 等の追加が必要になる点はフェーズ 2-2 の設計で扱う。

---

### 2-1 cast → User UUID 記録(cast-ownership.mjs)(フェーズ2-1)

**日付**: 2026-05-25
**レビュー対象**: `scripts/module/cast-ownership.mjs`、`scripts/data/actor/cast.mjs`、`scripts/tnx.mjs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 既存 player 向け処理との分離(player Actor の手札権限同期・EXP 同期は無変更か)
- 記録先フィールドの選定(playerId 転用 vs 新規フィールド)
- テスト可能性(純粋関数と Foundry 依存の分離)
- 無限ループリスク(ownerUserId 更新が再び hook を呼ばないか)

#### 決定事項

##### 論点1: 記録先フィールドの選定

**決定**: `cast.system.ownerUserId`(新規 `StringField`)を追加。`playerId` は転用しない。

決定理由: `playerId` は `_onDrop` / EXP 同期フック / `tnx-cast-sheet.mjs` の 10 箇所以上で
player Actor UUID として参照されている。2-1 で転用すると全て破壊される。
`ownerUserId` を独立フィールドとして設けることで、2-5 player 廃止まで安全に並存できる。
2-2 は `ownerUserId` を EXP 同期の参照先として使う。

##### 論点2: 純粋関数と Foundry 依存の分離

**決定**: `cast-ownership.mjs` を3層に分離。
- `pickFirstOwnerUserId` — ownership オブジェクトから最初の非 GM userId を選出(純粋関数)
- `resolveOwnerUserIdAction` — 新旧 UUID を比較してアクション(set/confirm-overwrite/none)を決定(純粋関数)
- `recordCastOwnerUser` — 上記を組み合わせて実際の更新・ダイアログを行う(Foundry 依存)

決定理由: ownership ロジックは `game.users` に依存しないため純粋関数でテスト容易。
ダイアログ(`Dialog.confirm`)は UI 層のため Foundry 依存関数に隔離。

##### 論点3: 無限ループリスク

**評価**: リスクなし。

`recordCastOwnerUser` が `castActor.update({ "system.ownerUserId": uuid }, { calcExp: false })` を呼ぶと
`updateActor` フックが再び発火するが、このとき `diff.ownership` は含まれない(`diff.system.ownerUserId` のみ)。
`if (diff.ownership)` ガードにより所有者記録分岐はスキップされる。
`{ calcExp: false }` オプションにより EXP 再計算も抑制される。

##### 論点4: hook 登録箇所

**決定**: 既存 `ready` フック内の `updateActor` に新分岐を追加。

決定理由: `game.users` が確実に利用可能な `ready` 以降での登録が適切。
既存の EXP 同期フックと同じ `updateActor` 内に追記することで `updateActor` の全体像が一箇所に集中する。

#### 課題

- 2-5 player 廃止時に `playerId` フィールドの扱い(廃止 / `ownerUserId` への統合)を決定する。
- `ownerUserId` の矯正用 UI(手動 UUID 入力)は 2-4 以降の検討事項。

---

### EXP・履歴の二フック同期(フェーズ2-2)

**日付**: 2026-05-25
**レビュー対象**: `scripts/module/exp-sync.mjs` / `scripts/tnx.mjs` / `scripts/actor/tnx-cast-sheet.mjs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

##### 論点1: `syncCastExpToUser` の同一性チェック

**決定**: `currentSpent === newSpent` のとき `ownerUser.update` をスキップする。

決定理由: 変更がなければ `updateUser` が不要に発火するのを防ぐ。
`currentTotal` は User flag から取得するため、totalが変わっていなくても spent/value が
一致していれば書き込まない。この早期リターンにより updateUser フックの発火回数を最小化できる。

##### 論点2: `_performHistoryUpdate` のキー変換アプローチ

**決定**: `system.history.X` → `flags.TNX.history.X` のキー置換を直接実施する。

決定理由: TnxHistoryMixin が生成する updateData は `system.history.*` のフラット dot-notation。
User flag の対応パスは `flags.${TNX_FLAG_SCOPE}.history.*` であり、プレフィックス置換で機械的に変換できる。
追加で `system.exp.total`(historySum)は User flag の `exp.total` に対応させ、
`additional` の加算は行わない(additional は cast 固有で User flag 非管轄)。

cast ローカルの `system.history` も同時に更新することで、TnxHistoryMixin が `this.document.system.history`
を直接参照する add/delete/change 操作の整合性を保つ。

##### 論点3: `updateCastExp` の `linkedPlayer` パラメータ削除

**評価**: 安全。

`linkedPlayer` は全コールサイトで使われておらず(デフォルト `null`)、削除しても既存呼び出しに
影響しない。フォールバック参照(`fromUuid(playerId)`)も削除することで不要な非同期 I/O を排除できる。

##### 論点4: 二フックまたぎの無限ループ防止

**決定**: `{ syncing: true }` フラグを updateUser と syncCastExpToUser が発する update に付与し、
各フックの先頭でガードする。

決定理由: 往復の順路が固定しているため、片道ずつ遮断すれば十分:
- `syncCastExpToUser` → `ownerUser.update({syncing:true})` → `updateUser` が `options.syncing` で return
- `updateCastExp` → `cast.update({calcExp:false})` → `updateActor`: `calcExp:false` で EXP 再計算スキップ;
  cast→User 同期は `!options.syncing` なので実行し `syncCastExpToUser` → `ownerUser.update({syncing:true})` で止まる

##### 論点5: `_getHistoryData`/`_saveHistoryData` の死コード解消

**評価**: 解消できた。

`ownerUserId` 設定時は User flag を参照・書き込む実装を追加。
これらは `_performHistoryUpdate` からは直接呼ばれていないが、
将来 `_performHistoryUpdate` のリファクタリング時にこれらを活用できる。
`_saveHistoryData` は `saveUserFlagHistory` + cast ローカル更新に一本化している。

#### 課題

- cast の `system.history` はローカルコピーであり、レコードシートで直接 User flag を変更した場合に
  cast ローカルとの乖離が生じる可能性がある。2-4(データ移行)または 2-5(廃止)でキャストシートの
  history 表示を User flag 参照に完全移行することで解消する予定。
- `_performHistoryUpdate` の key 変換ロジックは、TnxHistoryMixin が生成するパス形式に依存している。
  Mixin の実装変更時はこの翻訳ロジックの追従が必要。

---

### CSS 全体監査 (3-CSS-0 前提)(フェーズ3)

**日付**: 2026-06-06 (監査) / 2026-06-07 (実装完了)
**レビュー対象**: `css/tnx.css` (3,723 行 → 約 3,570 行)
**ステータス**: 実装完了

**実装コミット（本体リポジトリ）**:
- `57ffbb1` 分類C: 重複ルール除去・矛盾重複解消
- `293a238` 分類D: 主要セレクターへの親スコープ付与
- `915593f` グループ4・6・7: セレクターリスト統合
- `6c06e63` グループ1・3: アイコンコントロールスタイル統一
- `5ee6954` 分類G: 死コード・ステールコメント除去
- `67d0124` 分類E: appearance: none 標準プロパティ補完
- `dad4a40` 分類F: !important を詳細度引き上げで削減
- `90d99f9` 分類B: ハードコードカラー変数化（4変数追加）

**グループ8**: `.text-item-controls` の `align-items: center` 欠落は意図的差異として保持。コンテナ差異はフェーズ4 デザイン刷新時に整理する。
**グループ2**: グループ4 の統合作業内で実質的に完了。

#### 概要

3-CSS-0（CSSリファクタリング）着手前の全行監査。ファイル変更は行っていない。
課題を A〜M の分類と、「外見同一・クラス別」のコンポーネント化候補グループ 1〜8 に整理した。

---

#### 分類A: セクション番号の不整合

ファイル先頭の Section 6 ヘッダーが Section 2〜5 より前に出現している。
番号付けコメント体系が複数混在（`/* === */`、`/* --- */`、全角バナー等）。

---

#### 分類B: ハードコードカラー（CSS変数バイパス）

主な箇所（フェーズ4の視覚デザイン刷新前に変数化しておく必要がある）:

- `#555` (トグルボタン背景, HUDパネル枠)
- `#fff` (トグルノブ, 履歴ヘッダー文字)
- `#a2f3ff` (ホバー色)
- `#333`, `#333a41`, `#2a2a2e`, `#444` (サイドバー・メニュー)
- `#f0f0f0`, `#ccc` (作成メニュー)
- `#222`, `#888` (HUDカード・プレースホルダー)
- **`white`, `#ccc`, `#000`** (L2754–2756) — ダークテーマ中にライトテーマ値が混入
- `rgba(255,255,255,0.3)` (履歴リストラッパー背景)
- `#444`, `#fff` (履歴ヘッダー)
- `#4a5057`, `#3e444a`, `#888` (view-mode グレーアウトボタン)
- `#777`, `#e0e0e0`, `#d0d0d0` (`.tnx-bs-button` — ライトテーマの配色が混入)
- `#858585`, `#999`, `#3e444a` (各所)

---

#### 分類C: 重複ルール（15ペア以上）

| 重複対象 | 1回目 | 2回目 | 備考 |
|---|---|---|---|
| `.style-skill-display` | L2830 | L2929 | 完全重複 |
| `.style-skill-display-grid` | L2837 | L2936 | 完全重複 |
| `.style-skill-display label` | L2842 | L2949 | 2回目に `flex-shrink: 0` 追加 |
| `.style-skill-inputs`（grid定義）| L2869 | L2956 | 完全重複 |
| `.style-skill-inputs .full-width-row` | L2874 | L2962 | 完全重複 |
| `.tnx.sheet.item .style-skill-inputs .form-group` | L2877 | L2965 | 完全重複 |
| `.array-row` | L3037 | L3107 | 完全重複 |
| `.array-row select` | L3044 (`flex: 0 0 auto`) | L3115 (`flex: 1`) | **値が矛盾**。コメントに「ここを変更」とあるが前の定義が残留 |
| `.row-controls` | L3069 | L3126 | 2回目に `flex: 0 0 auto` 追加 |
| `.add-array-item, .delete-array-item` | L3079 | L3135 | 2回目は不完全 (`/* ... 他のスタイルはそのまま ... */`) |
| `.handout-item` | L2398 | L2454 | 2回目がリファクタ版（flex追加）、1回目は旧版 |
| `.skills-list-edit .item-list .item` | L2717 | L3170 | 2回目に `gap: 10px` 追加 |
| `.abilities-grid.edit-mode-layout` | L574 | L1364 | 完全重複 |
| `.number-input-spinner input` | L325 | L341 | プレフィックス違いで重複 |
| `.number-input-spinner .tnx-button` | 同上 | 同上 | 同上 |

---

#### 分類D: スコープなしセレクター

`.tokyo-nova` 親セレクターを持たないルールが大量に存在し、Foundry 全体またはサードパーティと衝突するリスクがある。主な対象クラス:

- カード系: `.deck-card`, `.card-in-hand`, `.main-area`, `.neuro-area`, `.hand-area`, `.trump-area`, `.card-empty-placeholder`
- スキル系: `.skills-section`, `.suit-selection`（後半にスコープ付き版と共存）、`.suit-icon`, `.basic-info`, `.cell`（非常に汎用的な名前）
- 履歴系: `.history-list-wrapper`, `.history-grid`, `.history-header`, `.history-row`
- ハンドアウト系: `.handout-item`, `.handout-row` 等
- シナリオシート全体: `.scenes-container`, `.phase-section`, `.scene-table`, `.scene-item` 等
- その他: `.tnx-create-menu`, `.mirror-character`, `.cardpool-sheet`, `.abilities-section`

加えて、**スコープなし版とスコープあり版が共存**している例:
- `.suit-selection` (L2759, unscoped) と `.tokyo-nova.sheet.item.skill .suit-selection` (L2797, scoped)
- `.level-display input` (L2749, unscoped) と `.tokyo-nova.sheet.item.skill .level-display input[readonly]` (L2822, scoped)
  → unscoped 版は scoped 版の登場後は実質死コード

---

#### 分類E: ベンダープレフィックス不備

`-webkit-appearance: none` が `appearance: none` の対応なしに使用されている箇所が4件（L223, L595, L782, L1697 付近）。

---

#### 分類F: `!important` 乱用

- `.small-button` に4件（L2448–2452）
- `.usage-limit-group { align-items: end !important }` (L2899)
- ダイアログカード選択の width/height
- `.window-content` の background-image/color
- view-mode 入力欄の複数箇所
- 表示切り替え `display: none !important`

---

#### 分類G: 死コード・ステールコメント

- 変更経緯コメントが残存: `▼▼▼ 変更: ...`, `▼▼▼【重要】...` が本文中2〜3箇所
- 不完全ルールブロック: `.add-array-item` の2回目定義 (L3135) が `/* ... 他のスタイルはそのまま ... */` のまま
- `.abilities-section` (旧フィールドセットパターン)。現在は `.tnx-fieldset` が使われており実質不使用
- unscoped `.suit-selection`, `.level-display` の初期定義はスコープ付き版で上書きされ死コード化
- stale コメント: `.history-row .cell-title { justify-content: center; /* テキストの長い項目は左寄せ */ }` — コメントと値が矛盾 (L3487)

---

#### 分類H: 命名不統一

- `.cell` (L3333) — Foundry・他ライブラリと衝突リスクのある汎用名
- `.array-row`, `.checkbox-row`, `.form-fields-stack`, `.row-controls` 等に `.tnx-` プレフィックスなし
- コメントスタイルが混在: `/* === */`, `/* --- */`, 全角バナー, 日本語見出し

---

#### 分類I: CSS カスタムプロパティ名の混乱

- `--tnx-color-text-dim-sheet: #353535`（暗い背景向けの**暗い**テキスト色）が `--tnx-color-text-dim: #cecece`（暗い背景向けの**明るい**テキスト色）と紛らわしい命名

---

#### 分類J: 初期指摘の再利用性ギャップ

- `.texts-header .add-text-button` と `.info-header .add-info-button` がほぼ同一（後述グループ2）
- `.tnx-fieldset` と旧 `.abilities-section` の重複パターン

---

#### 分類L: ライトテーマ値のダークテーマへの混入

- `.sheet-header .level-display input` (L2754): `background-color: white; border: 1px solid #ccc; color: #000;` — 明示的なライトテーマ配色
- `.tnx-bs-button` (L3581): `background: #e0e0e0; border: 1px solid #777;` — ライト背景のボタンが全体のダークテーマと乖離

---

#### 分類M: スート選択パターンの乱立

同一UI（隠しチェックボックス＋アイコン選択）が3パターン存在:
1. `.suit-selection` (L2759, unscoped, レガシー)
2. `.tokyo-nova.sheet.item.skill .suit-selection` (L2797, scoped, 改善版)
3. `.suit-checkbox-tiny` (L3156, 別実装)

---

#### コンポーネント化候補（外見同一・クラス別）

同一の見た目を定義しながらクラス名が異なるケース。統一コンポーネントへの整理候補。

**グループ1: 丸型アイコンボタン**（ユーザー確認済み、揃えるなら `.skill-control` 方式）

| クラス | サイズ | ボーダー |
|---|---|---|
| `.skill-control` (L2299) | 22×22px | `1px solid border-accent` |
| `.add-handout-button` (L2437) | 26×26px | `1px solid border-accent` |
| `.add-scene-button` (L1908) | 26×26px | なし(`border: none`, `line-height`で中央揃え) |

**グループ2: ピル型セクション追加ボタン**（`.add-text-button` と `.add-info-button` はほぼ完全一致）

共通定義:
```css
display: inline-flex; align-items: center; gap: 8px;
height: 32px; padding: 0 12px; font-size: 13px; font-weight: bold;
background-color: var(--tnx-color-primary-darker);
border: 1px solid var(--tnx-color-primary-dark);
border-radius: var(--tnx-border-radius-md);
cursor: pointer; transition: all 0.2s;
```

差異: `.add-text-button` のみ `flex-shrink: 0`。

**グループ3: インラインアイコンコントロール**

`.text-control` (L2056)、`.info-control` (L2183)、`.handout-control` (L2426)、`.scene-control` (L1946) が全て `font-size: 1.2em; color: text-muted; cursor: pointer; transition: color 0.2s` + data-action 別のホバー色という同一パターン。`.scene-control` のみ `transition` なし。

**グループ4: セクションヘッダー行**（`.texts-header` と `.info-header` は完全一致）

```css
display: flex; justify-content: space-between; align-items: center;
border-bottom: 1px solid var(--tnx-color-border-accent); padding-bottom: 8px;
```

配下の `h3` スタイルも同一。

**グループ5: カード型アイテムコンテナ**

`.text-item` (L2017) と `.info-item` (L2142) がほぼ同一（`.info-item` のみ `transition: all 0.3s ease-out` 追加）。

**グループ6: カードアイテムヘッダー行**（完全一致）

`.text-item-header` (L2025) と `.info-item-header` (L2151) が完全同一。

**グループ7: カードアイテム内テキストエリア**（完全一致）

`.text-item-body textarea` (L2071) と `.info-item-body textarea` (L2198) が完全同一。

**グループ8: コントロールアイコン行**

`.text-item-controls` (L2051)、`.info-item-controls` (L2177)、`.handout-controls` (L2425) がほぼ同一（`display: flex; gap: 15px; align-items: center; flex-shrink: 0`）。

---

#### 推奨アクション

**3-CSS-0 で対処（優先度順）**:

1. **分類C の矛盾重複を先行解消** — `.array-row select` の値矛盾は機能バグの原因になりうる
2. **分類D: スコープ付与** — 全セレクターに `.tokyo-nova` 親を追加
3. **分類C: 完全重複の除去** — 後出し定義を正として前の定義を削除
4. **グループ 4・6・7（完全一致）の統一** — 共通クラスに一本化
5. **グループ 1・2・3・8 のコンポーネント化** — `.tnx-` プレフィックス付き共通クラス新設
6. **分類G: 死コード・ステールコメント除去**
7. **分類E: ベンダープレフィックス補完**
8. **分類F: `!important` 削減**（削除できないものは注釈を付ける）
9. **分類B: ハードコードカラーの変数化**（フェーズ4 デザイン刷新の準備）

**フェーズ4 で対処**:
- 分類L（ライトテーマ値）、分類M（スートパターン統一）の視覚的側面
- 分類I（変数名の整理）はフェーズ4 のカラーパレット再設計と同時に行う

---

### 3-CSS-0 CSS リファクタリング実装(フェーズ3)

**日付**: 2026-06-07
**レビュー対象**: `css/tnx.css`（全体）、`.stylelintrc.json`、`postcss.config.mjs`
**ステータス**: レビュー済(問題なし)

#### 実施内容

CSS 全体監査（前エントリ）で特定した分類 B〜H を複数セッションにわたって実施した。

**構造整理（分類C・D・G）**:
- `.array-row select` / `.array-row input` 等 15+ 箇所の重複・矛盾ルールを統合
- 全セレクタへの `.tokyo-nova` / `.tokyo-nova.sheet` 親スコープ付与
- dead CSS ルールの除去（`.abilities-section`、ステールコメント、孤立定義）
- `.scenario-sheet .sheet-body` 二重定義をマージ

**再利用性（グループ統合）**:
- `.tnx-btn` / `.tnx-linked-btn` 共通基底をコンマセレクタに統合（hover・disabled 状態も統合）
- `.text-item` / `.info-item` 完全重複統合
- `form-group-checkbox` / `.checkbox-row` 共通基底統合
- `textarea:focus` 3セレクタ統合
- webkit スピンボタン 8セレクタ 1ブロックに統合
- HUD `.main-area` / `.neuro-area` / `.hud-panel` 共通基底統合

**変数化（分類B）**:
- `:root` に 10+ カスタムプロパティを追加（`--tnx-overlay-*`、`--tnx-shadow-glow-text-*`、`--tnx-transition-*`、`--tnx-color-primary-glow` 等）
- マジックナンバー（hardcoded color / transition / shadow 値）を変数参照に置換

**開発ツール整備（build: 8917c4f）**:
- `stylelint` + `.stylelintrc.json`: 重複セレクタ・空ブロック・無効 hex を検出（`npm run lint:css`）
- `postcss` + `postcss-cli` + `postcss-combine-duplicated-selectors`: 同一セレクタ分散を自動マージ（`npm run css:merge`）
- `postcss-merge-rules` は設定から除外: 無関係なセレクタの過剰マージ・コメント削除の副作用が確認されたため

#### 技術的判断

**`postcss-merge-rules` 除外について**
同じ宣言セットを持つ別セレクタをコンマ結合する機能自体は有用だが、テスト実行で `:root` 内のセクションコメントおよび行末 why コメントが削除されることを確認。コードの意図を保持する価値の方が行削減より高いと判断し、設定ファイルから除外した。

**`replace_all` の安全性**
変数化作業で `Edit.replace_all` を使用する際、`:root` の変数定義行が置換対象に含まれると循環参照になる。対策: `transition: all 0.2s ease-out;` のように値だけでなくプロパティ名込みのフル宣言で置換対象を指定し、`:root` 行（`--tnx-transition-all: all 0.2s ease-out;`）とパターンが一致しないことを確認してから実行した。

#### 成果

- `stylelint lint:css`: 警告 0（重複セレクタ・空ブロックなし）
- `postcss-combine-duplicated-selectors`: 構造変更 0（残存する分散セレクタなし）
- 推奨アクションリスト（前エントリ）の 1〜9 をすべて対処済み

#### 残留ハードコードカラー（9 件）

以下はすべて単一箇所のみで使用される設計値であり、変数化の DRY メリットがないため対処を保留した。
フェーズ4 のカラーパレット再設計時に一括で見直す。

| 値 | 箇所 | 用途 |
|---|---|---|
| `#a2f3ff` | L420 | edit-mode-toggle ホバー時ノブ背景 |
| `rgba(0,180,255,0.15)` | L824 | カード選択ハイライト |
| `#333a41` | L1345 | コントロール形状背景 |
| `#585f68` / `#666` | L1719-1720 | セカンダリボタンホバー背景・枠線 |
| `rgba(0,170,255,0.08)` | L2044 | info-item 開示ヘッダー背景（`--tnx-color-primary-glow` の 0.1 に統一可） |
| `#222` | L2296 | カード背景 |
| `rgba(255,255,255,0.03)` | L2939 | スタイルスキル偶数行ストライプ |
| `rgba(255,255,255,0.3)` | L3051 | 履歴リスト背景 |

#### 推奨アクション

3-3（CastSheet V2 化）に進む。視覚デザインの刷新はフェーズ4 に分離済み。

---

### 3-3 TokyoNovaCastSheet ApplicationV2 移行 + KI-017 解消（フェーズ3）

**日付**: 2026-06-07
**レビュー対象**: `scripts/actor/tnx-cast-sheet.mjs`, `scripts/module/tnx-history-mixin.mjs`, テンプレート3点, `css/tnx.css`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: V1からV2へのAPI変換が完全かつ一貫しているか
- 拡張性: 静的ACTIONSパターンで将来のハンドラ追加が容易か
- FVTT慣行: V13 ActorSheetV2 の推奨パターンに準拠しているか
- 既存設計との整合: EffectsSheetMixin / TnxHistoryMixin の ACTIONS 統合が一貫しているか

#### 良かった点

- `HandlebarsApplicationMixin(ActorSheetV2)` 継承、`DEFAULT_OPTIONS` / `PARTS` / `tabGroups` の設計がV2慣行に沿っている
- `...EffectsSheetMixin.ACTIONS` / `...TnxHistoryMixin.ACTIONS` のスプレッド展開で KI-009 解消パターン(3-1)を踏襲
- jQuery依存を完全排除。クラスベースハンドラは `_onRender()` 内でネイティブDOMイベントに変換
- `form: { submitOnChange: true }` で growth 等 named input の変更を自動処理（`_onGrowthChange` 削除）
- KI-017 デッドコード（`_getHistoryData` / `_saveHistoryData` / `_updateExpTotal`）を削除
- `foundry.applications.api.DialogV2.confirm` / `foundry.applications.ux.TextEditor.enrichHTML` への移行
- プレイヤーアクター廃止に伴い `_onDrop` override・`unlinkPlayer` action・`_performHistoryUpdate` B ブランチを完全削除（ライフサイクルの簡素化）
- CSS 特異度問題の根本修正: `.suit-checkbox-tiny input` → `input[type="checkbox"]` でグローバルルールとの競合を解消
- 再描画時の UX 対策として `_preRender` / `requestAnimationFrame` パターンを確立（スクロール位置保存・復元 + `.tnx-no-transitions` によるトランジション再生抑制）

#### 課題

- ⚠️ `_onToggleAbilityDetails` のアニメーション: V1 の `panel.slideToggle(200)` を `style.display` トグルに降格。視覚的退行だが機能的には問題なし。フェーズ4 CSS でCSSトランジションに置き換え予定
- ⚠️ ContextMenu コールバックの型: V13 ContextMenu がコールバックに渡す要素の型（HTMLElement か jQuery か）は実機確認が必要。現実装は `header.dataset.itemId` を前提
- ⚠️ サイドバーチラツキ: `.tnx-no-transitions` による対策を実施したが実機での解消は未確認。根本対応は PARTS 分割（sidebar / body を独立した PART にする）だが、フェーズ4 以降での対応を想定

#### 推奨アクション

3-4（TnxHud V2 化）に進む。ContextMenu の実機動作とサイドバーチラツキは 3-7 最終検証で確認する。

---

### 3-4 TnxHud ApplicationV2 移行（フェーズ3）

**日付**: 2026-06-07
**レビュー対象**: `scripts/module/tnx-hud.mjs`, `templates/hud/hud.hbs`, `scripts/tnx.mjs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: V1（Application）から V2 への変換が完全かつ一貫しているか
- 拡張性: static ACTIONS パターンで将来のハンドラ追加が容易か
- FVTT慣行: `window: { frame: false, positioned: false }` が正しい `popOut: false` 相当か
- 既存設計との整合: TnxActionHandler / ContextMenu との接続が維持されているか

#### 良かった点

- `HandlebarsApplicationMixin(ApplicationV2)` 継承、`window: { frame: false, positioned: false }` による `popOut: false` 相当の実現。CSS `position: fixed` での自己管理も既存のまま維持
- `getData()` → `_prepareContext()` への変換が完全。`accessCardPileId` / `gmTrumpDiscardId` 未使用変数を同時に削除
- jQuery 依存を完全排除。ContextMenu / dragDrop ともネイティブ DOM（`this.element`）を直接利用
- sidebar collapse 連動リスナー（V12 時代の死コード）を削除。コードが簡潔になった
- hud.hbs の `id="tnx-hud"` を DEFAULT_OPTIONS に移管し、V2 の外側要素が ID を保持する正しい構造を確立（テンプレート内での重複 ID を回避）
- `data-action` を camelCase（`drawFromDeck` / `takeFromDiscard` / `drawNeuro` / `playCard` / `useTrump`）に統一し、V2 の命名規則に準拠
- `tnx.mjs` の `render(true)` → `render({ force: true })` により呼び出し側も V2 準拠
- lint 警告が移行前比 -4 件（tnx-hud.mjs 由来の未使用変数 `actor` / 未使用パラメータ `header` / `event` が解消）

#### 課題

- ⚠️ V2 で `window: { positioned: false }` が実際に Foundry の位置管理を無効化するかは実機確認が必要。CSS `position: fixed` で自己管理している想定（3-7 最終検証で確認）
- ⚠️ ContextMenu の動作: V13 ContextMenu へのネイティブ DOM 渡しが正しく機能するかは実機確認が必要（3-7 最終検証で確認）

#### 推奨アクション

3-5（TnxScenarioSheet V2 化）に進む。ContextMenu の実機動作は 3-7 最終検証で一括確認する。

---

### 3-5 TnxScenarioSheet ApplicationV2 移行（フェーズ3）

**日付**: 2026-06-07
**レビュー対象**: `scripts/journal/tnx-scenario-sheet.mjs`, `templates/journal/scenario-sheet.hbs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: V1（JournalSheet）から V2 への変換が完全かつ一貫しているか
- 拡張性: 24 アクションの静的メソッド分割が将来の機能追加で扱いやすい形か
- FVTT慣行: `HandlebarsApplicationMixin(DocumentSheetV2)` 継承・`Journal.registerSheet()` による登録が正しいか
- 既存設計との整合: UnlinkConfirmDialog / TnxScenarioSettingWizard との接続が維持されているか

#### 良かった点

- `HandlebarsApplicationMixin(DocumentSheetV2)` 継承で JournalSheet 相当の V2 移行を実現。`Journal.registerSheet()` は V1/V2 ともに有効で引き続き使用
- `getData()` → `_prepareContext()` への変換が完全。`this.object` → `this.document` の全面置換も漏れなし
- 動的 import（`user-flag-schema.mjs`）をトップレベル静的 import に移動し、アクションごとの非同期 import を排除。コードが簡潔になった
- jQuery 依存を完全排除。ContextMenu / change イベントリスナーともネイティブ DOM で統一
- `Dialog.confirm()` → `DialogV2.confirm()` への移行を全 4 箇所で実施
- `_handleActionClick()` switch 文（24 case）を 24 の静的メソッドに分割。各アクションの責務が独立し、テストや個別デバッグが容易
- `_onSendInfoToChat()` の重複ロジック（開示済み版・未開示版の同一ループ処理）を `buildSkillHtml()` ヘルパーで統一し、約 30 行のコード削減
- `_onDrop()` は V2 でも同名メソッドのオーバーライドで継続動作
- シナリオシート CSS は 3-CSS-0 で実施済みのため追加作業不要

#### 課題

- ⚠️ `TnxScenarioSettingWizard` はまだ V1（FormApplication）。`render(true)` での呼び出しは動作するが、3-6 での V2 化で正式対応予定
- ⚠️ `Journal.registerSheet()` が V2 ベースクラスに対して正しく機能するかは実機確認が必要（3-7 最終検証で確認）
- ⚠️ ContextMenu の `condition` コールバックが V13 ネイティブ DOM（`el.dataset.uuid`）で正しく機能するかは実機確認が必要

#### 推奨アクション

3-6（ダイアログ系 V2 化）に進む。Journal.registerSheet の実機動作と ContextMenu は 3-7 最終検証で一括確認する。

---

### [ダイアログ系 V2 化: tnx-dialog.mjs / TnxScenarioSettingWizard](フェーズ3-6)

**日付**: 2026-06-07
**レビュー対象**: `scripts/module/tnx-dialog.mjs`, `scripts/module/tnx-scenario-setting-wizard.mjs`, `templates/dialog/card-selection-dialog.hbs`, `templates/parts/scenario-setting-wizard.hbs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: V1 の `new Dialog({...}).render(true)` から `DialogV2.wait()` への変換が全 8 クラスで一貫しているか
- 拡張性: ウィザードの多ステップ状態管理が V2 で維持できているか
- FVTT慣行: `DialogV2.wait()` の `callback` / `close` パターンが標準に準拠しているか
- 既存設計との整合: 各ダイアログが呼び出し元（TnxActionHandler / TnxScenarioSheet / TnxHud）と引き続き正しく接続されているか

#### 良かった点

- `DialogV2.wait()` パターンを全 8 クラスで統一。`close: () => null` でダイアログを × で閉じた場合の戻り値を `null` に統一
- `TnxScenarioSettingWizard` の `FormApplication` → `HandlebarsApplicationMixin(ApplicationV2)` 移行において、多ステップ状態（`this.step` / `this.formData`）がインスタンスプロパティとして V2 でも完全に機能する
- ステップナビゲーションの jQuery `click` リスナー → `static ACTIONS` 変換が完全。スピナーも `data-action="increment"/"decrement"` として統合
- `_updateObject()` の巨大な処理本体を `_executeSetup()` に分離し、`_onFinishSetup` アクションがそれを呼ぶ形に整理。責務の分離が明確
- `CardSelectionDialog` の jQuery 選択トグル処理を DOM レベルで解決: テンプレートを `<label>` ラップ構造に変更し、CSS の `:has(input:checked)` で選択状態を表現。JS が不要になった
- 動的 import（`user-flag-schema.mjs`）をウィザードのトップレベルに移動。V1 時代の `await import()` パターンを排除

#### 課題

- ⚠️ `AmountInputDialog` のバリデーション挙動が V1 と微妙に異なる。V1 はダイアログを閉じずに再入力を促したが、V2 は `null` を返して閉じる。実用上の問題はないが、UX 変更として認識しておく
- ⚠️ `:has(input:checked)` の CSS は現代ブラウザでは広くサポートされているが、Foundry VTT の Electron バージョンによっては動作しない可能性がある。3-7 実機確認で検証する

#### 推奨アクション

3-7（最終検証）に進む。実機確認でカード選択ダイアログの `:has()` 動作、ウィザードのステップ遷移、全ダイアログの開閉を確認する。

---

### 3-7 最終検証・アクトシートタブバグ事後検討（フェーズ3）

**日付**: 2026-06-07
**レビュー対象**: `scripts/journal/tnx-scenario-sheet.mjs`, `templates/journal/scenario-sheet.hbs`（タブ不具合対応の事後検討）
**ステータス**: レビュー済(問題なし)

#### 背景

3-5 で TnxScenarioSheet を V2 移行した後、アクトシートのタブが一切機能しない不具合が3ラウンドのデバッグで修正された（コミット 932da9a → 8587b82 → 00b16c0）。本エントリはその根本原因と設計判断を記録する。

#### 根本原因分析

##### 原因1: `static TABS` 誤記法（コミット 932da9a で修正）

3-5 の実装コミット (a5a51a2) には `static TABS = { primary: { tabs: [...], initial: 'setting' } }` が含まれていたが、これは V1 形式の記法で V2 では機能しない。CastSheet（動作確認済みの V2 参照実装）は `static TABS` を持たず、代わりに `tabGroups` インスタンスプロパティと `_onRender` での `changeTab` 呼び出しで初期タブを設定している。`static TABS` を削除し同パターンを適用した。

##### 原因2: nav 要素の配置ミス（コミット 8587b82 で修正）

3-5 の template では `<nav>` が `<aside class="scenario-sidebar">` の子になっており、`nav.parentElement = <aside>` だった。Foundry V13 の `_onClickTab` は `button.closest(".tabs")` で nav を探し、その `parentElement` 内の `.tab[data-group]` コンテンツを探索する。nav の親が `<aside>` であるとコンテンツが見つからないため、`<aside>` を除去して nav を root section の直接子にした。

##### 原因3: `changeTab` の private field 問題（コミット 00b16c0 で最終修正）

Foundry V13 の `changeTab` 実装 (`foundry.mjs` L27851-27875) は `this.#content.querySelector('.tabs [data-group][data-tab]')` を使うため、nav 要素に `.tabs` CSS クラスが必要。しかし Foundry コアの `body.game .app .tabs { flex-flow: row nowrap }` は特異度 (0,3,1) で我々の `.scenario-sheet .vertical-tabs { flex-direction: column }` (0,2,0) を上書きするため、`.tabs` クラスを追加すると縦型レイアウトが崩壊する。

**解決策**: `TnxScenarioSheet` で `changeTab` をオーバーライドし、`this.#content`（private field = アクセス不可）の代わりに `this.element.querySelectorAll` を使う独自実装に置き換えた。これにより `.tabs` クラス不要で、V13 の private API を回避できる。

#### 設計上の評価

- **Foundry V13 の changeTab は実質 non-public API**: `.tabs` CSS クラス要件・`#content` private field のいずれも公式ドキュメントに明記されていない。V2 移行時はこの仕様を前提にした設計が必要
- **`changeTab` オーバーライドパターンの汎用性**: 縦型タブ・カスタムレイアウトが必要な場合のパターンとして今後も参照できる
- **`_onRender` での `force: true` 再適用**: V2 はレンダー後にタブ `active` クラスを自動付与しない。`_onRender` で `changeTab(tab, group, { force: true })` を呼ぶパターンは全 V2 シートで一貫して適用済み

#### 推奨アクション

フェーズ3 完了。フェーズ4（デザイン刷新）に進む。

---

### 4-0 デザイン方針確定（フェーズ4）

**日付**: 2026-06-07
**レビュー対象**: デザイン方針（実装ファイルなし）
**ステータス**: レビュー済(問題なし)

#### 確定事項

##### 全体トーン: Tokyo Noir Cyber

「ネオン看板が滲む雨夜の東京」— ダーク基調にシアン単色アクセントが光る抑制型サイバーパンク。
スタイル（職業）ごとの色分けは、システム側がスタイル名を認識する必要があるため不採用。アクセントはシアン 1 本に統一。

##### カラーパレット（確定値）

```
/* 背景 */
--tnx2-bg-0: #080b10   /* アプリ最底面 */
--tnx2-bg-1: #0d1117   /* シート本体 */
--tnx2-bg-2: #131920   /* セクション・パネル */
--tnx2-bg-3: #1a2230   /* 入力フィールド */
--tnx2-bg-hover: #1e2a3a

/* テキスト（ハレーション防止: 彩度を落とした cool white） */
--tnx2-text-primary:   #c8d4e0
--tnx2-text-secondary: #7a90a8
--tnx2-text-muted:     #4a5e72
--tnx2-text-accent:    #78c8d8  ← 文字用シアン（明度を下げる）

/* アクセント（装飾・ボーダー専用） */
--tnx2-accent:       #00c8e0  ← ボーダー・グロー専用。文字に直接使わない
--tnx2-accent-dim:   #007a8a
--tnx2-accent-glow:  rgba(0,200,224,0.25)

/* 状態色 */
--tnx2-color-warn:    #e07830
--tnx2-color-danger:  #cc3344
--tnx2-color-success: #44aa77

/* ボーダー */
--tnx2-border-subtle: #1e2d3e
--tnx2-border-active: #2a4a60
--tnx2-border-glow:   #006880
```

**ハレーション防止ルール**: `--tnx2-accent` (#00c8e0) はボーダー・装飾のみ。文字には必ず `--tnx2-text-accent` (#78c8d8) を使う。

##### タイポグラフィ

- 本文・ラベル: システムフォントスタック（日本語必須のためカスタムフォント不使用）
- 数値・コード・見出し: `"Consolas", "Menlo", monospace`
- 見出しデコレーション: フォント変更なし、左ボーダー + `> ` プレフィックステキスト (CSS `::before`)

##### 装飾

| 要素 | 採否 |
|---|---|
| スキャンライン | ✅ (CSS only、気に入らなければ削除) |
| `> ` プレフィックス見出し | ✅ (CSS `::before`、気に入らなければ削除) |
| `[ LEGEND ]` フィールドセット凡例 | ✅ |
| ネオン系ボーダーグロー | ✅ |
| グリッチエフェクト | ❌ (JS 必要 + 可読性低下) |
| アニメーション | 最小限（フォーカス・ホバーの CSS transition のみ） |
| 角丸 | 2px（角張ったサイバーパンク感を優先） |

##### 実装アプローチ

- `tnx2.css` を新設し `system.json` の `styles` に追記（`tnx.css` の後）
- `tnx.css` は移行期間中保持。シートごとに適用完了後、対応ブロックを削除
- フェーズ4 終盤（4-6）で `tnx.css` を廃止し `system.json` から除去
- 新変数系は `--tnx2-*` プレフィックスで管理（旧 `--tnx-*` と衝突しない）

### 5-x TnxRollTable（カード引きロールテーブル）（フェーズ5）

**日付**: 2026-06-11
**レビュー対象**: `scripts/module/tnx-draw-table.mjs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

RollTable ドキュメントのカードベース化。当初 DataModel サブタイプ
（`documentTypes.RollTable.tnxTable` + `CONFIG.RollTable.dataModels`）で実装したが、
**RollTable は `type` / `system` フィールドを持たずサブタイプ非対応**であることが
v13 API リファレンスで確定（ドキュメント作成自体が失敗）。flags ベースに再設計した。

#### 設計判断

- データは `flags.tokyo-nova-axleration.rollTable` に保存
  （`deckMode` / `deckId` / `docResults` 14 スロット / `deckResults` 動的配列）。
- flag 読み出しは `getTableFlagData()` に集約し、欠損キーをデフォルト補完。
- コアの `foundry.applications.sheets.RollTableSheet`（v13 で AppV2 化済）を
  unregister し、`TnxRollTableSheet` を makeDefault で登録 → 全 RollTable が TNX シートで開く。
- シート・flag ヘルパー・ドローユーティリティを 1 モジュールに統合
  （DataModel 廃止により `scripts/data/roll-table/` は削除）。

#### 良かった点

- DocumentSheetV2 + `tag: "form"` + form handler 構成はカードセットアップアプリと同パターンで整合。
- flag スキーマがヘルパー 1 箇所に集約され、User flag ヘルパー方式（CLAUDE.md §3）と思想が揃う。

#### 課題

- ⚠️ コアシート完全置換のため、ユーザーが通常の RollTable（ダイス表）を使いたくなった場合は
  シート選択 UI から戻せない（unregister 済）。TNX では使用場面がない前提で許容（ユーザー承認済）。
- ⚠️ `lookupFromCard` は「開いているシート」のみ対象。閉じているテーブルは発火しない仕様
  （意図的: 表を開いている＝有効化、のセマンティクス）。

#### 推奨アクション

- 対応不要。シーンカード特殊効果システム実装時（~~フェーズ11~~ フェーズ14＝シナリオ進行管理 想定・旧番号 2026-07-03 訂正）に再評価。

#### 差し戻し記録（2026-06-11）

flags ベース実装を含む全コミットを revert（d66636c）。技術的教訓:

- `documentTypes` に RollTable を宣言するとサーバーが documentTypes 全体を無効化し、
  ワールドの全カスタム型ドキュメントが invalid になる（データは温存されるが読み込まれない）。
  system.json の documentTypes に置けるのは system フィールドを持つ Document のみ。
- system.json の変更はブラウザリロードでは反映されない。Foundry 本体（ワールド）の再起動が必要。
- コアシートの unregister + 置換は要求超過だった。既存 UI 保持の要求がある場合、
  render フックでの注入など非破壊的な拡張を第一候補とすること。

#### 再実装記録（2026-06-11・ドロー表）

コミット 8907d2e。差し戻し後の再設計はコア保持・非破壊注入を徹底:

- ドキュメント・シートの置換なし。`preCreateRollTable`（デフォルト名「ドロー表」/
  アイコン card-hands.svg）と `renderRollTableSheet`（DOM 注入）の 2 フックのみ。
- 概要タブ: ロール式 form-group を hidden にしてデッキ選択を後挿入（フォーム値は温存）。
- 結果行: 設定デッキモードは `results.{i}.flags.tokyo-nova-axleration.cardName` を name に
  持つ select を注入し、コアの保存ボタンで他フィールドと一緒に永続化。
  仮想 DoC モードは range 入力に値を転記するだけ（保存フローはコアのまま）。
- フッターのドローボタンは capture リスナー + stopImmediatePropagation で
  コアの drawResult アクションより先に横取り。設定デッキモードでは disabled。
- チャット出力は `table.draw({roll, results})` / `({results, roll: null})` で
  コアのメッセージをそのまま使用。結果の編集 UI（別ウィンドウ）もコアのまま。

課題（軽微）:
- ⚠️ 仮想 DoC で replacement OFF のままだと一度出た結果がロックされる。
  DoC 再現としては replacement ON 推奨（ユーザー操作で設定可能、自動化はしない）。

---

### 5-6 キャストシート詳細タブ + 技能セクション刷新（フェーズ5）

**日付**: 2026-06-12
**レビュー対象**: `scripts/actor/tnx-cast-sheet.mjs`, `scripts/data/actor/cast.mjs`, `templates/actor/cast-sheet.hbs`, `css/tnx2.css`, `scripts/tnx.mjs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: コンテキストメニュー定義の重複が解消されているか
- 拡張性: ライフパスのデータ構造がフェーズ6（ライフパスアイテム実装）に耐えるか
- FVTT慣行: ProseMirror カスタム要素・ContextMenu condition・コア設定デフォルト変更が標準パターンか
- 既存設計との整合: 閲覧/編集モードの切替パターン・既存 CSS クラスの再利用

#### 良かった点

- **コンテキストメニューの共通化**: 閲覧/編集/複製/削除の option オブジェクトを 1 度定義し、
  `baseItemMenu` / `skillMenu` の配列合成で 4 セレクタに配布。`getItemFromHeader()` で
  itemId 解決も一本化（従来は 3 メニューがそれぞれ重複定義）。
- **閲覧/編集をアイテムシートの `_isEditMode` 切替で区別**: シート再利用でドキュメント閲覧と
  編集を 1 つの導線に統合。アイテムシート側の API（`_isEditMode` + `render`）をそのまま使用。
- **技能セクション編集モードのレイアウト再設計**: 閲覧モードの `skills-list-view` /
  `style-skills-list-wrapper` 構造・クラスを再利用し、編集対象セル（Lv・スート）だけ差し替え。
  dragSelector も `.skills-list-view .item` 追加のみで両モードのソートに対応。
- **ケバブメニュートリガーの汎用化**: `.item-menu-trigger` → `closest("[data-item-id]")` に
  contextmenu をディスパッチする方式で、行構造に依存しない。
- **`core.chatNotifications` デフォルト変更**: 設定定義の `default` プロパティのみ変更し、
  ユーザーの明示的選択は保存値が優先される非破壊的手法。存在チェック付きで将来のキー変更にも安全。
- **ProseMirror 固定高さ + `overflow: hidden`**: 自動拡張（height auto / overflow visible）は
  flex 内部スクロールの前提を壊し枠線はみ出しを起こすことを確認 → 固定高さ + 内部スクロールに確定。

#### 課題

- ~~⚠️ ライフパスの `name` はドロップ時のキャッシュであり、参照先アイテムの改名に追従しない。
  フェーズ6 のライフパスアイテム実装時に同期方針（render 時 fromUuid 解決 or キャッシュ維持）を確定する。~~
  → **フェーズ6-0 で方針確定（2026-06-12）**: 表示は毎回 fromUuid で現物解決。キャッシュ名は
  参照先削除時のフォールバック表示のみ。キャッシュ保持方式は全アイテム共通で今後一切採用しない（ユーザー明言）。
- ~~⚠️ `summary` フィールドはドロップ時に空文字を保存するのみ（編集 UI 未提供）。フェーズ6 で連携。~~
  → **フェーズ6-0 で方針確定**: 概要はライフパスアイテムの description をそのまま表示する
  （アイテム側に summary フィールドは追加しない）。実装はフェーズ6-5。

#### 推奨アクション

- ~~フェーズ6 ライフパスアイテム実装時に name/summary の同期設計を行う。それまでは対応不要。~~ → 設計確定済。実装はフェーズ6-5。

---

### 6-0 フェーズ6 着手時の DataModel 変更（フェーズ6）

**日付**: 2026-06-12
**レビュー対象**: `scripts/data/actor/cast.mjs`(player_name 削除)、
`scripts/data/actor/troop.mjs`(heads 追加)、
`scripts/data/item/*.mjs` 12 種(identificationKey 追加)、
`scripts/data/item/common/outfit-base.mjs`(構造刷新)、
`scripts/data/item/outfit-categories.mjs`(新設)、
`scripts/data/item/life-path.mjs`(lifePathType choices 化)、対応テスト一式
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: カテゴリ定数が単一モジュールに集約され、DataModel とシートの両方から参照できるか
- 拡張性: buy / hide の 3 状態が将来のシート UI(ドロップダウン + 数値入力)に耐えるか
- FVTT 慣行: StringField の choices / blank の併用、リソースバー用の {value, max} 構造が標準パターンか
- 既存設計との整合: B-0「共通 template の継承戦略」、フェーズ4 の i18n ハードコード方針

#### 良かった点

- **「なし/数値/解説参照」の 3 状態を `{mode, value}` SchemaField で表現**: 文字列 1 本で
  「-」「数値文字列」「解説参照」を持ち分けるより型安全で、シート側の表示分岐も mode の switch で済む。
  `threeStateValueField()` として buy / hide で共用。
- **カテゴリ定数の単一正本化**: `outfit-categories.mjs` の `OUTFIT_CATEGORIES`(大分類 → {小分類: type})
  が choices とシートの連動ドロップダウン(フェーズ6-1)の両方の供給源。Object.freeze で不変化。
  値は日本語表記をそのまま格納(フェーズ4 の i18n ハードコード方針に整合)。
- **lifePathType のキーをキャスト側スロットキーと統一**: origin / experience / encounter で揃え、
  フェーズ6-5 のドロップ時スロット・型検証が単純な文字列比較になる。
- **heads は {value, max} 構造**: トークンリソースバー割り当ての FVTT 標準要件に準拠。バー連携自体は 6-8。
- **template-integrity テストの除外を定数化**: `NON_DATAMODEL_FILES` で helpers / outfit-categories を
  まとめて除外し、今後の非 DataModel ファイル追加にも 1 行で対応できる。
- **テスト 637 → 654 件**: 全変更にスキーマ検証を併設。カテゴリ定数には一意性・type 妥当性・凍結の検証を含む。

#### 課題

- ⚠️ `hack` / `timing` / `exclusive` は StringField(初期値 "-")のまま。形式の確定は
  実装サブフェーズ着手時のユーザー確認に依存(Phase_6_Tasks_Detail.md 残課題)。
- ⚠️ 装備品データは実運用前のためマイグレーション不要と確認済み(2026-06-12 ユーザー回答)。
  以後 outfitBase の構造を変える場合はマイグレーション要否の再確認が必要。

#### 推奨アクション

- フェーズ6-1(装備品シート共通基盤 + general)に進む。

---

### 6-1 アウトフィット共通シート + general(フェーズ6)

**日付**: 2026-06-12
**レビュー対象**: `scripts/item/tnx-outfit-sheet.mjs`(新設)、`templates/item/outfit-sheet.hbs`(新設)、
`scripts/data/item/common/outfit-base.mjs`(hack/timing/part/exclusive 微調整)、
`scripts/module/tnx-skill-utils.mjs`(formatTimingLabel 切り出し)、`scripts/tnx.mjs`(登録)、`css/tnx2.css`(§35 追加)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 拡張性: 固有フィールドを持つ型(weapon 等)が継承で済む構成か
- 保守性: タイミング UI・選択肢がスタイル技能と二重管理にならないか
- FVTT 慣行: ApplicationV2 PARTS/TABS、submitOnChange と配列フィールドの併用
- 既存設計との整合: 既存アイテムシートのタブ構成・view/edit モード・CSS クラス再利用

#### 良かった点

- **継承前提の基底設計**: general は固有フィールドがないため共通シートをそのまま登録。
  6-2 以降は本クラスを継承し PARTS 差し替え + データタブ固有部分の追加で済む。
- **タイミング整形の共通化**: prepareStyleSkillView 内のインライン整形を
  `TnxSkillUtils.formatTimingLabel()` に切り出し、スタイル技能の挙動を変えずにアウトフィットと共用。
  選択肢マップも getSkillOptions() を共用し二重管理を回避。
- **配列操作はスタイル技能の実証済みパターンを踏襲**: 行追加/削除・value 変更時の下位フィールド
  リセットを item.update(配列全体) で行い、submitOnChange の index 消失問題を回避。
- **CSS は既存クラスを再利用**: style-skill-display-grid / settings-grid / array-row /
  tnx-row-btn 等を流用し、新規 CSS はヘッダーカテゴリ行(§35)のみに抑制。
  view-mode の入力無効化も既存 §26 のルールがそのまま適用される。
- **カテゴリ連動の文脈フィルタ**: `_categoriesForType()` が OUTFIT_CATEGORIES から
  その type に有効な大分類・小分類のみを提示。大分類変更時は小分類をリセット。

#### 課題

- ~~実機確認未了~~ → 実機確認済み(2026-06-14)。
- ~~extensible スロット UI~~ → 6-2 で実装済み。
- ~~真偽値フラグの表示ラベル~~ → 全フラグの意味をユーザーが言語化し確定(Outfits.md 参照)。

#### 推奨アクション

- ユーザー実機確認の後、フェーズ6-2(weapon / armor / cyborg)に進む。

---

### 6-2 戦闘系装備シート(weapon / armor / cyborg)(フェーズ6)

**日付**: 2026-06-12
**レビュー対象**: `scripts/item/tnx-outfit-sheet.mjs`(型対応拡張)、`templates/item/outfit-sheet.hbs`、
`scripts/data/item/common/extensible.mjs`(プール方式再設計)、`scripts/data/item/weapon.mjs`、
`scripts/data/item/common/outfit-base.mjs`、`scripts/tnx.mjs`(登録・ヘルパー)、対応テスト一式
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 拡張性: 6-3/6-4 の型追加が同じパターン(サマリ組み立て + 型別フィールドセット)で済むか
- 保守性: サブクラス分割せず単一シート + type 判定にした判断の妥当性
- FVTT 慣行: 配列フィールド(slots / damageType)の保存方法

#### 良かった点

- **サブクラスではなく単一シート + type 判定**: 6-1 時点ではサブクラス前提だったが、
  型差分が「サマリの並び」と「フィールドセットの有無」に収まるため、`view.summary` の
  {label, value} 配列組み立てと {{#if isWeapon}} 等の条件分岐で吸収。テンプレート 1 枚・
  クラス 1 つで 4 型(将来 10 型)を扱え、登録も types 配列の拡張のみ。
- **スロットのプール方式**: 装備側が数のみを持つというユーザー確定を
  `slots[] {kind, count}` で素直に表現。型ごとの既定プールは `SLOT_PRESETS` 定数 +
  `_normalizedSlots()` の表示時正規化で生成し、保存データを汚さない。
- **配列フィールドの保存は全体更新**: slots / damageType / timing とも、
  チェックボックス・スピナーから配列全体を組み立てて update する方式で統一
  (submitOnChange の index 部分更新によるデータ破壊を回避)。
- **quantity {value, max} の二層**: 「消費しても経験点が復活しない」を、現在個数と
  常備化個数の分離で構造的に保証。

#### 課題

- ~~実機確認未了~~ → 実機確認済み(2026-06-14)。
- ⚠️ オプション装備連携は 6-6(キャストシート)へ移管。消費アイテムのセッション復活・
  EXP 連動も未実装(構造のみ)。
- ~~旧データの validation エラー~~ → migrateData 実装済み(6-4 DataModel 全面統一)。

#### 推奨アクション

- ユーザー実機確認の後、フェーズ6-3(ianus / tron / tap)に進む。

---

### 6-4 生活系装備シート(vehicle / residence / housingArea / combiner)(フェーズ6)

**日付**: 2026-06-13
**レビュー対象**: `scripts/data/item/{vehicle,residence,housing-area,outfit-categories}.mjs`、
`scripts/item/tnx-outfit-sheet.mjs`(_prepareView 型別 switch 化)、
`scripts/item/tnx-item-sheet.mjs`(スピナー汎用化)、`tnx-housing-area-sheet.mjs`(新設)、
`templates/item/{outfit-sheet,housing-area-sheet}.hbs`、`scripts/tnx.mjs`(登録)、対応テスト
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: 型ごとに異なる概要順序の表現方法
- 拡張性: アウトフィットでない型(housingArea)の扱い
- 整合: 数値スピナーの重複解消

#### 良かった点

- **_prepareView を型別 switch に再整理**: 6-2/6-3 の加算的な条件分岐では型ごとに異なる
  概要の「順序」(例: cyborg は防→攻、vehicle は攻→SF→防)を正確に出せなかったため、
  型ごとに push 順を明示する switch に変更。可読性と正確性が両立。
- **housingArea を専用シートに分離**: 「アウトフィットではない」というルールに忠実に、
  outfitBase を前提とするアウトフィットシートを流用せず、説明 + 修正値のみの軽量シートを新設。
  MOD_FIELDS 定数でラベル・順序を一元管理。
- **数値スピナーの基底汎用化**: incrementField/decrementField を TokyoNovaItemSheet に移し、
  アウトフィット・住宅エリア両シートで共用(重複定義を解消)。今後の全アイテムシートで再利用可。
- **カテゴリ配列化の効果**: 6-2 で小分類→type を配列化していたため、general への移動(住宅
  オプション/アクセサリ・IANUSオプション)が対応表 1 行の修正で済んだ。
- **攻撃/防御フィールドの出し分け**: hasAttack/hasGuard/hasDefence フラグで、vehicle(攻あり
  受けなし)のような型差分をテンプレートの条件分岐 1 つで表現。

#### 課題

- ~~コンバイナーの合成連携は未実装~~ → 6-4 追補・追補 2 で実装済み(住宅エリア連携・コンバイン連携エントリ参照)。
- ~~実機確認未了~~ → 実機確認済み(2026-06-14)。
- ⚠️ residence/combiner は outfitBase の appearancePenalty/hack を内部的には保持(非表示なだけ)。
  データ的な無駄だが、型ごとに outfitBase を分割するより単純さを優先した。

#### 推奨アクション

- ユーザー実機確認 + コンバイナー設計確認の後、フェーズ6-5(lifePath シート + キャスト連携)に進む。

---

### 6-4 追補 住宅エリア連携・コンバイン連携(フェーズ6)

**日付**: 2026-06-13
**レビュー対象**: `housing-area.mjs`(area + MOD_FIELDS 集約)、`combiner.mjs`(combine 構造)、
`tnx-outfit-sheet.mjs`(住宅エリア解決・コンバインプレビュー・ドロップ受付)、`system.json`(pack)、
`tnx-cast-sheet.mjs`(ドロップ拒否)、対応テスト
**ステータス**: レビュー済(問題なし)

#### 良かった点

- **ドロップ受付の汎用化**: `_onDropZone` を data-drop-area で分岐させ、住宅エリア(residence)と
  コンバイン元(combiner)を 1 つのハンドラ + DragDrop バインドで処理。V2 の手動バインドは
  ドロップゾーン存在時のみ行う。
- **修正値ラベルの単一情報源**: `HOUSING_AREA_MOD_FIELDS` を DataModel モジュールに置き、
  住宅エリアシート(編集)と住宅施設シート(供給値表示)で共用。
- **合算の局所化**: 住宅施設の実効値算出は `_prepareView` の residence ケースに areaMods を渡して
  加算するのみ。エリア未設定時は加算 0 で同じ経路。
- **compendium ドロップダウンの動的構築**: pack index から {uuid: name} を生成。
  ドロップ(ワールド item)時は UUID 接頭辞でドロップダウンを無効化する素直な判定。
- **コンバインプレビューの確定的合成**: 二つの元を live 解決し、ルールが一意な項目
  (部位/分類/電制/隠)のみ算出。曖昧な param 取捨選択は構造(combine)を用意しつつ UI を保留。

#### 課題

- ~~コンバイナーの param 取捨選択 UI は未実装~~ → コンバイナーシート UI 刷新(5bb2455)で実装済み。
- ⚠️ residence の `housingArea` は単一リンク。取得ごとに自由組み合わせという原則とは別に、
  アイテムとしては 1 エリアを保持(実効値算出は 6-6)。
- ~~実機確認未了~~ → 実機確認済み(2026-06-14)。compendium は空で出荷(著作権配慮)。

#### 推奨アクション

- 実機確認 + コンバイナー param UI の対象確認の後、フェーズ6-5 へ。

---

### 6-4 追補 2: modeValueField 一般化・全 DataModel {mode,value} 統一・シート全面対応(フェーズ6)

**日付**: 2026-06-14
**レビュー対象**: `scripts/data/item/helpers.mjs`（modeValueField 昇格）、全 DataModel(outfit-base / extensible /
armor / cyborg / tap / vehicle / ianus / weapon / housing-area / residence)、
`scripts/item/tnx-outfit-sheet.mjs`・`templates/item/outfit-sheet.hbs`（全面 modeValueField 対応）、
`scripts/actor/tnx-cast-sheet.mjs`・`templates/actor/cast-sheet.hbs`（報酬点ボタン + modeValueField 表示）、
`css/tnx2.css`（対応スタイル）
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: modeValueField の単一情報源化
- 整合: migrateData の実装統一性
- パフォーマンス: シート描画時の mode チェック
- 拡張性: ActiveEffect フェーズとの接続

#### 良かった点

- **modeValueField の helpers.mjs 昇格**: outfit-base.mjs に局所定義されていた関数を
  `helpers.mjs` へ昇格し、全 DataModel で共用。DRY を徹底し同じパターンが複数ファイルに
  点在するリスクを解消。
- **migrateData の統一パターン**: 各 DataModel に `static migrateData(source)` を実装。
  旧 NumberField → `{mode: "value", value: n}` / `{mode: "none", value: 0}` への変換ロジックが
  一貫しており、実運用前データへの対応として信頼性が高い。
- **defenceField への mode 追加**: defence(S/P/I 三値)に mode フィールドを追加することで、
  「防がない」状態と「防が 0/0/0」を区別できる。シート表示の正確性が向上。
- **報酬点 ±1 ボタン**: 閲覧モードのまま報酬点を調整できる。ホバー表示で常時目に入らず、
  かつ下限 0 の無効化もある。D&D 5e の装備済みトグルと同じ「閲覧モード中の操作」パターン。
- **isCyber ※マーク修正**: サイバーウェア大カテゴリに自動的に ※ が付くのを防止。
  カテゴリ自体がサイバーウェアであることを示すため二重マークになる問題の修正。

#### 課題

- ⚠️ `{mode,value}` フィールドの ActiveEffect による修正方式（`OVERRIDE` か `ADD` か）は未定。
  ActiveEffect フェーズで設計が必要。
- ~~実機確認未了~~ → 実機確認済み(2026-06-14)。mode 切り替えの動作を確認。

#### 推奨アクション

- ユーザー実機確認の後、フェーズ6-5(lifePath シート + キャスト詳細タブ連携)に進む。

---

### 6-5 lifePath シート + キャスト詳細タブ連携（フェーズ6）

**日付**: 2026-06-14
**レビュー対象**: `scripts/item/tnx-life-path-sheet.mjs`(新設)、`templates/item/life-path-sheet.hbs`(新設)、
`scripts/actor/tnx-cast-sheet.mjs`(型検証・fromUuid 解決・ContextMenu)、
`templates/actor/cast-sheet.hbs`(カードレイアウト)、`scripts/data/actor/cast.mjs`(summary 削除)、`css/tnx2.css`(カード CSS)
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: live 解決パターンが既存の fromUuid 方針と整合しているか
- 拡張性: lifePathType 種別チェックが将来の型追加に対応できるか
- FVTT 慣行: ContextMenu の data-context-menu 属性・_activateContextMenus のバインドパターン
- 既存設計との整合: 閲覧/編集モードの toggling・ContextMenu の閲覧/編集/削除セット

#### 良かった点

- **summary フィールドの削除**: cast DataModel の lifePath スロットから `summary` を削除し、
  description の表示を完全に fromUuid live 解決に一本化。データ二重管理の問題を根本から排除。
  キャッシュは削除時フォールバック用の `name` フィールドのみに限定し、フェーズ6-0 のユーザー方針を貫徹。
- **型検証の単純設計**: `lifePathType` 空文字→どのスロットでも受付、非空文字→スロットキーと比較、
  lifePath 以外の型→即拒否、という 3 段階のチェックを `_onDropItem` に局所化。
  lifePathType のキーとスロットキー（origin/experience/encounter）を統一したことで比較が単純な `!==` のみ。
- **ContextMenu の一貫したパターン**: 既存の skills / style-skills の閲覧/編集/削除と同じ構造を
  `_activateContextMenus` に追記。`data-context-menu="lifepath-item"` で対象を絞り、
  `header.dataset.lifepathKey` でスロットキーを解決する方式。
- **カードレイアウトの CSS 変数活用**: 既存の `--tnx2-*` 変数のみで構成（新変数なし）。
  `.lifepath-card` の左ボーダーに `--tnx2-accent-dim` を使い、ヘッダー下のボーダーで description を分離。
  空スロットは `opacity: 0.6` で簡潔に表現。
- **lifePath アイテムシート**: TokyoNovaItemSheet を継承し、説明（ProseMirror）+ 設定（種別・習得技能・識別キー）
  の 2 タブ構成。organization-sheet.hbs と同じ `tnx-fieldset` パターンで実装。

#### 課題

- ⚠️ `skillName` の自動化（固有名詞技能の自動生成）は設計未確定。フェーズ未定の実装タスク。
  文字列フィールドのままにしておけばよく、シートUIへの影響はない。
- ⚠️ 実機確認は未了（ユーザー確認待ち）。

#### 推奨アクション

- ユーザー実機確認の後、フェーズ6-6(キャストシート「アウトフィット」タブ整備)に進む。

---

### 6-6 キャストシート「アウトフィット」タブ整備（フェーズ6）

**日付**: 2026-06-14
**レビュー対象**: `scripts/actor/tnx-cast-sheet.mjs`（アウトフィットタブ・集計ハンドラ）、
`scripts/data/actor/cast.mjs`（新フィールド群）、`scripts/data/item/common/outfit-base.mjs`（フィールド追加）、
`scripts/data/item/combiner.mjs`（isCombineActive / combatSpeedMod 追加）、
`scripts/item/tnx-outfit-sheet.mjs`（オプション品 UI・deactivateCombine）、
`scripts/tnx.mjs`（集計関数・フック）、`templates/actor/cast-sheet.hbs`（タブ改称・UI 追加）、`css/tnx2.css`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: アウトフィット集計ロジックの lokalization（hook 内 vs DataModel vs tnx.mjs）
- 整合: OUTFIT_TYPES の定義重複（cast-sheet.mjs と tnx.mjs で同一セットを持つ）
- 拡張性: outfitMod フィールドと ActiveEffect フェーズの接続可否
- FVTT 慣行: updateItem フック内での `actor.update` 呼び出し（再帰防止の syncing フラグ）

#### 良かった点

- **syncing フラグによる再帰防止**: `updateCastAppearanceModifier` / `updateCastOutfitMods` が
  `actor.update(…, { syncing: true })` を発行し、updateActor フック側の既存ガード（`options.syncing` チェック）と
  整合。EXP 再集計と同じパターンを踏襲しており一貫性がある。
- **差分チェックによる不要 update の回避**: 集計値が変わらない場合は `actor.update` を呼ばない。
  初期化時（ready）・フック時の両方で同じガードを通すことで無限ループおよびパフォーマンス劣化を防ぐ。
- **グループ別列定義を OUTFIT_GROUP_CONFIG として宣言**: 各大分類の表示列（キー・ラベル）を
  static 定数にまとめ、コンテキスト準備とテンプレートが同じ設定を共有する。
  将来の列追加・並び替えが定数一箇所の変更で済む。
- **グループ内ソート制限の実装**: `_onSortItem` でアウトフィット同士のドロップを
  「同じ表示グループ（同 DOM ルート）内のみ」に限定。グループをまたぐドロップは黙って無視し、
  ユーザーへの混乱を防ぐ。コメントでグループ内限定の理由も明記。
- **isGhost → CS 修正の再集計**: ゴースト登場中は物理現場不在のため TAP の CS 修正を無効化する
  ゲームルールの挙動を、`isGhost` 変更時の updateActor フック内で `updateCastOutfitMods` を
  再実行することで正確に反映。
- **bountyBase 再計算ボタン**: 外界値（mundaneTotalValue）は常に現在値から動的に計算される一方、
  `bountyBase` は手動確定値として DB に永続化する設計を明確化。
  ボタンのツールチップに現在の外界値を表示し、確定の意味が視覚的にわかる。

#### 課題

- ⚠️ `OUTFIT_TYPES` セットが `tnx-cast-sheet.mjs`（`OUTFIT_TYPES` static）と `tnx.mjs`
  （`OUTFIT_ITEM_TYPES` const）に重複定義されている。単一ソース化は後日整理タスク
  （どちらも正確なセットであり現在の動作には影響しない）。
- ⚠️ `outfitMod.reason` / `.passion` / `.life` / `.mundane` の集計ロジックは現在 0 返し
  （対応フィールドが各アウトフィット型に未定義）。ActiveEffect フェーズで各アウトフィット型に
  対応修正値フィールドが追加された際に集計関数を拡張する。
- ⚠️ `isCombineActive` フィールドは追加されたが、コンバイナー「起動/停止」UI は未実装。
  `deactivateCombine` アクションのハンドラ実装も 6-6 スコープ外（フェーズ未定）。

#### 推奨アクション

- ~~フェーズ6-7（guest シート）へ進む。~~ → **フェーズ11 に移管（2026-06-14）**
  guest / troop / extra シートと戦闘タブは判定・神業システム完成後に実施する。
- ~~`OUTFIT_TYPES` 重複（`tnx-cast-sheet.mjs` の `OUTFIT_TYPES` と `tnx.mjs` の `OUTFIT_ITEM_TYPES`）は
  6-10 仕上げ時またはフェーズ11 着手前にリファクタする。現在の動作には影響なし。~~
  → **解消（2026-07-03・フェーズ11-0、本体コミット `5b07a9f`）**: `scripts/data/helpers.mjs` の
  `OUTFIT_ITEM_TYPES`（9-2 で tnx.mjs から helpers に移設済み）に一本化し、cast シートの
  static `OUTFIT_TYPES` を削除。挙動不変・テスト980件 green。

---

### [TnxCheckEngine — 判定計算エンジン](フェーズ8-2)

**日付**: 2026-06-15
**レビュー対象**: `scripts/module/tnx-check-engine.mjs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: 計算ロジックを UI から完全分離。純粋関数 + 定数で構成。テストが書きやすい。
- 拡張性: AE ボーナス追加時は呼び出し側の `abilitiesCtx` を拡張するだけ。エンジン関数の署名変更は不要。
- パフォーマンス: 単純な算術計算のみ。問題なし。
- FVTT 慣行: Foundry の Cards API や Actor 型に依存しない純粋計算層として設計。
- 既存設計との整合: `SUIT_TO_ABILITY` を `TnxCheckFlow` と `TnxHud` が共有インポートで参照。重複定義なし。

#### 良かった点

- FUMBLE_VALUE / FIXED_21 を Symbol で表現したことで、`if (result === FUMBLE_VALUE)` という型安全な判定が可能。マジックナンバーや文字列定数に比べて誤マッチがない。
- `getComboSuits` を純粋関数として分離したことで、usage 側のロジックから独立してテスト可能。

#### 課題

- ⚠️ `getAbilityBySuit` は `abilitiesCtx[key].totalValue` を参照するが、`totalValue` の計算は `TnxCheckFlow._buildAbilitiesCtx` で行っており、それが AE を考慮しない暫定実装。ActiveEffect フェーズで修正が必要。

#### 推奨アクション

- ActiveEffect フェーズ: `_buildAbilitiesCtx` を廃止して `prepareDerivedData` の出力（`actor.system[key].totalValue`）を使うように移行する。

---

### [TnxCheckFlow / TnxCheckDialog — 判定フロー制御](フェーズ8-3)

**日付**: 2026-06-15
**レビュー対象**: `scripts/module/tnx-check-flow.mjs`, `scripts/module/tnx-check-dialog.mjs`
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: `TnxCheckFlow` が Singleton 静的クラスとして状態を一元管理。`_context` / `_trumpMode` の二変数のみ。
- 拡張性: ダイアログクラスを `tnx.mjs` でインジェクションすることで循環 import を回避。将来のダイアログ実装変更に強い。
- パフォーマンス: 判定ごとに1回だけ `_buildAbilitiesCtx` を呼び出す。HUD の `_prepareContext` でも呼ぶが、Actor のアイテムスキャンは許容範囲。
- FVTT 慣行: `ApplicationV2 / HandlebarsApplicationMixin` 準拠。`_onClose` のオーバーライドでキャンセル処理を実装（V2 のライフサイクル活用）。
- 既存設計との整合: `TnxActionHandler.playCard` を介してカードプレイ操作を委譲。HUD の `_onPlayCard` が `TnxCheckFlow.isPending` を確認してインターセプト。

#### 良かった点

- HUD 中心フローにより、カード選択のフィードバック（ゴールドグロー・グレースケール）を HUD の CSS で一元管理できる。
- 報酬点フローをカード選択後の別プロンプトに移動したことで、スート不一致時のスキップが自然に実装される。

#### 課題

- ⚠️ `TnxCheckFlow._buildAbilitiesCtx` はプライベート命名だが `TnxHud` から直接呼ばれている。将来的に `tnx-check-engine.mjs` に `buildAbilitiesCtx` をエクスポートする関数として昇格させると依存関係が明確になる（現時点では機能上の問題なし）。

#### 推奨アクション

- ActiveEffect フェーズ: `_buildAbilitiesCtx` を廃止し `actor.system[key].totalValue` を直参照するよう移行。`TnxHud` 側のプレビュー計算も同時に修正。
- 次は **フェーズ6-10（仕上げ・wiki 整合・v0.7.0）**。

---

### キャストシート ProseMirror 編集→閲覧切替時の自動保存(バグ修正)

**日付**: 2026-06-15
**レビュー対象**: `scripts/actor/tnx-cast-sheet.mjs`（`_onToggleEditMode`）
**ステータス**: レビュー済(問題なし)

#### レビュー観点

- 保守性: 修正が最小範囲に収まっているか
- FVTT 慣行: ProseMirror カスタム要素の `value` / `getAttribute("name")` の正しい使用
- 既存設計との整合: `actor.update()` トリガーの render タイミングと `_isEditMode` トグルの順序

#### 問題

`_onToggleEditMode` が同期で `this.render()` を呼ぶため、ProseMirror 要素に入力した内容が
未保存のまま再描画で消えていた。`form.submitOnChange` は ProseMirror の変更を拾わないため、
明示的な保存処理が必要だった。

#### 修正（初回）

- メソッドを `async` に変更。
- 編集→閲覧切替時（`this._isEditMode === true`）に、DOM 内の `prose-mirror` 要素を全走査。
- 各要素の `getAttribute("name")` でフィールドパス、`pm.value` で現在の HTML 内容を取得。
- `this._isEditMode = false` を**先にセット**してから `actor.update(updates)` を呼ぶ。
  → `actor.update` が発火する render フックの時点で既に閲覧モードのため、フラッシュなし。
- ProseMirror 要素が存在しない場合（他タブ等）は元の `toggle + this.render()` にフォールバック。

#### 追補バグ修正（2026-06-15）

**問題**: `await actor.update()` の後に `this.render()` を呼ばず `return` していたため、
説明文が未変更の場合に Foundry が「差分なし」と判定して更新フックを発火せず、
シートが再描画されないまま視覚的に編集モードのまま固まっていた。
`_isEditMode` は `false` に更新済みのため「閉じて開くと閲覧モード」になる副症状もあった。

**修正**: `await actor.update()` の直後に `this.render()` を追加。
`actor.update` の差分最適化の有無に関わらず確実に view モードで再描画するようにした。

#### 良かった点

- `_isEditMode = false` を先にセットすることで `actor.update` トリガーの再描画が直接ビューモードで
  行われ、二重描画・フラッシュを回避。
- 変更範囲が `_onToggleEditMode` 1 メソッドのみ。ProseMirror 設定コードは無変更。
- ProseMirror 要素が存在しない場合にフォールバックする後方互換性を保持。

#### 課題

なし。

---

### 部位（part）管理システム（フェーズ10-1）

**レビュー対象**: `outfit-base.mjs`（part 行スキーマ・isOption 派生）、`part-helpers.mjs`（表示/占有 純粋関数）、`cast.mjs`/`tnx-cast-sheet.mjs`（partSlots・占有パネル・一覧入れ子）、`tnx-outfit-sheet.mjs`（部位エディタ・装備先UI）、`part-slot-preset-app.mjs`（プリセット/編集App）、`tnx.mjs`（流し込み・既定自動設定）
**ステータス**: ✅ 実装完了・実機確認済（2026-06-27。基盤＋UI＋占有計算①②③＋部位表示ルール＋準備状態入れ子＋スロットゲージ）

**追記（2026-07-01）**: 部位スロットプリセット/キャスト部位の編集リスト（`PartSlotPresetApp`）に**行の手動並び替え（グリップのドラッグ＆ドロップ）**を追加。専用グリップのみ `draggable`（テキスト入力の選択を奪わない）、drop 時に `_harvest` で未保存編集を保ってから `_rows` を並べ替え→ render→保存で順序永続化。**TNX 標準方針＝表・リスト系は既定で並び替え可能**（ユーザー確定・[[feedback-tables-sortable-by-default]]）。あわせて `DEFAULT_PART_SLOT_PRESET` 外付け系に「住宅」追加（住宅施設の装備先部位・既存ワールドは要再適用）。

#### レビュー観点（保守性/拡張性/FVTT慣行/整合）

#### 良かった点

- **加算的スキーマ変更で後方互換**: value/slots/isOption/parentItemId を温存し新フィールドを追加。kind 既定 "other" で旧 {value,slots} データを自由記入へ無損移行（§4.2 文字列推測なし）。既存 857 テスト green。
- **エイリアス（occupiesOther/targetPart/targetCount）で排他部位を汎用化**: 両手持ち=片手持ち×2 をデータで表現でき、公式に無い排他やルール追加にコード変更なしで対応（YAGNI を侵さず将来対応力を確保）。
- **プリセット編集 App は既存カード設定様式を踏襲**（standard-form・number-input-spinner・行追加/削除）。設定は world Array、preCreateActor で新規キャストへ deepClone 流し込み。
- **対象欄の文脈連動（D&D 消費 UI 参考）**: ワールド直下=自由入力／キャラ所属=所有物ドロップダウン。アイテム名は名前ライブ照合（UUID キャッシュ非保持＝[[feedback-no-stale-caches]] と整合）。

#### 課題

- part 行スキーマがフィールド多数（kind/value/slots/isOptional/host*×5/refSubKind）。表示・占有・エディタの各所で kind 分岐が増えるため、純粋関数（占有計算・表示ラベル組み立て）へ早期に切り出してテストで固定する。
- 旧 isOption/parentItemId/parentSlotKind は当面温存（退行回避）。新オプションモデル（part kind=option＋host*）への一本化と旧フィールド撤去は UI 完成後に別途。
- プリセットは config:false の world Array で DataModel 検証が効かない。編集 App の #onSubmit で整形（空 value 除去・非エイリアス行の target* クリア）して担保。

#### 推奨アクション

- 占有計算・部位名表示ルールを Foundry 非依存の純粋関数（helpers）に実装し単体テスト（任意・非消費3型・エイリアス展開・relation・表示コネクタ）。
- UI 安定後に旧 isOption 系の撤去を別コミットで計画。

#### 実装結果（2026-06-27）

- 占有/表示の純粋関数を実装（`computePartOccupancy`／`computeHostOccupancy`／`formatPartDesignation`／`formatOptionLabel` 等・part-helpers 40 テスト）。推奨どおり Foundry 非依存で固定。
- **旧 isOption は「撤去」でなく「部位 kind=option から派生」へ転換**（`prepareDerivedData`）。`parentItemId`/`parentSlotKind` は孤児化していたが、撤去された装備先 UI を**部位カード最下段に再建して活用**（撤去せず）。→ 課題「旧フィールドの一本化/撤去」は派生＋再建という形で解消（別コミットでの撤去計画は不要に）。
- 部位表示は正本（ユーザー提供の部位名リスト）に逐条準拠（**例外は特殊弾のみ**）。一覧は準備済みオプションのみ入れ子（回路トレース調）＋スロット保有ホストにスロット使用ゲージ（種別ごと色分け）。
- UI 整合（[Sheets.md](../01_Wiki/UI/Sheets.md) 等）の最終更新は 10-8 で実施予定。

---

### 技能の組み合わせ解決・用途エディタの自動構成（フェーズ10-2＋判定）

**レビュー対象**: `skill-chain-resolution.mjs`（純関数）、`skill-dictionary.mjs`（辞典ローダ）、`tnx-usage-sheet.mjs`（用途設定時の解決 enforcement・個数制限・トリムダイアログ）、`tnx-skill-utils.mjs`（comboSkill 表示逆引き）、`tnx-style-skill-sheet.mjs`/`style-skill-sheet.hbs`（カスケードUI・対決分離）、`tnx-cast-sheet.mjs`/`item.mjs`（ロール挙動・用途不備検知）、`style-skill-acquisition.mjs`（自動取得）
**ステータス**: ✅ 実装完了（純関数層テスト済み）・実機レビュー済（2026-07-02・問題なし＝enforcement 収束も確認。具体データ依存の細部は未検証のまま持ち越し）

#### 良かった点

- **解決ロジックを Foundry 非依存の純関数に分離**（`skill-chain-resolution.mjs`）: `resolveSkillChain`（クロージャ走査・アクション=ベース・循環防止）／`resolveUsageSkills`（actor アイテムへ橋渡し・seed 閉包・全体でのベース再決定）／`comboLockAnalysis`+`isComboRequired`（トリムのロック）／`mandatoryComboNames`/`alternativeComboNames`。ユーザー裁定をテストで固定し UI と分離。
- **判定は用途解決済みを前提**（解決は用途設定時）。判定起動側は据え置きで augment（既存解決経路を壊さない）。
- **表示の都度逆引き**（識別キー→技能名）で保持名キャッシュを持たない（[[feedback-no-stale-caches]] と整合）。
- **ベース固定はハードロックでなく候補制限**（指定技能＋代用）。「自動確定＝既定値・ロックではない／候補が増えれば選択可能」の合意を満たす。
- **アクション技能はコンボから一律除外**（ベース専用）で、代用ベース時の旧アクション残留・ベース入れ替わりでの降格を構造的に防止。

#### 課題

- **enforcement を `_onRender` で実行**（冪等・変更時のみ update→再レンダリングで収束）。リアルタイム反映優先だが、レンダリング中の自己 update は収束を冪等性に依存。実機で無限レンダリングが無いことの確認が必要（あれば再入ガード追加）。
- **トリムダイアログのインタラクティブ部（DialogV2 actions のロック再計算）は engine 未検証**。data-action トグルボタン方式（既存 AmountInputDialog の actions 慣行に倣う）。
- `tnx-usage-sheet`/`tnx-cast-sheet` は複数機能が同居し肥大化。将来 enforcement 周りの責務分離を検討。

#### 推奨アクション

- 実機で render時 enforcement の収束とトリムダイアログのインタラクティブ挙動を確認。問題あれば再入ガード／ダイアログ機構を調整。
- 10-8 で関連 wiki（Usage_System 反映済み・[Check_System.md](../01_Wiki/Game_Mechanics/Check_System.md)）の整合最終化。

### 派生データ・変異器官・式神装備フラグ／専用の辞典参照化／部位プリセット並び見直し（フェーズ10-2 後追い）

**レビュー対象**: `outfit-base.mjs`（フラグ群＋`SHIKI_TYPES`）、`part-helpers.mjs`（`PART_HOST_FEATURE_LABELS`）、`tnx-outfit-sheet.mjs`/`outfit-sheet.hbs`（特性セクション・派生データ参照UI）、`style-skill-acquisition.mjs`（`autoImportDerivedData`）、`tnx.mjs`（`createItem` フック）、`tnx-cast-sheet.mjs`（経験点除外）、`part-slot-preset-app.mjs`（並び）
**ステータス**: ✅ 実装完了（全テスト green・eslint 0 error）・実機レビュー済（2026-07-02・問題なし。派生データ等の具体データ依存項目は未検証のまま持ち越し）

#### 良かった点

- **自動取得の既存パターンを踏襲**（[[feedback-reuse-existing-ui-precedent]]）: 派生データ参照UI（ドロップエリア＋参照ボタン＋右クリック閲覧/編集/削除）はスタイル技能の取得UI（`tnx-style-skill-sheet`）と同形。ドロップは既存 `_onDropZone`（`data-drop-area` ディスパッチ）に `derived-data` 分岐を足すだけで流用。
- **自動派生インポートは自動取得モジュールに同居**（`autoImportDerivedData`）。`createItem` フックは `styleSkill`→自動取得 / `hasDerivedData`→派生インポートに分岐し、責務が一貫。
- **派生データ参照は名前ライブ解決**（`_resolveDerivedRefs`＝fromUuid・削除済みは「(不明)」フォールバック）でキャッシュを持たない（[[feedback-no-stale-caches]] と整合）。
- **経験点二重計上回避を集計側で1行除外**（cast-sheet の `isDerivedData→0`）。既存の `isCheckAcquired`/`preserveExp.mode` 判定列に同居し augment。
- **式神タイプの表示位置は summary 配列へ splice 挿入**（部位行の直前）。型別の push 分岐を増やさず1箇所で「部位の1つ前」を実現。
- **専用の辞典プルダウンは前フェーズの辞典基盤を再利用**（`loadSkillChoices([STYLE_PACK])`/`[ORGANIZATION_PACK]`）。`exclusive` を `{type,key}[]` 配列化し、行ごとに2オプショングループ＋選択フラグを事前計算（hbs で `eq`/`concat` を使わず `opt.selected` をベイク）。**UI はタイミング欄の `.array-row`/`.row-controls`/`.tnx-row-btn`(+)・`tnx-row-btn--delete`(ゴミ箱) を再利用**（1行目＝＋、2行目以降＝削除）——独自 CSS を持たず比率も view-mode 無効化も既存 CSS に委ねる（初版で `.tnx-btn` の `width:100%` を流用し削除ボタンが伸びた件を是正）。保存は部位行と同じ全体更新（`_updateExclusive`・空でも1行から）。`migrateData` で旧 string→[] を吸収。

#### 課題

- 特性セクションは純フラグ（サイバーウェア/消費/変異器官/式神/派生データ）に整理し、**表示に関わる設定（式神タイプ・派生データを持つ＋派生データ参照UI）を「アウトフィット 詳細設定」へ集約**（タイプ＝電脳制御値の直下／派生データを持つ＝使用回数の上）。`hasDerivedData` ON 時は `isDerivedData` チェックを非表示（派生元は自身が派生データでない）。（2026-06-29 再配置）
- 派生データのドロップは型を限定していない（任意 Item を受ける）。`isDerivedData` は outfit-base にしか無いため非アウトフィットでは無視されるが、誤ドロップ時の警告は今は出さない（YAGNI・必要になれば追加）。

#### 推奨アクション

- 実機で特性セクション・派生データUI・説明タブのタイプ・自動派生インポート・「その他特徴」変異器官を確認。
- 部位プリセット並びは既存ワールドへは自動反映されない（初回ロードのみ）。既存ワールドはプリセット編集 UI で再適用。

### スタイル技能の取得・経験点（経験点消費なし／取得数表示／レベル自動参照）（フェーズ10-3）

**レビュー対象**: `style-skill.mjs`（`expFree`/`excludeFromCount`/`levelRef`＋`prepareDerivedData`）、`style-skill-acquisition.mjs`（純関数 `resolveLevelRef`/`groupStyleSkillsByStyle`）、`tnx-style-skill-sheet.mjs`/`style-skill-sheet.hbs`（取得・経験点UI）、`tnx-cast-sheet.mjs`/`cast-sheet.hbs`（EXP・連動ガード・スタイル別グループ化＋取得数見出し）、`tnx2.css`（`.style-skill-group-header`）
**ステータス**: ✅ 実装完了（全テスト green 967・eslint 0 error・stylelint clean）・実機レビュー済（2026-07-02・問題なし）

#### 良かった点

- **レベル参照を派生パイプラインで解決**（[[feedback-representation-belongs-to-pipeline-phase]] と整合）: `StyleSkillDataModel.prepareDerivedData` で `level`/`levelTotal` を1回上書きし、判定・表示・組み合わせ個数上限（usage-sheet が読む `system.level`）・AE バフ着地（`levelTotal`）が全て確定値を読む。**判定フローは無改変**（ユーザー懸念の「毎判定参照」を回避）。
- **純関数を既存モジュールに同居**（`style-skill-acquisition.mjs`）。`resolveLevelRef`/`groupStyleSkillsByStyle` は Foundry 非依存で、`idKeyPrefix`/`hasDuplicateStyleWeapon` と同じテスト基盤に乗る。
- **EXP 0 化は既存の除外列に augment**（[[feedback-never-consider-overwriting]]）: `_calcSingleItemCost`/`_getSkillExpCost` の styleSkill 分岐に `expFree || levelRef.enabled → 0` を足すのみ。EXP 同期機構（KI-008 圏）の構造は不変。
- **アクター上の参照先はドロップダウン**（識別キー値・既存キー自動選択）。辞典/直下は識別キー入力——「辞典は識別キー・アクターはプルダウン」という既存方針（`substituteTarget`・自動取得の出し分け）に沿う。
- **取得数はスタイル単位・グループ見出しに表示**（[[feedback-reuse-existing-ui-precedent]]）。表示場所はユーザーと2往復で確定: 見出し1か所への全体合算（混雑）も、スタイル表示エリア（セクションから遠く関連不明）も却下し、**スタイル技能リストをスタイル別グループ化**して各群小見出しに「秘技 n/上限・奥義 n/上限」を出す＝該当技能の直上・スタイル単位・一か所集中を回避。`groupStyleSkillsByStyle` がスタイル表示順でグループを返し（未一致は末尾「その他」）、template は `styleSkillGroups` を回す。グループ loop ネストに伴い edit-mode の `../TNX.SUITS` を `../../TNX.SUITS` へ補正。表示のみ・非強制（部位占有超過と同思想）。CSS は `.style-skill-group-header` にスコープ（[[feedback-css-scoped-changes]]）。

#### 課題

- **suit↔level 連動のガード**: cast シートのインライン技能行は suit トグルで level=スート数を再計算し EXP を増減する。`levelRef.enabled` の技能では level が派生確定するため、`_onSkillPropertyChange` 冒頭でガードし suit のみ更新・level/EXP に触れない。level 入力欄は両シートで `disabled`。
- **連鎖参照**は非対応（規約で連鎖しない）。`resolveLevelRef` は base レベルを読むため循環でも無限ループはしない。

#### 推奨アクション

- ~~実機で「経験点消費なし」「取得数に含まない（種別別ラベル）」「他スタイル技能のレベルを参照（辞典＝識別キー／アクター＝ドロップダウン）」と、レベル上書き・EXP/取得数の二重計上回避・suit 手動選択を確認~~ → **実機レビュー済（2026-07-02・問題なし）**。
- ~~残: 10-4 住宅エリア表示 → 10-5 combatSpeed UI（KI-010）→ 10-6 戦闘タブ（要 layout 言語化）→ 10-7 棚卸し → 10-8 仕上げ（v0.10.0）~~ → **フェーズ10 完了（v0.10.0・2026-07-02）**。

**追記（2026-07-01・設定タブの再分類）**: ユーザー指摘「詳細設定にチェックが集中しすぎ／ヘルプ(ⓘ)有無で行が不揃い」を受け、`style-skill-sheet.hbs` の `setting` タブを**関心別フィールドセットへ再分類**: 分類（種別/スタイル/特別な/上限）・発動（タイミング/対象/射程/目標値/使用回数）・特技専用（`{{#if special}}` で gate＝ワークス/代用/書き換え）・取得・経験点（取得と同時に/経験点消費なし/取得数に含まない/レベル参照）。上3トグル（アクション/報酬点/組み合わせ不可）は1行で崩れていないため上部に据置。**ブロックの深い移動はせず**、既存の連続境界にフィールドセット境界を挿入＋特技 special 群を自前 fieldset で包む（`name`/`data-*`/ハンドラ不変＝挙動リスク無し）。**不揃いの根因**は、`tnx-form-field > .form-group` の grid（`130px 1fr`）が `:has(checkbox)` の flex 規則より display を勝ち取り**ラベル列 130px 固定→長い和文ラベルが折返し**だった点。`.tokyo-nova.sheet.item.skill` スコープで toggle 行のみ `grid-template-columns: 1fr auto` に上書きし、ラベル可変・チェック右端で折返し解消（[[feedback-css-scoped-changes]]）。stylelint clean・全テスト green(968)。実機レビュー済（2026-07-02・問題なし）。

### コンバットスピード 3層パイプラインと入力 UI（フェーズ10-5・KI-010）

**レビュー対象**: `helpers.mjs`（`combatSpeedField`/`computeOutfitAggregates`）、`tap.mjs`、`item/helpers.mjs`（cs 名前空間）、`cast.mjs`（CS 派生・AE 着地・準備ゲート）、`attributes.mjs`（フォールバック）、`tnx-cast-sheet.mjs`/`cast-sheet.hbs`（戦闘タブ）、`tnx-outfit-sheet.mjs`/`outfit-sheet.hbs`、`system.json`、`tnx2.css`
**ステータス**: ✅ 実装完了（全テスト green 973・eslint 0 error・stylelint clean）・実機レビュー済（2026-07-02・問題なし。AE 配線は具体データ未整備で未検証のまま持ち越し）

#### 良かった点

- **「決定値＋ライブ修正値」を保存/派生の分離にそのまま写像**: 決定値＝スキーマ保存（base/value/current）、修正値＝派生実効値（baseTotal/valueTotal/currentTotal・毎 prepare 組み立て）。読み飛ばしは派生側の条件除外で、保存値に一切触れない＝スナップショット原則がデータ構造で保証される。
- **AE の CS 着地は仮想名前空間 `system.cs.*`**: 実スキーマパス（system.combatSpeed.\*）と分離し、アクター自身の効果が Foundry ネイティブ適用と custom 適用パスで**二重に効く事故を構造的に回避**。既存の ability/control 名前空間と同じ手法。
- **フラグ集計は純関数のまま**: `computeOutfitAggregates` は Foundry 非依存を維持し、通常分と読み飛ばし可能分を分離して返すだけ。ゴースト判定は消費側（CastDataModel）に置き、関数は状態を知らない。
- **initiative の全型解決**: 式が `displayTotal` を読むため、cast 以外（guest/troop）でも解決できるよう `AttributesTemplate.prepareDerivedData` に素のフォールバックを設置（cast は自前計算で上書き）。mixin のメソッドコピー＋クラス本体優先の既存挙動に乗る。
- **スピナー必須の原則**（[[feedback-number-input-spinner]]）: CS/カレント/手動修正すべて number-input-spinner。アイテムシートと同名の `incrementField`/`decrementField` アクションをキャストシートにも追加（同方式・data-field パス）。
- **単一「CS」表示＋表示層の自動制御**（初版=3値のラベル並び→第2版=能力値スタイル流用＋手動チップ、いずれもユーザー却下を経て確定）: シートには「CS」1つだけを出し、どの層を表示するかは**戦闘トラッカー連動で自動決定**（カット進行中＝カレント／それ以外＝CS。`isActorInStartedCombat`＋派生 `displayTotal`・AE 適用後に確定・initiative が読む）。手動の層切替 UI は持たない。カット開始/参加時にカレントへ CS を自動セットする戦闘フック（`combatStart`/`createCombatant`＝GM 書込、`deleteCombat`/`deleteCombatant`＝ローカル再準備＆再描画。表示層の切替はドキュメント更新を伴わないため reset+render で追従）。UI は**CS リードアウトパネル**として専用デザイン（左アクセントバー・大型モノスペース数字・カット進行中の状態チップ・読み飛ばし「X（Y）」併記・編集モードで内部データ4項目とプレアクト初期化を1行に）。見出し「コンバットスピード」は撤去（CS 1つに節見出しは不自然というユーザー指摘）。

#### 課題

- **cs.base の準備ゲートは AE 適用パス内の唯一の特例**（`_applyEffectBuffs` 冒頭の1条件）。一般化は解説効果全体の扱い（準備で常時効果解禁）に波及するため CS に限定した。将来一般化する場合はこの分岐が原型になる。
- プレアクト初期化は base と value を同時更新する1ボタン。オーバーレイ分（実効−base）を保って value に写すため、初期化時点のゴースト状態・準備状態が写り込む（プレアクトでは通常ゴースト登場前なので実用上問題ないと判断）。
- 旧 `mod` フィールドは migrateData なしで削除（KI-010 により UI が存在せず保存データは初期値のみのはず）。万一手動 DB 編集データがあれば消えるが、リスクは無視できる規模。

#### 推奨アクション

- ~~実機確認: 戦闘タブの3層表示・プレアクト初期化（ボタン表示名「CSを決定」）・タップのフラグ（携帯起点/準備起点の切替とゴースト時の括弧表示・読み飛ばし）・AE `system.cs.base|value|current` の着地・initiative（コンバットトラッカー）~~ → **実機レビュー済（2026-07-02・問題なし。AE 着地のみ具体データ未整備で持ち越し）**。
- ~~残: 10-7 棚卸し → 10-8 仕上げ（v0.10.0）~~ → **フェーズ10 完了（v0.10.0・2026-07-02）**。

### 戦闘タブ本体（CS・参照中の武器・防御力・タイミング別再表示）（フェーズ10-6）

**レビュー対象**: `cast.mjs`（`weaponRefs`）、`tnx-cast-sheet.mjs`（`_prepareCombatData`）、`cast-sheet.hbs`（戦闘タブ・全幅縦積み）、`tnx2.css`
**ステータス**: ✅ 実装完了（全テスト green 977・eslint 0 error・stylelint clean）・**レイアウト実機レビュー済＝確定（2026-07-02・問題なし）**。生身変更装備・義体の実データ検証はフェーズ12 へ持ち越し

#### 良かった点

- **「プレイヤーが答えたい質問」から要素を導出**（数回の是正の到達点）: 一覧（アウトフィットタブの仕事）でなく「いまの攻撃力・受け値・防御力」＝**参照中の武器2行**（攻撃/パリー・別武器可）と**防御力の種別別合計1行**のみ。効果系は状態タブの領分で置かない。
- **参照解決はルールの文言どおり**: 「宣言が要るのは複数準備時」→ 候補（準備済み weapon/cyborg）が1つなら自動参照・複数で未宣言なら「未宣言」。無効な宣言 ID は解決時に読み飛ばしてフォールバック（残置しても実害なし）。編集モードは name バインドのドロップダウン（submitOnChange）で専用アクション不要。
- **参照宣言は最小の状態**: `cast.weaponRefs{attackItemId,parryItemId}`（§4.3 対象）。表示のみ・非強制で、消費者（12 ダメージ算出）が後から読む——「状態は先に確定・消費は後続」パターン（[[feedback-representation-belongs-to-pipeline-phase]]）。
- **タイミング分類は検索補助として設計**: 標準8種（プロセス/アクション）は空でも常設・差し込みタイミング（自由記述）はテキストごとに非空のみ・「その他」（自由記述なし＋解説参照）は末尾非空のみ——のユーザー確定規則をそのまま実装。各群内は item.sort 尊重・名前クリックは既存 `startSkillCheck` へ委譲。新フィールドなし。
    - ⚠️ **この行の「差し込みタイミング（自由記述）」は誤り（2026-07-20 是正・下記エントリ参照）**。差し込みタイミングの実体は大半が `getSkillOptions().timing` の**名前つき選択肢**（`preCalcDamge`＝「ダメージ算出の直前」等）であり、自由記述（`other`＋`timingOther`）はその一部でしかない。実装も同じ取り違えをしており、名前つき選択肢の用途は該当があっても表示されなかった。標準群のうちリアクションは「該当する用途があるときのみ表示」（2026-07-20 同日にユーザーが撤回して確定した形）。

#### 追記（2026-07-02 後半・CS ボックスと生身ラインへの再設計）

- **CS**: 公式プロファイルシートの CS ボックスを tnx2 の語彙で写す（2段ラベル＋値・式キャプションなし）。カット進行中は既存のグロートークン（`--tnx2-accent-glow`）で発光＝チップ廃止（ユーザー案）。編集モードはボックス内が「決定の連鎖」縦積みに変形（閲覧/編集で同じ要素が変形する報酬点・能力値のパターンに一致）。
- **生身ライン**: `weaponRefs` の意味を「生身の書き換え元の選択」へ変更（空＝未変更の生身が既定→「未宣言」消滅）。生身はアイテムでなくアクターのデータ（休眠 `baseAttack`/`baseGuard` の正式化＝新設ゼロ）。書き換え元候補＝準備済み cyborg＋`isFleshChange` 武器。攻撃/パリーで別選択可。既存プルダウン流用。
- **§4.3**: `weapon.isFleshChange` 追加（特性セクション・武器のみ表示）。

#### 課題

- 全身義体の置き換え派生（キャラ実効攻/受/防）・〈二刀流〉等の合算・パリーの都度宣言は12（判定/リアクションフロー）と併せて。防御合計は複数義体を単純合算のまま。
- **反省（経緯）**: 初版は武器/防具の一覧表＋「カットの流れ」レール＋状態セクションを実装し、全て誤りだった。原因は (a) 未承認の誤記録への追従 (b)「表示＝一覧」という安易な読み (c) 検索補助要望の可視化ギミックへの取り違え。UI 要素はルールとユーザーの言葉から導出する。

#### 推奨アクション

- ~~ユーザーの実機レビューでレイアウトを確定~~ → **実機レビュー済・確定（2026-07-02）**。最終形は全幅縦積み（CS ボックス → 武器2行 → 防御力 → タイミング別）。初版記載の「2カラム」「BS 表示」は再設計で撤去済み（上記追記参照）。

### 住宅アウトフィットのエリア表示（フェーズ10-4）

**レビュー対象**: `tnx-cast-sheet.mjs`（`_resolveHousingAreaMods` に `area` 追加・`_prepareOutfitRow` で `housingAreaRank` 付与）、`cast-sheet.hbs`（バッジ）、`tnx2.css`（`.outfit-area-badge`）
**ステータス**: ✅ 実装完了（全テスト green 968・eslint 0 error・stylelint clean）・実機レビュー済（2026-07-02・問題なし。追補修正5点を含む）

#### 良かった点

- **既存の解決経路に最小 augment**（[[feedback-never-consider-overwriting]]）: 住宅施設行は既に `_resolveHousingAreaMods` でエリアを live 解決していたため、戻り値に `area` を1行足し `HOUSING_AREA_RANKS` でラベル化するだけ。新規の解決・キャッシュを作らない（[[feedback-no-stale-caches]]）。
- **バッジは既存 `.outfit-slot-tag` の作りを再利用**（[[feedback-reuse-existing-ui-precedent]]）: `--c` カスタムプロパティ方式のベース宣言を共有セレクタに足し、ランク別 `--c` のみ追加（宣言の重複なし）。色はランク名に対応（レッド/グリーン/ホワイト/サンクチュアリ）＋なし=ミュート。CSS は `.tab[data-tab="outfits"]` にスコープ。
- **名前置換でなくバッジ**を採用＝アイテム名を保ち、判定起動リンク（名前の `<a>`）を壊さない。

#### 課題

- リンク無し（住宅エリア未設定）の住宅施設はバッジ非表示＝意図どおり。ランク「なし」も可視色（`--tnx2-text-secondary`）で明示バッジ表示（エリアが「なし」と設定されている事実を示す。当初ミュート色で見えづらかったため修正）。
- **実装中に既存不具合 KI-022 を発見・解消**: `_resolveHousingAreaMods` が `useHousingAreaDrop` 分岐で `pack.getDocument(UUID)` を呼び、辞典で紐づけた住宅エリアを解決できず**供給値もバッジも欠落**していた（住宅施設シートは `fromUuid` で正常＝乖離）。`fromUuid` 無条件解決へ統一。バッジが出ない真因はこれであり、CSS ではなかった（ユーザー指摘どおり）。正本 KI-022。

#### 推奨アクション

- ~~実機でランク別バッジ色・名前横配置・未設定時の非表示、および辞典で紐づけた住宅施設の供給値（登/セ）が出ることを確認~~ → **実機レビュー済（2026-07-02・問題なし）**。
- ~~残: 10-5 combatSpeed UI（KI-010）→ 10-6 戦闘タブ（要 layout 言語化）→ 10-7 棚卸し → 10-8 仕上げ（v0.10.0）~~ → **フェーズ10 完了（v0.10.0・2026-07-02）**。

### 「判定」英名の Check 統一リネーム（フェーズ10-11 間の軽微修正）

**日付**: 2026-07-03
**レビュー対象**: 旧 `tnx-judgment-engine/flow/dialog.mjs` → `tnx-check-engine/flow/dialog.mjs`、旧 `judgment-dialog/request/result.hbs` → `check-*.hbs`、`tnx2.css`、関連識別子全般
**ステータス**: ✅ 実装完了（全テスト green 980・eslint 0 error・stylelint clean）

#### 経緯と根拠

- 「判定」の正しい英訳は **check**（TRPG 輸入時の訳語対応。[Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md) 冒頭の用語注記で確定済み）。フェーズ7-8 実装時の Judgment 系命名はこの用語確定（フェーズ9）より古く、混在状態だった。
- AE キー（`check.` / `controlCheck.`）・判定種別（`skillCheck` 等）・用途タイプ（`"check"`）は既に check 系であり、本リネームで命名が一本化された。

#### 変更内容

- クラス: `TnxJudgmentEngine/Flow/Dialog` → `TnxCheckEngine/Flow/Dialog`。関数・変数・typedef（`JudgmentCheckResult` → `CheckResult`）・CSS クラス（`judgment-*` → `check-*`）・グローバル（`game.tnx.judgment` → `game.tnx.check`）。
- ファイル 7 本を git mv（scripts 3・test 1・templates 3）。wiki も [Check_Rules.md](../01_Wiki/Game_Rules/Check_Rules.md)・[Check_System.md](../01_Wiki/Game_Mechanics/Check_System.md) に改名し全参照を更新。
- ChatMessage フラグキー `judgmentRequest`/`judgmentResult` → `checkRequest`/`checkResult`、ソケット種別も同名変更。
- 本体 CHANGELOG は当時の名称の記録として原文保持（wiki の過去記録は検索性優先で機械的に新名称へ置換）。

#### 課題

- ⚠️ **既存ワールドの過去チャットの「判定する」ボタンは失効**（旧フラグキーのため新コードが認識しない）。開発中 0.x のため migration は設けない（意図的に妥協）。

#### 推奨アクション

- 実機で判定フロー一巡（シート起動・RL 要求・山札判定・チャットカード表示・CSS 崩れなし）を確認。

### 戦闘タブ AR ボックス＋actionRank データモデル（フェーズ11 追補）

**日付**: 2026-07-03
**レビュー対象**: `scripts/data/helpers.mjs`、`scripts/data/actor/common/attributes.mjs`、`scripts/data/actor/cast.mjs`、`scripts/data/item/helpers.mjs`、`scripts/tnx.mjs`、`templates/actor/cast-sheet.hbs`、`css/tnx2.css`、テスト3ファイル
**ステータス**: レビュー済（実機確認待ち）

#### 設計

- **CS パイプラインの相似形で最小追加**: 定数／フィールド／表示解決関数を `helpers.mjs` に CS の対で配置し、attributes テンプレートの素の派生 → cast の AE 適用後確定、という CS と同じ2段構え。AE の仮想名前空間 `system.ar.max`（実スキーマパスと分離＝ネイティブ適用との二重適用防止）も cs.\* と同型。
- **カット開始時の付与は既存フックに統合**: `seedCombatSpeedCurrent` を `seedCombatStartValues` に拡張し、CS カレントと現在AR を1回の `actor.update` にまとめ書き（フック増設なし）。
- **UI は cs-box ブロックを共有**（[[feedback-reuse-existing-ui-precedent]]）: 修飾 `.cs-box--ar`（幅・間隔）と `.cs-box` への `vertical-align: top` 追加のみで、新規コンポーネント・重複セレクタなし（[[feedback-css-scoped-changes]]）。編集モードは「付与＋修正＝付与値 → 現在AR」の決定の連鎖（CS の cascade 構造を踏襲）。定数 1 は派生 `grantBase` としてテンプレートへ単一ソースで供給。
- テスト 15 件追加（フィールド構造・定数・表示解決・AE キー解析）。980 → 995 green。

#### 課題

- ⚠️ 現在AR の閲覧モードに増減操作を設けていない（CS の「閲覧モードに手動操作なし」裁定を踏襲）。フェーズ13 の自動化までカット中の消費管理は編集モード経由——実運用で煩雑なら再検討。
- 初版の表示解決関数（resolveActionRankDisplayTotal）は「なし」表示への修正（`b0132e8`）で撤去。カット進行外は数値でなく「—」を出すため表示分岐はテンプレート側（inCombat）に置き、データ層は maxTotal / value のみ供給する。テストは 992 green。

#### 推奨アクション

- ~~実機確認: 戦闘タブの AR ボックス（閲覧/編集の両モード・カット進行中のグローと現在AR への表示切替・カット開始時の自動セット）~~ → **実機確認済み（2026-07-03）**。非戦闘時に付与値が数値表示されていた点をユーザー指摘で「なし」表示に修正（`b0132e8`）。

### キャストシート共通部品化（フェーズ11-2）

**日付**: 2026-07-03
**レビュー対象**: `scripts/actor/tnx-character-sheet-base.mjs`（新設）、`scripts/actor/tnx-cast-sheet.mjs`（縮小）、`templates/actor/cast-sheet.hbs`（シェル化）、`templates/actor/parts/`（新設 8 partial）、`scripts/tnx.mjs`（loadTemplates）
**ステータス**: ✅ 実装完了（本体 `7c35ccc`・全テスト green 992・eslint 0 error）・**実機回帰待ち（cast シート全タブ・挙動不変が完了条件）**

#### 設計（11-2 設計どおり・2026-07-03 承認）

- **テンプレート**: `templates/actor/parts/` に sidebar / header / tab-abilities / tab-combat / tab-outfits / tab-status / tab-history / tab-profile の 8 partial を新設。移設は sed の行範囲抽出で**バイト同一**（転記ミス排除）。cast-sheet.hbs はタブ nav＋`{{> }}` 差し込みの薄いシェル。PARTS の複数分割はせず単一 part 描画を維持（DOM・スクロール保持・submitOnChange 不変）。
- **features ゲート**: header の EXP ブロック＝`{{#if features.exp}}`、tab-outfits の部位占有パネル＝`{{#if features.parts}}`。cast は `SHEET_FEATURES = { exp: true, history: true, parts: true }` のため出力不変。
- **クラス**: `TnxCharacterSheetBase` は旧 cast シートのコピーから cast 固有部を機械的に除去して作成（Python スクリプト・全置換に一意性アサーション付き）。cast に残したのは設計どおり **EXP 系**（`updateCastExp`・`_calcSingleAbilityCost`/`_calcSingleItemCost`・`_getSkillExpCost`）と**セッション履歴**（TnxHistoryMixin ACTIONS・リスナー・`_performHistoryUpdate`・history コンテキスト）のみ。
- **技能レベル変更の EXP 連動**は基底のフック `_applySkillLevelExp(item, oldLevel, newLevel, targetKind, input)` に分離（既定＝素通し。cast がオーバーライドして消費・不足時の入力巻き戻し＝従来と同一挙動）。EXP を持たない guest（11-3）はフック未定義のまま自然に非消費となる。
- **DEFAULT_OPTIONS** は ApplicationV2 の継承チェーンマージを利用。classes 配列は上書きのため cast がフル指定。

#### 課題

- ⚠️ partial 内の構造クラス名（`cast-sheet` / `cast-sidebar`）は CSS 再利用のため cast 名のまま共有する（アプリルートの `.cast`/`.guest` クラスで型は区別可能）。改名は CSS 全域に波及するため見送り。
- 未使用の `_prepareGenericSlots` は基底へそのまま移設（挙動不変の原則。整理は別途）。

#### 推奨アクション

- 実機回帰（cast シート）: 全タブ表示・編集モード切替・技能レベル/スート変更の EXP 連動・アウトフィット D&D 並び替え・部位占有・戦闘タブ（CS/AR）・セッション履歴・判定起動。

### GuestDataModel／TokyoNovaGuestSheet と共通基底（フェーズ11-3）

**日付**: 2026-07-03
**レビュー対象**: `scripts/data/actor/common/character-base.mjs`、`scripts/data/actor/cast.mjs`・`guest.mjs`、`scripts/actor/tnx-guest-sheet.mjs`、`templates/actor/guest-sheet.hbs`、`scripts/module/user-flag-schema.mjs`、`scripts/tnx.mjs`、テスト2ファイル
**ステータス**: レビュー済（実機確認待ち）

#### 設計

- **DataModel はクラス継承で共通化**（シート側 11-2 の TnxCharacterSheetBase と対の構図）: `CharacterBaseDataModel extends SystemDataModel.mixin(Biography, Attributes, ActorBase)` に共有スキーマ＋派生パイプライン一式。mixin のプロトタイプコピー（後勝ち・super 連鎖なし）に依存しない単純な継承のため、prepareDerivedData の実行順が明確（attributes フォールバック → 基底パイプライン）。
- **作成は 11-2 と同じ機械変換**（cast.mjs のコピーに一意性アサーション付き置換: クラス名・import パス・セッション履歴クラスタ除去）。cast.mjs / guest.mjs は薄い派生に書き換え。**cast のスキーマは不変**（既存 cast テストが無変更で通過＝裏付け）。
- **guest シート**: `TokyoNovaGuestSheet` は classes / PARTS / SHEET_FEATURES の宣言のみ（約30行）。シェル guest-sheet.hbs はセッション履歴タブを除いた partial 差し込み。11-2 の部品化がそのまま効いた形。
- **スキーマ差分テスト**: 「cast との差分＝セッション履歴クラスタ＋lifePath の5フィールドに限る」を不変条件としてテスト化（lifePath はユーザー再訂正 `9dc0928` で cast 固有へ移動し features.lifePath ゲートを追加）。今後 cast にフィールドを足すと guest 側の判断を強制する仕掛け（基底に足せば自動で両方に入る）。

#### 課題

- ⚠️ guest-sheet.hbs の構造クラス名（`cast-sheet` 等）は CSS 再利用のため cast 名のまま（11-2 記録どおり。型はアプリルート `.guest` で区別）。
- ⚠️ troop（11-4）は biography 追加・エニグマモードなど固有差分が大きく、CharacterBaseDataModel でなく attributes 直 mixin ＋必要部分の選択共有になる見込み（11-4 冒頭で設計）。

#### 推奨アクション

- 実機確認: guest の新規作成（部位プリセット・基本13技能・リンクトークンが入る）→ シート一巡（履歴タブ・EXP 表示が無いこと、報酬点・ライフパス・部位管理・戦闘タブ CS/AR・判定起動・ゴーストトグルが cast 同様に動くこと）→ 技能レベル変更で EXP 消費が発生しないこと → ゲストに手札上限 AE を付けたとき RL の HUD 上限が変わること。

### TokyoNovaTroopSheet と features 拡張（フェーズ11-4）

**日付**: 2026-07-03
**レビュー対象**: `scripts/actor/tnx-troop-sheet.mjs`、`scripts/data/actor/troop.mjs`、`scripts/actor/tnx-character-sheet-base.mjs`、`templates/actor/parts/sidebar.hbs`・`header.hbs`、`templates/actor/troop-sheet.hbs`、`css/tnx2.css`、`scripts/tnx.mjs`、テスト
**ステータス**: レビュー済（実機確認待ち）

#### 設計

- **SHEET_FEATURES を 7 フラグに拡張**（exp/history/lifePath/parts/miracles/bounty/heads）し、**基底既定とのマージ解決 `sheetFeatures`** を導入（派生クラスの宣言漏れで既定が欠ける事故を防止。cast/guest は宣言追加不要で従来挙動維持）。
- **troop シートも宣言のみの薄い派生**（guest と同型・約30行）。シェルは履歴タブなし5タブ。神業・報酬点は partial 内 features ゲート、部位占有は既存の features.parts ゲートが効く。
- **heads パネルは報酬点ボックスの意匠（bounty-\* クラス）を共有**（[[feedback-reuse-existing-ui-precedent]]）。追加 CSS は heads-max / heads-mode-toggle の2ルールのみ（[[feedback-css-scoped-changes]]）。±は汎用 incrementField/decrementField アクション（data-min=0）を再利用し、専用リスナーなし。編集モードは number-input-spinner（[[feedback-number-input-spinner]]）。
- **神業ガードはデータ入口で二重化**（ドロップ拒否＋スタイル由来自動取得のスキップ）——シート非表示だけでは辞典ドラッグで混入しうるため。
- **トークンバー**: `CONFIG.Actor.trackableAttributes = { troop: { bar: ["heads"] } }` ＋新規作成時に bar1=heads・OWNER_HOVER を既定化。エニグマモードでもフィールドが同一のため割当は不変。

#### 課題

- ⚠️ 11-3 レビューの見込み（attributes 直 mixin）を撤回し CharacterBaseDataModel 継承へ（理由は監査エントリ）。死蔵フィールド（bounty/partSlots/生身/isGhost/handMaxSizeMod）はスキーマに残る——判定フローの bountyAvailable は 0 になり実害なし。
- ⚠️ 戦闘タブの武器/防御は troop でも表示される（分身に有用）。生身表記「生身 I+0／受0」が集団としては違和感の可能性——実機確認で判断。

#### 推奨アクション

- 実機確認: トループ新規作成（bar1=heads・13技能）→ サイドバーの人数±/エニグマモード切替（ラベルとトークンバーの追従）→ スタイルをドロップして神業が自動取得されないこと・神業ドロップが拒否されること → 判定起動・戦闘タブ表示。

#### 是正記録（2026-07-03・同日・本体 `46a8584`）

- SHEET_FEATURES を 12 フラグに拡張（growth / personalData / citizenRank / handle / troopLevel を追加）。features は基底既定とのマージ解決のため cast / guest は宣言変更不要。
- 名前固定は `nameLocked` コンテキスト（header の identity 編集を disabled 化）＋ `tnx.mjs` の `syncTroopName`（updateActor / スタイルの create・update・delete で導出名へ同期。導出名一致で収束するため再帰安全）。
- 能力値式は `_prepareAbilityTotals` フック抽出（基底=キャスト式、troop=growth 枠にトループレベルを入れて `computeAttributeFinal` を再利用＝式の単一ソース維持）。
- 反省: 「エニグマモードにしたら」というユーザー発言を Boolean フラグに直訳した。3類型（トループ/エニグマ/分身）が正本 Troops.md に明記されていた以上、種別切替として設計すべきだった（記録された類型構造 > 発言の字面）。

### TokyoNovaExtraSheet と features.abilities/combat・fixedResult（フェーズ11-5）

**日付**: 2026-07-03
**レビュー対象**: `scripts/actor/tnx-extra-sheet.mjs`、`scripts/actor/tnx-character-sheet-base.mjs`、`templates/actor/extra-sheet.hbs`・`parts/tab-abilities.hbs`、`scripts/data/item/common/usage.mjs`、`templates/app/usage-sheet.hbs`、`scripts/tnx.mjs`、テスト
**ステータス**: レビュー済（実機確認待ち）

#### 設計

- **features を 14 フラグに拡張**（abilities / combat を追加）。「能力値を持たないシート」を基底の3ゲート（能力値コンテキスト・戦闘データ・部位占有の準備）＋テンプレート1ゲート（能力値グリッド）で表現し、extra も**宣言のみの薄い派生**（guest/troop と同型）に収まった。技能タブは tab-abilities partial の再利用（ナビラベルだけ「技能」）。
- **fixedResult は usage 基盤への最小追加**（フィールド1本＋発動タブ入力1行＋起動フローの早期分岐。§4.3 の usage 基盤改変は 11-1 一括承認の「固定値判定」項どおり）。用途不備検知は固定達成値のとき対象外（技能・スートを使わないため）。
- **チャットは専用の軽量カード**（check-result.hbs はカード画像等を前提とするため再利用せず）。flags は checkResult 互換で情報系統を揃えた。
- エキストラの通常判定ガードは**用途選択後・固定値分岐の後**に配置（解説のチャット表示と固定値判定はエキストラでも通る）。

#### 課題

- ⚠️ エキストラのスタイルスロット・市民ランク・ハンドル等は表示のまま（基本は名前のみだが持てるデータのため。過剰なら troop 同様に実機指摘で絞る）。
- ⚠️ 固定達成値入力はスピナー原則の例外（nullable「空欄=通常判定」のため）。

#### 推奨アクション

- 実機確認: エキストラ新規作成（リンクトークン・技能シードなし）→ 3タブ表示 → 技能/アウトフィットのドロップと一覧 → 神業使用 → 固定達成値を設定した用途の判定（チャットに「達成値 N（固定値）」）→ 固定値なし用途の判定が警告で止まること。


## フェーズ11-6: NPC取得・消費先設定・分身共有・EXP算入（2026-07-05）

#### 設計判断

- **消費解決レイヤーの一元化**: usage-consumption.mjs に解決（resolveConsumeRows・純関数）／確認ダイアログ／適用（平プラン）を集約。分身→本体差し替え（resolveConsumeRowsForActor）は同じ解決関数の getItem 差し替えで実現し、ダイアログ残数表示と実消費が必ず同じ対象を見る。プランは平データ（actorId/itemId/kind/amount）で ctx を通す（deepClone 安全）。
- **純ロジックの分離**: npc-acquisition-logic.mjs（所有者逆引き・取得帰結）を Foundry 非依存で分離。フロー本体（npc-acquisition.mjs）は check-flow を import するため、テスト可能性と import 連鎖の Foundry グローバル問題を分けて解決。
- **循環回避**: check-flow → npc-acquisition の完了継続は動的 import（npc-acquisition が TnxCheckFlow.open を static import するため）。
- **トークン配置**: dnd5e TokenPlacement の簡易版（プレビュー追従・クリック確定・右クリックスキップ・TOP_LEFT_VERTEX スナップ）。ホイール回転は見送り。actor.getTokenDocument → scene.createEmbeddedDocuments。
- **EXP 再計算トリガ**: 所有者参照/種別の変更は preUpdateActor で旧所有者 uuid を options に捕捉し、新旧双方を再計算（付け替えで計上が移動するため）。per-item ループは _sumActorItemsCost に共通化（キャスト本体と所有トループ級で共用・挙動不変）。
- **UI 踏襲**: 消費設定=D&D Consumption（行追加/削除・スピナー）。取得アイテム/所有者/エキストラアクター参照=フェーズ10 ドロップエリア方式＋fromUuid(Sync) ライブ解決（name は削除時フォールバックのみ）。

#### 既知の留意点

- ⚠️ 消費設定の保存は用途シート _onSubmit のホワイトリストに行入力の再構成として配線（新フィールドは必ず保存経路も配線・11-5 の教訓を適用済み）。
- ⚠️ トークン配置は canvas API（PIXI プレビュー・getSnappedPoint）依存のため実機確認が最重要。gridless はスナップなし。
- ⚠️ 分身共有の照合は identificationKey 規約前提。キー無しの自作アイテムは名前一致→それも無ければローカル消費（ダイアログの「（本体と共有）」表示の有無で判別可能）。

#### 推奨アクション（実機確認）

1. 消費先設定: 既存 check 用途が「親×1」で従来同一に動くこと／組み合わせ技能が自動消費**されない**こと／神業 usageCount の消費。
2. NPC取得（トループ/エニグマ）: 所有者設定→用途使用→判定→heads=達成値転記・レベル転記・配置。
3. NPC取得（分身）: 目標値10・成功時のみ配置・sourceName 自動設定・使用回数が本体から減ること。
4. NPC取得（エキストラ）: 式神符型アウトフィット→派生取得＋共有アクターの配置が同時に起きること。
5. EXP: トループ側の技能/アウトフィット追加で所有者キャストの消費経験点が増えること・「うち所有トループ級」小計・トループシートの所有者経験点表示（分身に出ないこと）。

#### ステータス更新（2026-07-08・フェーズ11 完了）

- 推奨アクション（実機確認 1〜5）は反復実機レビューで全消化（ユーザー明言「残作業なし」）。レビュー起因の主な是正: 使用ボタンの無条件描画・アイテムロールからの NPC取得起動・分身の直接召喚化（ensureBunshinActor・13技能シードのスキップで二重取得解消）・取得対象の用途側一本化（autoAcquireActors/sourceName 廃止）・チャットカード文字色の一般防御（[class*="tnx-"]）・ドロップの種別ルーティング。


### 判定バフのグループ参照（check.style / check.works）（フェーズ12 補）

**日付**: 2026-07-10
**レビュー対象**: `scripts/data/item/helpers.mjs`（`parseEffectTargetKey` / `checkChangeMatches`）、`scripts/module/tnx-check-flow.mjs`（`_computeCheckBonus`）
**ステータス**: レビュー済(問題なし)

#### 良かった点

- **後方互換で criteria を拡張**: `_computeCheckBonus` の criteria を `skillKeys`（キー配列）→ `skills`（`{key,style,organization}`）に拡張しつつ、`checkChangeMatches` は `criteria.skills` を優先し**旧 `skillKeys` も `{key}` として受理**。既存テスト・呼び出し（`computeCheckBonus`/`gatherCheckBonusSources`）を壊さない。
- **既存文法への最小追加**: `parseEffectTargetKey` の check 分岐に `group:"style"|"works"` を足すだけ（前方一致 `*` の解釈は据え置き）。ドット区切りで値バフの `category.<cat>` と表記を揃えた（`check.style.<キー>`）。`controlCheck` はグループ不可（能力値のみ）で早期 null。
- **辞典由来の構造化フィールドを再利用**: 照合キーは style skill の `system.style`・works の `system.special.works.organization`（[Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md) §6 が「後続」としていた構造化の実体化）。

#### 留意点

- ⚠️ styleKey/orgKey は技能側フィールドの生値直書き（入力補助の辞典ドロップダウンは未提供）。
- 値バフ（`system.*`）側は未対応＝**判定キーのみ**。ダメージの対象依存は下記の別エントリで実装。


### ダメージ対象バフ（damage.vsStyle / damage.vsWorks）（フェーズ12 補）

**日付**: 2026-07-10
**レビュー対象**: `scripts/data/item/helpers.mjs`（`_gatherBonusSources`/`damageVsChangeMatches`/`gatherDamageVsSources`/`collectActorEffectBuffs`）、`scripts/module/damage-flow.mjs`（`collectDamageVsBonuses`）、`scripts/data/actor/common/character-base.mjs`（除外）、`scripts/module/tnx-check-flow.mjs`（effects 収集の共通化）
**ステータス**: レビュー済(問題なし)

#### 良かった点

- **判定バフと同型の「実行時評価の別系統」に載せた**: `check.*` が確立したパターン（適用パスから除外し、アクション時にエフェクトを集めて照合）をダメージへ横展開。恒常適用パスの `CHECK_SCOPES` に `damageVs` を足すだけで焼き込みを防げる。
- **重複排除ロジックを共通化**: `gatherCheckBonusSources` の中身を `_gatherBonusSources(effects, predicate)` に抽出し、判定バフ／ダメージ対象バフが同じ重複規約（identity 最大1回・stackable スタック）を共有。`gatherCheckBonusSources` の外部挙動は不変（既存テスト green）。
- **effects 収集も共通化**: `collectActorEffectBuffs(actor)` を新設し、`_computeCheckBonus` の inline push を置換＋ダメージフローでも再利用（DRY・正規形の一元化）。
- **台帳への合流が自然**: ダメージ対象バフを既存の `damageBonusRows`（用途のダメージ修正）に合流させ、`f.damageBonuses` の描画・`damageRollTotals` の集計にそのまま乗せた（新しい表示経路を作らない）。

#### 留意点

- ⚠️ 対象解決は `fromUuid(targetUuid)`＋`resolved?.actor ?? resolved`（トークン/アクター両対応）。命中確定済みの攻撃カードからのみ呼ばれる。
- 値はリテラルのみ（式評価しない＝判定バフと同じ簡潔さ）。式・攻撃固有の対象依存は次エントリ（`@target`）で実装。


### 用途ダメージ修正の @target 参照（buildFormulaData の target 引数）（フェーズ12 補）

**日付**: 2026-07-10
**レビュー対象**: `scripts/module/tnx-formula.mjs`（`buildFormulaData`/`buildTargetData`/`evaluateBonusRows`）、`scripts/data/item/helpers.mjs`（`targetStyleWorksKeys`）、`scripts/module/damage-flow.mjs`（`resolveTargetActor`・配線）
**ステータス**: レビュー済(問題なし)

#### 良かった点

- **欠損キーを 0 にする Proxy**: `@target.style.<未所持キー>` は式評価でエラーになりがち（未定義参照）だが、`new Proxy({}, {get})` で「所持=1・非所持=0」を返すため、どのスタイルキーを書いても安全に 0/1 に評価される。symbol アクセスも 0 で無害化。
- **対象の style/works 抽出を AE 対象バフと共通化**: `targetStyleWorksKeys(target)` を helpers に置き、ダメージ式（`buildTargetData`）と AE 対象バフ（`collectDamageVsBonuses`）が同一定義を共有（「スタイル＝type:style／ワークス＝ワークス技能の組織」の判定が分岐しない）。
- **対象を一度だけ解決**: `openDamageRollDialog` で `resolveTargetActor` を1回呼び、damageBoost の `buildFormulaData`・ダメージ修正の `evaluateBonusRows`・AE 対象バフの3者へ同じ actor を渡す（`collectDamageVsBonuses` は uuid 受けの async → actor 受けの sync に整理）。
- **引数追加のみで後方互換**: `buildFormulaData(actor, result, bearer, target)` / `evaluateBonusRows(..., target)` とも末尾任意引数。既存呼び出し（AE 値・判定ボーナス・damageReduce）は無変更で `@target` 非供給。

#### 留意点

- ⚠️ `data.target` に Proxy を載せるため、この data を **JSON 直列化する経路では使わない**（ダメージ式の Roll は都度生成・評価のみで直列化しない＝問題なし）。
- `damageReduce` は防御側視点（actor=対象）のため `@target` 非供給。攻撃者参照が要れば別途。
- v0.11.0 同期（package.json/system.json・CHANGELOG）。タグ付け・push はユーザー。


### 用途自身の修正値欄・効果供給元拡張（フェーズ12 補）

**日付**: 2026-07-10
**レビュー対象**: `usage.mjs` / `tnx-formula.mjs` / `tnx-check-flow` / `attack-flow` / `damage-flow` / `tnx-usage-sheet` / `usage-sheet.hbs` / `tnx-item-sheet`
**ステータス**: レビュー済(問題なし)

#### 良かった点

- **専用欄は行配列と別スキーマ**（`checkBonusSelf`/`damageBonusSelf`＝単一 StringField）＝「常設・非増減」を素直に表現（行の source を seed して疑似再現しない＝ユーザー要望どおり別枠フィールド）。
- **`@item.self` を bearer で通す**: 既存 `buildFormulaData` の bearer（AE 値用の `@item.self`/`@item.parent`）を用途にも流用。`evaluateBonusRows` に bearer 引数、`evaluateSelfBonus` を新設。`sourceItemId` を ctx／攻撃ペイロードに載せ実行時に親を解決（識別キーの無いアイテムでも参照可）。
- **effects の `itemId` 明示**: `effectId` 単独では供給元アイテムを解決できないため `{itemId, effectId}` に。空=親で旧データ互換（migrate 不要＝新フィールドの initial ""）。選択値 `itemId|effectId`・解除は両者一致で1件。
- **後方互換**: `evaluateBonusRows`/`buildFormulaData` の追加引数は末尾任意。既存呼び出し（AE 値・判定・damageReduce）は無変更。effects の grep 掃討で `tnx-item-sheet` の旧 `{effectId}` 前提も供給元アイテム解決に更新。

#### 留意点

- ⚠️ 対象への効果付与本体はフェーズ13。現状の item-sheet 用途使用は供給元アイテム上で効果を enable（`byHost` でまとめて `updateEmbeddedDocuments`）。
- ⚠️ 専用欄の帰属名は `bearer.name`（親アイテム名）。親が解決不能（`sourceItemId` なし）なら "用途"。


### ヴィークル搭乗ドッジの操縦上書き（フェーズ12 補）

**日付**: 2026-07-10
**レビュー対象**: `scripts/module/attack-flow.mjs`（`startReaction` / `promptVehicleDodgeFallback`）
**ステータス**: レビュー済(問題なし)

#### 良かった点

- **既存の技能候補パイプラインに1点差し込み**: dodge のとき `candidates` を `[操縦技能]` に差し替えるだけ（役割検出・全技能フォールバックは `candidates` 未設定時のみ実行）。既存の選択ダイアログ・判定フローをそのまま流用。
- **`operateSkillKey`/`resolveOperateSkill` を再利用**（操縦移動と同じ解決経路）。準備ヴィークルは `find` 1件（部位1枠の前提＝ユーザー確定）。
- **フォールバックは既存の反応導線へルーティング**（`startReaction("parry")` / `handleNoReaction`）＝新経路を作らない。DialogV2 は既存様式（form-footer ボタン・callback 返り値）。
- **循環なし**: attack-flow→vehicle-move は静的・逆向きなし。check-flow→attack-flow/vehicle-move は動的 import。

#### 留意点

- ⚠️ `startReaction` は Foundry グローバル依存の UI フローでユニットテスト外（純関数 `resolveOperateSkill`/`resolveOpposed` はテスト済み）。
- ⚠️ 制御値受けフォールバックの権限は `handleNoReaction` 側（対象操作者/GM 限定）に従う。


### 用途効果のターゲット付与（usage-effects.mjs）（フェーズ12 補）

**日付**: 2026-07-10
**レビュー対象**: `scripts/module/usage-effects.mjs` / `tnx-socket-handler`（`usageEffectApplied`）/ `tnx.mjs` 描画フック / `tnx-check-flow`・`attack-flow`・`tnx-character-sheet-base`・`tnx-item-sheet` の配線
**ステータス**: レビュー済(問題なし)

#### 良かった点

- **単一の共通フラグ `usageEffects` ＋ 単一の `renderChatMessageHTML` フック**でカード種別（判定結果/攻撃/用途使用）を横断。ボタン描画・適用・適用済み表示を1モジュールに集約（カードごとに実装しない）。
- **付与＝AE 複製**（`toObject`・`_id` 除去・`disabled=false`・`transfer=false`）。純ロジック `resolveUsageEffectData` はテスト済み。
- **権限は既存パターン踏襲**: 対象所有者/GM が適用（ダメージ適用と同じ）。適用済みフラグ更新は author/GM は直接・非作者はソケット委譲（`emitDamageUpdate` と同型の `usageEffectApplied` を追加）。循環なし（`usage-effects`→`socket-handler` のみ）。
- **各フローは末尾で捕捉**: `prepareUsageEffectPayload` を check（sheet-base）・attack（useAttack）・declaration（item-sheet）で呼び、`"cancel"` を見て用途中止。カードのフラグに載せるだけで描画はフックに委譲。

#### 留意点

- ⚠️ 結果カードのボタン描画・適用は Foundry フック/グローバル依存でユニットテスト外（純ロジックのみテスト）。
- ⚠️ item-sheet の旧「供給元アイテム上で効果を有効化」は本モデル（対象へ付与＝使用カード投稿）に置換。
- ⚠️ 二重付与防止は `applied` フラグ＋ローカル再描画に依存（人手運用前提）。


### 用途の「無視する指定技能」（ignoreComboSkills）（フェーズ12 補）

**日付**: 2026-07-10
**レビュー対象**: `skill-chain-resolution.mjs`（`resolveUsageSkills` 第4引数 `ignoreComboKeys`）、`usage.mjs`（`ignoreComboSkills`）、`tnx-usage-sheet.mjs`、`usage-sheet-combo.hbs`
**ステータス**: レビュー済(問題なし)。**初版（per-row トグル `c79e55f`）は設計不良で `c4611e3` にて全面差し替え**

#### 初版の欠陥（記録・再発防止）

- per-row トグル＝「技能を追加した後に行のトグルを ON」という順序が前提だが、**親の指定技能がアクションだと `_addComboCheck` が追加時点で弾き、トグルに到達できない**（ユーザー指摘）。解決側（リゾルバ）だけ作り、**追加ゲート（手前の経路）を検証しなかった**。追加時ダイアログでの回避案も「毎回聞くのは誤り」で却下。

#### 現設計の良い点

- **設定は用途側の永続リスト**（`ignoreComboSkills`＝辞典識別キーの `ArrayField(StringField)`・modifiableParams と同型）。一度設定すれば、追加判定（`_addComboCheck`）・チェーン解決（`_resolveComboChain`）・auto-fill（`deriveUsageAutoFill`）の**全経路が同じ設定を参照**し、該当技能は黙って単体参加になる（ダイアログ・トグル無し）。
- **リゾルバは識別キー集合で除外**: seed 連鎖の必須集約で `ignoreSet.has(k)` をスキップ（seed 自身は常に参加・起点連鎖には不適用＝ベース決定を壊さない）。第4引数は末尾任意で後方互換。
- **UI は既存の即時追加セレクト＋行削除の踏襲**（skill-ref-select / effect-select と同型）。保存＝識別キー・表示＝辞典逆引き名（生キー非表示、未収載キーのみそのまま＝confrontation 表示と同じ規約）。
- **解除の安全弁**: 設定削除で指定技能（アクション）が引き込まれてアクション重複になる場合は `_comboActionConflict` で拒否。

#### 留意点

- ⚠️ 候補セレクトは辞典全技能のフラット一覧（general/style/works）。長いが機能上は十分。絞り込み UI は必要になれば別途。
- ⚠️ `comboLockAnalysis`（削除ロック）には ignore を渡していないが、無視した指定技能はそもそも combo に居ないためロック対象にならず、実害なし。


### 一般技能の外部ドロップで正規ソート順が効かない不具合の修正（フェーズ12 補）

**日付**: 2026-07-10
**レビュー対象**: `scripts/actor/tnx-character-sheet-base.mjs`（`_onDropItem` / `_calcInsertSortValue`）
**ステータス**: レビュー済(問題なし)・本体 `71b3481`

#### 仕組みの整理（正本）

- 一般技能の**正規順の正本**は `TnxSkillUtils.GENERAL_SKILL_SORT_PREFIXES`（医療→射撃→知覚→電脳→製作→心理→意志→交渉→芸術→運動→回避→操縦→白兵→謀略→貫禄→隠密→社会→コネ）。
- **シート表示は `item.sort` 昇順のみ**＝正規順は「作成時に正しい sort 値を振る」ことでのみ実現される。振る経路は ①基本13技能シード（`setupDefaultSkills`・正規順×1000刻み） ②シート＋ボタン（`_calcInsertSortValue`＝同グループ末尾と次グループ先頭の中間値） ③今回追加の**外部ドロップ**。手動ドラッグ並び替えは仕様どおり自由（並びは永続化）。

#### 不具合と修正

- **不具合**: `_onDropItem` は一般技能の sort を調整せず、`item.toObject()` が**辞典側ドキュメントの sort 値を持ち込む**ため、辞典からのドロップだけ並びが崩れていた（シード・＋ボタンは正常）。
- **修正**: `_onDropItem` に一般技能分岐を追加し、`_calcInsertSortValue` で正規位置の sort を振ってから作成（＋ボタンと同じ挿入挙動）。`_calcInsertSortValue` は prefix 渡しから**識別キー直渡し**に一般化（"other"の Infinity 番兵は「空キー→`getSkillSortPosition`＝Infinity＝末尾」に自然吸収・＋ボタン側の挙動は不変）。識別キーなしの自作技能は従来どおり末尾。


### 再判定システム（checkRecheck）（フェーズ12 補）

**日付**: 2026-07-11
**レビュー対象**: `tnx-check-flow.mjs`（`_buildRecheckContext`/`startRecheck`/`renderRecheckButton`）、`attack-flow.mjs`（`postAttackCard`）、`tnx.mjs`、テンプレート/CSS
**ステータス**: レビュー済(問題なし)。※初回コミット `d409992` は `_postResultChat` の引数受け漏れ（no-undef 2 errors）を含んだままコミットしてしまい `fbfbb75` で即修正——**lint を最後まで確認してからコミットする**こと（反省）。

#### 良かった点

- **既存フローの再入で実現**: 再実行は `TnxCheckFlow.open(保存済みctx)` を呼ぶだけ（専用の再判定フローを作らない）。判定ダイアログ・HUD 手札プレイ・山札・報酬点プロンプト・チャットカードの全既存装置がそのまま効く。
- **スナップショットは serializable な最小構成**: `_buildRecheckContext` が ctx から必要フィールドだけを写す。`consumeUses` は含めない（再消費防止）。報酬点は `bountyAllowed`（真偽）だけ保存し、量は `startRecheck` が現在の所持から再計算（元判定の消費を自然に反映）。
- **継続処理つき判定は null で自衛**: reaction/npcAcquire/treatment/movement/controlNegate はスナップショット自体を作らない＝ボタンが出ない（状態機械の巻き戻し問題を UI 段階で回避）。
- **usageEffects/checkRecheck/usageEffectButton は同型のフラグ＋単一フックのパターン**で統一（カード種別を横断する装飾は全部この形）。
- 再判定カードには `cr-badge--recheck`（既存バッジ体系に1色追加）。

#### 留意点

- ⚠️ 攻撃の再判定は新しい攻撃カードを出す＝旧カードは pending のまま残る（卓運用）。
- ⚠️ `startRecheck` はワールドアクター前提（`game.actors.get`）。合成アクターの再判定はフェーズ13 のトラッカー文脈（リアクションと同じ制限）。


### コンディション効果の無視ゲート（`ignore.*`）（フェーズ12 補）

**日付**: 2026-07-16
**レビュー対象**: `conditions.mjs`（`getEffectiveConditions`＝正規アクセサ・`ignoreRuleMatches`/`gatherIgnoreRules`）、`helpers.mjs`（`parseEffectTargetKey`）、`tnx-check-flow.mjs`／`character-base.mjs`（消費点）、`tnx-character-sheet-base.mjs`／`bad-status-list.hbs`／`css`／`tnx.mjs`
**ステータス**: ✅ 実装完了（本体反映済み）・実機確認済み（2026-07-21・v0.12.0）

#### 設計判断（ユーザーとの対話で確定）

- **「効果が各所で個別に読まれる」設計は据え置きが妥当**: コンディションの効果は異種サブシステムに異なる時点で着地する（達成値＝判定時・制御値/部位＝派生値算出時・恐慌/邪毒＝13）。文脈依存の実行時効果（萎縮＝対象次第・重圧＝能力値次第）は事前計算できないため、値バフのような単一パイプラインには畳めない。無理な統一より個別読みが妥当と判断。
- **堅牢性は「どこで適用するか（分散・据え置き）」と「無視されるか（中央化）」の分離で確保**: 前者は各消費点のまま、後者を**単一の正規アクセサ `getEffectiveConditions` に中央化**。各行に `effectIgnored` を付与し、消費側は尊重するだけ。将来コンディションに実効果を足す＝新消費点をアクセサ経由で書けば ignore が自動で効く。担保レベルは `readFlag`/`AE_FLAG_PARAMS` と同じ「必ず経由＋追加時 grep 全数掃引」（TNX にコンディション→効果の単一パイプラインが無いため物理的バイパス不能にはできない＝規約で担保）。

#### 良かった点

- **既存前例の踏襲**: 照合は回復対象の `recoveryKindMatches` と同じ group/kind の考え方。キーは `check.suitChange` と同じ値不要マーカー。パーサ（`parseEffectTargetKey`）・実行時系統の除外（`_applyEffectBuffs` から外す）も既存系統に揃えた。
- **純ロジックの分離**: `ignoreRuleMatches`/`gatherIgnoreRules` は Foundry 非依存でテスト可（`getEffectiveConditions` は最小モックアクターで woundSource 解決までテスト）。
- **`gatherPartSlotMods` の入力を actor→実効コンディション行に変更**して、部位スロットもアクセサ経由に一本化（無視が部位にも効く）。

#### 留意点

- ⚠️ `getEffectiveConditions` は prepareDerivedData で 2 回（制御値・部位）呼ばれる＝毎 derive の軽い重複。必要になれば 1 回に集約（現状 YAGNI）。
- ⚠️ 消費点の網羅はフェーズ13 依存（恐慌/捕縛/狼狽/戦闘不能/邪毒）。実装時にアクセサ経由を守ること。

---

### 戦闘タブ「タイミングごとの使用技能・アウトフィット」の是正＋用途の「戦闘タブに表示しない」

**日付**: 2026-07-20
**レビュー対象**: `tnx-character-sheet-base.mjs`（`_prepareCombatData` のタイミング分類）、`usage.mjs`（`hideInCombatTab`）、`usage-sheet.hbs`、`tnx-usage-sheet.mjs`（`_onSubmit`）
**ステータス**: ✅ 実装完了（テスト green 1497・eslint 0 error）・**実機確認済み（2026-07-21・v0.12.0）**。分類の全数確認は実ソースの `classify` を抜き出して選択肢の正本（`getSkillOptions()`）を全キー流して実施済み

#### 設計判断

- **リアクションは「該当する用途があるときのみ表示」（ユーザー確定・同日の撤回を経た最終形）**: 当初「リアクションの使用はチャットカードからの起動に一本化されたので常設群から除去」としたが、ユーザーが同日撤回し「**表示はするが、表示できる用途が存在したら表示**（差し込みタイミングと同じ条件）」へ。実装は `fixedBuckets` の `onlyWhenFilled` フラグ＋組み立て時のフィルタ。<br>**位置はアクションの並び（メジャー→リアクション→オート）を保つ**——表示条件が差し込み群と同じでも、末尾へ送るとルール上の行動順から外れて読み取れなくなるため（Code 判断・ユーザー裁定を仰いでいない）。用途タイプのリアクション（`isReactionType`）とは別物＝ここで見ているのは**タイミング**である点に注意。
- **「戦闘タブに表示しない」は用途の表示専用フラグ**（`actions[].hideInCombatTab`・既定 false）: 起動・判定・タイミングの意味には一切影響させない。置き場所は**タイミング fieldset**——戦闘タブの群分けはタイミング駆動なので、設定の所属が一致する。
- **差し込み群はラベルで束ねる**: 自由記述に「ダメージ算出の直前」と手入力した用途と、同名の選択肢を選んだ用途は同じタイミングなので同じ群へ。キーを分けると同じ見出しの群が二つ並ぶ。並び順は選択肢の定義順 → 自由記述（同ラベルが両方から来たら選択肢の位置を採る）。

#### 是正した誤り（ユーザー指摘）

- **名前つきタイミング選択肢の取りこぼし**: `classify()` が `timing.value` の `action`/`process`/`other` しか処理せず、それ以外（`preCalcDamge`/`onCalcDmage`/`postClacDamge`/`preDamage`/`postDamage`/`always`/`alwaysSelect`/`initiativeMajor`/`miracle`/`appearance`/`backstage`）は無言で `return` していた。**該当する用途があっても群ごと出ない**——ユーザーの「該当するものがあれば表示、のはずが表示されない」はこれ。
- **原因は正本を読まずに書いたコメント**: 実装時のコメントに「差し込みタイミング（その他の自由記述。例「ダメージ適用の直前」）」とあり、**選択肢 enum と自由記述テキストを取り違えた状態で符号化**していた。`getSkillOptions().timing` を一度引いていれば「ダメージ適用の直前」が enum のラベルだと分かった（[[feedback-logical-consistency-first]]・CLAUDE.md §5.1「正本に当てて検証する」）。

#### 留意点

- ⚠️ タイミング分類は Foundry のシートメソッド内に埋まっており単体テストが無い（本リポジトリの慣行＝純ロジックは `scripts/module/` に置いてテストする）。今回は実ソースを抜き出しての全数確認で代替した。再発防止を固めるなら純関数への切り出しが要るが、独断では行わない（§4.3）。
- ⚠️ `combat-timing-chip--move`（旧・操縦移動の合成アクション）の CSS が未使用のまま残っている。今回の変更とは無関係のため触っていない。

---

### コンボボックス共通部品（自由入力＋候補）— ネイティブ datalist の置き換え

**日付**: 2026-07-23
**レビュー対象**: `scripts/module/combobox.mjs`（`enhanceComboboxes` / `Combobox`）、`css/tnx2.css`（`.tnx-combobox*`）、`scripts/tnx.mjs`（`renderActiveEffectConfig` 配線）、`scripts/module/part-slot-preset-app.mjs`（`_onRender` 配線）
**ステータス**: ✅ 実装完了（eslint 0 error・stylelint clean・test 1773 green）・**実機確認済み（隔離 Foundry v13.351＋Playwright・2026-07-23）**

#### 背景

AE 設定シートの変更キー入力（`changes.*.key`）は任意文字列を許すため、ベタ打ち回避に HTML ネイティブ `<datalist>`（`<input list>`）を充てていた。狙いは妥当だが、ネイティブ datalist のドロップダウンは見た目・スクロール・▼位置がブラウザ任せで一切制御できず、候補が多い（cast では 200 件超）とスクロール不能で使い物にならない＝ユーザー報告の不具合（KI-036）。

#### 設計判断（ユーザー確定）

- **適用範囲＝明示配線**（number-input-spinner と同じ house style）。グローバル自動フックは採らない——他モジュール由来やネイティブの datalist へ無差別適用する副作用を避ける。今後の同型入力は `<input list>`＋`<datalist>` を書いて `enhanceComboboxes` を1行呼ぶだけ。
- **既存マークアップを候補ソースに再利用（progressive enhancement）**。標準の `<input list>`＋`<datalist>` をそのまま書かせ、昇格時に候補を読む。呼び出し側の書き換えは最小。

#### 良かった点

- **元 `<input>` を器として保持**（name/value/フォーム送信経路を壊さない）。`list` は外してネイティブUIを無効化し、`data-tnx-combo` を冪等マーカ兼出所として付与。部位スロットの `_harvest()` は元 input を querySelector で読むため無改修で通る。
- **ドロップダウンは body 直下 `position:fixed`**＝親の overflow（AE 設定タブ・app のスクロール枠）にクリップされない。`max-height`＋`overflow-y:auto` で必ずスクロール可。位置は入力直下（下端が画面外なら上へ反転）、幅・左端を入力に一致。スクロール/リサイズで追従、外側 pointerdown/Esc/blur/アンカー切断で閉じる。
- **視覚はアンカー入力の computed style を継承**（背景は透明なら祖先を辿って solid を採る・枠と選択ハイライトは `currentColor` 混色）＝ネイティブシートの light/dark と TNX ダークの双方でテーマ非依存に馴染む。CSS は `.tnx-combobox*` の自立スコープ（`.tokyo-nova` 非依存＝body 直下要素に効かせるため）。
- **自由入力を守る**（select 化しない・ハイライト既定は -1）。↑↓/Enter/Esc・クリック・ホバーに対応。確定時は input へ代入し `input`/`change` を bubbling 発火＝AE の `syncChangeValueInputs`（キー→値入力の型切替）や submitOnChange が正しく連鎖。
- **レイアウト保全の2分岐**: 親が flex のときはラッパが元 input の `flex-grow/shrink/basis` を継承（部位スロット targetPart＝`flex:1 1 4rem` を保持・実機で `1 1 64px` を確認）、それ以外はラッパ `display:flex;width:100%`（AE のセル幅を満たす）。両分岐を実機検証。

#### 留意点

- ⚠️ AE 設定は submitOnChange で change 時に全体再描画＝各描画で input が作り直されるため、`enhanceComboboxes` は毎描画フレッシュに再適用（頭で開いているドロップダウンを閉じる）。冪等マーカで同一描画内の二重昇格を防ぐ。
- ⚠️ `color-mix()` を枠/ハイライトに使用（V13 の Chromium で可）。旧環境では効かないが配布対象は V13。

#### 実機検証（隔離 Foundry＋Playwright・2026-07-23）

- 稼働中のユーザー実インスタンス（port 30000）に無干渉で、物理コピー＋別 port（30099）＋新規空ワールドで隔離起動。
- AE 効果シート: 変更キー入力3件が昇格（`list` 除去確認）・▼が右端内側に配置・ドロップダウンは入力直下同幅・候補 217 件を `overflow:auto`（scrollHeight 5774 > client 258）でスクロール可・「part」入力で 172 件へ絞り込み・↑↓＋Enter で `system.partSlot.head-part` を確定・JS エラー0。partAdd 値入力も昇格（wrapper 4個）。
- 部位スロットプリセット: 占有部位入力3件が昇格・flex 継承（`1 1 64px`）・86 件をスクロール表示・整列/JS エラー0。

#### 追補: AE キー候補の全キー網羅化（2026-07-23・ユーザー追加指摘「中途半端」）

- **問題**: コンボボックス化しても候補が `name`／部位キー／フラグ／`ignore.*`／`damage.dealt|taken` のみで、§2 の全キー一覧に対し判定バフ・キャラ値・型別パラメータ・識別キー系がごっそり欠けていた（旧「識別キー系は候補外」＝§2.8 の documented 判断）。
- **設計判断（ユーザー確定・2段階）**: ①旧「識別キー系は候補外」を撤回し全キーを候補に。②当初「任意入力部は `<key>` 雛形で統一」としたが、ユーザーが「`<key>` は識別キーの指定のみ／2個含むキーは何を指す？」と指摘——**リスト形式の思想＝完全性なので有限の組合せは実キーで全列挙**、`<key>` は**識別キー（唯一の任意入力軸）専用・1キーに最大1個**へ是正。乗り先3モード（素のキー／`item.<識別キー>`／`system.category.<分類>`）×着地属性（型別パラメータ＋フラグ）を掛け合わせ、分類は §2.4 全数のため総計 cast で約2,200件。
- **良かった点**: 正本（`parseEffectTargetKey`／wiki §2）に一つずつ当てて全系統を洗い出し、有限集合（能力値・型別パラメータ・フラグ・タグ・BS種別・分類キー・部位キー）を定数（`AE_FLAG_PARAMS`／`CONDITION_KINDS`／`OUTFIT_CATEGORIES`／部位プリセット）から機械生成。属性軸（`attrs`）を素のキー・`item.<key>`・`system.category.<cat>` の3モードで共有＝重複記述なし。`<key>` を含む option 値は属性エスケープ（`&lt;`）。**機構と内容の分離**——コンボボックスは候補ソースを増やしただけで不変（数千件でも同じ）。並びは「よく使う順→分類狙いを末尾」で表示上限の先頭を有用キーにする。
- **留意点**: ⚠️ 分類×パラメータは意味の薄い組合せ（`system.category.social.attack` 等）も含むが、AE は型を強制しないため「書けるキー＝全部出す」の思想を優先（ユーザー裁定）。⚠️ `<key>` 雛形はそのまま保存すると死にキー（`parseEffectTargetKey` 空振り）だが打ち替え前提。⚠️ **描画上限は設けない**（当初 200件上限＋「他N件」を入れたが、ユーザーが「指示文はUI方針に反する／上限撤廃」を確定＝一致は全件描画・マーカーなし）。datalist は毎描画（submitOnChange 再描画）で再構築、ドロップダウンも一致全件を innerHTML 一括描画するが、実機計測で ▼全件≈69ms・狙った語≈17ms・広い接頭辞（一致≈2,000件）で最大≈80ms＝実用上許容（GPU無効の headless 計測のため実機はより速い見込み）。
- **実機検証**: cast の効果シートで候補 **2,234 件**・`system.category.melee.attack`／`bodyArmor.defence.S`／`item.<key>.system.suits.spade` 等の実キー欠け0・**`<key>` を含む44件は全て識別キー位置＝2個含む候補0**・▼で全2,234件を全件描画（マーカー無し）・「category.melee.」絞り込みで melee 属性36件・再描画時間 狙った語17ms/広い接頭辞最大80ms・JS エラー0（隔離 Foundry v13.351＋Playwright・2026-07-23）。
- **正本の追随**: [Active_Effects.md](../01_Wiki/Game_Mechanics/Active_Effects.md) §2.8／§5 の旧「候補外」「datalist は partSlot/part/name/フラグのみ」記述を更新済み。

### AR・メジャー・割り込み・支援の再設計（フェーズ13・2026-08-05）

**日付**: 2026-08-05
**レビュー対象**: `combat-progression.mjs`・`combat-turn-order.mjs`・`tnx-combat.mjs`・`tnx-check-flow.mjs`・`interrupt-grant.mjs`・`focus-system-result.mjs`・`tnx-rl-request-app.mjs`
**ステータス**: レビュー済(問題なし)・実機確認待ち

#### 良かった点

- **純ロジックとグルーの分離を維持**: フェーズ遷移・記帳・割り込みスタックの判断は Foundry 非依存の `combat-progression.mjs`（TDD）に集約し、`TnxCombat` は適用のみ。`isMajorActionTiming` も純関数（`combat-turn-order.mjs`）。既存の 13-3/13-4 の分離方針を踏襲。
- **記帳の一本化**: 旧 `buildEndMainUpdate`＋`buildCantActUpdate`（同一挙動の重複）を `buildArDecrementUpdate` に統合。呼び出し元（メイン終了・行動不能ペナルティ・挿入メイン終了）が同じ関数を共有し、AR減少⟺CS0 の不変条件が一箇所で保証される。
- **所有モデルを consumesAr 一点に外部化**: 自己/他者の割り込みを自動導出せず、用途宣言（`interruptConsumesAr`）→ combatant フラグ → 挿入メインへ伝搬。将来の神業例外を「用途で明示」で吸収でき、分岐の増殖を避ける（ユーザー方針＝2026-07-26）。
- **割り込みスタックで入れ子に耐える**: 単一フレーム（旧 `interruptReturn`）から配列スタック（`interruptStack`）へ。各フレームが親の `interruptConsumesAr` まで退避するため、入れ子でも復帰時に親の有償/無償が正しく戻る。`popInterruptFrame` は入力を破壊しない。
- **既存の起動集約を尊重**: メジャー記帳は `_execute`（唯一のカードプレイ経路）に1点フック。支援対象は既存の `extra→ctx→result` 伝搬に相乗り（新経路を作らない）。委譲は既存の socket 集約（`markMajor` 1種追加）。
- **冪等性**: `_addMajorActed` は重複無視、`markMajorAction` は `_execute` 経由と FS 経由の二重呼びを吸収。`autoApplyFocusSupport` は `focusSupportApplied` で二重記帳を防ぐ。

#### 留意点

- ⚠️ `tnx-check-flow.mjs`／`focus-system-result.mjs` が `TnxCombat` を静的 import。循環は `tnx-combat → tnx-socket-handler →（動的）focus-system-result` が動的のため成立せず（確認済み）。
- ⚠️ 支援を答える時点でカット進行の phase が initiative でない場合、`markMajorAction` は現 phase の `majorActed` に積む（現実には支援はイニシアチブで答える前提）。要実機確認。
- ⚠️ 再判定（allowRecheck）を持つ支援用途では、`focusSupportApplied` の冪等ガードにより支援 AE の付け直しは起きない（成否が変わっても再適用しない）。支援用途は通常 recheck を持たない想定。
- 〔2026-08-05 追補〕支援ボーナスの表現を二転した。①当初 `pendingSupport` カウンタ → ②「フラグだけの AE」= **中身のない見えない効果**（ユーザーが「効果の中身が無い／見えない効果を勝手に付けるな」と是正）→ ③**中身のある本物の change を持つ AE**。最終形：実フィールド `AttributesTemplate.focus.progressBonus`（素値0・integer）を追加し、支援 AE は `change: system.focus.progressBonus +1`（ネイティブ AE 適用が着地）。中核 AE パーサ（`parseEffectTargetKey`/`_applyEffectBuffs`＝§4.3）は**非改変**——アクター上の素の `system.*` 効果は `_resolveBuffApplications` の self 分岐（bearer≠Item）で空返しのためカスタム適用は走らず、ネイティブ適用のみ＝二重加算なし（cs/ar が仮想名前空間で回避しているのと逆に、実フィールド＋actor-borne self の組で自然回避）。進行判定時に実効値 `system.focus.progressBonus` を加算し、その change を持つ AE を除去して消費。可視で RL が手当てでき、consume-once に自然。カット終了の自動失効はフェーズ15 の一般機構へ委譲。
- ⚠️ 旧 13-5 の割り込みは「実機確認済」だったが、AR/戻り/終了ボタンを置換したため**この部分は再確認が要る**（監査エントリに明記）。

### 効果の下書き置き場を「常設の器」から「使い捨ての器」へ（KI-037・2026-08-06）

**日付**: 2026-08-06
**レビュー対象**: `effect-authoring.mjs`・`tnx.mjs`（init/ready）
**ステータス**: レビュー済・実機確認済（隔離 Foundry v13.351＋Playwright）

#### 良かった点

- **器と保存先の分離が元々正しかった**ため、器の寿命だけを変えれば済んだ。効果データの保存先は呼び出し側（アクトのプリセット＝ジャーナルのフラグに実体・付与は `buildGrantedEffectDataFrom` がデータから複製）で、器を参照する経路は存在しない。**「すぐには付与されない」プリセットの性格に対しても、器の生存期間は無関係**（ユーザー指摘に対する確認結果）。
- **標準シートを使うという設計判断は据え置き**（独自エディタを作らない＝注入 UI の二重実装回避）。器が要る理由そのものは変わっていないため、変更は器の寿命と隠蔽の登録時点だけに閉じた。
- **掃除の巻き添えを構造で防いだ**: 置き忘れの掃除対象を `_stats.lastModifiedBy === game.user.id` に限定。時間ヒューリスティクス（「N 分以上前のものを消す」）を使わず、「自分が置き忘れたものだけ片づける」という一文で意味が閉じる。
- **RED→GREEN で検証した**: 修正版が「隠れている」ことだけを見ると、**下書きが存在しないから見えないだけ**の空振りと区別できない。`init` 登録を外した RED 版で「初回描画の一覧に見える」ことを先に再現し、観測が有意であることを担保した上で GREEN を確認した。

#### 留意点

- ⚠️ 隠蔽は**DOM からの除去**であり、Foundry のディレクトリに正規の除外機構が無いことによる。器の寿命が組み立て中だけに縮んだため露出窓は小さいが、機構としては依然ハック。将来コアが除外 API を持てば置き換える。
- ⚠️ 掃除は GM のみ・自分が最後に触った器のみ。**別 GM が置き忘れた器は、その GM が次にワールドを開くまで残る**（隠れてはいる）。単一 RL 運用ではまず起きない。
- ⚠️ 器の作成・削除は組み立てのたびに 1 往復の DB 操作になる（旧＝初回のみ）。効果を組む頻度は低く、体感差は無い。
- 〔経緯〕本件の調査中に「プリセットの効果は付与が後になるので、組み立て時点の持続時間起点（`duration.startTime`/`startRound`）が焼き付くのでは」という懸念を挙げたが、**誤り**として撤回した。コアは期限切れで AE を削除せず（`duration.expired` を立てて changes を抑止するだけ）、本システムは `game.time`/`worldTime` に一切触れないため実時間で持続時間が減る経路が無く、周期による自動失効の実処理もまだ無い（フェーズ15）。現状どこにも作用しない。

### シナリオテキストを RL プリセットへ合流（フェーズ14-8・2026-08-09）

**日付**: 2026-08-09
**レビュー対象**: `templates/journal/scenario-sheet.hbs`・`scripts/journal/tnx-scenario-sheet.mjs`・`scripts/module/request-presets.mjs`・`scripts/module/tnx-scenario-panel.mjs`
**ステータス**: レビュー済・実機確認済（隔離 Foundry v13.351＋Playwright）

**経緯**: アクトシートのテキストタブには送信導線が無く（実行系は 14-7 でパネルへ完全移行済）、同種の「卓で RL が使う素材」は RL プリセットタブに集まっている。ユーザー指示によりテキストタブを廃し、RL プリセットタブの先頭へ合流（構造・配置ともユーザー承認）。**送信機能そのものは未着手**——シーンコントロールバーまたはシナリオコントロールパネルに新規追加する（2026-08-09 ユーザー明言・本作業は内容の移動のみ）。

#### 良かった点

- **形でなく機能で括った**: シナリオテキストは「名前が付いた、RL が卓で選んで送る素材」であり、判定要求・報酬点・ダメージ・効果と同じ役割段にある。よって `.preset-item` 意匠に載せ、保存・削除・並べ替えを既存の汎用プリセット機構（`data-preset-kind`＝保存先フラグキー駆動の `_onPresetFieldChange`／`_onDeletePreset`／`_movePreset`）へ委ねた。専用アクション2種・専用 change リスナー・専用 CSS4 規則が消え、正味の追加は新規行ファクトリのみ。
- **並べ替えが無償で付いた**（TNX 標準＝表・リスト系は既定で手動並び替え可）。旧テキストタブには無かった。
- **データを書き換えていない**: 保存先フラグは `scenarioTexts` のまま、名前欄のキーも既存データの `title` を保持（他プリセットの `label` に合わせる改名をしない）。パネル側の既存読み出し（`sendText`）は無改修で通る。
- **prose-mirror の配線を一箇所に寄せた**: `change`＋`save` 両購読の `bind` ヘルパーをプリセット群にも適用（従来はシーン行・テキスト・情報項目のみ）。プリセットに長文欄が増えても同じ経路で拾える。
- **`presetLabel` の一般化を最小に留めた**: `label ?? title` の1段フォールバックのみ。既存の全プリセット（`title` を持たない）の挙動は不変。

#### 留意点

- ⚠️ **削除の確認ダイアログはテキストだけ残した**（`deleteTextPreset`）。他プリセットの削除は無確認だが、ここで消えるのは設定値ではなく書き溜めた本文そのもののため、旧テキストタブの安全弁を落とさない判断。意匠は共通のままで挙動だけ違う点は意図的。
- ⚠️ 名前欄のキーだけが他プリセットと異なる（`title` / 他は `label`）。テンプレートとファクトリにコメントを置いたが、将来プリセット横断の処理を書くときは `presetLabel` を通すこと。
- ⚠️ **本文エディタは1件あたり 200px 固定**のため、テキストが増えると同タブ下部の判定要求以下へ到達するまでのスクロールが伸びる（旧タブでは独立していたので顕在化しなかった）。折り畳み等の圧縮は未対応——実運用で邪魔になったら扱う。
- 〔申し送り〕送信導線（シーンコントロールバー or シナリオコントロールパネル）は別途新規実装。パネル側の一覧ラベルは未入力時「テキストn」に揃えた（従来は全件「テキスト」で送信先を選べなかった）。

### シナリオコントロールパネルの現在シーン表示（シーンプレイヤー／登場判定）（フェーズ14-8・2026-08-09）

**日付**: 2026-08-09
**レビュー対象**: `templates/app/scenario-panel.hbs`・`scripts/module/tnx-scenario-panel.mjs`・`scripts/module/appearance-logic.mjs`（`formatAppearanceSummary`）・`scripts/module/skill-dictionary.mjs`（`formatGroupedSkillNames`）
**ステータス**: レビュー済・隔離描画で組版確認済（実テンプレート＋実 CSS・Playwright スクリーンショット）

**経緯**: ユーザー指摘「シーン表示ですが、現状必要な情報が一切表示されていません」。上演中に卓が見るべき値＝**シーンプレイヤー・登場判定の目標値・登場に使用する技能**。表記はユーザー指定＝「シーンプレイヤー：キャスト名」（ルーラーシーンはラベルなしで「ルーラーシーン」とだけ）・「登場：〈技能名A〉〈技能名B〉 10」。**同カテゴリの固有名詞技能はまとめて表記**（社会なら〈社会：N◎VA、ストリート〉・コネなら〈コネ：キース・シュナイダー、エウラリア〉）。

#### 良かった点

- **目標値は判定側と同じ算出を呼ぶ**（`appearanceCheckParams`）。表示のために TN の表を再実装せず、エリア準拠／数値指定／不可の分岐も判定と一致する。アクター依存の引数（危険値・サンクチュアリの装備）は渡さない＝シーン設定だけで決まる部分を出す、という切り分け。
- **シーンプレイヤーの解決を1関数に集約**（`resolveScenePlayer`）。パネルの表示と切替チャット（`_performSceneEntryEffects`）が同じ判定・同じ表示名を使う（従来は同じ式が2箇所に複製され、片方だけ直す余地があった）。**ルーラーシーンの述語は `session-state.isRulerScene` に抜き出して3箇所目（`_applySceneEntry` の isScenePlayer 付与）とも共用**——表示と状態機械が「ルーラーシーンとは何か」で食い違わないことを構造で保証する。
- **表示名はキャスト名**（ユーザー指定）＝`user.character`。担当未設定はユーザー名、旧自由文字列 `player` を最後のフォールバック（旧データを書き換えない従来方針のまま）。
- **技能名のまとめは辞典側の語彙で実装**（`formatGroupedSkillNames`）。小分類は既存の `idKeyPrefix`＋`ONOMASTIC_TYPES` をそのまま使い、名前の接頭（「社会：」）を落として「、」で連ねる。逆引きできないキーは落とす（生キーを表示しない＝識別キー規約）。まとめる位置はその小分類の初出位置で、無条件取得技能は個別に並ぶ。
- **技能と目標値の間は不改行スペース**（U+00A0）。隔離描画で普通の空白だと 420px 幅で目標値だけが次行に取り残される（「10」が単独で浮く）ことを実際に確認してから入れた。
- 純関数2つに TDD（テスト +8・全 1920 green）。表示文字列は**実装から生成して描画検証**した（手打ちの文字列で見た目を確かめない＝最初の描画では手打ちの普通空白で検証してしまい、実装との差に気付いて取り直した）。

#### 留意点

- ⚠️ 目標値も指定技能も無い（エリア未設定＋指定なし）シーンでは「登場：」行そのものを出さない。空ラベルの器を残さない方針だが、**RL から見て「設定し忘れ」と「意図的な未設定」は区別できない**（区別が要るならアクトシート側の警告で扱う）。
- ⚠️ 「登場：不可」は `appearanceMode === "none"` の表示。サンクチュアリの装備による登場不可は**アクターごと**に決まるため、この行には出さない（判定実行時の警告が担当）。
- ⚠️ エリア表示の区切りを他項目と揃えて全角コロンにした（「エリア: 」→「エリア：」）。既存の切替チャット（`buildSceneSwitchMessage`）は `<strong>シーンプレイヤー:</strong>` のままで、そちらは太字ラベル書式のため触っていない。

### RL による登場・名前の非公開・シーン行の登場キャラクター事前設定（フェーズ14-8・2026-08-09）

**日付**: 2026-08-09
**レビュー対象**: `appearance-logic.mjs` / `appearance-state.mjs` / `session-state.mjs` / `tnx-scenario-panel.mjs` / `tnx-scenario-sheet.mjs`
**ステータス**: レビュー済(問題なし)

#### 設計判断

- **状態の置き場**: 名前の非公開は登場に付随する状態なので、登場状態と同じ器（Actor フラグ）に `appearingHidden` として置き、窓口も `appearance-state.mjs` 一本に保った（フラグ直読み・直書きを散らさない既存規範）。`setAppearing(actor, true, {hideName})` の **hideName 省略時は現在の指定を保つ**——トークン表示切替からの再登場（`updateToken` フック）が RL の指定を落とさないため。退場は両フラグを落とす。
- **表示解決の一本化**: `displayActorName(actor)` を唯一の解決点にし、パネルの「登場中」一覧と**チームのメンバー表示の両方**を通した。隣で実名が出ていたら伏せる意味がないため、表示箇所を漏らさないことが機能要件そのもの。RL には実名を返し、状態は目のアイコンで示す（判断材料を RL から奪わない）。
- **純関数の共用**: パネルの登場候補と台本の事前設定プルダウンは同じ「キャラクターを type 別に並べる」問題なので、`groupCharacterChoices(actors, {labelOf, excludeAppearing})` 1本に集約した（初版は候補列挙用に `appearanceCandidates` を別に作っていたが、重複するため撤去）。i18n と `game.actors` は呼び元が渡す＝純関数のままテストできる。
- **シーン入場の集合**: シーンプレイヤーの自動登場と事前設定を `sceneEntryAppearances` に集約し、`_applySceneEntry` は集合を回すだけにした。「誰が登場するか」の規則が1関数に閉じ、テストで固定できる。
- **UI は既存意匠の流用**: アクトシートは指定技能とまったく同じタグ入力（`tnx-tag-field` ＋ 追加プルダウン ＋ チップ）で、チップ内トグルだけ `.tnx-tag-toggle` を新設。パネルは舞台裏・チーム編成と同じ `scp-load-row`（プルダウン＋アイコン）と `scp-chip`／`scp-chip-remove`。新規 CSS は3ルールのみ。
- **追加行の「名前を伏せる」トグルは DOM ローカル**: 保存先を持たない一時的な選択なので、再描画を挟まず class を反転させるだけにした（登場のたびに既定＝伏せないへ戻る）。付け間違いはチップ側のトグルで直せるため、往復の導線は閉じている。
- **`CHARACTER_ACTOR_TYPES`** を `data/helpers.mjs` に新設（`OUTFIT_ITEM_TYPES` と同じ「単一ソースの型集合」の位置）。既存の同義のインライン配列（`condition-edit.mjs`）は今回の範囲外として触っていない。

#### 検証

- 純ロジックのテスト +10（`isAppearanceBlockedScene` / `normalizeAppearanceActors` / `sceneEntryAppearances` / `groupCharacterChoices` と `normalizeSceneRow` の新フィールド）。全体 **1956 green**（92ファイル）・eslint 0 error（警告15は既存）・stylelint clean。
- **隔離 Foundry（実ワールド複製・GM と PL の2クライアント）で実操作を確認**: 候補プルダウンの4群と登場中の除外／名前を伏せた登場と PL 側の「？？？」／PL に RL 操作が描画されないこと／チップの目トグル往復／✕退場で両フラグ消滅／「登場：不可」シーンでチーム免除が効かず RL 登場は通ること／事前設定がシーン入場で `hideName` 込みで適用されること／「登場：不可」の行でも登場キャラクター欄が出ること。

#### 課題・申し送り

- 盤面トークンのネームプレートは非公開の対象外（前掲メカニクス監査の申し送りと同じ）。実装するなら `token.displayName` の一時変更になり、RL のトークン管理と競合するため設計が要る。

### 巡回シーンとイベントシーン（フェーズ14-8・2026-08-09）

**日付**: 2026-08-09
**レビュー対象**: `session-logic.mjs` / `appearance-logic.mjs` / `session-state.mjs` / `scene-entry-dialog.mjs`（新設） / `residence-area.mjs`（新設） / `tnx-scenario-panel.mjs` / `tnx-scenario-sheet.mjs`
**ステータス**: レビュー済(問題なし)

#### 設計判断

- **足りないのは「行を増やす手段」ではなく「行の種類」**。シーンの数・シーンプレイヤー・エリア/目標値・イベントの起動点の4つが台本に固定できないという診断から、`kind` を1つ足して既存の行モデルを拡張した。実行時に台本へ行を追記する案（ユーザーの正本データが上演のたびに伸びる）と、リサーチを台本の外の巡回機構にする案（`sceneId` が台本行 id である前提が全経路にあり、仮想行の概念を通すコストが高い）は却下。**既定 `normal` で既存の台本の挙動は不変**。
- **巡回状態は位置ではなく集合**。`scenePlayerDone`（消化済みユーザー）を正本にしたので、「イベントで務めた分だけ順番が飛ぶ」が**規則の帰結として出る**（特例処理を書かない）。記帳は種別を問わずシーン入場時に一律で行う——シーンプレイヤーが立った事実そのものを記録する形にすると、後からどの種別が増えても取りこぼさない。
- **その場で決めた値は台本を汚さない**。`sceneOverride` を実行状態に置き、行との合成は `resolveSceneAppearance(row, override)` 1本に集約した。パネルの「登場：」表示・登場判定・切替チャットが同じ関数を通るので、値の出所が食い違わない（14-8 の「表示のための再実装をしない」方針の踏襲）。
- **住宅施設は既存の判定経路に乗せる**。`{area, mode:"fixed", fixedValue}` の形に落として `appearanceCheckParams` へ渡すだけにしたので、危険値係数（グリーン×1・ホワイト×2）とサンクチュアリの装備ゲートが**新しいコードなしで**効く。住宅エリアの解決子はシートクラスの静的メソッドだったものを `residence-area.mjs` へ切り出し、シートとダイアログで共用（呼び元の挙動は変えていない）。
- **舞台候補はシーンプレイヤーに依存するので、ダイアログ内で完結させる**。ユーザーごとの「見えるキャラクター」表を先に作って渡し、シーンプレイヤーを選び直したらクライアント側で絞り込み直す（ダイアログ内で非同期の再取得をしない）。
- **ボタンの出し分けは純関数に置く**。`canShowNextScene` / `nextSceneTarget` / `eventSceneCandidates` / `areEventScenesDone` はすべて台本と実行状態だけを見る純関数で、パネルは結果を描くだけ。`eventSceneCandidates` は現在地がイベント行なら**その群の先頭まで遡る**——想定と違う順で起動しても、同じ群に残ったイベントが次の候補として出る。
- **UI は既存意匠の流用**: シーン開始ダイアログの指定技能はシートと同じ `tnx-tag-field`、イベント選択の縦積みボタンは `.tnx-scene-end-dialog` で確立した `.form-footer` の縦積みパターン、巡回の消化状況は舞台裏の枠（`scp-backstage`）と同じ意匠。

#### 検証

- 純ロジックのテスト **+47（全 2003 green・92ファイル）**・eslint 0 error（警告15は既存）・stylelint clean。
- **隔離 Foundry で実操作を確認**: 台本の種別列と巡回/イベント行の欄の出し分け／シーン開始ダイアログ（舞台=任意でエリア＋目標値、住宅施設を選ぶと両欄が畳まれる、エリア選択で目標値が 8/10/12 へ、初期値 10、指定技能の追加）／住宅施設の実効目標値 14＝基本12＋エリア修正2 とエリア=ホワイトの適用／シーンプレイヤーを選び直すと舞台候補が入れ替わること／パネルの巡回状況（済み／次）／3ボタンの出し分け（巡回＝次のシーンへ＋イベント起動、イベント→イベント＝次のシーンへを出さない、イベント→巡回＝両方、全イベント消化＝クライマックスへ）／イベント選択ダイアログと「起動せず実行済みにする」／フェイズ移動での消化リセット。
- **レイアウトの是正（描画して見つけたもの）**: ①種別列を足したことで見出し「シーンプレイヤー」が省略された → シート幅を 830→924（列 88px＋gap 6px の分）に広げて既存列の幅を元に戻した ②ダイアログのスピナーだけが右端に寄る（共通 `.number-input-spinner` の `justify-content: flex-end` ＋ Foundry の `.form-fields` の `flex-end`）→ ダイアログ内だけ先頭寄せ＋スピナーを内容幅に ③イベント選択のラジオが見えず選択状態が分からない → ラジオに寸法を与え `:has(:checked)` で行を強調 ④「起動せず実行済みにする」が横並びで2行に折れる → footer を縦積み。

#### 課題・申し送り

- ⚠️ **イベント行の群を消化しきった状態でイベント行にいると、前進ボタンが出ない**（`次のシーンへ` は「次がイベント」で塞がれ、起動候補は空）。シーン一覧のジャンプが逃げ道。ユーザー指示の帰結なので実装は素直に従い、扱いは裁定待ち。
- ⚠️ `doneEventSceneIds` は行を削除しても残る（無害だが掃除はしない）。
- ~~キャストシートの「登場」列が住宅施設の実効値を読んでいない~~ → **KI-039 として起票し同日中に解消**（下記エントリ）。

### 表示の実効値読みの取り残しを一掃（KI-039・2026-08-09）

**日付**: 2026-08-09
**レビュー対象**: `residence-area.mjs` / `tnx-character-sheet-base.mjs` / `tab-abilities.hbs`
**ステータス**: レビュー済(問題なし)

#### 設計判断

- **規約は既にあった**——`_computeColValue` の doc コメント「表示は AE 込み実効値(total)。編集入力は base のまま(フェーズ9-3)」。今回直したのは規約の追加ではなく、**規約から漏れていた3系統の一掃**。同じ規約の下にある `_computeCombinedColValue` が `.value` 読みのままだったのは、2026-07-13 に `attack` だけ実効値へ直した**部分的な修正の取り残し**だった。
- **住宅施設は算出そのものを1関数へ**（`residenceEffectiveValues`）。読み手が2箇所（キャストシートの列・14-8 の舞台）あるので、`…Total ?? 素値` の読み方をどちらにも書かず、住宅エリア修正の加算ごと関数に閉じた。**住宅エリアが無くても実効値を返す**ので、旧実装にあった「エリアが無ければ素値」というフォールバック分岐が消えた（分岐が消えたこと自体が、同じ間違いを再発させない形）。
- **コンバイン行は `mvT` を導入して全列を通した**。列ごとに `.total ?? .value` を書き散らすと同じ取り残しが再発するので、`_computeColValue` と同名・同定義のローカルヘルパーを置いて「この関数の中では素値を直接読まない」形にした。
- **編集入力欄は触っていない**（`outfit-sheet.hbs` の増減ボタン付き入力・`tab-abilities.hbs` のレベル入力・防具の防御値入力など）。base を書くのが正しいため。

#### 検証

- 隔離 Foundry: 住宅施設の列＝実効99＋エリア修正2で **101**、**住宅エリアを外しても 99**（旧実装は素値 12 に戻っていた＝この分岐が本丸）、セキュリティ **77／66**、技能レベル **Lv9**。
- コンバイン列は `_computeCombinedColValue` を直接呼び、**素値のみのときの出力が従来と一字一句同一**であること（回帰なし）と、`.total` があるとき11列すべてが実効値になることを確認。
- テスト 2003 green・eslint 0 error・stylelint clean。

#### 課題・申し送り

- 全数掃引は「素値と実効値で綴りが異なるフィールド」（`level`/`FAValue`/`appearanceTarget`/`cyberSecurity`/`analogSecurity`/`uses.max`/`attack.damageType`/`defence.*`）と modeValue の `.value`/`.total` を対象に行った。残った素値読みはすべて**編集入力欄と書き込み経路**で、意図どおり。

### 巡回・イベント進行の是正とシーン番号の自動カウント（フェーズ14-8・2026-08-09）

**日付**: 2026-08-09
**レビュー対象**: `session-logic.mjs` / `session-state.mjs` / `tnx-scenario-panel.mjs` / `tnx-scenario-sheet.mjs`
**ステータス**: レビュー済(問題なし)

#### 設計判断

- **「形」でなく「機能」で塞ぐ**。「イベントの次がイベントなら次のシーンへを出さない」を*次の行の種別*だけで実装したのが誤りだった。規則の機能は「**起動条件を踏んでいないイベントへ順送りで滑り込ませない**」ことなので、判定対象は「次の行の種別」ではなく「**読み飛ばした先が未実行のイベントか**」。`nextSceneTarget` に実行済みイベントのスキップを入れ、`canShowNextScene` はその結果を見るようにして、行き先の決定と可否の判定が同じ1本を通るようにした（2つの規則が食い違わない）。
- **候補の範囲は「上は全部・下は次の巡回まで」の非対称**。上へ広げるのは「踏まれずに残ったイベントを失わない」ため、下を止めるのは「先の段階のイベントを今出さない」ため。目的が違うので範囲も非対称でよい。実装は前方走査と後方走査の2ループで、走査順がそのまま台本順になるので**先頭＝初期選択＝最も上にある未実行のイベント**が自然に決まる（並べ替えを書かない）。
- **番号は2種類あることを認めた**。台本の No.（行の並び順）と上演中の SCENE n（何シーン目か）は、巡回シーンがあると必ず食い違う。前者は `sceneSequenceNumbers`（純関数・台本から導出）、後者は `sessionState.sceneNumber`（入場のたびに +1）で、**別物として別の場所に置いた**。手入力の `number` は書かなくなったが読み出し互換のためフィールドは残す（既存データを壊さない）。
- `buildSceneSwitchMessage` は `number` を任意引数で受け、未指定なら従来どおり行の `number` を使う（純関数のまま・呼び元が上演中の番号を渡す）。

#### 検証

- 純ロジックのテスト +10（**2013 green**）。`nextSceneTarget`/`canShowNextScene` の実行済みスキップ、候補の上方向拡張と台本順、`sceneSequenceNumbers`、`buildSceneSwitchMessage` の番号上書き。
- **隔離 Foundry**: 上演 SCENE 1→2→3→4→5（巡回の再入場を含む）／台本 No. 1〜8 の自動採番／巡回2 で上の未実行「襲撃」と下の「決意」が候補に並び初期選択が「襲撃」／実行済みイベントを読み飛ばして「次のシーンへ（リサーチ）」が出ること（是正前は前進ボタンが皆無だった位置）。

#### 課題・申し送り

- 台本の No. は読み取り専用表示になったので、行の並べ替え UI は無いまま（順序を変えるには行を作り直す）。従来と同じ制約で、今回広げていない。

### ダイアログの行間と器の扱い（2026-08-09 ユーザー指摘）

**日付**: 2026-08-09
**レビュー対象**: `css/tnx2.css`（ダイアログ共通） / `condition-edit.mjs`
**ステータス**: レビュー済(問題なし)

#### 原因

DialogV2(v13) の `.dialog-content.standard-form` は `display:flex; gap:16px` で、**直下の子**に行間を与える。TNX のダイアログは CSS スコープ用の器（`<div class="...">`）で本文を包むことが多く、その場合 `.form-group` は器の子になるため gap を受け取れず、**行が隙間なく密着する**。器を使うかどうかという実装都合が、そのまま見た目の差になっていた。

#### 設計判断

- **器が `.form-group` を直に並べているときだけ**、同じ縦並び＋間隔を器側に持たせる（`.tokyo-nova .dialog-content > div:has(> .form-group)` と `> form:has(...)`）。器ごとに個別ルールを足すと同じ抜けが再発するので、条件を「form-group を直に並べている器」という**機能**で書いた。
- `gap: inherit` を使い、`.dialog-content` の値をそのまま引き継ぐ（Foundry 側が変えても追従し、16px という数字をこちらで持たない）。
- **直下に `.form-group` を持たない器は対象外**＝`.tnx-damage-form`（行の器を挟む）・`.tnx-event-choices`（gap 4px の独自意匠）は影響を受けない。実測で確認済み。
- 影響のあった器: `.tnx-scene-entry`（14-8）・`.tnx-joker-wild`・`.usage-type-select-dialog`（単一行のため見た目の変化なし）。

#### 併せて見つかった別件（同じ「ダイアログの器」の問題）

- **`condition-edit.mjs` の器が `<form>` だったため、クラスごとパーサに落とされていた。** DialogV2 は本文を自前の `<form>` の中に描くので、入れ子の `<form>` は不正で除去される。結果 `.tnx-select-dialog`（2026-07-15 ユーザー指摘で作った「ラベルを全幅で上に置き、プルダウンは広げない」意匠）が**一度も効いていなかった**。器を `<div>` に変更して復活させた。
- 復活させた意匠を初めて実機で見たところ、`.form-fields` の `justify-content: flex-end`（Foundry 既定）で**幅を絞ったプルダウンだけが右へ飛び**、ラベルと縦がずれていたので先頭寄せに是正した。
- `tnx-check-flow.mjs` の `<form autocomplete="off">` も同様に落ちているが、`.form-group` は直下に出るため**レイアウトの実害はない**（`autocomplete` は select に効かないので機能的にも無害）。今回は触っていない。

#### 検証

- 隔離 Foundry で、器あり／器なしの行間を実測して**一致（16px）**を確認。`.tnx-damage-form` が対象外のままであることも確認。
- 効果の編集ダイアログ・ジョーカーのワイルド指定・シーン開始ダイアログを描画して確認。シーン開始ダイアログはタグ入力行のラベルをプルダウンの行に合わせた（シートの `.scene-tag-row` と同じ規則。中央揃えのままだとチップが増えるたびにラベルが動く）。
- テスト 2013 green・eslint 0 error・stylelint clean。

### ルーラーシーンの分離と消化記帳のリサーチ限定（フェーズ14-8・2026-08-10）

**日付**: 2026-08-10
**レビュー対象**: `session-logic.mjs` / `session-state.mjs` / `tnx-scenario-sheet.mjs`＋`scenario-sheet.hbs`
**ステータス**: レビュー済(問題なし)

#### 設計判断

- **1つの選択で2つのことを表さない。** 14-7 の「GM ユーザー選択＝ルーラーシーン」は、シーンプレイヤー欄に「誰が主役か」と「ルーラーシーンか」を同居させていた。RL がキャストを持ちハンドアウトの割り当てもある場合、この2つが衝突して**RL のキャストを主役にできない**。ルーラーシーンを独立した選択肢（`@ruler`）に切り出して含意を消した。以後 `isRulerScene` は `@ruler` と旧 `isMasterScene` だけを見る。
- **参照の型は変えない**（当初「キャスト参照にすべきか」を検討したが見送り）。バグの原因は参照の型ではなく暗黙の含意だった。ハンドアウトも巡回順もユーザー参照なので、ここだけキャスト参照を混ぜる方が筋が悪い。代わりに**表示をキャスト名主体**（「キャスト名（ユーザー名）」）にして、欄の読まれ方と実装の参照を揃えた。
  - 手放した点: ユーザーの割当キャラクターを差し替えると台本のシーンの主役が黙って変わる。ユーザーに提示済み。
- **記帳は「リサーチのシーンだけ数える」**。旧実装は全フェイズで記帳し「フェイズが変わったら捨てる」形だったため、**破棄より前に読むシーン開始ダイアログがオープニングの記録を引きずり**、リサーチ先頭の巡回シーンで無関係な人が「済」になっていた。条件を「どこで数えるか」に置き換えたことで、順序に依存する破棄そのものが不要になった（読む側の順序を気にしなくてよくなる）。判断は純関数 `recordScenePlayerDone` に切り出してテストで固定。
- **巡回順の入口を1本にした**（`getRotationOrder()`）。順番・「済」表示・既定値・候補の4つが必ず同じ並びを見る。キャスト未割当のユーザーを落とす条件もここ1箇所。以前は `rotationOrder(_activeHandouts())` を3箇所で個別に呼んでおり、フィルタを足すと食い違う余地があった。
- **GM を特別扱いしない**。当初「巡回順から GM を除外する」案を出したがユーザー指摘で撤回——RL がキャストとハンドアウトを持つなら巡回の正当な参加者。除外すべきは「キャストを割り当てていないユーザー」であって GM ではない。
- 巡回シーンの候補が0件なら**警告して切替を中止**する（空のセレクトを出して無言で壊れるのを避ける）。

#### 検証（隔離 Foundry・RL にキャストとハンドアウトを割り当てた状態）

- 巡回順に `RLのキャスト / Gamemaster` が**含まれる**こと。キャスト割当を外すと落ち、戻すと復帰すること。
- オープニング2シーンを通しても消化記録が空のままで、リサーチ先頭の巡回ダイアログに**「済」が1つも付かない**こと・既定が先頭のキャストであること。
- 巡回ダイアログの候補が**キャスト名表示**で、**「ルーラーシーン」が出ない**こと。
- 巡回で「RLのキャスト」を選んでも**ルーラーシーンにならず**（`isRulerScene: false`）、「シーンプレイヤー：RLのキャスト」と表示され GM に `isScenePlayer` が立つこと。
- 通常シーンで `@ruler` を選ぶとルーラーシーンになり誰にもフラグが立たないこと。**GM ユーザーを選ぶとその RL のキャストが主役**になること。
- 旧 `isMasterScene` の行がルーラーシーンとして読まれること。
- アクトシートの候補が「- 未選択 -／ルーラーシーン／キャスト名（ユーザー名）…」で並ぶこと。
- テスト **+11＝2024 green**・eslint 0 error・stylelint clean。

#### 課題・申し送り

- ⚠️ **14-7 から本日までの間に「GM ユーザー選択」で作られたルーラーシーンの行は、意味が変わる**（その RL のキャストが主役として読まれる）。該当する行はシートで「ルーラーシーン」を選び直す必要がある。旧 `isMasterScene` の行は影響なし。データの自動書き換えはしていない。

### シーンプレイヤーの指定をハンドアウト参照にする（フェーズ14-8・2026-08-10）

**日付**: 2026-08-10
**レビュー対象**: `session-logic.mjs` / `session-state.mjs` / `tnx-scenario-sheet.mjs`＋`scenario-sheet.hbs` / `css/tnx2.css`
**ステータス**: レビュー済(問題なし)

#### 設計判断

- **台本の言葉で書けるようにする。** シナリオは「SCENE 2：シーンプレイヤー＝HO①」の形で書かれ、担当はプレアクトで決まる。ユーザー参照だと担当が変わるたびに台本を書き直すことになり、**巡回シーン（ハンドアウト順に回す）と通貨も揃わない**。ハンドアウト参照にすると、固定シーンも巡回も同じ言葉（ハンドアウト）で表せる。
- **解決は1段増えるだけで、進行の機構には触らない。** 実行状態は従来どおり解決済みのユーザー id（ルーラーシーンは `@ruler`）を持つので、パネル表示・切替チャット・`isScenePlayer` の付与・舞台候補・巡回・消化の数え方はすべて無改修。解決点は `_requestSceneEntry` の1箇所に閉じ、`_applySceneEntry` はその結果を使うだけにした。
- **旧データは読み替えで吸収**（一括書き換えなし）。優先順は `playerHandoutId` → 旧 `playerUserId`（ユーザー参照／`@ruler`）→ 旧 `isMasterScene` → 旧 `player`（自由文字列）。シートでは旧指定を空選択肢のラベル「旧: 〜」で見せ、選び直せば新形式へ移る（**何が指定されていたかを隠さない**）。
- **担当が解決できない指定は警告する**（対象ユーザー未設定・参照切れのハンドアウト）。台本はプレアクト前に書くので未割当のハンドアウトも指定できる必要があり、**候補から外すのではなく実行時に気づかせる**形にした。
- ラベルはハンドアウト名（「①〈カブト〉用ハンドアウト」）ではなく **「①スタイル名（キャスト名）」**（ユーザー指定）。台本の欄には「用ハンドアウト」が冗語で、担当が見えることの方が上演直前に効く。純関数 `handoutPlayerLabel` に切り出し、辞典のスタイル名解決とキャスト名解決は呼び元から渡す。

#### 検証

- 純ロジックのテスト +13（**2037 green**）。ラベルの4形（通常・キャストなし・自由記述・スタイル未選択）と解決の8経路（ハンドアウト／対象未設定／参照切れ／`@ruler`／旧ユーザー参照／旧 `@ruler`／旧 `isMasterScene`／優先順）。
- **隔離 Foundry**: 候補が「- 未選択 -／ルーラーシーン／①カゼ（キャスト）／②カブキ（キャスト (2)）／③マネキン／黒幕（RLのキャスト）」で並び**共通ハンドアウトが出ない**こと／旧データ行が「旧: キャスト (2)」と出ること／HO 指定・旧ユーザー参照・ルーラーシーンがそれぞれ正しく解決されること／**ハンドアウトの担当を差し替えると台本を触らずにシーンプレイヤーが追随する**こと／対象ユーザー未設定の指定で警告が出てシーンプレイヤー不在になること。
- **レイアウト**: 新ラベルが従来の列幅（2fr=124px）に収まらず切れたため、列配分を `3:2:3` → **`5:5:4`** に変更（シート幅は 924 のまま・シーンプレイヤーをシーン名と同幅にし、値の短い使用シーンから分けてもらう）。あわせて使用シーンの空選択肢を「- 使用シーンなし -」→**「- なし -」**に（列見出しと重複していた）。実測で見出し・値ともに省略なし・横スクロールなしを確認。

### 情報項目の技能行のレイアウト（2026-08-10 ユーザー指摘）

**日付**: 2026-08-10
**レビュー対象**: `templates/journal/scenario-sheet.hbs` / `css/tnx2.css`
**ステータス**: レビュー済(問題なし)

#### 原因

技能行は「左半分の器（チップ領域＋追加プルダウン）｜右に目標値」という横並びだった。器が行の半分しか占めないため、**チップが右へ伸びず**（目標値スピナーの上が空く）、**追加プルダウンの右にも空白**が残っていた（`.tnx-tag-add` の共通ルールが `max-width: 180px`）。

#### 設計判断

- **段の役割で組み直した**。上段＝チップ領域（行の全幅）、下段＝追加プルダウン＋目標値。器（`.tnx-tag-field`）で左右に分けるのをやめ、`.skill-check-row` を縦並びにして `.skill-check-bottom` を新設。チップは行の全幅を使い、プルダウンは下段の残り幅を埋める。
- `.tnx-tag-add` の共通ルール（左寄せ・180px 上限）は**この行の中だけ**で外した（`.skill-check-bottom > .tnx-tag-add`）。他の場所のタグ入力（FS エディタ・シーン行）は従来どおり。

#### 検証

隔離 Foundry で実測: チップ領域の幅が行の内幅と一致（610/628・右端まで到達）／プルダウンの右端 402 と目標値の左端 410 が 8px の隙間で接する／プルダウンと目標値が同じ行に揃う。未設定・1件・6件の3状態で確認し、6件でも1行に収まった。stylelint clean。

### コネの参照先にワールド直下を含める（2026-08-12 ユーザー要望）

**日付**: 2026-08-12
**レビュー対象**: `scripts/module/skill-dictionary.mjs` / `scripts/journal/tnx-scenario-sheet.mjs` / `scripts/module/tnx-scenario-panel.mjs` / `scripts/module/session-state.mjs`
**ステータス**: 実装済（`d11f629`）・隔離 Foundry で検証済

#### 原因

アクトコネクションは「そのアクト限りのコネ技能」なのに、参照先が一般技能辞典だけだった（2026-08-08「辞典格納が運用前提」）。恒久的な辞典に使い捨ての NPC コネを積むことになる。

#### 設計判断

- **1つの索引に3経路を載せる**。`actConnection`（識別キー）を使う場所は、選択肢（アクトシート）・表示名（パネルの送信）・コピー付与（アクト開始）の3つ。**どれか1つだけ広げると必ず壊れる**——選べるのに表示が「（参照切れ）」になり、アクト開始で「見つかりません」と警告される。新設 `loadContactSkillIndex()` に3箇所とも寄せ、絞り込み・優先順位の規則が1箇所にしか無い状態にした。
- **共有の逆引き（`loadGeneralSkillNameByKey`）は触らない**。範囲が「コネだけ」に確定した以上、シーン指定技能・情報項目と共有する関数を広げるのは範囲の食い違い。コネ専用索引を新設して差し替えるほうが、変更の届く先と要望の範囲がぴったり合う。
- **絞り込み条件は辞典とワールドで同一**（一般技能かつ固有名詞技能かつ識別キー `contact_`）。出所で条件を変えると「ワールドだと出るのに辞典だと出ない」類の説明できない差が生まれる。
- **純関数への切り出し**: `mergeContactEntries(packEntries, worldEntries)` が絞り込み・辞典優先・並びを担い、Foundry 依存（`game.packs` / `game.items`）は収集だけ。規則をテストで固定できる。
- **uuid は `pack.getUuid()` で作る**（`Compendium.<pack>.Item.<id>` の文字列組み立てをしない）。書式の権威は Foundry 側にあり、v13 の API リファレンスで `getUuid` の存在を確認したうえで実機でも `typeof === "function"` を確認した。
- **解決は `fromUuid` の単品取得**。列挙は `getIndex`・単品は `getDocument`（キャッシュ優先）という KI-026 の規範に沿う（`fromUuid` は辞典 uuid に対して内部で `getDocument` に落ちる）。

#### 却下した案

- **プルダウンを出所（辞典／ワールド）で optgroup 分割**（Code 初案・ユーザー指摘で撤回）。辞典優先で同一キーが1件に畳まれる以上、分割が解くのは「同名・別キー」だけで、それは辞典の中だけでも起こる。**この変更が新しく生む問題ではないものを、この変更のために UI で説明しようとしていた**＝求められていない表示。
- **保存値を UUID に変える**（＝2026-08-08 の D&D 案に戻す）。識別キーの無いワールドアイテムも選べるようになるが、既存データの読み替えがもう一段増える。ユーザーが「プレフィックス統一は前提」と明言したため不要。

#### 検証

- 純ロジック **+6＝2043 green**（コネだけを集める／辞典優先／小分類違いを採らない／識別キー無しを採らない／名前順／空入力）。
- **隔離 Foundry**: 辞典2件＋ワールド2件（うち1件は辞典と同キー）を作って index を確認——`contact_kaburi` が辞典側の名前で1件に畳まれ、ワールド固有の `contact_worldtaro` が「ワールド」出所で並ぶ。アクトシートのプルダウンに optgroup が無く4択（空＋3件）で出る／選択が `actConnection` に保存される／送信 HTML が `〈コネ：ワールド太郎〉`／`startAct()` でキャストに `isActLimited: true` でコピーされ「コネを 1 件配布しました」／参照切れキーで「コネ技能が見つかりません(RLのキャスト)。」。
- ワールドアイテムの `toObject()` に乗る `folder`（null）・`ownership`（`{default:0, <GM>:3}`）は埋め込みで問題を起こさない（事前に懸念していた点を実測で解消）。
- eslint 0 error（警告15は既存）。CSS 未変更のため stylelint は対象外。

### 技能の源を収集層で一般化する（2026-08-12・前項の範囲是正）

**日付**: 2026-08-12
**レビュー対象**: `scripts/module/skill-dictionary.mjs`
**ステータス**: 実装済（`96a4be2`）・隔離 Foundry で検証済

#### 原因

コネだけワールド直下を読む実装（`d11f629`）が、呼び出し側（選択肢・逆引き・付与）を個別に差し替える形だった。範囲が技能全体へ広がったとき、同じやり方では**9 箇所の呼び出し側を順に直す**ことになり、直し漏れが即バグになる。加えて、コネ索引が収集層の返り値を全て辞典項目と見なして uuid を組み立てており、収集層が広がると壊れる構造だった。

#### 設計判断

- **呼び出し側でなく収集層に源を足す**。`loadSkillEntries` と `loadDictionarySkillItems` の 2 つが技能を読む唯一の口なので、そこで合成すれば上に乗る選択肢・逆引き・カスケード・用途候補は**無改修で**両方を見る。個別対応は「どこを直したか」を人が覚えている必要があり、覚え違いが露出しない。
- **源の有無をパック単位のテーブルで持つ**（`WORLD_SKILL_SOURCES`）。技能 3 辞典にだけ述語を置き、スタイル辞典・組織辞典には置かない。これで「技能だけ」という範囲が**データとして1箇所に**書かれ、`loadSkillChoices([STYLE_PACK])` のような既存呼び出しは何もしなくても従来どおりになる。範囲を条件分岐として散らさない。
- **キャッシュは辞典分だけ**。ワールドはライブなので都度読み、合成は呼ぶたび。キャッシュに混ぜると「作ったのに出ない」が起きる。
- **uuid はエントリが持つ**。組み立てを収集層に置き、辞典は `pack.getUuid`・ワールドは `item.uuid`。呼び出し側が出所を推測して uuid を作る形をやめた（今回のバグの型そのもの）。
- **正規位置関数を合成の引数にする**。スタイル辞典だけ `styleSortPosition` を使うため、並び規則を合成側にハードコードしない。

#### 却下した案

- **呼び出し側を1つずつ広げる**（前項のやり方の継続）。9 箇所あり、うち 3 箇所は逆引き・カスケードなど「選択肢ではないが同じ源を見る」もので、直し漏れが表示だけの不整合として静かに残る。
- **`loadContactSkillIndex` を畳んで作り直す**。収集層が一般化した以上その大半は不要になるが、アクトコネは別途 HO からの自動生成を検討中で「今回触らない」指示があるため、**uuid の受け取り方だけを直す最小の変更**に留めた。

#### 検証

- 純ロジック **+6＝2069 green**（辞典＋ワールドの合成・同キーは辞典優先で uuid も辞典側・識別キー無しの除外・正規位置→名前順・正規位置関数の差し替え・空入力）。
- **隔離 Foundry**: ワールドに 一般/固有名詞/コネ/スタイル/ワークス/キー無し の 6 件を作り、3 辞典それぞれに正しく振り分かること（スタイルとワークスは `works.value` で分岐）・`medicine` の同キー衝突が辞典側で畳まれること・**uuid が出所どおりで `fromUuid` が実体を返すこと**（修正の要点）・スタイル辞典が不変であること・`loadGroupedGeneralSkillChoices`／`loadGeneralSkillNameByKey`／`buildSkillCascadeSteps`／`loadDictionarySkillItems` まで広がっていることを確認。
- eslint 0 error（警告15は既存）。CSS 未変更のため stylelint は対象外。

### シナリオコントロールの送信カードを共通の骨格へ揃える（2026-08-15）

**日付**: 2026-08-15
**レビュー対象**: `scripts/module/session-logic.mjs`・`scripts/module/tnx-scenario-panel.mjs`・`templates/chat/scene-switch-card.hbs`・`text-card.hbs`・`info-card.hbs`・`handout-card.hbs`・`css/tnx2.css`
**ステータス**: 実装済・隔離 Foundry で検証済

#### 原因

パネルから送るチャットは 5 経路あるのに、専用テンプレートを持っていたのは**ハンドアウトだけ**（2026-08-12）。残る 4 経路は素の HTML 文字列（`<h2>`/`<h3>`＋`<hr>`）か、シナリオテキストに至っては**入力された本文をそのまま投げるだけ**で、同じ操作盤から出たものが卓では別の見た目になっていた。

#### 設計判断

- **骨格は判定要求カード（`.tnx-check-request` の header/body）に揃える**。ハンドアウトカードが既にこれを踏襲しており、報酬点カード・判定要求カードとも同じ骨格。新しい意匠を作らずに揃えるのが「参考にしてください」の意図に合う。
- **ヘッダーの役割を固定した**: 左のタグ＝**種別**（ハンドアウト／SCENE n／トレーラー／シナリオテキスト／情報）、右のタイトル＝**その配布物の名前**（ハンドアウト名・シーン名・アクト名・情報項目名）。タイトルを主役として出す指定（`font-size: 1.05em`）は 4 種の共通ルールに統合した（ハンドアウト専用だったものを引き上げ）。
- **トレーラーとシナリオテキストは 1 つのテンプレートで足りる**（`text-card.hbs`）。どちらも「種別タグ＋見出し＋入力された長文」で、段の構造も条件分岐も持たない＝**形が同じなのではなく担う機能が同じ**。カードを分けると同じ修正を 2 箇所に当てることになる。
- **組み立ては純関数、HTML はテンプレート**。`buildSceneSwitchCardData` / `buildTrailerCardData` / `buildScenarioTextCardData` / `buildInfoCardData` が描画コンテキストを返し、`renderChatCard(name, data)` が描く。既存の `buildHandoutCardData` と同じ形になり、テストは HTML 文字列の一致でなく**データの形**を見るようになった（体裁を直すたびにテストが割れない）。
- **共通部品を 2 つ切り出した**: ラベル付きの行 `.cr-req-field`（ハンドアウトの `.ho-field` を引き上げ・ラベル列の幅だけカード側で決める）と、長文ブロック `.cr-req-prose`（ProseMirror の段落の上下端の余白を落とす）。ハンドアウトカードも同じクラスに載せ替えたので、同じ見た目を 2 つの名前で持つ状態を作っていない。
- **見出しは名前が空なら出さない**（`{{#if title}}`）。シナリオテキストの一覧ラベルは未入力時に「テキスト3」という**並べるための連番**になるため、それを卓に出さない（内部の都合の露出）。
- **本文はテンプレートで `{{{ }}}`**。リッチテキストを `<p>` で囲まない＝旧実装で空段落が挟まっていた原因を、囲いを持たない構造にすることで消した。

#### 却下した案

- **情報カードの「枝」ごとに `<hr>` を置く**（旧実装の見た目の踏襲）。カード内では罫線の要素を足すより**隣接ブロックの `border-top`** で切るほうが、空ブロックのときに線が浮かない。ハンドアウトの本文/PS の切り方と同じ。
- **シーンカードのシーン番号をタイトル側に混ぜる**（`SCENE 3 : 追跡` の 1 行）。旧見出しの形だが、種別タグ＋名前という 4 種共通の役割分担から外れる。番号はタグに入れた。

#### 検証

- 純ロジック **+5＝2117 green**（4 builder の描画コンテキスト・ルーラー読み替え・段の昇順と開示・本文を囲わないこと・空入力の null）。
- **隔離 Foundry（実データの複製ワールド・GM で実操作）**: パネルの送信ボタンから 4 種＋ハンドアウトを実際に送り、チャット幅 292px で描画して確認。シーンプレイヤーあり／ルーラーシーン／行だけ（本文も担当も無い＝ヘッダーのみ）の 3 形、テキストは名前あり／名前なし、情報は目標値モード／開示済みモード（段つき）。**カード内の横はみ出しは 0**（全要素の `scrollWidth` を実測）。
- **是正 1 件（自己精査で発見）**: シーン名が長いと種別タグが「SCE / NE 2」と割れていた。`.cr-req-type-tag` に `flex: 0 0 auto` と `white-space: nowrap`、`.cr-req-title` に `flex: 1 1 auto; min-width: 0` を足して、**折り返すのはタイトルだけ**にした（判定要求・報酬点カードにも同じ骨格として効く）。
- 検証: テスト 2117 green・eslint 0 error（警告15は既存）・stylelint 0 error。

#### 追補（2026-08-15 同日・ユーザー指摘の是正）

- **「目標値の送信」が本文まで送っていた**件（詳細は監査の同日エントリ）は、**2 つの分岐を共通ヘルパーに畳んだときに、出力の差が消えた**という型の不具合だった（`a5a51a2`・2026-06-07）。**共通化してよいのは「同じものを 2 回書いている」ときだけで、分岐ごとに出すものが違うならそれは仕様の差**——今回の是正では共通ヘルパーは残したまま、モードを引数（`onlyDisclosed`）で受けて**出す/出さないをデータの形で表す**ようにし、テストで両モードを別々に固定した（畳んだことで消えた差を、テストが見張る位置に置き直した）。
- **パネルのセクションの出し分け**（情報・チームを上演中だけに）はテンプレートの `{{#if actStarted}}` で行い、`_prepareContext` のデータ組み立ては触っていない。表示条件を 1 箇所（テンプレート）に置くほうが、「どの段階で何が出るか」を読むときに 2 ファイルを突き合わせずに済む。

### 枠(fieldset)が中身の長さで伸びる問題の構造的是正（2026-08-15）

**日付**: 2026-08-15
**レビュー対象**: `css/tnx2.css`（`.tnx-fieldset` / `.tnx-wrap-text` / シナリオコントロールパネルのラベル群）
**ステータス**: 実装済・隔離 Foundry で全画面を測って検証済

#### 原因

シナリオコントロールパネルの「情報」セクションで、技能を 1 つ足すたびに枠が横へ広がり、親からはみ出して見切れていた（ユーザー報告・スクリーンショットつき）。原因は **2 つの自動最小幅**が重なっていたこと。

1. `fieldset` は UA スタイルで `min-inline-size: min-content` を持つ（`min-width: 0` を明示しない限り、中身の最小内容幅より狭くならない）。
2. flex/grid の子は自動最小サイズ（`min-width: auto` = min-content）を持つ。

そこへ `white-space: nowrap` のラベル（技能名を `/` で連結した動的文字列）が入ると、その 1 行の全長が枠の最小幅になる。**枠の幅が中身の文字数で決まる**状態だった。実測: パネルの内側 370px に対し「情報」枠 415px・「シーン」枠 417px（`window-content` が横スクロール）。

#### 設計判断

- **枠の幅は置かれた場所の幅で決まり、中身では決まらない**（2026-08-15 ユーザー指示「この種のレジェンドが自動で幅を調節することは絶対にありません」）。`.tnx-fieldset` に `min-width: 0` を置いて 2 つの自動最小幅を同時に解除した。**共有クラスに 1 行**なので、システム内の全 fieldset（用途シート 18・アウトフィット 13・スタイル技能 10・パネル 5 ほか計 60 以上）に同時に効く。個別の画面で対症療法をしない。
- **入りきらないときは枠でなく中身を折り返す**。パネルの動的ラベル（アクト名・シーン名・送信ラベル・情報の内容ラベル・チップ）から `white-space: nowrap` と `text-overflow: ellipsis` を外し、`overflow-wrap: anywhere` にした。**省略記号での切り詰めも「見切れ」**であり、ユーザー指示（「はみ出したり見切れたりすることは絶対に避けて」）に反する。
- 共通クラス `.tnx-wrap-text`（`flex: 1 1 auto; min-width: 0; overflow-wrap: anywhere`）を用意し、今後の動的ラベルはこれを付けるだけで済むようにした。
- **静的ラベルの `max-content` 列は残す**（RL プリセットの共有ラベル列など）。あれは**固定文字列**の最長で決まるので実行時に伸び縮みせず、「幅が一定」の要件を満たす。ユーザー承認済みの配置（2026-08-09）でもある。
- fieldset の UA マージン（`margin-inline: 2px`）は残した。全 fieldset に一定量なので幅は一定であり、外すと全シートが 2px ずれるだけで得がない。

#### 検証（測って確かめた）

机上で「直したはず」と言わないために、**隔離 Foundry で幾何を実測する監査スクリプト**を書いて全画面に当てた。判定は 3 つ:
`app-scroll`（ウィンドウの中身が横スクロールする）／`escape`（要素の右端がウィンドウのコンテンツ枠より外）／`box-wide` `box-narrow`（fieldset の幅が親のコンテンツ幅と一致しない）。

- **仕込み**: 全アイテム型 17・全アクター型 4 を「とても長い名前のテスト用アイテム《スペシャル・エディション》」で作成し、キャストには同じ長さの技能・武器・神業を持たせた。情報項目にはユーザー報告と同じ「〈社会：ストリート〉/〈社会：企業〉/〈社会：真教〉＞5」を入れた。
- **対象**: アイテムシート 17 型／アクターシート 4 型 × タブ 6／用途シート（タブ 3）／アクトシート（タブ 4）／シナリオコントロールパネル／サブシーンパネル／FS 判定パネル／経験点配布／RL 判定要求／カードセットアップ／レコードシート／判定ダイアログ／シーン開始ダイアログ／状態異常編集／戦闘トラッカー／HUD。
- **結果**: 是正後は `app-scroll` 0・`escape` 0・`box-wide` 0。`box-narrow` は全画面一律 4px（fieldset の UA マージン 2px×2）で、Foundry コアの `UserConfig` も同じ値。つまり**幅は中身に依らず一定**。
- 是正前後の実測（パネル）: 情報 415px / シーン 417px → **どちらも 366px**（他の枠と同値）・`window-content` の横スクロール解消。
- テスト 2120 green・eslint 0 error（警告15は既存）・stylelint 0 error。

#### 申し送り

- 監査スクリプト（`sweep2/3/4/5` 相当）は scratchpad にあり恒久化していない。同種の是正を行うときは**同じ 3 判定で全画面を測り直す**のが再現手順（この記録が手順書を兼ねる）。

### 指定技能の表示規則を 1 つに集約（2026-08-15）

**日付**: 2026-08-15
**レビュー対象**: `scripts/module/skill-dictionary.mjs`（`formatDesignatedSkills`）・`session-logic.mjs`・`tnx-rl-request-app.mjs`・`focus-system-request.mjs`
**ステータス**: 実装済・隔離 Foundry で検証済

#### 原因

「同じ小分類の固有名詞技能は一つに束ねる」表記（〈社会：ストリート、警察〉）は 2026-08-09 にユーザー指示で作ったのに、**使っていたのは登場判定の 1 箇所だけ**だった（`formatGroupedSkillNames`）。情報項目・判定要求カード・FS 判定は、それぞれ技能ごとに `formatSkillName` を掛けて「・」や " / " で連ねる**別実装**を持っていた。同じ「指定技能」が画面ごとに違う書き方で出ていた。

#### 設計判断

- **表示規則は 1 つ**（`formatDesignatedSkills(keys, nameByKey) → string`）。束ね方も連結も返り値の形（配列でなく文字列）もここで決め、呼び出し側は文字列を置くだけにした。配列を返して呼び出し側が join すると、**join の仕方が呼び出し側の数だけ増える**（実際そうなっていた）。
- 置き場は `skill-dictionary.mjs`（小分類テーブル `ONOMASTIC_TYPES` と `idKeyPrefix` があるモジュール）。`session-logic` からも読むが、辞典モジュールは import 時に Foundry へ触らないので純ロジックのテストから使える。
- **操作単位のチップは対象外**と明記した（1 チップ＝1 技能で ✕ を持つ＝束ねると外せない）。規則の適用範囲を JSDoc に書き、次に触る人が同じ判断を再現できるようにした。
- 逆引き不能キーの扱いは呼び出し側で分ける: カードの見出しは空にできないので「（参照切れ）」、情報項目は旧い自由記述 `name` へフォールバック。**規則本体は「解決できないキーは落とす」だけを持つ**（生キーを出さないという一点）。

#### 情報項目の表示モデル

- 目標値に上下関係が無い（ユーザー裁定）ので、**入口の目標値と段の目標値を区別しない** `infoValueList` を作り、`infoSkillGroups(content)` が「技能行ごとの表示単位（技能ラベル＋目標値の並び）」を返す。パネルもチャットカードもこの 1 つの関数から描く——**同じものを 2 度組み立てない**。
- 保存形式（`content.text` ＋ `content.tiers[]`）は無改修。表示だけを等価にした。保存形式まで「目標値の集合」に寄せるには移行が要るので、必要になったら設計フェーズで扱う（監査に申し送り）。

#### 検証

- 純ロジック **+2＝2122 green**（目標値の横並び・昇順・枝ごとの技能行・開示済みの 2 列・技能を持たない枝・束ね表記・逆引き不能の扱い・`infoSkillGroups` の 4 ケース）。
- **隔離 Foundry**: ユーザー提示のデータ（〈社会：ストリート、警察〉5/10/12 と〈電脳〉21）で、パネル・目標値の送信・開示済みの送信・アクトシートの編集欄を実描画して確認。幅の監査（`app-scroll`/`escape`/`box-wide`）も再走して 0。
- テスト 2122 green・eslint 0 error（警告15は既存）・stylelint 0 error。

### シナリオコントロールパネルの固定ヘッダ＋頻度別タブ（フェーズ14-8・2026-08-15）

**日付**: 2026-08-15
**レビュー対象**: `templates/app/scenario-panel.hbs`・`scripts/module/tnx-scenario-panel.mjs`・`scripts/tnx.mjs`（設定登録）・`css/tnx2.css`
**ステータス**: 実装済・隔離 Foundry で検証済

#### 背景と情報設計

- RL がセッション中ずっと開いておくパネルなのに、全セクションが縦積みでスクロールが頻発していた（ユーザー指摘）。ユーザー提示の使用頻度＝シーン送り・情報の開示（毎シーン）／キャスト・チーム（シーン切替時）／アクト読込（セッション1回）を、そのまま画面の階層にした: **固定ヘッダ（常時）→ 進行タブ（毎シーン・既定）→ キャストタブ（切替時）→ アクトタブ（1回）**。
- 実装前に構成案3案（タブ分割・折りたたみ・常設操作帯）をテキスト提示し、タブ分割案の承認を得てから着手（制約: シーン情報の表示は一番上から動かさない）。

#### 設計判断

- **スクロールの器を1つに**: `window-content` を overflow hidden の flex 列にし、スクロールは本文 `.scp-body` だけが持つ。ヘッダ（`.scp-status`）とタブ nav は自然に固定される——「固定」を position: sticky 等で作らない（既存の max-height 上限とリサイズがそのまま効く）。共有セレクタだったサブシーンパネルの規則は分離し従来のまま。
- **タブは既存機構の流用**: コア V13 の changeTab は nav に `.tabs` クラスを要求しコア CSS と競合するため、アクト（シナリオ）シートで確立した独自 changeTab を踏襲。意匠も sheet-tabs の踏襲でタブの見た目を新造しない。レンダー後の active 復元も既存シートと同型（`_onRender` で force 適用・nav が無い PL/開始前はスキップ）。
- **状態の置き場＝クライアント設定**: 最後に開いたタブ・巡回/シーン一覧の開閉は HUD 折りたたみと同じ器（scope: client・config: false）。パネルは sessionState の onChange で頻繁に re-render されるため、DOM トグルでなく**設定→ _prepareContext → テンプレート**の一方向で描く（再描画で状態が飛ばない）。
- **キャスト操作の分離**: 登場中チップ（表示＝シーンの状態）はヘッダ残置、操作（登場判定・手動登場行・チーム）は castSection インライン部品に切り出し、GM はキャストタブ・PL はヘッダ直下に同じ部品を置く——**同じ内容を2度書かない**。開始前の構成（アクト→配布・送信）は無改修。
- **折りたたみの範囲**: 畳んだのは巡回とシーン一覧（毎シーンの操作でない参照系）だけ。舞台裏・「次のシーンへ」等の進行ボタンは常時表示のまま＝提示した構成案どおり。

#### 検証

- テスト 2122 green（変更はテンプレート・UI 配線・設定登録のみで新規純ロジックなし）・eslint 0 error（警告15は既存）・stylelint 0 error。
- **隔離 Foundry**（develop 複製・GM/PL 2クライアント・Playwright）: 3タブの表示と既定タブ／タブ切替と close→reopen での復元（設定値も確認）／シーン一覧・巡回（巡回行を注入して実データ描画）の開閉と設定永続／高さ480 で本文のみスクロール・ヘッダとタブの座標不変・横はみ出し 0／PL: タブなし・ヘッダ直下に登場判定＋チーム／開始前: タブなし・アクト＋配布・送信。スクリーンショット8枚を目視で精査。

### 判定の強制失敗機構（forcedFailure）（フェーズ14-8・2026-08-15）

**日付**: 2026-08-15
**レビュー対象**: `scripts/module/tnx-check-flow.mjs`・`appearance-logic.mjs`・`appearance-check.mjs`・`templates/chat/check-result.hbs`
**ステータス**: 実装済・隔離 Foundry で検証済

#### 設計判断

- **ブロック分岐の削除でなく、判定フローの汎用文脈として実装**: `ctx.forcedFailure`（表示ラベル文字列）。判定できない状況は今後も出うる（ルール上「ほぼ存在しない」が、シーン制約系は登場判定以外にも波及しうる）ため、登場判定専用のフラグにせず、フローの結果確定点 1 箇所で `success=false` に固定する形にした。達成値・内訳・カードプレイ・報酬点の各機構は無改修＝**成否だけが落ちる**。
- **理由コードとラベルの分離**: 純ロジック（`appearanceCheckParams`）は理由コード（`"none"|"sanctuary"`）を返し、UI 文字列（「登場：不可」等）は起動側（appearance-check）が対応表で与える。純ロジックに表示文字列を持ち込まない。
- **覆らないことの保証を 3 経路で**: 初回計算（結果確定点）・再判定（スナップショット `_buildRecheckContext` → `startRecheck` の引き継ぎ）・事後修正（`_applyCheckModify` の成否再計算ガード）。手動修正で達成値自体は動く（卓の裁定ツールとして残す）が成否は失敗のまま・「修正後の成否」行も出さない。
- **成否バナーの表示規則を拡張**: 従来「目標値があるときだけ成否を出す」だったが、強制失敗は目標値なし（登場：不可は TN を持たない）でも必ず「失敗＋理由」を出す。長い注記による「失敗」の文字割れは `.cr-result` の flex-wrap で注記ごと次行へ逃がす（全結果カード共通・注記が短ければ従来と同じ1行）。

#### 検証

- テスト 2122 green（appearanceCheckParams の返り値変更を既存テスト書き換えで固定・blocked は全消滅）・eslint/stylelint 新規指摘なし。
- **隔離 Foundry E2E**（GM/PL 2クライアント・実カードプレイ）: ①登場：不可＝フロー完走・スート一致 達成値10 でも「失敗｜登場：不可」・登場フラグなし ②GM 手動修正 +99 → 達成値109・成否は失敗のまま ③サンクチュアリ＋危険値装備＝「失敗｜サンクチュアリ・危険値装備 目標値 12」。結果カード2種をスクリーンショットで目視精査（文字割れ是正後を再確認）。

### 登場判定の専用チャットカード（フェーズ14-8・2026-08-16）

**日付**: 2026-08-16
**レビュー対象**: `scripts/module/appearance-logic.mjs`（`appearanceCardInfo`）・`tnx-check-flow.mjs`（`_renderResultContent`）・`templates/chat/check-result.hbs`
**ステータス**: 実装済・隔離 Foundry で検証済

#### 設計判断

- **用途（usage）を新設せず、判定文脈で描画を分岐**: 攻撃カード（ctx.attack）・移動カード（ctx.movement）と同じ既存パターン。登場判定は「任意の技能＋注入文脈」で起動される判定であり、アイテム側にデータを持たせる理由がない（指定技能がシーンごとに変わるため持たせられない＝ユーザー指示）。専用テンプレートも新設せず、`check-result.hbs` に `{{#if appearance}}` の行を足す形＝**基底に情報を足す形式**（2026-07-19 基底化の原則）を守る。
- **表示情報は純関数 `appearanceCardInfo`** が組み立てる（エリアは設定時のみ `hasInfo`・帰結は `success===true` のみ）。フローは typeLabel の差し替えとコンテキスト1個の追加だけで、判定計算・投稿経路は無改修。
- **`areaLabel` は継続文脈（ctx.appearance）に載せる**: 継続キーは再判定スナップショットに常時保存されるため、再判定の置き換え描画でも専用カードの見た目が自動的に保たれる（新しい保存経路を作らない）。
- 帰結行「シーンに登場した」は投稿時点の静的描画。事後修正で成否が変わるケースは既存カードと同じ規約（「修正後の成否」行が生きた真実・登場の自動適用は継続再実行 `rerunOnSuccessOnly` が担う）に乗る。

#### 検証

- 純ロジック **+4＝2126 green**（null 文脈・エリア未設定/設定・ゴースト・success true/null/欠落）・eslint/stylelint 新規指摘なし。
- **隔離 Foundry E2E**（PL の実カードプレイ・成功するまで再試行）: タグ「登場判定」・エリア行の出し分け（未設定＝行なし／サンクチュアリ＝表示）・ゴースト宣言行・成功時の帰結行と登場/isGhost フラグ・強制失敗との併存（理由注記＋目標値）。カード3種をスクリーンショットで目視精査。
