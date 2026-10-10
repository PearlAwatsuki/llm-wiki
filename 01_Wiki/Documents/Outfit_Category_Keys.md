# アウトフィット分類キー一覧

2026-10-09 時点の `scripts/data/item/outfit-categories.mjs`（`OUTFIT_CATEGORIES`）に対応。大分類8種・小分類40種。保存する値は表示名ではなく、以下のキー。大文字・小文字もそのまま使用する。

## 保存先と識別キーとの違い

| 内容 | 保存先 |
|---|---|
| 主分類の大分類／小分類 | `system.majorCategory` / `system.minorCategory` |
| 主分類の「その他」の本文 | `system.majorCategoryOther` / `system.minorCategoryOther` |
| 副分類 | `system.additionalCategories[]` の `{rowId, major, minor, majorOther, minorOther}` |

`Item.type` は DataModel の種類、分類キーは装備の分類、`system.identificationKey` は個々のアイテムの識別キー。それぞれ別の値であり、分類キーから識別キーを生成しない。

主分類と副分類を合わせた分類集合を、ホスト照合・修理・改造・AEの分類照合などで使用する。小分類キーは全体で一意で、大分類キーとも重複しない。用途の `repairableCategories` では大分類・小分類キーを同じ配列に保存する。

## 大分類・小分類・Item type

`types` はコードに定義された対応型。分類を変更しても `Item.type` は変わらず、別型のフィールドは追加されない。

| 大分類 | 大分類キー | 小分類 | 小分類キー | 対応する Item.type |
|---|---|---|---|---|
| 武器 | `weapon` | 白兵武器 | `melee` | `weapon` |
| 武器 | `weapon` | 射撃武器 | `ranged` | `weapon` |
| 武器 | `weapon` | 搭載兵器 | `mounted` | `weapon` |
| 武器 | `weapon` | 武器オプション | `weaponOption` | `weapon` |
| 武器 | `weapon` | 特殊弾 | `specialAmmo` | `weapon` |
| 防具 | `armor` | ボディアーマー | `bodyArmor` | `armor` |
| 防具 | `armor` | アーマーギア | `armorGear` | `armor` |
| 防具 | `armor` | ファッション | `fashion` | `general` |
| サイバーウェア | `cyberware` | IANUS | `ianus` | `ianus` |
| サイバーウェア | `cyberware` | IANUSオプション | `ianusOption` | `general` |
| サイバーウェア | `cyberware` | ニューラルウェア | `neuralware` | `general` |
| サイバーウェア | `cyberware` | アーティフィシャルボディ | `artificialBody` | `general` |
| サイバーウェア | `cyberware` | オーガニックウェア | `organicware` | `general` |
| サイバーウェア | `cyberware` | サイコアプリ | `psychoApp` | `general` |
| サイバーウェア | `cyberware` | コスメティック | `cosmetic` | `general` |
| サイバーウェア | `cyberware` | 全身義体 | `fullCyborg` | `cyborg` |
| トロン | `tron` | ポケットロン | `pocketron` | `tron` |
| トロン | `tron` | PTオプション | `ptOption` | `tron` |
| トロン | `tron` | タップ | `tap` | `tap` |
| トロン | `tron` | ソフトウェア | `software` | `tap` |
| トロン | `tron` | ハードウェア | `hardware` | `tap` |
| ヴィークル | `vehicle` | 地上車両 | `groundVehicle` | `vehicle` |
| ヴィークル | `vehicle` | 航空機 | `aircraft` | `vehicle` |
| ヴィークル | `vehicle` | 船舶 | `ship` | `vehicle` |
| ヴィークル | `vehicle` | ウォーカー | `walker` | `vehicle` |
| ヴィークル | `vehicle` | ドローン | `drone` | `vehicle` |
| ヴィークル | `vehicle` | ヴィークルオプション | `vehicleOption` | `vehicle` |
| 住宅 | `housing` | 住宅施設 | `residence` | `residence` |
| 住宅 | `housing` | 住宅オプション | `housingOption` | `general` |
| 住宅 | `housing` | 住宅アクセサリ | `housingAccessory` | `general` |
| アイテム | `item` | ツール | `tool` | `general` |
| アイテム | `item` | マジックアイテム | `magicItem` | `general` |
| アイテム | `item` | 妖的外姿 | `mysticalAppearance` | `general` |
| アイテム | `item` | 生体装備 | `biotech` | `general` / `weapon` / `armor` |
| アイテム | `item` | ドラッグ | `drug` | `general` |
| アイテム | `item` | フーズ | `food` | `general` |
| サービス | `service` | ソーシャル | `social` | `general` |
| サービス | `service` | バックグラウンド | `background` | `general` |
| サービス | `service` | エキストラ | `extra` | `general` |
| サービス | `service` | コンバイナー | `combiner` | `combiner` |

## 特殊な選択肢と編集

| 保存値 | 表示 | 補足 |
|---|---|---|
| `""` | 未選択 | 分類指定なし |
| `reference` | 解説参照 | 大分類・小分類の両方で使用可能 |
| `other` | その他 | 対応する `…Other` / `majorOther` / `minorOther` に自由記述 |

選択肢の順番は未選択、通常分類、解説参照、その他。型から分類が決まるアウトフィットも分類欄を無効化せず編集できる。`reference` と `other` には通常の親大分類はない。

## AEで指定する場合

新規の分類変更は `system.outfitCategories`（分類の行操作）を使用する。主分類の指定IDは `primary`、副分類は各行の `rowId`。配列番号で指定しない。

例えば主分類を射撃武器へ変更する値は次のとおり。通常の小分類から大分類 `weapon` を補完する。

```json
{"operation":"update","selector":{"mode":"id","id":"primary"},"changes":{"minor":"ranged"}}
```

特殊値を小分類に指定した場合は既存の大分類を保持する。大分類と通常の小分類が矛盾する操作は適用しない。詳細は [Active_Effects.md](../Game_Mechanics/Active_Effects.md) を参照。

## 関連ページ

- [Item 型と保存フィールド](Item_Types.md)
- [装備部位・表示・占有](../Game_Rules/Outfits.md)
