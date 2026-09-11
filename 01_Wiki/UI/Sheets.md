# シート一覧

## Application API 状況

| API | 使用状況 |
|---|---|
| ApplicationV1 | フェーズ3 で**全廃済み**。コード内に一切残存しない |
| ApplicationV2 | 全シート・HUD・ダイアログが V2 に移行完了（フェーズ3）|

## 実装済みシート

| クラス | 継承 | 対象 type | ファイル |
|---|---|---|---|
| TnxCharacterSheetBase | ActorSheetV2 | 基底（直接登録なし・フェーズ11-2 新設） | `scripts/actor/tnx-character-sheet-base.mjs` |
| TokyoNovaCastSheet | TnxCharacterSheetBase | cast | `scripts/actor/tnx-cast-sheet.mjs`（EXP 系＋セッション履歴のみ） |
| TokyoNovaGuestSheet | TnxCharacterSheetBase | guest | `scripts/actor/tnx-guest-sheet.mjs`（宣言のみの薄い派生・フェーズ11-3） |
| TokyoNovaTroopSheet | TnxCharacterSheetBase | troop | `scripts/actor/tnx-troop-sheet.mjs`（宣言のみの薄い派生・フェーズ11-4） |
| TokyoNovaExtraSheet | TnxCharacterSheetBase | extra | `scripts/actor/tnx-extra-sheet.mjs`（名前＋説明のみ・全ドロップ拒否・フェーズ11-5） |
| TokyoNovaItemSheet | ItemSheetV2 | 基底（直接登録なし） | `scripts/item/tnx-item-sheet.mjs` |
| TokyoNovaStyleSheet | TokyoNovaItemSheet | style | `scripts/item/tnx-style-sheet.mjs` |
| TokyoNovaMiracleSheet | TokyoNovaItemSheet | miracle | `scripts/item/tnx-miracle-sheet.mjs` |
| TokyoNovaGeneralSkillSheet | TokyoNovaItemSheet | generalSkill | `scripts/item/tnx-general-skill-sheet.mjs` |
| TokyoNovaStyleSkillSheet | TokyoNovaItemSheet | styleSkill | `scripts/item/tnx-style-skill-sheet.mjs` |
| TokyoNovaOrganizationSheet | TokyoNovaItemSheet | organization | `scripts/item/tnx-organization-sheet.mjs` |
| TokyoNovaOutfitSheet | TokyoNovaItemSheet | weapon/armor/ianus/cyborg/tron/tap/vehicle/residence/combiner/general | `scripts/item/tnx-outfit-sheet.mjs` |
| TokyoNovaHousingAreaSheet | TokyoNovaItemSheet | housingArea | `scripts/item/tnx-housing-area-sheet.mjs` |
| TokyoNovaLifePathSheet | TokyoNovaItemSheet | lifePath | `scripts/item/tnx-life-path-sheet.mjs` |
| TnxScenarioSheet | DocumentSheetV2 | JournalEntry（アクト） | `scripts/journal/tnx-scenario-sheet.mjs` |
| TnxRecordSheet | ApplicationV2 | User（レコード） | `scripts/module/tnx-record-sheet.mjs` |

## コアデフォルト（カスタム未実装）

**なし** — 全 Actor / Item type がカスタムシートを持つ（フェーズ11-5 で完了）

## 共通 UI パターン

