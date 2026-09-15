# LilyPond WASM / Safari対応の保存記録

2026-09-16。Tetorica VGM Analyzerで試したLilyPondプレビューの再現用資料です。
Analyzer本体は軽量なMusicXML表示に移行しましたが、LilyPondの組版結果と
Safari対応を再利用できるように、このforkに保存しています。

## cloneして使う

ビルド済みWASMと必要なruntime packを `wasm/web/vendor/lilypond/` に同梱しています。
このリポジトリをcloneしたら、ルートで以下を実行してください。NixやDockerは不要です。

```sh
python3 -m http.server 38089 --directory wasm/web
```

http://localhost:38089/ を開いて `.ly` を貼り、Previewを押します。
同梱エンジンはソースビルド後にBinaryen 132で後処理した版です。
ファイルのハッシュは `wasm/web/vendor/lilypond/DISTRIBUTION_SHA256SUMS` に記録しています。

## このforkとWASM移植の関係

このリポジトリはLilyPond本体のソースです。記録追加時のHEADは
`a108be336eb879650aaf760539eea89c0c817642`。
保存したプレビューのWASMは既存バイナリの後処理で作りました。
その後、移植側が固定したLilyPondのC++ソースからもWASMをビルドできました。
このforkのルートにあるC++チェックアウトとはリビジョンが異なります。
使った移植は https://github.com/hlolli/lilypond-wasm です。

- npm: `@hlolli/lilypond-wasm@0.1.0-alpha.1`
- LilyPond 2.27.2 / Guile 3.0.11 / WASI Preview 1
- `web/vendor/lilypond/upstream-source.tar.gz`: 移植側のeditor/buildスナップショット
  (`bc1f9483776a2415c12cb43886497d6f0f6b3f22`)。完全なエンジンソースではありません。
  このスナップショットのRELEASING.mdはalpha.4を対象にしており、alpha.1と混同しないこと。
- alpha.1の完全な対応ソースとSHA-256: [SOURCE.md](web/vendor/lilypond/SOURCE.md)
- コピー元: hello_ymfmの `docs/vgm_analyzer/vendor/lilypond` と専用スクリプト。
  アセット保存時のハッシュ: [SNAPSHOT_SHA256SUMS](web/vendor/lilypond/SNAPSHOT_SHA256SUMS)

## Safari対応で変更したこと

C++ソースの変更ではなく、既存WASMにBinaryen **132**で後処理を施しました。

```sh
wasm-opt upstream.wasm --coalesce-locals --vacuum -o lilypond.wasm
```

失敗時のスタックに現れた関数index 6130は、ローカル変数スロットが420から21に減りました。
生存期間が重ならないローカルを共有して呼び出しスタックの負担を減らす処理です。
linear memory内のC/C++用スタック8MiBは変更していません。
ブラウザエンジンの呼び出しスタックとlinear memoryのスタックは別です。

`--all-features` は付けません。試行時に対応環境で読めない出力になったため、
入力WASMのtarget_featuresを維持しました。

| | bytes | SHA-256 |
|---|---:|---|
| upstream | 61,235,199 | `e957ee1839f0102d9fd543821019e7ecb5fac71584dcc7a92983dd7525499961` |
| 最適化後 | 60,150,063 | `1d4d5d576ac40a720f2a1224061e5d068b5e805c4d2554f5980d0e8039384854` |

関連するworker側の修正も保存しています。

- `runtime_pack.js`: gzipのままの応答と、ホストが既に展開した応答の両方を受け付け、長さを検証。
  itch.ioでは既に展開された応答を観測しました。
- `wasi_stdio.js`: fd 0–2へのfd_tellは元と同じNOTCAPABLE (76)を返し、不要な例外ログを抑制。
- `lilypond.worker.js`: 上記helperの使用、失敗時のError.stack送信、最適化後WASMのURL。
- `lilypond_preview.js`: 1回の描画ごとにworkerを起動。終了・キャンセル・180秒タイムアウトでterminate。
  SVGはBlob URLのimgとして表示し、再表示・終了でURLを解放します。

## WASMとruntimeを再生成する（確認済みの経路）

必要: Python 3、tar、curl、Binaryen 132、Node（検証時）。以下はこのforkのルートで実行します。
Binaryenは https://github.com/WebAssembly/binaryen/releases/tag/version_132 の
実行環境に合った配布物を使ってください。macOS arm64で検証しました。

