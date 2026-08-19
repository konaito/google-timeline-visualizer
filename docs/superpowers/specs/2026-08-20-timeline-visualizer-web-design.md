# Timeline Visualizer Web 版 設計書

- 日付: 2026-08-20
- 対象リポジトリ: `konaito/google-timeline-visualizer`（`mahlernim/google-timeline-visualizer` の fork）
- ブランチ: `feat/web-version`
- 配置: fork 内の `web/` サブディレクトリ（upstream の更新を取り込む際に衝突させないため）

## 1. 目的と成功条件

Android アプリ限定だった Timeline Visualizer を、ブラウザだけで完結する Web アプリとして提供する。

成功条件:

1. ユーザーが `Timeline.json` を選び、期間を指定し、アニメーションをプレビューし、MP4 をダウンロードできる
2. Timeline データが端末外に一切出ない（サーバーへのアップロードなし、テレメトリなし）
3. PC ブラウザと**スマートフォンのブラウザ（iOS Safari 16.4+ / Chrome Android）の両方**で成立する
4. 出力される動画が、既存 Android 版と同じ見た目・同じカメラ挙動になる

## 2. スコープ

### MVP に入れる

- Timeline ファイルの読み込み（`Timeline.json` および Takeout 形式）
- 外れ値フィルタ（Android 版 `LocationOutlierFilter` と同等、デフォルト設定固定）
- 期間選択（年月レンジ、および正確な日付指定）
- タイトル入力と動画長の指定
- アニメーションのプレビュー
- MP4 の生成とダウンロード
- 日本語 / 英語のみ

### MVP に入れない（YAGNI）

- マイビデオ一覧（IndexedDB への動画保存）
- カメラ設定 UI（fixed / steady / dynamic の切り替え）、長距離圧縮の切り替え、画質選択
  - **Android 版の既定値をそのまま内部固定する**（`CameraSettings.DEFAULT` を実測して確認）:
    カメラ `STEADY`、長距離圧縮 `BALANCED`（指数 0.85）、画質 `STANDARD`（480×480 / 2.5 Mbps）、24 fps
- 1080×1080 の旅程サマリ画像
- 9 言語対応
- 前回ファイルの記憶（Android 版の `TimelineSourceStore` 相当）
- WebM フォールバック（`VideoEncoder` 非対応環境）

## 3. 検証済みの前提

すべて本設計時に一次ソースで実測した。推測ではない。

