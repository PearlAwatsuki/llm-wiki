# Handlebars テンプレート

## ディレクトリ構成

```
templates/
  actor/
    cast-sheet.hbs          # cast シート本体
    player-sheet.hbs        # player シート
  item/
    style-sheet.hbs
    miracle-sheet.hbs
    general-skill-sheet.hbs
    style-skill-sheet.hbs
    organization-sheet.hbs
  journal/
    scenario-sheet.hbs      # アクトシート
  hud/
    hud.hbs                 # 常駐 HUD
  chat/
    scene-card.hbs          # チャットカード表示
  user/
    record-sheet.hbs        # レコードシート (フェーズ2)
  dialog/                   # 8 ファイル
  parts/                    # 共通パーシャル
```

## 共通パーシャル (`templates/parts/`)

| ファイル | 用途 | 使用シート |
|---|---|---|
| active-effects-list.hbs | ActiveEffect CRUD | miracle, generalSkill, styleSkill, organization, cast |
| bad-status-list.hbs | バッドステータス | cast |
| history-list.hbs | EXP 履歴 | cast, player (レコードシートは独自実装) |
| usage-list.hbs | 用途リスト | miracle, generalSkill, styleSkill |
| prosemirror-editor.hbs | リッチテキスト | 各所 |
| scenario-setting-wizard.hbs | セットアップ | ウィザード |

## data-action パターン

シートは `[data-action]` 属性 + リスナーディスパッチが主流。

### 修正済み不一致

`active-effects-list.hbs` の action 名を EffectsSheetMixin の switch 文と一致 (KI-001 解消):

| テンプレート | ハンドラ |
|---|---|
| create | create |
| toggle | toggle |
| edit | edit |
| delete | delete |

### 未解決

`usage-list.hbs` の `data-action="edit"` — ハンドラ不在 (KI-002)

## フォーム name 属性

DataModel 移行後、フィールドパスは `system.xxx` 形式 (type 名プレフィックスなし)。

例: styleSkill の comboSkill → `system.comboSkill.N.value` (NOT `system.styleSkill.comboSkill...`)

## テンプレート読み込み

`loadTemplates()` でプリロード (v13 deprecation 警告のみ、動作は v15 まで保証)。

## ProseMirror / HTMLField

description フィールドは HTMLField。編集 UI は prosemirror-editor パーシャル等を使用。

`system.json` の `htmlFields: ["description"]` でサニタイズ宣言。