```sh
mkdir -p wasm/build
curl -fsSL https://registry.npmjs.org/@hlolli/lilypond-wasm/-/lilypond-wasm-0.1.0-alpha.1.tgz -o wasm/build/upstream.tgz
tar -xzf wasm/build/upstream.tgz -C wasm/build
python3 wasm/scripts/pack_lilypond_runtime.py wasm/build/package wasm/web/vendor/lilypond
python3 wasm/scripts/optimize_lilypond_wasm.py wasm/build/package/dist/lilypond.wasm wasm/web/vendor/lilypond/dist/lilypond.wasm --wasm-opt /path/to/binaryen-version_132/bin/wasm-opt
```

最後のスクリプトは入力WASMのSHA-256とBinaryenのバージョンを確認します。
出力ハッシュが上表と一致することを確認してください。
pack処理は543個のruntimeファイルを束ねます（展開後57,779,421 bytes）。
WASM単体では動かず、Scheme、フォント、設定、worker、JS helperも必要です。
作業用 `build/` はGit管理から除外しています。プレビューに必要な
`dist/lilypond.wasm` と `runtime/runtime-files.json`、`runtime/runtime-files.pack.gz`
はGit管理対象です。上の再生成コマンドは旧npm版の再現用なので、実行すると同梱の
ソースビルド版エンジンを旧版で上書きします。

## ソースからエンジンをビルドする（確認済み）

Docker + Nixでalpha.1の固定ソースから `.#lilypond` のビルドに成功しました。
実行コマンド、入力リビジョン、生成物と描画検証の結果は
[SOURCE_BUILD.md](SOURCE_BUILD.md) に記録しています。
フォント・Schemeなどのruntimeは保存済みのものを再利用しています。
以下のnpm配布物全体の生成は今回の検証対象ではありません。

[SOURCE.md](web/vendor/lilypond/SOURCE.md) に記載したalpha.1の完全ソースを取得し、
SHA-256を確認して展開し、その中のビルドプロジェクトの指示に従ってください。
移植側はNix flakeで依存ライブラリ、WASI/LLVM runtime、LilyPondを組み立てます。
保存スナップショットで確認できるターゲット例は以下です（そのflakeのルートで実行）。

```sh
nix build .#lilypond-npm-tarball --no-update-lock-file
nix build .#lilypond-source-bundle --no-update-lock-file
nix build .#checks.aarch64-darwin.lilypond-npm-smoke --no-update-lock-file --print-build-logs
```

これらをこのLilyPond本体forkのルートで実行することはできません。
本体forkの変更を組み込むには、移植側のLilyPond source指定とパッチを合わせる作業が別途必要です。
再ビルド後のバイナリはalpha.1とハッシュが変わるので、固定入力用optimizerをそのまま使わず、
変更内容を確認して後処理と検証を適用します。

## 簡単な使い方・検証

```sh
node --test wasm/web/lilypond_preview.test.mjs
node wasm/scripts/check_lilypond_wasm.mjs
node --liftoff-only wasm/scripts/check_lilypond_wasm.mjs
# 自分のスコアを検証する場合
node --liftoff-only wasm/scripts/check_lilypond_wasm.mjs /path/to/score.ly
python3 -m http.server 38089 --directory wasm/web
```

http://localhost:38089/ を開き、テキスト欄に `.ly` を貼ってPreviewを押します。
SVGを表示し、各ページを保存できます。通信は同じサーバーのruntime取得に使い、スコアは送信しません。
通常のHTTPサーバーで試せます。`file://` で直接開かないでください。

これまでの確認: 通常Nodeで4音の譜面1ページ、`--liftoff-only`でも1ページ、
Jungle全9段で8ページ。修正後に利用者がSafari実機で生成成功を確認しました。
Safariのバージョンは記録していないため、全バージョン対応の保証ではありません。
Chrome DevToolsを開くと実行方式が変わる場合があり、スタック限界の再現条件になります。
Node検証はworkerとWASM/runtimeの確認であり、Safari UIや配信ヘッダの検証を代替しません。

## ライセンス

コピーしたGPL対象コードは元のライセンスを維持します。依存物の通知は
`web/vendor/lilypond/` のLICENSE、COPYING、THIRD_PARTY_NOTICES.md、licenses/、editor-licenses/を参照。
バイナリ配布時の対応ソースについてはSOURCE.mdも引き継いでください。
ここに保存したeditor/buildスナップショットだけを完全な対応ソースとして扱わないでください。
