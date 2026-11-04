# GMO FX Monitor (v1: 口座＋ポジション)

OANDA版 FX Monitor（github.com/mokushun/test）の GMOコインFX版。
v1 は口座とポジションの表示のみ。確定損益の履歴・手法別集計（PF／月別等）は含まない
（GMO API の約定履歴は直近1日分しか遡れないため、やるなら日次蓄積が別途必要）。

## 構成

| ファイル | 役割 | 置き場所 |
|---|---|---|
| `gmo_monitor.py` | GMO Private API（参照のみ）→ `gmo_stats.json` 生成 | Pi（gmo_nakane.py と同じフォルダ） |
| `index.html` | 表示フロント（OANDA版の見た目を踏襲、ポジション画面のみ） | GitHub Pages 等の静的ホスティング |
| `gmo_stats.json` | 生成物。フロントが URL＋トークンで取得 | トークンで守れる場所（下記） |

## 生成側（Pi）

前提: `gmo_nakane.py` 用の `config.ini`（`[gmo] api_key / api_secret`、chmod 600）が
同じフォルダにあること。キーはそのまま流用し、新しい設定は不要。

```bash
# 疎通確認（ファイルは書かない）
python3 gmo_monitor.py --check
# → OK: 純資産 ... / 維持率 ...% / 建玉 N件 含み損益 ... / 未約定 N件

# 生成（既定の出力先は同じフォルダの gmo_stats.json）
python3 gmo_monitor.py

# 出力先を明示する場合
python3 gmo_monitor.py --out /path/to/served/gmo_stats.json
```

cron 例（5分おき。ボットの毎分 cron とは別行で追加）:

```cron
*/5 * * * * cd /home/pi/github/dev-lab/GMO_Trade && python3 gmo_monitor.py >> monitor.log 2>&1
```

- 取得は `GET /v1/account/assets`、`GET /v1/openPositions`（prevId で全件）、
  `GET /v1/activeOrders` の3つだけ。注文系 API は呼ばない。
- assets / openPositions が失敗したら何も書かずに終了コード1（前回の JSON が残る）。
  activeOrders だけ失敗した場合は `errors` に記録して本体は出す。
- 標準ライブラリのみ。Python 3 があれば Pi でそのまま動く。

## データ形式メモ（OANDA版との差）

- `account`: `margin_ratio / nav(時価評価総額) / margin_used / balance / available /
  unrealized(評価損益・価格分) / swap(未決済スワップ)`
- `positions.total.unrealized` は**価格分のみ**。スワップは `swap` に分離し、
  合計は `unrealized_with_swap`。OANDA版 JSON と同じキー名に揃えてある。
- GMO には手法タグがないため、建玉は全件を単一タグ `NAKANE_GMO`
  （表示名「仲値買い (GMO)」）で集計する。手動建玉が混ざると実態とずれる点に注意。
- `positions.detail` に建玉ごとの明細（建玉ID・建値・建てた時刻）も入れる
  （ボットの約定確認用。OANDA版にはない GMO 版の追加分）。

## フロントと公開方法（未決定・運用側で決める）

フロントの設定ダイアログに `gmo_stats.json` の URL とトークンを入れると、
`Authorization: Bearer <token>` 付きで取得する（OANDA版と同じ方式、
localStorage のキーは `gmo_fx_url` / `gmo_fx_tok` で OANDA版と分離済み）。

口座データなので**認証なしの公開URLには置かないこと**。OANDA版 stats.json と
同じ配信の仕組み（トークンを検証する場所）に GMO 用ファイルを1つ足すのが
一番素直。それが使えない場合は、生成を VPS 側でなく Pi で行い、
scp 等で VPS の保護されたパスへ送る構成でもよい（その場合もトークン検証は必須）。

フロントの設置先は、test リポジトリに `gmo.html` として追加するのが早い
（別リポジトリでも可）。

## テスト状況（2026-11-04、Muse VM 上）

- 生成側: モック API レスポンス（公式ドキュメントの形式）で集計を検証済み。
  ペア別平均建値・合計・スワップ分離・空口座ケースのアサーション通過。
- フロント: 生成したモック JSON を実際に読み込ませて描画を確認済み。
- **未実施**: 実キーでの `--check`（Pi 上でのみ可能）。Pi に置いたらまずこれを実行。
