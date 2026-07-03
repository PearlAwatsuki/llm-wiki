# Handlebars テンプレート

## ディレクトリ構成

```
templates/
  actor/
    cast-sheet.hbs          # cast シートのシェル（タブ nav＋partial 差し込み。フェーズ11-2 で部品化）
    guest-sheet.hbs         # guest シートのシェル（履歴タブなし。フェーズ11-3）
    troop-sheet.hbs         # troop シートのシェル（神業/報酬点/部位なし・heads あり。フェーズ11-4）
    parts/                  # アクターシート共通部品（フェーズ11-2。cast/guest 等で共有・features でゲート）
                            #   sidebar / header / tab-abilities / tab-combat / tab-outfits /
                            #   tab-status / tab-history(cast 専用) / tab-profile
  item/
    style-sheet.hbs / miracle-sheet.hbs / general-skill-sheet.hbs /
    style-skill-sheet.hbs / organization-sheet.hbs /
    outfit-sheet.hbs / housing-area-sheet.hbs / life-path-sheet.hbs   # フェーズ6
  journal/
    scenario-sheet.hbs      # アクトシート
  hud/
    hud.hbs                 # 常駐 HUD
  chat/
    scene-card.hbs / check-request.hbs / check-result.hbs       # フェーズ8
  user/
    record-sheet.hbs        # レコードシート (フェーズ2)
  app/                      # usage-sheet.hbs / usage-sheet-combo.hbs / rl-request-app.hbs
  apps/                     # damage-chart-text.hbs
  dialog/                   # 各種ダイアログ
  parts/                    # 共通パーシャル
```

## 共通パーシャル (`templates/parts/`)

| ファイル | 用途 | 使用シート |
|---|---|---|
| active-effects-list.hbs | ActiveEffect CRUD | miracle, generalSkill, styleSkill, organization, cast |
| bad-status-list.hbs | バッドステータス | cast |
| history-list.hbs | EXP 履歴 | cast (レコードシートは独自実装) |
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

### 解消済み

`usage-list.hbs` の用途編集ハンドラは実装済み（`_onUsageEdit`・KI-002 解消・フェーズ8-6）

## フォーム name 属性

DataModel 移行後、フィールドパスは `system.xxx` 形式 (type 名プレフィックスなし)。

例: styleSkill の comboSkill → `system.comboSkill.N.value` (NOT `system.styleSkill.comboSkill...`)

## テンプレート読み込み

`loadTemplates()` でプリロード (v13 deprecation 警告のみ、動作は v15 まで保証)。

## ProseMirror / HTMLField

description フィールドは HTMLField。編集 UI は prosemirror-editor パーシャル等を使用。

`system.json` の `htmlFields: ["description"]` でサニタイズ宣言。
