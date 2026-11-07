# 指示書: FX Monitor の統合（OANDA版 + GMO版 → 1つのアプリ）

- 作成: 2026-11-07（JST）、Claude
- 担当: Muse（コーディング）
- 依頼者: ユーザー

## 1. 目的

次の2つの `index.html` を1つに統合する。

| 呼び名 | 場所 | 内容 |
|---|---|---|
| OANDA版（元） | test リポジトリの `index.html` | 4タブ（ポジション／チャート／仲値／手法別）。PWA（manifest.json・sw.js）。Chart.js と lightweight-charts を使用 |
| GMO版 | `GMO/index.html`（`GMO/README.md` も読むこと） | 1ページ縦並び（合計／GMO／OANDA／チャート）。チャートは自前のcanvas描画 |

**背景**: 今後の運用は OANDA から GMOコインFX に移行していく。APIトレードのテストは今のところ問題なし。
したがって **GMOが主、OANDAは移行期間中の副（残ポジションの確認と過去実績の閲覧用）** という位置づけで統合する。

## 2. 作業前に必ずやること

1. `AGENTS.md`、`docs/status.md`、`GMO/README.md` を読む。
2. 両方の `index.html` を最後まで読む（下の「3. 現状の差分」は要点であり、正は実コード）。
3. `sw.js` と `manifest.json` を読む（キャッシュ対象と更新方法を確認）。
4. 実データの形を確認する。**実キーは使わない・要求しない**。次のどちらかで進める。
   - `GMO/` に残っているモック JSON（前回の検証で使ったもの）
   - ユーザーに、個人情報や金額をダミーに置き換えた `gmo_stats.json` / `stats.json` のサンプルを依頼する
5. 着手前に、変更方針を `handoff/` に1ファイルで書く（AGENTS.md の「既存ファイルを大きく書き換える前」の規定）。

## 3. 現状の差分（要点）

### データ源

| 用途 | OANDA版 | GMO版 |
|---|---|---|
| 口座・ポジション | `fx_url`（`/oanda/stats.json`）。Bearer認証 | `gmo_stats.json`（`/oanda/gmo_stats.json`）と `stats.json` の2本。Bearer認証（共通トークン） |
| チャート | `/public/h1.json` `d1.json` `strength.json` `status.json` `positions.json`（認証なし） | `/public/charts.json`（認証なし） |
| 履歴・手法別 | `stats.json` の `strategies`（equity／by_month／by_weekday） | GMOには `history`（約定の累計件数）のみ。手法別集計はまだ無い |

### localStorage のキー

- OANDA版: `fx_url` `fx_tok` `fx_syms`
- GMO版: `gmo_fx_url` `gmo_fx_tok` `fx_oanda_url` `fx_chart_type`

→ 統合後は **キーを整理するが、旧キーも読み取りのフォールバックとして残す**（ユーザーのスマホに設定が入っているため、統合後に再入力させない）。

### 表示ロジックの違い（注意）

- GMOの含み損益 `positions.total.unrealized` は **価格分のみ**。スワップは別項目（`swap`、`unrealized_with_swap`）。OANDAにはこの区別がない。
- GMOの未約定注文は `G.pending_orders`（トップ階層）、OANDAは `PUB.po.pending_orders`（`/public/positions.json`）。場所が違う。
- GMOのポジションには `detail`（建玉明細）がある。OANDAにはない。
- GMOは手法タグが無く、全件が `NAKANE_GMO`（表示名「仲値買い (GMO)」）。
- 仲値の合算（`TEST_NAKANE_BUY` → `NAKANE_BUY`）の処理が、両ファイルに重複して存在する。OANDA版の方が完全（`mergeStrat` など履歴の合算を含む）。
- GMOの `margin_ratio` が極端に大きい値（>1e12）のときは `null` 扱いにしている（建玉なしの口座）。この処理は残すこと。
- 維持率の警告ライン（500%／300%）は定数。両口座で同じ値を使う。

## 4. 統合後の仕様

### 4.1 構成（タブ）

ベースは **OANDA版のタブ構成とPWAの作り**。そこにGMO版の表示を移植する。

| タブ | 内容 |
|---|---|
| ポジション | 先頭に合計（純資産・含み損益。GMO版の `rSum` と同じ考え方）。続けて **GMOセクション（主）→ OANDAセクション（副）**。GMOには含み損益（価格）／スワップ／含み＋スワップ、建玉明細、未約定注文を出す。OANDAは現状の表示のまま |
| チャート | 切り替え（2択）: 「GMO（charts.json）」と「OANDA（H1/D1・通貨強弱）」。既定は GMO。それぞれ既存の描画をそのまま使い、片方を作り直さない |
| 仲値 | 現状のOANDA版そのまま（曜日別・月別。TEST_NAKANE_BUY は NAKANE_BUY に合算済み）。見出しに「OANDA実績」と明記 |
| 手法別 | 現状のOANDA版そのまま。見出しに「OANDA実績」と明記。**GMO用の手法別は今回作らない**（下の6を参照） |