| 項目 | 結果 | 確認方法 |
|---|---|---|
| `VideoEncoder` | Chrome 94+ / Firefox 130+ / Safari 16.4+（iOS も同じ）/ **Firefox Android 未対応** | MDN browser-compat-data `api/VideoEncoder.json` |
| `VideoFrame`（canvas から作る側） | Chrome 94+ / Firefox 130+ / Safari 16.4+（iOS も同じ） | MDN BCD `api/VideoFrame.json` |
| `OffscreenCanvas` | Chrome 69+ / Firefox 105+ / Safari 16.4+（iOS も同じ） | MDN BCD `api/OffscreenCanvas.json` |
| `Blob.stream()` | Chrome 76+ / Firefox 69+ / Safari 14.1+ | MDN BCD `api/Blob.json` |
| `CanvasRenderingContext2D.roundRect` | Chrome 99+ / Firefox 112+ / Safari 16+ | MDN BCD `api/CanvasRenderingContext2D.json` |
| 地図タイル (CartoDB) の CORS | `access-control-allow-origin: *` が返る。canvas は汚染されない | `curl -I` で実測 |
| MP4 muxer | `mp4-muxer` は**非推奨**。後継は同一作者の [Mediabunny](https://github.com/Vanilagy/mediabunny)（ゼロ依存 TypeScript、v1.55.1 / 2026-08-18） | GitHub / MDN WebCodecs ページ |
| TanStack Start の静的配信 | `spa: { enabled: true }` で `/_shell.html` を生成。公式いわく "A CDN that can serve static assets is all you need." | [公式 SPA mode ガイド](https://tanstack.com/start/latest/docs/framework/react/guide/spa-mode) |

**足切りラインは Safari 16.4**。`VideoEncoder` / `VideoFrame` / `OffscreenCanvas` の 3 つが同じ 16.4 で揃うので、対応可否の判定が 1 本化できる。

### 未確認（実装時に確かめる）

- `createImageBitmap` の Worker 内対応。BCD から引けなかったため、**実機で確認するまで対応済みと書かない**
- 具体的なコーデック（H.264 のプロファイル/レベル）が通るかは端末依存。BCD では判定不能なので、実行時に `VideoEncoder.isConfigSupported()` で機能検出する
- iOS Safari のタブあたりメモリ上限。実機で大きい `Timeline.json` を食わせて測る

## 4. アーキテクチャ

```
┌─────────────────── main thread ───────────────────┐
│ UI (TanStack Start, SPA mode, 単一ルート)          │
│  ├ ファイル選択 → parse.worker へ File を postMessage │
│  ├ 期間選択 UI                                     │
│  └ プレビュー: <canvas> + requestAnimationFrame     │
│       └── core/render を直接呼ぶ                    │
└────────────────────────────────────────────────────┘
        │ File / TypedArray(transfer)      │ 描画命令は共有コード
        ▼                                  ▼
┌── parse.worker ──┐            ┌── export.worker ──────────┐
│ ストリーミング     │            │ OffscreenCanvas に描画     │
│ JSON パース       │            │ → VideoFrame               │
│ → SoA TypedArray  │            │ → VideoEncoder             │
│ → transfer で返す │            │ → Mediabunny で MP4 に mux │
└───────────────────┘            │ → Blob を返す              │
                                 └────────────────────────────┘
```

**プレビューはメインスレッド、書き出しは Worker** のハイブリッド構成。理由:

- プレビューを Worker に置くと、スクラブ操作のたびに postMessage の往復が挟まって応答性が落ちる
- 書き出しはメインスレッドに置くと数十秒 UI が固まる。スマホでは即座にタブが殺される

### レイヤ境界

`core/` は DOM にも Worker API にも依存しない純粋な TypeScript。ここが**プレビューと書き出しの共有単位**であり、テスト可能な単位でもある。

「プレビューでは正しいのに動画は違う」という事故は、描画コードを 2 本持った瞬間に必ず起きる。`core/render/painter.ts` は `CanvasRenderingContext2D | OffscreenCanvasRenderingContext2D` を引数に取り、どちらでも同じ絵を描く 1 本のコードにする。

## 5. データ表現

Android 版は v2.1.1 / v2.1.2 の 2 回にわたって「大きい Timeline を読むとメモリ不足で落ちる」を修正しており、対策の方向はオブジェクト保持からプリミティブ配列 + 遅延ビューへの置き換えだった（`TimelineModels.kt` の `points: List<GeoPoint>` + `cumulativeDistanceKm: DoubleArray` + `JourneyRenderPath`）。

同じ罠がブラウザにもある。JS のオブジェクト配列は 1 要素あたり数十〜百バイト超のオーバーヘッドを持つため、100 万点で数百 MB に達しうる。

したがって位置データは **SoA（Structure of Arrays）の TypedArray** で持つ:

```ts
interface TimelinePoints {
  readonly lat: Float64Array          // 8 bytes
  readonly lon: Float64Array          // 8 bytes
  readonly timeMs: Float64Array       // 8 bytes（Date ではなく epoch ミリ秒）
  readonly cumulativeKm: Float64Array // 8 bytes
  readonly length: number
}
```

1 点あたり 32 バイト。100 万点で約 32 MB に収まる。

制約:

- **生の JSON 文字列を保持しない**。`File.stream()` から読んだチャンクを逐次パースし、読み終えたチャンクは捨てる
- `GeoPoint` 相当のオブジェクトは、1 点を指す一時的なビューとしてのみ生成する（配列としては絶対に持たない）
- Worker からメインスレッドへは `ArrayBuffer` を transferable で渡す（コピーを作らない）

## 6. モジュール構成

```
web/
├ src/
│  ├ core/                     … DOM 非依存。vitest でテストする対象
│  │  ├ timeline/
│  │  │  ├ parser.ts           … ReadableStream → SoA。Android の TimelineParser 相当
│  │  │  ├ outlier.ts          … LocationOutlierFilter 相当
│  │  │  └ period.ts           … TimelinePeriod 相当（期間の切り出し）
│  │  ├ journey/
│  │  │  ├ mercator.ts         … WebMercator 相当
│  │  │  ├ journey.ts          … Journey 相当（累積距離、legs、位置補間）
│  │  │  └ timing.ts           … JourneyTiming 相当（長距離圧縮のエルミート補間）
│  │  ├ render/
│  │  │  ├ camera.ts           … ビューポート計算とカメラトラック（TimelinePainter の前半 400 行相当）
│  │  │  ├ painter.ts          … 実際の描画。Canvas2D の型を引数で受ける
│  │  │  └ animation.ts        … TimelineAnimation 相当（アウトロ 1.5 秒）
│  │  └ tiles/
│  │     └ source.ts           … タイル URL 生成、Cache API、メモリ LRU
│  ├ workers/
│  │  ├ parse.worker.ts
│  │  └ export.worker.ts
│  ├ features/                 … UI（React）
│  │  ├ FilePicker.tsx
│  │  ├ PeriodPicker.tsx
│  │  ├ Preview.tsx
│  │  └ ExportPanel.tsx
│  └ routes/
│     └ index.tsx              … 公開ルートはここ 1 本だけ
└ tests/                       … vitest
```

各モジュールは「何をするか / どう使うか / 何に依存するか」が単独で答えられる粒度に保つ。`core/render/painter.ts` は Android 版で 823 行あるが、カメラ計算（`camera.ts`）を分離して 400 行程度に抑える。

## 7. 描画とエンコードのパイプライン

### プレビュー

1. `requestAnimationFrame` で経過時間を取る
2. `animation.frameAtElapsedSeconds()` で `journeyProgress` / `outroProgress` を得る
3. `camera.viewportAt()` でビューポートを計算
4. 必要タイルを `tiles/source.ts` から取得（未取得のものは非同期でロードし、届いたフレームから反映）
5. `painter.draw(ctx, ...)`

プレビューはタイルの事前取得をしない。届いた分だけ描いて、遅れて出てくる。

### 書き出し（export.worker）

Android 版 `Mp4Exporter` と同じ 3 フェーズ構成にする:

1. **PREPARING_MAP**: 動画長からサンプル数を決め、全フレームで必要になるタイル ID を先に洗い出してまとめて取得する。ここで取り切らないと、エンコード中にネットワーク待ちが挟まって時間が読めなくなる
2. **CREATING_VIDEO**: 24 fps で 1 フレームずつ OffscreenCanvas に描き、`new VideoFrame(canvas, { timestamp })` → `encoder.encode()`。**`frame.close()` を必ず呼ぶ**（呼ばないと GPU メモリが即死する）
3. **FINISHING_VIDEO**: `encoder.flush()` → Mediabunny で MP4 を組み立て → `Blob` を返す

### バックプレッシャ（必須）

`VideoEncoder` は非同期でキューを持つ。描画ループが encode を投げ続けると、エンコードが追いつかずキューが際限なく伸びてメモリを食い潰す。**スマホではこれが即クラッシュになる。**

```ts
while (encoder.encodeQueueSize > 4) {
  await new Promise(r => setTimeout(r, 0))  // dequeue イベントでも可
}
```

Android 版が 2 回連続でメモリ起因のクラッシュを出している以上、ここは「あとで気づいたら直す」ではなく最初から入れる。

### 進捗とキャンセル

- 進捗は `postMessage` でフェーズと割合を返す。Android 版と同じく残り時間の推定も出す
- キャンセルは `AbortController` の signal を worker に渡し、フレームループの先頭で確認する

## 8. エラー処理と機能検出

起動時に 1 度だけ判定し、非対応なら書き出しボタンを無効にして理由を出す:

```ts
const canExport =
  typeof VideoEncoder !== 'undefined' &&
  typeof OffscreenCanvas !== 'undefined' &&
  (await VideoEncoder.isConfigSupported({ codec: 'avc1.42001f', width: 480, height: 480, bitrate: 2_500_000 })).supported
```

- **コーデック**: `avc1.42001f` が通らず制約ベースラインの `avc1.42E01F` なら通る端末がある。非対応と判定する前に両方試す
- **Firefox Android**: `VideoEncoder` 未対応（実測済み）。プレビューまでは動くので、書き出し時に「このブラウザは動画の書き出しに対応していない。Chrome か Safari で開いてほしい」と明示する。黙って落とさない
- **パース失敗**: Android 版 `TimelineParseException` と同じ理由コード（形式不明 / 位置情報なし / 期間内にデータなし）を維持し、それぞれ違う文言を出す
- **タイル取得失敗**: 1 枚単位で握り潰し、背景グラデーションのまま進む（Android 版と同じ挙動）
- **メモリ不足**: 点数が閾値を超えたら、期間を狭めるよう案内する。**黙って間引いて「できました」とは言わない**

## 9. テスト戦略

### ユニットテスト（vitest）

Android 版には既にテストが揃っている。これを**仕様書として移植する**:

| 移植元 | 移植先 |
|---|---|
| `WebMercatorTest.kt` | `tests/mercator.test.ts` |
| `JourneyTest.kt` | `tests/journey.test.ts` |
| `JourneyTimingTest.kt` | `tests/timing.test.ts` |
| `TimelineParserTest.kt` | `tests/parser.test.ts` |
| `LocationOutlierFilterTest.kt` | `tests/outlier.test.ts` |
| `TimelineAnimationTest.kt` | `tests/animation.test.ts` |

`test-fixtures/*.json`（takeout-sample / seoul-bohol-sample / outlier-sample / android-ios-sample）がリポジトリに既にあるので、そのまま入力に使う。**TDD で進める**: 先にテストを移植して落として、それから実装する。

### 数値の一致確認

同じ fixture に対して Kotlin 実装と TypeScript 実装が同じ値を返すことを確認する。累積距離、ビューポート、`JourneyTiming.distanceAt()` の出力を比較する。ピクセル単位の画像比較はアンチエイリアスの差で必ず落ちるのでやらない。

### 実機確認（省略しない）

`grep が通った` `テストが緑` は「意図どおり動く」の証明ではない。最後に必ず:

1. 実際の `Timeline.json`（数百 MB クラス）を PC Chrome に食わせて MP4 を書き出す
2. **同じことを iPhone の Safari でやる**
3. 出た MP4 を再生して、Android 版の出力と並べて見比べる

## 10. デプロイ

- TanStack Start を `spa: { enabled: true }` でビルド
- 公開ルートは `/` だけにする。画面遷移は URL に載せずコンポーネント state で持つ。こうすると SPA フォールバック（`404.html` のコピーや `_redirects`）が一切不要になり、GitHub Pages にそのまま置ける
- GitHub Actions で `web/` をビルドして Pages に配置する

## 11. リスク

| リスク | 影響 | 対策 |
|---|---|---|
| iOS Safari のメモリ上限が想定より低い | スマホで書き出せない | 早い段階で実機測定する。実装の最初のマイルストーンに置く |
| `createImageBitmap` が Worker で使えない | タイル描画ができない | メインスレッドで ImageBitmap 化して transfer する経路にフォールバック |
| 端末が H.264 エンコードを持たない | MP4 が出ない | `isConfigSupported()` で検出し、理由を明示する |
| Android 版と絵が一致しない | 「同じものが作れる」が嘘になる | 数値一致テスト + 目視での並べ比較 |

## 12. 非スコープの再確認

サーバーは一切使わない。サーバーサイドレンダリング、サーバー関数、動画の受け渡し、解析、いずれも行わない。この制約は「プライバシー」が本アプリの中核価値であることに由来しており、後から緩めない。
