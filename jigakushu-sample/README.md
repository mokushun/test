# jigakushu-sample — Blogger→GitHub Pages 移管見本

「自学習の国語室」（jigakushu.blogspot.com）を GitHub Pages へ移管する際の持ち方の見本。
見本として「ごんぎつね」（新美南吉）の本文＋学習サポートの1組だけを変換してある。

## 構成

```
index.html                  トップ（見本では1作品のみ掲載）
works/gongitsune.html       作品本文
support/gongitsune.html     学習サポート
assets/css/site.css         サイト共通の枠（ヘッダー・フッター、Bloggerテーマ分の補い）
assets/css/works.css        作品本文用CSS（全作品共通）
assets/css/support.css      サポート用CSS（全サポート共通）
assets/img/gongitsune.jpg   本文画像（googleusercontent からローカル化）
```

## 変換ルール（Blogger記事 → 静的ページ）

1. 記事本文の断片（`<div class="book-container">…`）はそのまま使う。表示ページ全体のコピーはしない。
2. 先頭の `<style>` は取り除き、共通CSSファイルへ切り出す。
   - 本文用（4,297字）とサポート用（4,572字）は **統合しない**。`:root` の色テーマが違い
     （本文=緑 #1a4a20 系／サポート=赤 #8b1e3f 系）、混ぜると色が壊れるため2ファイルに分ける。
3. 内部リンク（blogspot宛て）は相対パスへ書き換える。本文→サポート、サポート→本文（元の
   `history.back()` リンクは本文ページへの固定リンクに置換）。
4. 画像は googleusercontent からダウンロードして `assets/img/` に置き、リンクも差し替える。
5. `<a name='more'>` など Blogger 専用の断片は除去。テーマ側にしかなかった `.separator` の
   最低限の定義は site.css で補う。
6. 各ページに共通ヘッダー／フッターと `<head>`（charset・viewport・title）を付ける。

## 検証済み（2026-11-15）

- blogspot / googleusercontent への参照は残っていない（外部リンクの青空朗読などは除く）
- jsdom で computed style を確認：本文のスライド見出し背景 #1a4a20、サポート #8b1e3f と
  Blogger 版のテーマ色と一致。本文スライド6枚・ふりがな（ruby）102箇所を保持
- ローカルサーバーで全ファイル 200 応答を確認

## 本移管で残る作業

- 残り全作品・サポート・固定ページへの同じ変換（スラッグ対応表を作って一括処理）
- トップの学年別一覧を現行「国語室」ページから生成
- 算数ページ（計算ドリル・思考力脳トレ）は内蔵 JavaScript を assets/js へ分離する必要あり
- 画像は合計38枚（unique）。この見本の画像は 2.5MB と大きいので、本移管では幅800px程度への
  縮小も検討する
