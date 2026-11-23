# CIとルール処理の再監査（2026-11-24）

開始時は本体 `main@e64a6ff`、Wiki `master@6f07c6d`。両方ともリモートと一致し、作業ツリーはクリーンだった。用途・技能の開発はローカルで再開できる。共有ブランチで自動検証を必須に戻すには GitHub 側で請求ロックを解除し、修正を含む実行を成功させる必要がある。

## CIの障害履歴と復旧条件

| 実行 | ジョブを確認した結果 |
|---|---|
| 9月16日 [#79](https://github.com/PearlAwatsuki/TNX-for-FVTT/actions/runs/35069107140) | ホストランナーが割り当てられ、インストール・lint・テストを完走した最後の成功 |
| 9月19日 [#80](https://github.com/PearlAwatsuki/TNX-for-FVTT/actions/runs/35438115259) | ランナーが起動し、ESLintで8エラー。CJSの不要なエスケープ7件、同名のシートフック1件。CSS lintとVitestには到達せず。Nodeアクションの非推奨注記は成功した#79にもある |
| 9月24日 [#81](https://github.com/PearlAwatsuki/TNX-for-FVTT/actions/runs/35951478240) 〜 [#90](https://github.com/PearlAwatsuki/TNX-for-FVTT/actions/runs/45174841130) | 10実行すべてでランナー名が空、ステップ0。各ジョブのChecks注記は “The job was not started because your account is locked due to a billing issue.” |

Actions RESTのruns・jobsと対応するcheck-runのannotationsを照合した。#90のジョブログの404と整合する。対象期間の `.github/workflows/ci.yml` は同一設定で、公開リポジトリの `main` へのpush・PRを対象に、標準 `ubuntu-latest` とNode 22を使う。現在のActions権限設定APIは認証なしで401。請求明細や決済拒否の理由は公開APIの確認範囲外。標準ランナーの公開リポジトリ利用は無料であり、分数枠の使い切りを示す証拠はない。[GitHub Actionsの料金](https://docs.github.com/en/actions/concepts/billing-and-usage)

**所有者の操作**　GitHubのSettings → Billing & licensingで、対象アカウントのPayment information、支払い履歴、失敗通知を確認する。未払い・決済拒否・与信失敗の表示があれば該当項目を解消する。残高ゼロで対応項目が表示されない場合は、GitHub Supportに所有者名、リポジトリ、#90のURL、ジョブID `137693309637`、上記注記を添えてロックの確認を依頼する。解除後、修正を含むSHAでジョブが起動し全工程を通ることが復旧条件。旧SHAの再実行では今回のコードを検証できない。[GitHubのロック解除手順](https://docs.github.com/en/billing/how-tos/troubleshooting/locked-account)

## ローカル検証

| 工程 | 結果 |
|---|---|
| 修正前 | 198ファイル、3,482成功・25失敗・3スキップ。ESLintは12エラー・11警告 |
| `npm ci` | ロックファイルから再現でき、終了コード0。npm auditの依存関係警告は16件（moderate 6、high 8、critical 2）。依存更新は別作業 |
| `npm run lint` | 終了コード0、エラー・警告ともに0 |
| `npm run lint:css` | 終了コード0 |
| 修正箇所・関連回帰 | 状態付与、シート、カード、報酬点、リアクション、用途移行等の16ファイル341件がすべて成功。全テストの内数 |
| `npm test` | WindowsでCI指定のNode 22.23.5とローカルの24.13.1の両方を実行。各200ファイル、3,533成功・0失敗・3スキップ |
| GitHub Actions | 修正コードのホスト実行は未検証。請求ロックの解除と対象SHAでの実行が必要 |

3件のスキップは変更前からある `item-sheet-submit`、`info-links`、`style-skill` のFoundry環境依存ケース。31件を追加し、廃止済み判定だけを呼ぶ5件を整理した。修正前には、AR不変、不正カードのワイルド指定、禁止された山札の消費、部分合計、軽減記帳の上書きを個別に再現した。

## 修正と再現

| 修正 | 原因・帰結 | 主な実装と回帰テスト |
|---|---|---|
| アクターシート再描画 | 後方の同名メソッドが複数領域の記録と遷移抑制を隠していた。親の前処理を1回呼び、サイドバー・本文・タブの位置を記録。親の描画後に新しいDOMへ戻し、遷移を再開する | `actor/tnx-character-sheet-base.mjs` の `_preRender`・`_onRender`。実シートクラスと差し替えたDOMで `actor-sheet-scroll.test.mjs` が検証 |
| ローカルCI | `script.cjs`・`script3.cjs` は旧テンプレート・メニューの一括置換用。固定絶対パスを持ち、現行の起動元もなく、移行先の形式は導入済みのため削除。現行HEADの別エラーはエクスポート名の正規表現と登場判定テストの未import。未使用変数・importも整理した | `app/tnx-record-sheet.mjs`、`appearance-logic.test.mjs` ほか。シナリオ進行・削除の単純確認は既存の `confirmDialog` を使用。規約テストの上限値は維持 |
| 精神12・17 | 負傷作成時、カット参加者の現在ARを「驚愕」で1減らし「士気喪失」で0にする。下限0。無視・無効・カット外では支払わない。元のダメージ行または個別の効果IDで記帳し、同じフックが再送されても重ねない。独立した命中は個別に効く | `data/damage-chart.mjs`、`core/item-transfer.mjs#applyWoundActionRank`、`flow/condition-resolution.mjs#applyDamageChartResult`、`flow/damage-flow.mjs`。作成・削除フック、無視、再送、次カットの付与値を `condition-application.test.mjs` で検証 |
| 状態ドローと邪毒 | 山札・捨て札なし、空、取得失敗、公開失敗、不正な数字・スートは通知して受付を未解決に保つ。実ジョーカーのみ指定・引き直し。邪毒の数字指定では存在しないスート欄を読んでいた点も修正 | `flow/condition-resolution.mjs#drawOneToDiscard`、`executeConditionDraw`、`resolvePoisonDraw`。チャットの邪毒ボタンから肉体負傷まで検証 |
| 重圧と技能使用禁止 | 判定開始後に付いた負傷も山札ドロー前、カード確定時に再照合する。引いて初めて確定する重圧では、宣言・変換後のスートで拒否した1枚を返却する。実絵札がファンブルになる経路でも変換後のスートを使う。拒否時は判定を選択待ちに保つ | `flow/tnx-check-flow.mjs#executeFromDeck`・`_execute`、`card-number-change.test.mjs`。Foundry 14.369の `Cards.draw`、`_drawCards`、`Card.recall` の実装を照合。返却は元文書のdrawnを解除し、捨て札のコピーだけを削除するため元のsortを保持する |
| 複数枚のダメージドロー | 《暴露》で2枚目を引けなくても1枚の合計で先へ進めた。ジョーカー指定のキャンセルも0相当になった。全枚数の数字が確定した場合のみ合計し、不足・例外・キャンセル時は停止。社会9・19の派生では親の負傷と未解決の派生を区別して表示 | `flow/damage-flow.mjs#drawDamageValueFromDeck`・`applyDerivedDamage`、`flow/miracle-flow.mjs#promptMiracleDamageResult`、`condition-application.test.mjs` |
| 同一クライアントの社会ダメージ軽減 | 残高4点に3点の要求を2件重ねると、実際は3点＋1点を支払い、遅いチャット更新で最後の1点しか残らなかった。メッセージ単位で資格確認・支払い・台帳更新の完了まで直列化した | `flow/damage-flow.mjs#promptBountyMitigation`、`check-bounty.test.mjs`。チャット更新を遅延させて2行と残高を確認 |

本体のパスは `scripts/` からの相対表記。付与型ARの根拠は [Combat_Flow](../01_Wiki/Game_Rules/Combat_Flow.md)、チャートは [Damage_Rules](../01_Wiki/Game_Rules/Damage_Rules.md)。治療により過去のARは払い戻さず、次カットは `rules/combat-seed.mjs#buildCombatSeedUpdate` の通常の付与に従う。「可能なら戦闘中止」は卓の裁定に残る。

## 横断監査と前回9項目

| 照合対象 | 実装経路と確認結果 |
|---|---|
| 判定・購入・再判定 | `rules/purchase.mjs#computeNoCardPurchase` と購入チャットの失敗差分はnull。通常判定のA固定21は報酬点を消費しない。スート不一致・ファンブルは不成立のまま、許可された再判定では新しいカードを使う。再判定で用途コストを重ねず、参加技能から報酬点資格を再評価する。山札の実J/Q/Kはファンブル、ジョーカーによる同ランク宣言は10。`check-modification`・`card-number-change`・`check-bounty`・`purchase-logic` の各テストで確認。[判定ルール](../01_Wiki/Game_Rules/Check_Rules.md) |
| 攻撃・リアクション・ダメージ | `flow/attack-flow.mjs` のパリー成立状況から `rules/reaction.mjs#resolveTargetDefense` が有効な受け値を解決する。敗北した成立パリーと不成立を区別。ダメージ算出は `rules/damage.mjs#computeDamage` で恒久軽減後に0を適用し、次に事後修正、最後に適用時軽減。複数対象への書込みはKI-073 |
| 負傷・BS・戦闘不能 | `rules/conditions.mjs#getEffectiveConditions` が無効・無視を反映。社会20は `character-base.mjs#effectiveCitizenRank` だけをXとし基礎値を保存する。電子妨害は `_computeJammingSource` で上方判定にだけ加算する。作成フックは負傷由来のBS・戦闘不能を紐付け、治療・制御による無効化・シーン境界の失効は別経路。旧Wikiの技能禁止「警告のみ」と「不明スートはジョーカー」を現行裁定に合わせて訂正。[状態仕様](../01_Wiki/Game_Mechanics/Conditions.md) |
| 報酬点と用途資源 | `spendBounty` は確定時の資格と残高から実消費額を返す。入力中の減額・口座凍結・信用失墜、用途の支払後の判定、再判定、購入、社会軽減を検証。2つの独立したモジュール・Actor複製が残高3点を同時に読んだ実験では双方が3点を算入し、保存要求は2件とも同じ残高だった。実ネットワーク速度や発生頻度は測定していない。KI-072、KI-074 |
| 用途と参加技能 | 判定起動・参加技能追加・カード確定時の禁止、リアクションの選択手段、技能定義から実効値を導く経路を確認。手動行、独立した旧用途の移行・復元、`settingsBackup` の回帰も実行。[数値供給の確定仕様](../01_Wiki/Game_Mechanics/Skill_Numeric_Provision_Review.md) |

### 既存テストの修復根拠

- 現行AE仕様では `check.melee` は技能、`check.meleeAttack` は白兵攻撃。次回失効のテストにも同じ区別を適用した。自動付与は準備先か選択対象を指定し、自身への実適用は実体化コピーで検証する。式評価のテストには準備済み武器を与え、期待攻撃力11と適用回数2を維持した。[AE仕様](../01_Wiki/Game_Mechanics/Active_Effects.md)
- 登場のテストは `setAppearing(actor,true,{ghost:false})` と登場失敗時のフラグ巻き戻しを検証する。廃止済みのドローン許可リストを呼ぶテストは、搭載分類・準備先を区別する実行テストに更新した。履歴入力ではDOMが持つタグ名をモックに与え、行、カーソル、フォーカス、タブ復元の各検査を維持した。
- 神業回復の「同一シーン限定」は卓判断へ移った。実装から呼ばれない旧判定関数とその単体テスト5件を整理し、現行の `recoveryCandidateAllowed` は旧シーンも候補となり神業由来の制限が働く既存テストで検証した。失敗テストのskip化や規約テストの上限緩和は行っていない。

## 実機確認と裁定

優先順位と受入条件の正本は [未解消課題](../01_Wiki/Issues/Active_Issues.md) と [詳細](../01_Wiki/Issues/Known_Issues_Detail.md)。KI-071は外部障害、KI-072はモック再現済みの競合、KI-073・074はコードから特定した部分更新リスク、KI-075は未実装の再開機能である。

1. 所有者とRLの2ブラウザで、同時決済、GM委譲時の切断・再接続、複数対象の権限差を確認する。単一決済者と要求IDの導入は既存の用途移行から独立して設計する。
2. プレイヤー権限で重圧対象の山札カードを公開し、同じ札だけが元の順序に戻り、別の手札で判定できることを確認する。返却しても公開済み情報は消えない。通常どおり公開して元位置に戻す方式を実装した。札をプレイヤーに一切知らせない運用を求める場合はRLによる非公開判定と、ジョーカー・任意のスート変更の宣言時点について別途裁定が必要になる。
3. カット進行中に精神12・17を加え、治療・カット更新・未リンクトークンのAR表示を照合する。入力中の用途コスト不足を再確認するか、中断した派生ドローの既出札を持ち越すかは、卓の判断を要する。今回の検証はVitest、DOM、インストール済みFoundryソースの照合まで。
