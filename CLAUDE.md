# video-compressor（動画圧縮ツール）

ブラウザ完結の動画圧縮ツール。動画を指定サイズ（8/10/25/50MB・カスタム）以下に圧縮する。
ffmpeg.wasm で**ブラウザ内処理**＝サーバーに動画を送らない。

## 構成・技術
- **`index.html` 1枚で完結**（静的）。
- 圧縮エンジン: `@ffmpeg/ffmpeg@0.12.10` + `@ffmpeg/util@0.12.1`（UMD・SRI 付き）。
  コア wasm は `@ffmpeg/core@0.12.6` を unpkg から `toBlobURL` で実行時取得。
  - シングルスレッド版コアを使用＝**SharedArrayBuffer 不要 → COOP/COEP ヘッダ不要**（CF Pages 無設定で動く）。
  - コア wasm は約 30MB → CF Pages の 25MB/ファイル制限に引っかかるため**自前ホストせず CDN ロード**。
- ビットレート算出: `目標バイト × 8 / 長さ秒` から音声 128k を引き、安全マージン 0.92。
  - 単一パス + 超過時に実サイズから補正して 1 回だけ再エンコード（2-pass より速く目標に収束）。
  - 目標が小さい場合は bits/pixel/frame 判定で解像度を自動ダウンスケール。

## デプロイ
- `main` に push → GitHub Actions（`deploy.yml`）で Cloudflare Pages へ自動デプロイ。
- URL: `https://video-compressor-580.pages.dev/`（`video-compressor.pages.dev` はグローバルに別人が取得済みのため `-580` が付与された）
- リポジトリは **public**（`uniboo-apps` 組織シークレット `CLOUDFLARE_API_TOKEN` を使用）。
- コミットメッセージは **ASCII（英数字）** で書く（CF Pages の非 ASCII デプロイ失敗を回避）。
- コード変更後は確認なしに `git commit & push`（自動デプロイ）。
- push 時に `notion-commit.yml` が Notion コミットDB へ記録し、AI DB「動画圧縮ツール」行（page id `3884b6f3-2895-81d1-a9e1-d69d69bf3fed`）に紐づく。
  - 必要な GitHub 設定: variable `NOTION_COMMITS_DB_ID` / `NOTION_APP_PAGE_ID`、secret `NOTION_TOKEN`（org 自動付与されなかったため repo secret に登録済み）。

## ルール
- **public なので秘密をコードに置かない**（動画は全てブラウザ内処理・送信なし）。
- CDN ライブラリはバージョン固定＋SRI（コア wasm は toBlobURL のため SRI 不可・バージョン固定で担保）。

## 高速化（2026-06）
- **マルチスレッドコア**（`@ffmpeg/core-mt@0.12.6`）を `crossOriginIsolated` 時に使用 → CPU 全コアでエンコード。
  - `_headers` で COOP=`same-origin` / COEP=`require-corp` を付与し cross-origin isolation を有効化（CF Pages 専用・ローカル http.server では効かないので ST にフォールバックする）。
  - 非対応環境（isolation 無効、MT ロード失敗）は自動でシングルスレッド版にフォールバック（`ensureFFmpeg` の try/catch）。
  - unpkg からの core 取得は cors fetch（ACAO:*）なので require-corp 下でも通る。
- **プリセットを圧縮率で自動選択**: 軽圧縮（目標/元 ≥ 0.5、例 700→500MB）は `ultrafast`、中間 `superfast`、強圧縮 `veryfast`。ABR(-b:v) なのでプリセットを速くしても目標サイズは維持される。
- **等倍時は `-vf scale` を省略**（ダウンスケール不要なら scaler を通さない）。
- エンコード中は経過時間を表示。

## 既知の制約（ffmpeg.wasm）
- ブラウザ内 wasm のため**ヒープ上限 ~2GB**（ST は 32bit。MT でも巨大ファイルは注意）。4K や長尺はメモリ不足で失敗しうる。
- iOS Safari はメモリ制約が特に厳しい。長い・高解像度の動画は PC 推奨。
- ネイティブ ffmpeg（PC ローカル）なら同じ処理が桁違いに速い。巨大・常用するなら PC 処理が本筋。
