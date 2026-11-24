# Hueday 詳しい地図モジュール

建物の形・すべての道路・鉄道・駅・お店の名前まで入った地図を、**無料**で表示するためのコードです。

## 構成

| ファイル | 役割 |
|---|---|
| `index.html` | 地図の本体。上の `CONFIG` だけ書き換えれば動く |
| `vendor/maplibre-gl.js`, `vendor/maplibre-gl.css` | 地図表示ライブラリ MapLibre GL JS 5.24.1（BSD-3） |
| `vendor/pmtiles.js` | 1ファイル形式の地図（PMTiles）を読むライブラリ 4.5.0（BSD-3） |
| `vendor/basemaps.js` | Protomaps の地図デザイン 5.7.2（BSD-3） |
| `r2-cors.json` | Cloudflare R2 に設定するCORS（ほかのサイトからの読み込み許可） |

## 仕組み

```
GitHub Pages（index.html）
   │ 地図の必要な部分だけを Range リクエストで取得
   ▼
Cloudflare R2（japan.pmtiles：OpenStreetMap の日本分）
```

- サーバー（プログラムを動かすもの）は不要。ファイルを置くだけ
- APIキー不要、従量課金なし（R2 は保存10GB/月まで無料、ダウンロードの転送料は無料）

## 手順

### 1. 日本の地図ファイルを作る（パソコン）
- ツール：https://github.com/protomaps/go-pmtiles/releases から自分のOS用をダウンロード
- 最新の地図データの日付：https://maps.protomaps.com/builds/
- 実行（日付は最新のものに置き換える）：
  ```
  pmtiles extract https://build.protomaps.com/20261120.pmtiles japan.pmtiles --bbox=122.9,24.0,146.0,45.6
  ```
  サイズを小さくしたい場合は `--maxzoom=14` を付ける

### 2. Cloudflare R2 に置く
- 場所：https://dash.cloudflare.com/ → 左メニュー「R2」→「バケットを作成」（例：`hueday-map`）
- 「設定」→「公開アクセス」→ r2.dev のURLを許可 → `https://pub-xxxx.r2.dev` を控える
- 「設定」→「CORSポリシー」→ `r2-cors.json` の中身を貼る（`YOUR-GITHUB-NAME` を自分のGitHub名に）
- 300MB を超えるファイルは管理画面から直接アップロードできないので rclone を使う
  - 使い方：https://developers.cloudflare.com/r2/examples/rclone/
  - `rclone copy japan.pmtiles r2:hueday-map/ -P`

### 3. GitHub Pages で公開する
- 場所：https://github.com/new でリポジトリ `hueday-map` を作る →「Add file」→「Upload files」でこのフォルダの中身をすべてアップロード
- 「Settings」→「Pages」→ Branch を `main` にして保存
- `https://YOUR-GITHUB-NAME.github.io/hueday-map/` で開ける

### 4. 設定を入れる
`index.html` の `CONFIG.PMTILES_URL` に `https://pub-xxxx.r2.dev/japan.pmtiles` を入れる。

## 確認ポイント（コードレビュー用）

- `pmtiles.Protocol` を `maplibregl.addProtocol('pmtiles', protocol.tile)` で登録している
- 地図デザインは `basemaps.layers('protomaps', basemaps.namedFlavor('light'), { lang: 'ja' })` で生成
- 文字（フォント）とアイコンは Protomaps 公式の `basemaps-assets`（GitHub Pages）から読み込む
- 投稿ピンは `maplibregl.Marker`。ズーム16以上で写真の丸ピンに切り替え、境目をまたいだときだけ描き直す
- 回転はオフ（`dragRotate:false`、`disableRotation()`）
- 表記：`© OpenStreetMap` を地図右下に表示（ODbL の条件）

## 本体アプリへの組み込み（次の段階）

本体（Hueday）の「真ん中のマップ」を、この地図に置き換える予定。
投稿データは `HuedayMap.setPosts(posts)` で渡す（`{id, lat, lon, cat, place, img, date}`）。
マイマップ（都道府県・市町村の色塗り）は、今の白地図をそのまま使う。