- **view/edit モード**: `_isEditMode` + `_render()` オーバーライド
- **ヘッダーの配置規約**（2026-08-08 ユーザー明示）: シートヘッダーのボタン（トグル操作・UUID をコピー・シートを閉じる、およびアウトフィットのフラグトグル）は**すべて右寄せ**。**左寄せは編集/閲覧切り替えトグルのみ**。Foundry 既定では `flex:1` の `.window-title` が余白を吸ってコントロールを右へ押し出すが、本システムはシートのタイトルを非表示にする（本文に名前を出す方針）ため、余白を吸う要素が無いとコントロールが左端へ寄る。編集モードトグルの `margin-right:auto` に依存すると、トグルが出ない場合（権限なし・閲覧のみ）に左寄せへ崩れるので、`.tokyo-nova.sheet .window-header { justify-content: flex-end; }` でヘッダー側を右寄せ既定にする。タイトルを再表示するシート（アクト・FS 判定）は `flex:1` のタイトルが余白を吸うため影響を受けない
- **UUID コピー**: 全シートヘッダー
- **テキスト自動スケール**: CastSheet（`_applyTextSqueezing()`）
- **フラグトグル**: アウトフィットシートのヘッダーアイコン（準備済み・携帯中・プレアクト購入）
- **ContextMenu**: 全シートの閲覧/編集/削除（type 依存で複製・リンク解除等を追加）
- **コンボボックス（自由入力＋候補）**: `scripts/module/combobox.mjs` の `enhanceComboboxes(root)`（2026-07-23・KI-036）。標準の `<input list="x">` ＋ `<datalist id="x">` を、スタイル可能な独自コンボボックスへ「昇格」させる横断機構（number-input-spinner と同じ明示配線方式）。ネイティブ `<datalist>` はドロップダウンの見た目・スクロール・▼位置がブラウザ任せで制御不能なため置き換える。元 `<input>` はそのまま値の器として使い（name/value/フォーム送信を壊さない）、候補は datalist から昇格時に読んで `list` 属性を外す（ネイティブUI無効化・冪等マーカ `data-tnx-combo`）。ドロップダウンは `document.body` 直下に `position:fixed`（親 overflow でクリップされない・`max-height`＋`overflow-y:auto` で必ずスクロール可）、視覚はアンカー入力の computed style を継承しテーマ非依存。自由入力は常に許容（select 化しない）・↑↓/Enter/Esc 対応。CSS は `.tnx-combobox*`（`.tokyo-nova` 非依存の自立スコープ）。配線先=AE設定シートの変更キー入力＋ partAdd 値入力（`tnx.mjs` の `renderActiveEffectConfig`）、部位スロットプリセットの占有部位入力（`part-slot-preset-app.mjs`）。今後の同型入力は `<input list>`＋`<datalist>` を書いて `enhanceComboboxes` を1行呼ぶだけで対応。

## キャストシートタブ構成（2026-07-03 実態に整合化）

サイドバー: ポートレート・基本情報・**報酬点**（基準＋現在・再計算ボタン）・**EXP**（cast 専用）。

| タブ | 内容 |
|---|---|
| 能力 | 能力値ブレークダウン・スタイル（役割トグル・神業）・一般/スタイル技能 |
| 戦闘 | **フェーズ10 で実装**。CS ボックス（表示は単一「CS」・カット進行中はカレント参照＋グロー・編集モードで決定の連鎖＝ベース/修正/CS/カレント）＋ **AR ボックス**（フェーズ11 追補・付与型＝カット進行中のみ現在AR・それ以外「なし」）→ 武器 2 行（攻撃/パリー・weaponRefs による使用武器選択・空欄は生身・「生身（名前）」表記）→ 防御力（全義体込み種別合計 1 行）→ タイミングごとの使用技能・アウトフィット（**標準 7 種常設＋リアクションは非空のみ**・差し込みタイミングは非空のみ・その他＋解説参照は末尾）。一覧・効果系（負傷/BS）は置かない（他タブの領分）。<br>**2026-07-20 是正（ユーザー指摘）**: ①**リアクションを「該当する用途があるときのみ表示」へ**——常設 8 種のうちリアクションだけ表示条件を差し込みタイミングと同じにする（当初「チャットカード起動に一本化したので戦闘タブから除去」としたが同日ユーザーが撤回・**表示はする**が条件付き）。**位置はアクションの並び（メジャー→リアクション→オート）を保つ**——表示条件が同じでも末尾送りにするとルール上の行動順から外れて読めなくなるため（`fixedBuckets` の `onlyWhenFilled`）。②**差し込みタイミングの取りこぼしを修正**——「ダメージ算出の直前」等は自由記述ではなく `getSkillOptions().timing` の**名前つき選択肢**（`preCalcDamge` 等）だが、`classify()` が `action`/`process`/`other` の 3 値しか見ておらず、該当する用途があっても**丸ごと落ちていた**。名前つき選択肢も差し込み群として扱い、群の並びは選択肢の定義順→自由記述の順。**群はラベルで束ねる**（自由記述に同名を手入力した用途と選択肢を選んだ用途は同じ群に入る）。③用途の**「戦闘タブに表示しない」**（`actions[].hideInCombatTab`）が立つ行は全群から除外する。詳細は [Combat_Flow.md](../Game_Rules/Combat_Flow.md) |
| アウトフィット | 大分類別グループ（武器/防具/サイバーウェア/トロン/ヴィークル/住居/その他）・部位占有パネル |
| 状態 | ActiveEffect 一覧・バッドステータス一覧（`templates/parts/` の partial） |
| セッション履歴 | 履歴一覧（TnxHistoryMixin）・レコードシート同期トグル（cast 専用） |
| 詳細 | ライフパス 3 スロット・キャラクター説明エディタ |

