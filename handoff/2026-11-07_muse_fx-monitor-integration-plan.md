# 変更方針: FX Monitor 統合（index.next.html の作成）

- 日時: 2026-11-07 17:15 JST 頃
- 担当: Muse
- 依頼: `2026-11-07_claude_fx-monitor-integration.md`（指示書）
- このファイルは着手前の変更方針（指示書2-5、AGENTS.md「既存ファイルを大きく書き換える前」の規定による）

## 読んだもの（指示書2の確認）

- ai-repo の `AGENTS.md`、`docs/status.md`、`docs/infra.md`
- test リポジトリの `index.html`（OANDA版、424行）、`GMO/index.html`（GMO版、335行。ローカルの `~/workspace/gmo_monitor/index.html` と同一であることを md5 で確認）、`GMO/README.md`、`sw.js`（CACHE = `fx-monitor-v2`、SHELL は `./, index.html, manifest.json, icon-192/512`）、`manifest.json`
- 実データの形: 実キーは使わず、ダミー金額の `gmo_stats.sample.json` と生成側コード（`gmo_monitor.py` / `chart_data.py`、いずれもユーザーの Pi/VPS で稼働中のもの）で確認した

## 指示書9節への回答（確認結果）

1. **`gmo_stats.json` の最新形**: test リポジトリの `GMO/README.md` は古い（「履歴は含まない」時点）。現行の `gmo_monitor.py` は次を出す。統合版はこの形に合わせる。
   - トップ階層: `source` / `generated_at`（ISO・Z） / `generated_at_unix` / `account` / `positions` / `pending_orders` / `errors` / `history`（約定蓄積がある場合のみ）
   - `history`: `executions_total` / `closed_total` / `added_this_run` / `first_ts` / `last_ts` / `db_file`（GMO版 HTML の参照と一致）
   - `account.margin_ratio` は生成側でも番兵値対策済み（証拠金0 または >1e12 で `null`）。フロント側の >1e12 → null 処理も指示書どおり残す
2. **`positions.detail` / `pending_orders`**: どちらも現行の生成物に入っている。
   - `detail[]`: `position_id` / `instrument` / `side` / `units` / `price` / `unrealized` / `swap` / `open_time`
   - `pending_orders[]`（トップ階層）: `instrument` / `type` / `order_type` / `settle_type` / `units`（売りは負） / `price` / `status` / `order_id` / `tag`
3. **push 権限**: 保存済みの git 認証情報で clone 済み。push は `index.next.html` 追加時に実地で確認し、結果を完了報告に書く

## 変更方針

新規ファイル **`index.next.html`** を1つ作るだけ。`index.html` と `sw.js` には触らない（ユーザーのOK後、置き換えと CACHE バージョン上げを同じコミットで行う）。

### 土台（OANDA版から引き継ぐ）

- タブ構成（ポジション／チャート／仲値／手法別）、PWA の作り（manifest・アイコン・theme-color・safe-area）、Chart.js（cdnjs）と lightweight-charts（unpkg）の読み込み、仲値タブ・手法別タブの描画コード、OANDA チャート（H1/D1・通貨強弱）の描画コード、CSS 変数とダークモード対応

### 移植（GMO版から）

- ポジションタブ先頭の合計（`rSum` の考え方。片方の口座しか取れていない場合は、取れた側だけの合計として出す）
- GMO セクション: 含み損益（価格）／スワップ／含み＋スワップの KPI、`positions.detail` の建玉明細、`pending_orders`（トップ階層）、`history` の1行、口座ごとの維持率警告バー（`alertHtml` 方式、WARN 500% / BAD 300% は両口座共通の定数）
- チャートタブの GMO 側: `charts.json` の自前 canvas 描画（チップ選択・1時間足/日足・ローソク/ライン切替、`fx_chart_type` の保存を含む）をそのまま移す

### 統合の要点

