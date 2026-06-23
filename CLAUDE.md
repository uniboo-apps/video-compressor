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

## ルール
- **public なので秘密をコードに置かない**（動画は全てブラウザ内処理・送信なし）。
- CDN ライブラリはバージョン固定＋SRI（コア wasm は toBlobURL のため SRI 不可・バージョン固定で担保）。

## 既知の制約（ffmpeg.wasm）
- ブラウザ内 32bit wasm のため**ヒープ上限 ~2GB**。4K や長尺はメモリ不足で失敗しうる。
- iOS Safari はメモリ制約が特に厳しい。長い・高解像度の動画は PC 推奨。