> 旧表（能力値/スタイル/一般技能/…/エフェクトの 7 分割）はフェーズ5 以前の構成で実態と乖離していたため更新（2026-07-03）。

## アウトフィットシートタブ構成

| タブ | 内容 |
|---|---|
| 説明 | 概要サマリ（型別）+ ProseMirror エディタ |
| 設定 | 全フィールド編集（カテゴリ・各数値・フラグ群） |
| 用途 | usage テンプレート一覧（usage を持つ 10 種のみ） |
| エフェクト | ActiveEffect 管理 |

housingArea は専用シート（説明 + 修正値 7 種）。lifePath は説明 + 設定（種別・習得技能・識別キー）の 2 タブ。

## アクトシート（TnxScenarioSheet）

- `JournalEntry` はサブタイプを持たない → 作成ダイアログでの型選択不可
- 導線: ジャーナルディレクトリの「アクトシートを作成」ボタン（フェーズ1-1）
- 構造: flag + 独自タブ UI（ハンドアウト・情報項目・シーン・テキスト）
- **JournalEntryPage カスタムサブタイプ化は不採用**（通常ジャーナルに混入するため）

## 参照

- テンプレート: [Templates.md](Templates.md)
- v13 移行: [Foundry_v13.md](../Migration/Foundry_v13.md)

### アクター画像（2026-09-11）

共通サイドバーの単一画像枠に立ち絵（actor.img）またはコマ画像を表示する。編集モードのみ画像上部のスイッチを表示し、クリックで切り替える。編集モードで表示画像をクリックすると立ち絵・コマのどちらも画像選択を直接開き、対象の画像フィールドだけを保存する。閲覧モードは選択中の画像だけを表示する。表示選択はシートインスタンス内で保持し、アクターデータには保存しない。通常アクターのコマ設定はプロトタイプ（以後配置するコマ）、非リンクトークンの合成アクターではそのトークンが対象。

### アクトシートの追加目標値（2026-09-11）

情報項目で「目標値を追加」を押すと、本文を持つ追加情報を作成する。各追加情報の技能名の選択欄で1段目の技能行を選び、その条件の目標値を入力する。「目標値だけを追加」は同じ本文へ別の条件を足す。各条件の×はその条件だけを削除し、追加情報の×は本文を含むブロックを削除する。旧データは「すべての使用技能（共通）」として表示される。

2026-09-11 表示訂正: 技能選択肢は技能名だけを表示する。先頭の連番・末尾の入口目標値・「元の技能＋目標値」ラベルは表示しない。

### 2026-09-11 アクトシートのエリア指定と開始時入力

アクトシート本体でエリアを選ぶと、そのエリアの登場判定目標値を自動的に使う。「登場判定」選択欄と数値欄は非表示となり、「エリア準拠」は選択肢から削除する。エリア未設定では未設定・数値指定・不可を選べ、数値指定時の未入力数値は10から開始する。以前の「エリア＋数値指定/不可」の併用よりエリア指定を優先する。

開始ダイアログは別の処理。シーンプレイヤー「任意」なら開始時に選択する。登場判定未設定なら目標値・指定技能をまとめて出し、既存の指定技能も編集可能な初期値として扱う。設定済みの項目は出さない。住宅施設の舞台選択は候補がある場合に利用でき、選択プレイヤーに応じて候補が変わる。詳細は KI-064。

### 2026-09-11 シーンのエリア選択幅

登場判定欄が非表示のとき、エリア選択欄の380px上限を解除し、ラベルを除く行の残り全幅に広げる。Stylelint通過。

### 2026-09-11 エンディングの登場・チーム解散

エンディングに登場判定はない。アクトシートではエリア・登場判定・指定技能を表示せず、開始時にも設定を求めない。パネルのPL操作は「登場」で担当キャラクターを自由登場させる。RLによる手動登場・事前の登場キャラクター指定は従来どおり利用できる。チームはクライマックス終了（別フェイズへの移行）時に自動解散し、エンディングへ直接移る場合も解散する。クライマックス内のシーン移動では維持する。詳細はKI-065。

### 2026-09-11 履歴入力のフォーカスとレコードのスクロール

キャスト履歴・レコードシートの保存同期による再描画時は、現在入力中の行と項目へフォーカスを復元し、未保存の値・カーソル位置を保持する。レコードシートは経験点サマリと列ヘッダーを固定し、履歴一覧だけをスクロールする。詳細はKI-066。