- 各セクションの見出しに、データの更新時刻（JST）を出す。30分以上古ければ赤にする（既存の仕様を踏襲）。
- 口座ごとの維持率警告バーを、各セクションの先頭に出す（GMO版の `alertHtml` 方式）。

### 4.2 設計の約束事

1. **GMOとOANDAの取得は互いに独立**させる。片方が失敗しても、もう片方は表示する（GMO版の `Promise.all` + 個別 `catch` の方式を踏襲。OANDA版の `if(!D)return` のように、片方が無いと全体が描画されない作りにしない）。
2. 仲値の合算処理は **1か所にまとめる**（重複を消す）。
3. 非表示のタブ・セクションに描画しても幅が0になる。**表示するタイミングで再描画**する（GMO版の `drawChart` は幅0なら何もしない作り）。
4. 外部ライブラリは今のまま（Chart.js は cdnjs、lightweight-charts は unpkg）。新しいライブラリは追加しない。
5. ダークモード・iPhoneの幅（375px）で崩れないこと。`env(safe-area-inset-*)` の指定を残す。
6. 単一の `index.html` に収める（ビルド工程なし）。

### 4.3 設定ダイアログ

- 入力項目: GMOデータのURL／OANDAデータのURL／トークン（共通）／チャートの通貨ペア（OANDA側の4つ）
- 既定URLはGMO版の `DEF_G` `DEF_O` を引き継ぐ。
- トークンが未設定ならダイアログを開く（既存の挙動）。

### 4.4 PWA

- `sw.js` のキャッシュ対象・バージョンを確認し、**キャッシュのバージョンを上げる**（古い画面が残らないように）。
- 更新後、スマホのホーム画面アプリで新しい画面になることを確認する方法を、完了報告に書く。

## 5. 進め方・成果物

1. 最初は **`index.html` を上書きしない**。新しいファイル名（例: `index.next.html`）で作り、ユーザーがスマホで確認できる状態にする。
2. ユーザーのOKが出てから、`index.html` を置き換える（`sw.js` のバージョン更新も同時）。
3. Museは test リポジトリの **main に直接コミットしてよい**（ブランチ・PRは不要。発注系の機能がなく、口座に影響しないため）。ただし次を守る。
   - GitHub Pages は main をそのまま公開するので、**ユーザーがOKを出すまで `index.html` と `sw.js` には触らない**（`index.next.html` の追加だけなら公開中の画面に影響しない）。
   - コミットは小さく分け、メッセージに何を変えたか書く（問題があれば `git revert` で戻せるように）。
   - 公開リポジトリなので、トークン・個人の金額が入ったJSON・スクリーンショットはコミットしない。
   - `index.html` を置き換えるコミットは、`sw.js` のバージョン更新と同じコミットにする。

## 6. 今回やらないこと（スコープ外）

- GMOの手法別集計・累積損益・月別の追加（`gmo_monitor.py` 側に履歴の蓄積が必要。`history` が貯まってから別タスクで）
- 注文・決済などの発注系の機能（このアプリは **表示のみ**。注文APIは呼ばない）
- OANDA側のデータ生成（`update.py`）の変更
- 通貨ペアの追加、デザインの全面的な変更

## 7. 検証（完了報告に結果を書くこと）

1. JavaScript の構文チェック（`<script>` を取り出して `node --check`）。
2. モックJSONで描画確認。次の4パターン:
   - 両方取得できる
   - GMOだけ失敗（OANDAは表示される）
   - OANDAだけ失敗（GMOは表示される）
   - 建玉なし（GMOの `margin_ratio` が極端な値／`null`）
3. ライト・ダーク両方、幅375pxで表示確認（可能ならスクリーンショットを `handoff/` に添える）。
4. 旧設定（`fx_url` `fx_tok` `gmo_fx_*` だけが入った状態）でも、再入力なしに表示される。
5. ブラウザのコンソールにエラーが出ていない。

## 8. ルール（AGENTS.md の再掲）

- 秘密情報（トークン・APIキー・個人の金額が入ったJSON）は、コード・md・ログ・スクリーンショットに入れない。
- 無料枠の範囲で作業する。
- 作業の終わりに、`handoff/YYYY-MM-DD_muse_<内容>.md` に次を書く: 日時（JST）／やったこと／結果・エラー／変更したファイル／次にやってほしいこと／注意点。
- 不明点があって止まる場合は、推測で進めず、質問を `handoff/` に書いてユーザーに伝える。

## 9. 確認が必要になりそうな点（Muse→ユーザーへ）

- 実際の `gmo_stats.json` の最新の形（`GMO/README.md` は「約定履歴は含まない」だが、HTMLは `history` を参照している。`gmo_monitor.py` が更新されている可能性）
- GMO側で `positions.detail` や `pending_orders` が実際に入っているか
- test リポジトリへの push 権限