- **取得は口座ごとに独立**: GMO と OANDA の `stats.json` は個別に fetch し、個別に catch して、そのセクションだけにエラー表示する。OANDA版の `if(!D) return` 型（片方が無いと全体が描画されない作り）にはしない。公開データ（OANDA の `/public/*.json`、GMO の `charts.json`）も同様に独立
- **仲値合算は1か所**: OANDA版の合算ブロック（`mergeStrat` / `mergeEq` / `mergeTag` / `combine` 等）を正として1組だけ置き、OANDA データの読み込み時に1回だけ適用する。GMO版にあった簡易版 `mergeTag` は持ち込まない。OANDA セクションのポジション表示はルート版 `rPos` を移植し、未約定注文はルート版どおり `/public/positions.json`（`PUB.po.pending_orders`）から出す（GMO の未約定とは置き場所が違う点に注意）
- **チャートタブは2択切替**: 「GMO（charts.json）」／「OANDA（H1/D1・通貨強弱）」、既定は GMO。片方の描画コードは作り直さない。非表示側は幅0で描画が壊れるため、表示に切り替えたタイミングで再描画する（GMO の `drawChart` は幅0なら何もしない既存ガードを利用。OANDA 側の lightweight-charts は初回表示時に構築する）
- **見出しに更新時刻**: ポジションタブの GMO／OANDA 各セクションとチャートに、データの更新時刻（JST）を出し、30分以上古ければ赤にする（既存仕様の踏襲）。維持率警告バーは各セクション先頭
- **仲値タブ・手法別タブ**: OANDA版そのまま。見出しに「OANDA実績」と明記する。GMO の手法別は今回作らない（指示書6）

### 設定と localStorage（再入力させない）

設定ダイアログ: GMO URL／OANDA URL／トークン（共通）／OANDA 側チャートの通貨ペア4つ。既定 URL は GMO版の `DEF_G` / `DEF_O`。トークン未設定ならダイアログを開く。

読み取りは旧キーをフォールバックとして残す:

| 項目 | 優先 | フォールバック |
|---|---|---|
| GMO URL | `gmo_fx_url` | （なければ `DEF_G`） |
| OANDA URL | `fx_oanda_url` | `fx_url` → （なければ `DEF_O`） |
| トークン | `gmo_fx_tok` | `fx_tok` |
| OANDA チャートペア | `fx_syms` | 既定4ペアで補完（OANDA版の既存処理） |
| GMO チャート形式 | `fx_chart_type` | `candle` |

保存時は新旧両方のキーに書く（`gmo_fx_url` / `fx_oanda_url` / `fx_url`、トークンは `gmo_fx_tok` / `fx_tok`）。置き換え前の旧ページ（ルート版・GMO版）と同じオリジンで localStorage を共有するため、プレビューで保存した設定が旧ページでもそのまま使える。

### PWA

- 今回は `sw.js` を変更しない。置き換え時に CACHE を `fx-monitor-v2` → `fx-monitor-v3` に上げ、`index.html` の置き換えと同じコミットに入れる
- 確認方法は完了報告に書く（ホーム画面アプリを開き直す／更新ボタンで再取得し、タブ構成が新しいものになっていることを見る、等）

## 検証計画（指示書7）

1. インライン `<script>` を取り出して `node --check`
2. jsdom 上でモック JSON（金額はダミー）を fetch スタブから返し、4パターンを確認: 両方OK／GMOだけ失敗／OANDAだけ失敗／建玉なし（GMO `margin_ratio` が番兵値→null 表示）
3. 旧設定だけが入った localStorage（`fx_url`+`fx_tok` のみ／`gmo_fx_*` のみ）で再入力なしに表示されること
4. コンソールエラーが出ないこと。幅375px とライト/ダークは、CSS が両版とも既存のもの（max-width 860px・safe-area・`prefers-color-scheme`）を引き継ぐことを確認し、可能ならスクリーンショットを添える

## スコープ外（指示書6の再掲）

GMO の手法別集計の追加、発注系機能、OANDA 側データ生成の変更、通貨ペア追加、デザイン全面変更はしない。