2026-09-11 履歴フォーカス追修正: キャストの履歴タブをDOM挿入前に表示状態へ戻す。フォーカスはpreventScrollで復元し、履歴タブのスクロール位置もPARTSの同期対象に含める。レコードのヘッダー固定は維持。KI-066参照。


### 2026-09-12 キーハンドアウトと既存ハンドアウトの再調査

ユーザー確定事項: 1対1のペルソナ/キー。キーはPSなし、公開条件と推奨タイミングは短い通常input。担当本人が公開可能。公開でペルソナ/PS/取得済みコネは変更しない。アクトシートはRLの原稿編集専用、PLはコントロールパネルから専用ApplicationV2で再閲覧する。

既存実装の追跡結果: handouts配列がアクトJournalのflagに保存され、フォーム変更時に行IDで更新される。userIdからuser.characterを解決し、旧actorIdは読み替える。シーンプレイヤー指定、巡回順序、PCへのコネは既存行IDを参照。読み込みと開始は別で、開始前にもパネルから配布できる。送信カードは内容・解決済みのコネ情報を保持。受け取り時にNPCは辞典から複製、PC/自由記述は技能生成。同名の重複を避ける。開始時のコネ自動取得は廃止済み。アクト終了で原稿やチャットは削除せず、経験点配布後にアクト限定技能を維持/削除する。

実装: useKeyHandouts既定false。原稿は各handouts行のkeyHandoutに置き、既存の編集部品を共用する。handoutGroupsで対応する2行を一つのhandout-pair枠にまとめ、内部区切りと組間余白で区別する。PSと対象ユーザーはペルソナ側のみ。公開/推奨タイミングはinputで保存・エスケープ表示し、本文だけをリッチテキストにする。

配布時の内容はkeyHandoutDeliveries.<handoutId>へ保存し、私信で本人とRLへ送信する。再閲覧はこの配布済み記録だけを読み、チャット削除後も維持。担当/公開状態を専用アプリと公開処理で確認する。公開要求はIDだけをソケットでGMに送り、再確認して配布済みカードを公開。元カードが削除済みなら公開カードを再作成する。キーの公開でActor/PS/コネを更新しない。担当変更時は保存記録と私信宛先が追随する。別Journalの作成やシート登録は撤去した。既にワールドに作成された一時版のJournalはこのコード変更では削除しない。

秘匿の範囲: Foundry標準の私信とTNXのユーザー別閲覧判定を使用。PLにアクト編集画面を提供しない。共有クライアントデータに対する暗号化や独立したサーバー認可を追加するものではない。

検証: 既存コードの作成/保存/参照/開始/終了の経路を照合。配布・本人以外拒否・公開二重実行・PS不変・再閲覧が原稿を読まないこと・チャット削除後の再閲覧と公開を回帰検証。全128ファイル2867テスト通過、ESLint/Stylelint通過。Foundry同梱Handlebarsでテンプレートをコンパイル。Edgeヘッドレスで既存CSSを読み込み、共通枠内のペルソナ/キー、2項目のinput、次の組との間隔を画像確認。Foundryの実セッション操作は未確認。


### 2026-09-12 キーハンドアウトのRL送信とPL公開を分離（追修正）

ユーザー指摘: 既存の配布・送信欄はアクト開始後に隠れる。RL向けは開始後も使える独立セクションが必要で、何度でも送信できなければならない。公開はPLが行い、内容全体を新しい公開チャットとして送る。

修正: タブ外の共通一覧を撤去。RLは開始前のボディと開始後のアクトタブに専用fieldsetを表示し、番号・スタイルから解決したタイトルと担当名、既存の紙飛行機アイコンで送信する。再送は現在の原稿を本人とRLへ私信送信し、配布済み記録を更新する。公開済みフラグは再送でも維持。PLはボディ内で再閲覧/公開。RLのパネル・閲覧アプリに公開ボタンは出さず、公開権限判定でもRLを拒否する。PLの公開は元の私信を変更せず、本文・コネ・スタイル・推奨スート・公開説明・推奨タイミングを含む新しい公開カードを送信し、保存記録を公開済みにする。既存ペルソナ配布欄の表示条件は維持。

検証: 実テンプレートをFoundry同梱Handlebarsでレンダリングし、RL/PL×開始前/開始後の4状態でボディ内配置、RLのアクトタブ内配置、RL公開ボタン不在を確認。回帰テストは再送可能、公開時の私信保持・全内容送信、二重公開防止、RL公開拒否、公開後の再送と状態維持を検証。Foundry実機操作は未確認。
