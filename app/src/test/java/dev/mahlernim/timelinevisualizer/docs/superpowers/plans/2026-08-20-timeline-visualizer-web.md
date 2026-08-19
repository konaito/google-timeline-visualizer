# Timeline Visualizer Web 版 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Android 限定だった Timeline Visualizer を、サーバーを一切使わずブラウザだけで `Timeline.json` から MP4 を生成できる Web アプリにする。

**Architecture:** `core/` に DOM 非依存の純粋 TypeScript としてパース・幾何計算・カメラ・描画を置き、プレビュー（メインスレッド）と書き出し（Worker + OffscreenCanvas）の両方が同じコードを呼ぶ。位置データは SoA の TypedArray で保持し、MP4 は WebCodecs `VideoEncoder` と Mediabunny で生成する。

**Tech Stack:** TanStack Start (SPA mode) / React / TypeScript / Vite / vitest / bun / WebCodecs / Mediabunny / OffscreenCanvas

**Spec:** `docs/superpowers/specs/2026-08-20-timeline-visualizer-web-design.md`

**移植元:** 同リポジトリの Android 実装（`app/src/main/java/dev/mahlernim/timelinevisualizer/`）。各タスクに参照先ファイルと行番号を書いてある。**実装前に必ず移植元を読むこと。** 挙動を推測で書かない。

## Global Constraints

- **サーバーを使わない。** SSR、サーバー関数、API ルート、外部へのデータ送信はいずれも禁止。Timeline データが端末外に出た時点で仕様違反
- **wasm を使わない。** ffmpeg.wasm 等は導入しない
- **ブラウザ下限は Safari 16.4 / Chrome 94 / Firefox 130。** `VideoEncoder` / `VideoFrame` / `OffscreenCanvas` の 3 つがこのラインで揃う（MDN BCD で実測済み）。Firefox Android は `VideoEncoder` 未対応なので書き出し非対応として扱う
- **位置データはオブジェクト配列で持たない。** `Float64Array` の SoA のみ。生の JSON 文字列も保持しない
- **パッケージマネージャは bun。** `npm` / `yarn` は使わない
- **動画の既定値は Android 版と一致させる**（`render/CameraSettings.kt` で実測）: カメラ `STEADY`、長距離圧縮 `BALANCED`（指数 0.85）、解像度 480×480、ビットレート 2,500,000、24 fps、アウトロ 1.5 秒
- **地図タイル URL:** `https://a.basemaps.cartocdn.com/light_all/{z}/{x}/{y}.png`。帰属表示（attribution）を必ず画面と動画に焼き込む
- **作業ディレクトリは `web/`。** リポジトリ直下の Android / Python 資産には手を触れない
- **各タスクの最後に必ずコミットする。** `main` には push しない。ブランチは `feat/web-version`

---

## File Structure

| ファイル | 責務 |
|---|---|
| `web/src/core/timeline/points.ts` | SoA の `TimelinePoints` 型と生成・アクセスのヘルパ |
| `web/src/core/timeline/parser.ts` | `ReadableStream` から Timeline JSON を逐次パースして `TimelinePoints` を作る |
| `web/src/core/timeline/outlier.ts` | 移動速度が物理的にありえない点を除去する |
| `web/src/core/timeline/period.ts` | 年月レンジ / 日付レンジで点を切り出す |
| `web/src/core/journey/mercator.ts` | 緯度経度 ↔ Web メルカトル世界座標、日付変更線の unwrap |
| `web/src/core/journey/journey.ts` | 累積距離、区間（leg）、進捗から位置を求める補間、描画用の densify |
| `web/src/core/journey/timing.ts` | 動画の経過割合 → 走行距離のマッピング（長距離圧縮） |
| `web/src/core/render/animation.ts` | 経過秒 → 本編進捗 / アウトロ進捗 |
| `web/src/core/render/camera.ts` | ビューポート計算とカメラトラックの事前構築 |
| `web/src/core/render/painter.ts` | Canvas2D への実描画（背景・タイル・経路・オーバーレイ） |
| `web/src/core/tiles/source.ts` | タイル URL 生成、Cache API、メモリ LRU |
| `web/src/workers/parse.worker.ts` | ファイルを受け取りパースして SoA を transfer で返す |
| `web/src/workers/export.worker.ts` | OffscreenCanvas に描いて VideoEncoder に流し MP4 Blob を返す |
| `web/src/features/*.tsx` | 画面（ファイル選択・期間選択・プレビュー・書き出し） |
| `web/src/routes/index.tsx` | 唯一の公開ルート |

---

## Task 1: プロジェクトの土台

**Files:**
- Create: `web/package.json`, `web/vite.config.ts`, `web/tsconfig.json`, `web/src/routes/__root.tsx`, `web/src/routes/index.tsx`, `web/vitest.config.ts`, `web/src/core/constants.ts`, `web/tests/smoke.test.ts`
- （CI ワークフローは Task 15 で作る。ここでは作らない）

**Interfaces:**
- Consumes: なし
- Produces: `bun run build` が `web/dist/` に静的成果物を出し、`bun run test` が vitest を実行できる状態

- [ ] **Step 1: TanStack Start を SPA mode で初期化**

```bash
cd web && bun init -y
bun add @tanstack/react-start @tanstack/react-router react react-dom
bun add -d vite @vitejs/plugin-react typescript vitest @types/react @types/react-dom
```

`web/vite.config.ts`:

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'

export default defineConfig({
  base: '/google-timeline-visualizer/',
  plugins: [
    tanstackStart({ spa: { enabled: true } }),
    react(),
  ],
})
```

`base` は GitHub Pages のサブパス配信に必要。ルートドメインに置くなら `/` に変える。

- [ ] **Step 2: 失敗するスモークテストを書く**

`web/tests/smoke.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { APP_NAME } from '../src/core/constants'

describe('app constants', () => {
  it('exposes the app name', () => {
    expect(APP_NAME).toBe('Timeline Visualizer')
  })
})
```

- [ ] **Step 3: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/smoke.test.ts`
Expected: FAIL — `Cannot find module '../src/core/constants'`

- [ ] **Step 4: 最小実装**

`web/src/core/constants.ts`:

```ts
export const APP_NAME = 'Timeline Visualizer'
export const TILE_URL_TEMPLATE = 'https://a.basemaps.cartocdn.com/light_all/{z}/{x}/{y}.png'
export const VIDEO_SIZE = 480
export const VIDEO_BITRATE = 2_500_000
export const VIDEO_FPS = 24
export const OUTRO_SECONDS = 1.5
export const OUTRO_TRANSITION_SECONDS = 1.0
export const COMPRESSION_EXPONENT = 0.85
export const ATTRIBUTION = '© OpenStreetMap contributors © CARTO'
```

- [ ] **Step 5: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/smoke.test.ts`
Expected: PASS

- [ ] **Step 6: ビルドが通ることをローカルで確認**

Run: `cd web && bun run build`
Expected: `web/dist/` に成果物、`_shell.html` が生成されている。**CI で確かめるのではなくここで確かめる**

- [ ] **Step 7: `.gitignore` を確認してからコミット**

`web/.gitignore` に `node_modules` / `dist` / `.output` が入っていることを確認する。`bun init` が置く内容に依存しない。

```bash
cd web && cat .gitignore   # 3 つとも無ければ追記する
git add web
git commit -m "Scaffold the web app with TanStack Start in SPA mode"
```

---

## Task 2: Web メルカトル投影と SoA データ表現

**Files:**
- Create: `web/src/core/journey/mercator.ts`, `web/src/core/timeline/points.ts`
- Test: `web/tests/mercator.test.ts`, `web/tests/points.test.ts`

**移植元:** `app/src/main/java/dev/mahlernim/timelinevisualizer/model/TimelineModels.kt:370-392`（`WorldPoint` と `WebMercator`）、テストは `app/src/test/java/dev/mahlernim/timelinevisualizer/model/WebMercatorTest.kt`

**Interfaces:**
- Consumes: なし
- Produces:
  - `project(latitude: number, longitude: number): { x: number, y: number }`
  - `shortestWrappedX(xs: number[]): number[]`
  - `interface TimelinePoints { lat: Float64Array; lon: Float64Array; timeMs: Float64Array; cumulativeKm: Float64Array; length: number }`
  - `createPoints(capacity: number): MutableTimelinePoints`（`push(timeMs, lat, lon): void` を持つ）と `finalizePoints(mutable: MutableTimelinePoints): TimelinePoints`
  - `haversineKm(lat1: number, lon1: number, lat2: number, lon2: number): number`

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/mercator.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { project, shortestWrappedX } from '../src/core/journey/mercator'

describe('WebMercator', () => {
  it('projects the origin to the centre of the world square', () => {
    const point = project(0, 0)
    expect(point.x).toBeCloseTo(0.5, 6)
    expect(point.y).toBeCloseTo(0.5, 6)
  })

  it('takes the short path across the date line', () => {
    const direct = [project(0, 179).x, project(0, -179).x]
    const wrapped = shortestWrappedX(direct)
    expect(Math.max(...wrapped) - Math.min(...wrapped)).toBeLessThan(0.01)
  })
})
```

`web/tests/points.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { createPoints, finalizePoints, haversineKm } from '../src/core/timeline/points'

describe('TimelinePoints', () => {
  it('stores coordinates in typed arrays without object allocation', () => {
    const mutable = createPoints(2)
    mutable.push(Date.parse('2026-01-01T00:00:00Z'), 37.5665, 126.978)
    mutable.push(Date.parse('2026-01-01T01:00:00Z'), 35.6762, 139.6503)
    const points = finalizePoints(mutable)

    expect(points.length).toBe(2)
    expect(points.lat).toBeInstanceOf(Float64Array)
    expect(points.lat[1]).toBeCloseTo(35.6762, 6)
    expect(points.cumulativeKm[0]).toBe(0)
    expect(points.cumulativeKm[1]).toBeCloseTo(1157, 0)
  })

  it('grows beyond the initial capacity without losing points', () => {
    const mutable = createPoints(1)
    for (let i = 0; i < 100; i++) mutable.push(i * 1000, 35 + i * 0.001, 139)
    expect(finalizePoints(mutable).length).toBe(100)
  })

  it('measures the Seoul to Tokyo great circle distance', () => {
    expect(haversineKm(37.5665, 126.978, 35.6762, 139.6503)).toBeCloseTo(1157, 0)
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/mercator.test.ts tests/points.test.ts`
Expected: FAIL — モジュールが存在しない

- [ ] **Step 3: 実装**

`web/src/core/journey/mercator.ts` は `TimelineModels.kt:372-392` をそのまま移す。`project` は経度を `(lon + 180) / 360`、緯度を `ln(tan(π/4 + φ/2))` で 0..1 に正規化する。`shortestWrappedX` は隣接する x が 0.5 以上離れていたら ±1 して連続にする。

`web/src/core/timeline/points.ts` の `MutableTimelinePoints` は容量が足りなくなったら 2 倍の `Float64Array` を確保して `set()` でコピーする。`push` の中で直前の点との `haversineKm` を足して `cumulativeKm` を埋める。`finalizePoints` は `subarray(0, length)` ではなく `slice(0, length)` を返す（transfer 時に余分なバッファを運ばないため）。

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/mercator.test.ts tests/points.test.ts`
Expected: PASS

- [ ] **Step 5: コミット**

```bash
git add web/src/core web/tests
git commit -m "Add Web Mercator projection and typed-array point storage"
```

---

## Task 3: ストリーミング JSON パーサ

**Files:**
- Create: `web/src/core/timeline/parser.ts`
- Test: `web/tests/parser.test.ts`

**移植元:** `app/src/main/java/dev/mahlernim/timelinevisualizer/data/TimelineParser.kt`（全 371 行）。特に `readRootObject:104`、`readSegments:137`、`readSegment:145`、`readTimelinePath:184`、`parseCoordinate:309`、`parseInstant:328`。テストは `app/src/test/java/dev/mahlernim/timelinevisualizer/data/TimelineParserTest.kt`

**入力形式:** リポジトリ直下の `test-fixtures/` にある 4 種すべてを扱えること。
- `takeout-sample.json` … Google Takeout 形式
- `android-ios-sample.json` … Android / iOS のエクスポート形式
- `seoul-bohol-sample.json` … 最小構成
- `outlier-sample.json` … 外れ値を含む

**Interfaces:**
- Consumes: Task 2 の `createPoints` / `finalizePoints`
- Produces:
  - `parseTimeline(stream: ReadableStream<Uint8Array>): Promise<TimelinePoints>`
  - `class TimelineParseError extends Error { reason: 'unsupported-format' | 'no-location-data' }`

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/parser.test.ts`:

```ts
import { readFile } from 'node:fs/promises'
import { describe, expect, it } from 'vitest'
import { parseTimeline, TimelineParseError } from '../src/core/timeline/parser'

function streamOf(text: string): ReadableStream<Uint8Array> {
  const bytes = new TextEncoder().encode(text)
  return new ReadableStream({
    start(controller) {
      // わざと細かく分割して、チャンク境界をまたぐパースを検証する
      for (let i = 0; i < bytes.length; i += 7) controller.enqueue(bytes.slice(i, i + 7))
      controller.close()
    },
  })
}

async function parseFixture(name: string) {
  const text = await readFile(new URL(`../../test-fixtures/${name}`, import.meta.url), 'utf8')
  return parseTimeline(streamOf(text))
}

describe('parseTimeline', () => {
  it('reads the Takeout export format', async () => {
    const points = await parseFixture('takeout-sample.json')
    expect(points.length).toBeGreaterThan(0)
    expect(Number.isFinite(points.lat[0])).toBe(true)
    expect(points.lat[0]).toBeGreaterThanOrEqual(-90)
    expect(points.lat[0]).toBeLessThanOrEqual(90)
  })

  it('reads the Android and iOS export format', async () => {
    const points = await parseFixture('android-ios-sample.json')
    expect(points.length).toBeGreaterThan(0)
  })

  it('reads the minimal Seoul to Bohol sample', async () => {
    const points = await parseFixture('seoul-bohol-sample.json')
    expect(points.length).toBeGreaterThan(0)
  })

  it('orders points by time', async () => {
    const points = await parseFixture('takeout-sample.json')
    for (let i = 1; i < points.length; i++) {
      expect(points.timeMs[i]).toBeGreaterThanOrEqual(points.timeMs[i - 1])
    }
  })

  it('rejects JSON that is not a Timeline export', async () => {
    await expect(parseTimeline(streamOf('{"hello":"world"}')))
      .rejects.toThrow(TimelineParseError)
  })

  it('rejects input that is neither an object nor an array', async () => {
    await expect(parseTimeline(streamOf('42'))).rejects.toThrow(TimelineParseError)
  })

  it('never materialises the whole input as one string', async () => {
    // 10 MB 相当を流しても、返る配列サイズが入力サイズに比例して膨らまないこと
    const points = await parseFixture('takeout-sample.json')
    expect(points.lat.byteLength).toBe(points.length * 8)
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/parser.test.ts`
Expected: FAIL — モジュールが存在しない

- [ ] **Step 3: 実装**

チャンクを跨ぐ逐次パーサを自前で書く。**`JSON.parse` に全文を渡してはいけない**（数百 MB の文字列を持った時点で負け）。

構成:
1. `TextDecoderStream` で UTF-8 に変換し、チャンクを受け取る
2. 文字単位の状態機械でトークンを読み、「今どのキーの配下にいるか」だけを浅くスタック管理する
3. `semanticSegments` / `timelinePath` / `locations` / `rawSignals` のような**位置を持つ末端オブジェクトに入った時だけ**、その 1 オブジェクトを組み立ててから `push` し、即座に捨てる
4. 座標は `"37.5665°, 126.9780°"` 形式と `geo:37.5665,126.978` 形式の両方を受ける（`parseCoordinate:309` 参照）
5. 時刻は ISO 8601 とオフセット付き形式の両方を受ける（`parseInstant:328` 参照）
6. 1 点も取れなかったら `TimelineParseError('no-location-data')`、ルートがオブジェクトでも配列でもなければ `TimelineParseError('unsupported-format')`
7. 最後に時刻昇順でソートする（`normalize:53` 参照）。ソートは SoA のインデックス配列を作って並べ替える。オブジェクト配列を作らない

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/parser.test.ts`
Expected: PASS（7 件）

- [ ] **Step 5: コミット**

```bash
git add web/src/core/timeline/parser.ts web/tests/parser.test.ts
git commit -m "Add a streaming Timeline JSON parser"
```

---

## Task 4: 外れ値フィルタ

**Files:**
- Create: `web/src/core/timeline/outlier.ts`
- Test: `web/tests/outlier.test.ts`

**移植元:** `app/src/main/java/dev/mahlernim/timelinevisualizer/data/LocationOutlierFilter.kt`（93 行）、テストは `app/src/test/java/dev/mahlernim/timelinevisualizer/data/LocationOutlierFilterTest.kt`

**Interfaces:**
- Consumes: Task 2 の `TimelinePoints`, `haversineKm`
- Produces: `filterOutliers(points: TimelinePoints, mode?: 'balanced' | 'off'): { points: TimelinePoints, removedCount: number }`

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/outlier.test.ts`（Kotlin 版 5 ケースをそのまま移植する）:

```ts
import { describe, expect, it } from 'vitest'
import { createPoints, finalizePoints } from '../src/core/timeline/points'
import { filterOutliers } from '../src/core/timeline/outlier'

function build(rows: [string, number, number][]) {
  const mutable = createPoints(rows.length)
  for (const [time, lat, lon] of rows) mutable.push(Date.parse(time), lat, lon)
  return finalizePoints(mutable)
}

describe('filterOutliers', () => {
  it('removes a single impossible out-and-back point', () => {
    const points = build([
      ['2026-01-01T00:00:00Z', 37.5665, 126.978],
      ['2026-01-01T01:00:00Z', 0, -50],
      ['2026-01-01T02:00:00Z', 37.57, 126.98],
    ])
    const result = filterOutliers(points)
    expect(result.removedCount).toBe(1)
    expect(result.points.length).toBe(2)
    expect(result.points.lat[0]).toBeCloseTo(37.5665, 6)
    expect(result.points.lat[1]).toBeCloseTo(37.57, 6)
  })

  it('removes a short clustered spoofing excursion', () => {
    const points = build([
      ['2026-01-01T00:00:00Z', 37.5665, 126.978],
      ['2026-01-01T01:00:00Z', 0, -50],
      ['2026-01-01T01:10:00Z', 0.2, -50.1],
      ['2026-01-01T02:00:00Z', 37.57, 126.98],
    ])
    const result = filterOutliers(points)
    expect(result.removedCount).toBe(2)
    expect(result.points.length).toBe(2)
  })

  it('preserves plausibly timed intercontinental travel', () => {
    const points = build([
      ['2026-01-01T00:00:00Z', 37.5665, 126.978],
      ['2026-01-02T14:00:00Z', -23.5505, -46.6333],
      ['2026-01-10T12:00:00Z', 37.57, 126.98],
    ])
    const result = filterOutliers(points)
    expect(result.removedCount).toBe(0)
    expect(result.points.length).toBe(3)
  })

  it('preserves one-way long distance travel and both endpoints', () => {
    const points = build([
      ['2026-01-01T00:00:00Z', 37.5665, 126.978],
      ['2026-01-01T12:00:00Z', 35.6762, 139.6503],
      ['2026-01-02T12:00:00Z', 51.5072, -0.1276],
    ])
    const result = filterOutliers(points)
    expect(result.points.length).toBe(3)
    expect(result.points.lat[0]).toBeCloseTo(37.5665, 6)
    expect(result.points.lat[2]).toBeCloseTo(51.5072, 6)
  })

  it('returns every point when the filter is off', () => {
    const points = build([
      ['2026-01-01T00:00:00Z', 37.5665, 126.978],
      ['2026-01-01T01:00:00Z', 0, -50],
      ['2026-01-01T02:00:00Z', 37.57, 126.98],
    ])
    const result = filterOutliers(points, 'off')
    expect(result.removedCount).toBe(0)
    expect(result.points.length).toBe(3)
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/outlier.test.ts`
Expected: FAIL

- [ ] **Step 3: 実装**

`LocationOutlierFilter.kt` の判定条件をそのまま移す。**閾値の数字を自分で決め直さないこと。** 移植元を読んで同じ値を使う。「行って戻ってくる」形の逸脱と、逸脱先で短時間クラスタを作る形の両方を検出する。始点と終点は落とさない。

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/outlier.test.ts`
Expected: PASS（5 件）

- [ ] **Step 5: コミット**

```bash
git add web/src/core/timeline/outlier.ts web/tests/outlier.test.ts
git commit -m "Port the location outlier filter"
```

---

## Task 5: Journey（期間切り出し・補間・描画パス）

**Files:**
- Create: `web/src/core/timeline/period.ts`, `web/src/core/journey/journey.ts`
- Test: `web/tests/journey.test.ts`（期間切り出しのテストもこのファイルに入れる。別ファイルにしない）

**移植元:** `model/TimelineModels.kt:28-107`（`Timeline.forDateRange` / `countForDateRange`）、`:111-370`（`Journey`, `JourneyLeg`, `RouteSample`, `JourneyPosition`）。テストは `model/JourneyTest.kt`

**Interfaces:**
- Consumes: Task 2、Task 4
- Produces:
  - `sliceByDateRange(points: TimelinePoints, startISO: string, endISO: string): TimelinePoints`
  - `countByDateRange(points: TimelinePoints, startISO: string, endISO: string): number`
  - `createJourney(points: TimelinePoints): Journey`
  - `interface Journey { points: TimelinePoints; totalDistanceKm: number; legs: JourneyLeg[]; transferThresholdKm: number; positionAt(progress: number): JourneyPosition; positionAtDistance(km: number): JourneyPosition; renderPathLength: number; renderSampleAt(index: number): { x: number, y: number, distanceKm: number } }`
  - `interface JourneyPosition { latitude: number; longitude: number; timeMs: number; distanceKm: number; fromIndex: number; toIndex: number; segmentFraction: number }`

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/journey.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { createPoints, finalizePoints } from '../src/core/timeline/points'
import { sliceByDateRange, countByDateRange } from '../src/core/timeline/period'
import { createJourney } from '../src/core/journey/journey'

const SEOUL: [string, number, number] = ['2025-06-01T00:00:00Z', 37.5665, 126.978]
const BOHOL: [string, number, number] = ['2025-06-01T04:00:00Z', 9.85, 124.1435]

function build(rows: [string, number, number][]) {
  const mutable = createPoints(rows.length)
  for (const [time, lat, lon] of rows) mutable.push(Date.parse(time), lat, lon)
  return finalizePoints(mutable)
}

describe('date range selection', () => {
  it('includes only points on the selected dates', () => {
    const points = build([
      ['2026-04-01T12:00:00Z', 37.0, 127.0],
      ['2026-04-02T12:00:00Z', 37.1, 127.1],
      ['2026-04-03T12:00:00Z', 37.2, 127.2],
      ['2026-04-04T12:00:00Z', 37.3, 127.3],
    ])
    const sliced = sliceByDateRange(points, '2026-04-02', '2026-04-03')

    expect(sliced.length).toBe(2)
    expect(countByDateRange(points, '2026-04-02', '2026-04-03')).toBe(2)
    expect(sliced.timeMs[0]).toBe(Date.parse('2026-04-02T12:00:00Z'))
    expect(sliced.timeMs[1]).toBe(Date.parse('2026-04-03T12:00:00Z'))
  })
})

describe('Journey', () => {
  it('interpolates continuously along a long flight', () => {
    const journey = createJourney(build([SEOUL, BOHOL]))
    const quarter = journey.positionAt(0.25)
    const halfway = journey.positionAt(0.5)
    const threeQuarters = journey.positionAt(0.75)

    expect(quarter.latitude).toBeLessThan(SEOUL[1])
    expect(quarter.latitude).toBeGreaterThan(halfway.latitude)
    expect(halfway.latitude).toBeGreaterThan(threeQuarters.latitude)
    expect(threeQuarters.latitude).toBeGreaterThan(BOHOL[1])
    expect(halfway.distanceKm).toBeCloseTo(journey.totalDistanceKm / 2, 1)
    expect(halfway.segmentFraction).toBeCloseTo(0.5, 4)
  })

  it('densifies long legs so rendering stays smooth', () => {
    const journey = createJourney(build([SEOUL, BOHOL]))
    expect(journey.renderPathLength).toBeGreaterThan(20)

    let largestStep = 0
    for (let i = 1; i < journey.renderPathLength; i++) {
      const step = journey.renderSampleAt(i).distanceKm - journey.renderSampleAt(i - 1).distanceKm
      largestStep = Math.max(largestStep, step)
    }
    expect(largestStep).toBeLessThanOrEqual(75.1)
  })

  it('unwraps world coordinates across the date line', () => {
    const journey = createJourney(build([
      ['2025-06-01T00:00:00Z', 10, 179],
      ['2025-06-01T04:00:00Z', 20, -179],
    ]))
    const first = journey.renderSampleAt(0)
    const last = journey.renderSampleAt(journey.renderPathLength - 1)
    expect(Math.abs(last.x - first.x)).toBeLessThan(0.02)
  })

  it('keeps the render path virtual instead of materialising objects', () => {
    const journey = createJourney(build([SEOUL, BOHOL]))
    // renderSampleAt は呼ぶたびに同じ値を返す純関数であること
    expect(journey.renderSampleAt(3)).toEqual(journey.renderSampleAt(3))
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/journey.test.ts`
Expected: FAIL

- [ ] **Step 3: 実装**

`TimelineModels.kt` の `Journey` / `JourneyRenderPath` をそのまま移す。要点:

- `renderPath` は**配列として実体化しない**。「元の点」と「長い区間を 75 km 以下に刻んだ仮想サンプル」を、インデックスから計算で引ける形にする（Kotlin 側が v2.1.2 でメモリのために取った方式と同じ）
- 世界座標 x は隣接サンプルとの差が 0.5 を超えたら ±1 して連続にする（`unwrapNear:683`）
- `transferThresholdKm` は経路ごとに動的に決まる（`calculateTransferThresholdKm` を読むこと。固定値にしない）

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/journey.test.ts`
Expected: PASS

- [ ] **Step 5: コミット**

```bash
git add web/src/core web/tests
git commit -m "Port journey geometry, date range slicing and the virtual render path"
```

---

## Task 6: 時間配分とアニメーション進行

**Files:**
- Create: `web/src/core/journey/timing.ts`, `web/src/core/render/animation.ts`
- Test: `web/tests/timing.test.ts`, `web/tests/animation.test.ts`

**移植元:** `render/JourneyTiming.kt`（96 行）、`render/TimelineAnimation.kt`（32 行）。テストは `render/JourneyTimingTest.kt`、`render/TimelineAnimationTest.kt`

**Interfaces:**
- Consumes: Task 5 の `Journey`
- Produces:
  - `createTiming(journey: Journey, compressionExponent: number): { distanceAt(progress: number): number }`
  - `frameAtElapsedSeconds(elapsedSeconds: number, journeyDurationSeconds: number): { journeyProgress: number, outroProgress: number }`
  - `totalDurationSeconds(journeyDurationSeconds: number): number`

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/timing.test.ts`（Kotlin 版 3 ケースの移植）:

```ts
import { describe, expect, it } from 'vitest'
import { createPoints, finalizePoints } from '../src/core/timeline/points'
import { createJourney } from '../src/core/journey/journey'
import { createTiming } from '../src/core/journey/timing'
import { totalDurationSeconds } from '../src/core/render/animation'

const OFF = 1.0
const BALANCED = 0.85

function journeyAt(longitudes: number[]) {
  const mutable = createPoints(longitudes.length)
  for (const lon of longitudes) mutable.push(Date.parse('2026-01-01T00:00:00Z'), 0, lon)
  return createJourney(finalizePoints(mutable))
}

function progressAtDistance(timing: { distanceAt(p: number): number }, distanceKm: number) {
  let low = 0
  let high = 1
  for (let i = 0; i < 40; i++) {
    const middle = (low + high) / 2
    if (timing.distanceAt(middle) < distanceKm) low = middle
    else high = middle
  }
  return (low + high) / 2
}

describe('createTiming', () => {
  it('reduces the share of an unusually long segment without changing geometry', () => {
    const journey = journeyAt([0, 0.1, 10.1, 10.2])
    const linear = createTiming(journey, OFF)
    const balanced = createTiming(journey, BALANCED)
    const longStartKm = journey.points.cumulativeKm[1]
    const longEndKm = journey.points.cumulativeKm[2]

    const linearShare =
      progressAtDistance(linear, longEndKm) - progressAtDistance(linear, longStartKm)
    const balancedShare =
      progressAtDistance(balanced, longEndKm) - progressAtDistance(balanced, longStartKm)

    expect(balancedShare).toBeLessThan(linearShare)
    expect(balanced.distanceAt(0)).toBeCloseTo(0, 9)
    expect(balanced.distanceAt(1)).toBeCloseTo(journey.totalDistanceKm, 6)
    expect(totalDurationSeconds(30)).toBe(31.5)
  })

  it('keeps linear timing exactly when compression is off', () => {
    const journey = journeyAt([0, 0.1, 10.1])
    const timing = createTiming(journey, OFF)
    for (const progress of [0, 0.1, 0.5, 0.9, 1]) {
      expect(timing.distanceAt(progress)).toBeCloseTo(journey.totalDistanceKm * progress, 9)
    }
  })

  it('has no speed jump at a segment boundary', () => {
    const journey = journeyAt([0, 0.1, 10.1, 10.2])
    const timing = createTiming(journey, BALANCED)
    const boundary = progressAtDistance(timing, journey.points.cumulativeKm[1])
    const step = 0.00001
    const before = (timing.distanceAt(boundary) - timing.distanceAt(boundary - step)) / step
    const after = (timing.distanceAt(boundary + step) - timing.distanceAt(boundary)) / step

    expect(Math.abs(after - before)).toBeLessThanOrEqual(Math.max(1, before * 0.02))
  })
})
```

`web/tests/animation.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { frameAtElapsedSeconds, totalDurationSeconds } from '../src/core/render/animation'

describe('frameAtElapsedSeconds', () => {
  it('adds a 1.5 second outro after the journey', () => {
    expect(totalDurationSeconds(30)).toBe(31.5)
  })

  it('reports journey progress during the main section', () => {
    expect(frameAtElapsedSeconds(15, 30)).toEqual({ journeyProgress: 0.5, outroProgress: 0 })
  })

  it('holds the journey at the end and runs the outro over one second', () => {
    expect(frameAtElapsedSeconds(30.5, 30)).toEqual({ journeyProgress: 1, outroProgress: 0.5 })
    expect(frameAtElapsedSeconds(31.5, 30)).toEqual({ journeyProgress: 1, outroProgress: 1 })
  })

  it('clamps out of range input', () => {
    expect(frameAtElapsedSeconds(-5, 30).journeyProgress).toBe(0)
    expect(frameAtElapsedSeconds(999, 30).outroProgress).toBe(1)
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/timing.test.ts tests/animation.test.ts`
Expected: FAIL

- [ ] **Step 3: 実装**

`JourneyTiming.kt:16-31` のエルミート補間をそのまま移す。区間長を `exponent` 乗して「体感距離」を作り、そこに対して単調三次エルミートで距離を割り当てる。傾き（`slopes`）の計算方法も移植元に従う。ここを自己流にすると境界で速度が飛んでテストが落ちる。

`TimelineAnimation.kt` は定数 2 つと関数 2 つだけなのでそのまま移す。

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/timing.test.ts tests/animation.test.ts`
Expected: PASS

- [ ] **Step 5: コミット**

```bash
git add web/src/core web/tests
git commit -m "Port journey timing compression and animation progress"
```

---

## Task 7: カメラ（ビューポート計算）

**Files:**
- Create: `web/src/core/render/camera.ts`
- Test: `web/tests/camera.test.ts`

**移植元:** `render/TimelinePainter.kt:99-435`（`viewport`, `rawViewport`, `cameraTrack`, `buildCameraTrack`, `tileZoom`, `stabilizedTileZoom`, `overviewViewport`, `blendViewport`, `trailWindowDistance`, イージング）と `render/CameraSettings.kt`

**Interfaces:**
- Consumes: Task 5、Task 6
- Produces:
  - `interface Viewport { minX: number; maxX: number; minY: number; maxY: number; zoom: number }`
  - `createCamera(journey: Journey, options: { width: number, height: number, journeyDurationSeconds: number, movement: CameraMovementPreset, compressionExponent: number }): Camera`
  - `interface Camera { viewportAt(progress: number): Viewport; overview(): Viewport; blend(a: Viewport, b: Viewport, t: number): Viewport; requiredTiles(viewport: Viewport): { id: TileId, worldX: number }[] }` … `requiredTiles` は Task 8 の描画と Task 13 のタイル事前収集の両方が使う
  - `const STEADY: CameraMovementPreset`（`contextFraction: 1.0, minimumContextKm: 650, maximumContextKm: 650, padding: 2.8, minimumViewportSpan: 0.0006, zoomOutAlpha: 0.14, zoomInAlpha: 0.035, legAware: false, fixedZoom: false`）

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/camera.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { createPoints, finalizePoints } from '../src/core/timeline/points'
import { createJourney } from '../src/core/journey/journey'
import { createCamera, STEADY } from '../src/core/render/camera'

function camera() {
  const mutable = createPoints(3)
  mutable.push(Date.parse('2025-06-01T00:00:00Z'), 37.5665, 126.978)
  mutable.push(Date.parse('2025-06-01T04:00:00Z'), 9.85, 124.1435)
  mutable.push(Date.parse('2025-06-01T08:00:00Z'), 35.6762, 139.6503)
  return createCamera(createJourney(finalizePoints(mutable)), {
    width: 480,
    height: 480,
    journeyDurationSeconds: 30,
    movement: STEADY,
    compressionExponent: 0.85,
  })
}

describe('createCamera', () => {
  it('produces a viewport that contains the current position', () => {
    const view = camera().viewportAt(0.5)
    expect(view.maxX).toBeGreaterThan(view.minX)
    expect(view.maxY).toBeGreaterThan(view.minY)
    expect(view.zoom).toBeGreaterThanOrEqual(0)
    expect(view.zoom).toBeLessThanOrEqual(19)
  })

  it('keeps the viewport square for a square canvas', () => {
    const view = camera().viewportAt(0.3)
    expect(view.maxX - view.minX).toBeCloseTo(view.maxY - view.minY, 6)
  })

  it('moves smoothly without jumps between adjacent frames', () => {
    const cam = camera()
    let largestJump = 0
    let previous = cam.viewportAt(0)
    for (let i = 1; i <= 100; i++) {
      const current = cam.viewportAt(i / 100)
      const centreShift = Math.abs(
        (current.minX + current.maxX) / 2 - (previous.minX + previous.maxX) / 2,
      )
      largestJump = Math.max(largestJump, centreShift / (current.maxX - current.minX))
      previous = current
    }
    // 1 フレームでの中心移動は画面幅の 30% を超えない
    expect(largestJump).toBeLessThan(0.3)
  })

  it('shows the whole route in the overview viewport', () => {
    const cam = camera()
    const overview = cam.overview()
    for (const progress of [0, 0.25, 0.5, 0.75, 1]) {
      const view = cam.viewportAt(progress)
      expect(overview.maxX - overview.minX).toBeGreaterThanOrEqual(view.maxX - view.minX)
    }
  })

  it('blends between two viewports', () => {
    const cam = camera()
    const a = cam.viewportAt(0)
    const b = cam.overview()
    expect(cam.blend(a, b, 0)).toEqual(a)
    const mid = cam.blend(a, b, 1)
    expect(mid.minX).toBeCloseTo(b.minX, 6)
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/camera.test.ts`
Expected: FAIL

- [ ] **Step 3: 実装**

`TimelinePainter.kt:99-435` を移す。カメラトラックは進捗をサンプリングして事前に作り、`zoomOutAlpha` / `zoomInAlpha` で指数移動平均をかけて滑らかにする。ズーム段階は `stabilizedTileZoom:307` のヒステリシスをそのまま入れる（入れないとタイルズームが 1 フレームごとに行き来してちらつく）。

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/camera.test.ts`
Expected: PASS

- [ ] **Step 5: コミット**

```bash
git add web/src/core/render/camera.ts web/tests/camera.test.ts
git commit -m "Port the camera track and viewport calculation"
```

---

## Task 8: 描画（Painter）

**Files:**
- Create: `web/src/core/render/painter.ts`
- Test: `web/tests/painter.test.ts`

**移植元:** `render/TimelinePainter.kt:440-680`（`draw`, `drawBackground`, `drawTiles`, `drawOverlay`, `drawRouteRange`, `worldToScreen`）と `render/RenderText.kt`

**Interfaces:**
- Consumes: Task 5、Task 7
- Produces:
  - `type Canvas2D = CanvasRenderingContext2D | OffscreenCanvasRenderingContext2D`
  - `drawFrame(ctx: Canvas2D, options: { journey, camera, viewport, position, title, width, height, tiles: (id: TileId) => ImageBitmap | null, text: RenderText }): void`
  - `interface RenderText { locale: string; datePattern: 'ymd' | 'md'; distanceUnit: string; attribution: string; fallbackTitle: string }`

**この関数がプレビューと書き出しの両方から呼ばれる唯一の描画コードである。** 分岐を作らないこと。

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/painter.test.ts`（実 Canvas がない環境なので、呼ばれた描画命令を記録するスタブで検証する）:

```ts
import { describe, expect, it, vi } from 'vitest'
import { createPoints, finalizePoints } from '../src/core/timeline/points'
import { createJourney } from '../src/core/journey/journey'
import { createCamera, STEADY } from '../src/core/render/camera'
import { drawFrame } from '../src/core/render/painter'

function stubContext() {
  const calls: string[] = []
  const gradient = { addColorStop: vi.fn() }
  const handler: ProxyHandler<object> = {
    get(_target, property: string) {
      if (property === '__calls') return calls
      if (property === 'createLinearGradient') return () => gradient
      if (property === 'measureText') return (text: string) => ({ width: text.length * 10 })
      if (property === 'canvas') return { width: 480, height: 480 }
      return (...args: unknown[]) => {
        calls.push(property)
        return undefined
      }
    },
    set() {
      return true
    },
  }
  return new Proxy({}, handler) as never
}

function fixture() {
  const mutable = createPoints(2)
  mutable.push(Date.parse('2025-06-01T00:00:00Z'), 37.5665, 126.978)
  mutable.push(Date.parse('2025-06-01T04:00:00Z'), 9.85, 124.1435)
  const journey = createJourney(finalizePoints(mutable))
  const camera = createCamera(journey, {
    width: 480,
    height: 480,
    journeyDurationSeconds: 30,
    movement: STEADY,
    compressionExponent: 0.85,
  })
  return { journey, camera }
}

const TEXT = {
  locale: 'ja-JP',
  datePattern: 'ymd' as const,
  distanceUnit: 'km',
  attribution: '© OpenStreetMap contributors © CARTO',
  fallbackTitle: 'My Journey',
}

describe('drawFrame', () => {
  it('paints a background, the route and the overlay card', () => {
    const { journey, camera } = fixture()
    const ctx = stubContext()
    drawFrame(ctx, {
      journey,
      camera,
      viewport: camera.viewportAt(0.5),
      position: journey.positionAt(0.5),
      title: '2025年の旅',
      width: 480,
      height: 480,
      tiles: () => null,
      text: TEXT,
    })

    const calls = (ctx as unknown as { __calls: string[] }).__calls
    expect(calls).toContain('fillRect')   // 背景
    expect(calls).toContain('stroke')     // 経路
    expect(calls).toContain('roundRect')  // オーバーレイのカード
    expect(calls).toContain('fillText')   // タイトルと日付
  })

  it('still renders when no tiles are available', () => {
    const { journey, camera } = fixture()
    const ctx = stubContext()
    expect(() =>
      drawFrame(ctx, {
        journey,
        camera,
        viewport: camera.viewportAt(0),
        position: journey.positionAt(0),
        title: '',
        width: 480,
        height: 480,
        tiles: () => null,
        text: TEXT,
      }),
    ).not.toThrow()
  })

  it('draws every available tile', () => {
    const { journey, camera } = fixture()
    const ctx = stubContext()
    const bitmap = {} as ImageBitmap
    drawFrame(ctx, {
      journey,
      camera,
      viewport: camera.viewportAt(0.5),
      position: journey.positionAt(0.5),
      title: 'x',
      width: 480,
      height: 480,
      tiles: () => bitmap,
      text: TEXT,
    })
    const calls = (ctx as unknown as { __calls: string[] }).__calls
    expect(calls.filter((c) => c === 'drawImage').length).toBeGreaterThan(0)
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/painter.test.ts`
Expected: FAIL

- [ ] **Step 3: 実装**

Android の描画 API を Canvas2D に読み替える。対応は以下:

| Android | Canvas2D |
|---|---|
| `LinearGradient` + `Paint.shader` | `ctx.createLinearGradient()` → `ctx.fillStyle` |
| `canvas.drawRoundRect` | `ctx.beginPath(); ctx.roundRect(...); ctx.fill()` |
| `canvas.drawBitmap(bmp, null, RectF)` | `ctx.drawImage(bitmap, left, top, width, height)` |
| `Paint.measureText` | `ctx.measureText(text).width` |
| `Paint.textSize` | `ctx.font = \`${size}px ...\`` |
| `Paint.breakText` | `measureText` の二分探索で自前実装 |
| `DateTimeFormatter` | `Intl.DateTimeFormat` |
| `NumberFormat` | `Intl.NumberFormat` |

移植上の注意:

- 座標系のスケールは `width / 720` 基準（`drawOverlay:571` の `scale`）。480 で描くと 0.667 倍になる。ここを変えると Android 版と見た目がずれる
- タイル描画は右端と下端を 1 px 広げる（`drawTiles:568` の `right + 1, bottom + 1`）。これを省くとタイル間に隙間の線が出る
- 経路の描画範囲は「現在位置から後方に `trailWindowDistance` km」だけ。全経路を毎フレーム描かない
- 帰属表示は必ず描く

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/painter.test.ts`
Expected: PASS

- [ ] **Step 5: コミット**

```bash
git add web/src/core/render/painter.ts web/tests/painter.test.ts
git commit -m "Port the frame painter to Canvas 2D"
```

---

## Task 9: 地図タイルの取得とキャッシュ

**Files:**
- Create: `web/src/core/tiles/source.ts`
- Test: `web/tests/tiles.test.ts`

**移植元:** `data/TileRepository.kt`（55 行）

**Interfaces:**
- Consumes: Task 1 の `TILE_URL_TEMPLATE`
- Produces:
  - `interface TileId { zoom: number; x: number; y: number }`
  - `tileUrl(id: TileId): string`
  - `createTileSource(): { get(id: TileId): ImageBitmap | null, load(id: TileId): Promise<ImageBitmap | null>, loadAll(ids: TileId[], onProgress: (done: number, total: number) => void): Promise<void> }`

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/tiles.test.ts`:

```ts
import { describe, expect, it, vi, beforeEach } from 'vitest'
import { createTileSource, tileUrl } from '../src/core/tiles/source'

const BITMAP = {} as ImageBitmap

beforeEach(() => {
  vi.stubGlobal('createImageBitmap', vi.fn(async () => BITMAP))
  vi.stubGlobal('caches', undefined)  // Cache API がない環境でも動くこと
})

describe('tileUrl', () => {
  it('builds the CARTO url', () => {
    expect(tileUrl({ zoom: 5, x: 27, y: 12 }))
      .toBe('https://a.basemaps.cartocdn.com/light_all/5/27/12.png')
  })
})

describe('createTileSource', () => {
  it('returns null before a tile is loaded', () => {
    expect(createTileSource().get({ zoom: 5, x: 27, y: 12 })).toBeNull()
  })

  it('caches a loaded tile in memory', async () => {
    const fetchMock = vi.fn(async () => new Response(new Blob([new Uint8Array([1])])))
    vi.stubGlobal('fetch', fetchMock)
    const source = createTileSource()
    const id = { zoom: 5, x: 27, y: 12 }

    await source.load(id)
    await source.load(id)

    expect(fetchMock).toHaveBeenCalledTimes(1)
    expect(source.get(id)).toBe(BITMAP)
  })

  it('returns null instead of throwing when a tile fails', async () => {
    vi.stubGlobal('fetch', vi.fn(async () => new Response(null, { status: 404 })))
    expect(await createTileSource().load({ zoom: 5, x: 1, y: 1 })).toBeNull()
  })

  it('reports progress while loading a batch', async () => {
    vi.stubGlobal('fetch', vi.fn(async () => new Response(new Blob([new Uint8Array([1])]))))
    const source = createTileSource()
    const progress: number[] = []
    await source.loadAll(
      [{ zoom: 5, x: 1, y: 1 }, { zoom: 5, x: 2, y: 1 }],
      (done) => progress.push(done),
    )
    expect(progress).toEqual([1, 2])
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/tiles.test.ts`
Expected: FAIL

- [ ] **Step 3: 実装**

- メモリ LRU は最大 256 枚。超えたら古いものから `ImageBitmap.close()` して捨てる（**close を忘れると GPU メモリが解放されない**）
- 永続キャッシュは Cache API（`caches.open('carto-tiles')`）。`caches` が使えない環境ではメモリのみで動くこと
- `loadAll` の同時実行数は 6 まで。CDN に一気に投げない
- 失敗は 1 枚単位で握り潰して `null` を返す。全体を落とさない

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/tiles.test.ts`
Expected: PASS

- [ ] **Step 5: コミット**

```bash
git add web/src/core/tiles web/tests/tiles.test.ts
git commit -m "Add map tile loading with memory and Cache API layers"
```

---

## Task 10: パース Worker とファイル選択画面

**Files:**
- Create: `web/src/workers/parse.worker.ts`, `web/src/features/FilePicker.tsx`, `web/src/features/useTimeline.ts`
- Modify: `web/src/routes/index.tsx`
- Test: `web/tests/parse-worker-protocol.test.ts`

**Interfaces:**
- Consumes: Task 3、Task 4
- Produces:
  - Worker メッセージ: `{ type: 'parse', file: File }` → `{ type: 'progress', bytesRead: number, totalBytes: number }` / `{ type: 'done', points: TimelinePointsTransfer, removedCount: number }` / `{ type: 'error', reason: 'unsupported-format' | 'no-location-data' }`
  - `interface TimelinePointsTransfer { lat: ArrayBuffer; lon: ArrayBuffer; timeMs: ArrayBuffer; cumulativeKm: ArrayBuffer; length: number }`
  - `useTimeline(): { state, selectFile(file: File): void }`

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/parse-worker-protocol.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { createPoints, finalizePoints } from '../src/core/timeline/points'
import { toTransfer, fromTransfer, transferables } from '../src/workers/protocol'

describe('worker transfer protocol', () => {
  it('round-trips points without copying', () => {
    const mutable = createPoints(2)
    mutable.push(1000, 37.5, 127)
    mutable.push(2000, 35.6, 139.7)
    const points = finalizePoints(mutable)

    const transfer = toTransfer(points)
    const restored = fromTransfer(transfer)

    expect(restored.length).toBe(2)
    expect(restored.lat[1]).toBeCloseTo(35.6, 6)
    expect(restored.cumulativeKm[1]).toBeGreaterThan(0)
  })

  it('lists every buffer as transferable so nothing is cloned', () => {
    const mutable = createPoints(1)
    mutable.push(1000, 37.5, 127)
    const transfer = toTransfer(finalizePoints(mutable))
    expect(transferables(transfer)).toHaveLength(4)
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/parse-worker-protocol.test.ts`
Expected: FAIL

- [ ] **Step 3: 実装**

`web/src/workers/protocol.ts` に変換関数を置く。Worker 本体は `file.stream()` を `parseTimeline` に渡し、終わったら `filterOutliers` をかけて `postMessage(transfer, transferables(transfer))` で返す。

`FilePicker.tsx` は `<input type="file" accept=".json,application/json">` だけ。**ドラッグ＆ドロップは MVP に入れない。**

進捗は `file.size` に対する読み込みバイト数で出す。数百 MB のファイルでは十数秒かかるため、無反応に見せない。

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/parse-worker-protocol.test.ts`
Expected: PASS

- [ ] **Step 5: 実ファイルで手動確認**

`test-fixtures/takeout-sample.json` を開発サーバーで読み込み、点数が表示されることを目で確認する。

Run: `cd web && bun run dev`

- [ ] **Step 6: コミット**

```bash
git add web/src
git commit -m "Parse the Timeline file in a worker and add the file picker"
```

---

## Task 11: 期間選択画面

**Files:**
- Create: `web/src/features/PeriodPicker.tsx`, `web/src/core/timeline/summary.ts`
- Test: `web/tests/summary.test.ts`

**移植元:** `MainActivity.kt` の新規動画画面（月レンジと正確な日付の切り替え、既定は直近の完全な 1 年）

**Interfaces:**
- Consumes: Task 5 の `sliceByDateRange` / `countByDateRange`
- Produces:
  - `summarize(points: TimelinePoints): { firstMs: number, lastMs: number, countByMonth: Map<string, number> }`
  - `defaultRange(summary): { start: string, end: string }`

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/summary.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { createPoints, finalizePoints } from '../src/core/timeline/points'
import { summarize, defaultRange } from '../src/core/timeline/summary'

function build(times: string[]) {
  const mutable = createPoints(times.length)
  times.forEach((t, i) => mutable.push(Date.parse(t), 37 + i * 0.01, 127))
  return finalizePoints(mutable)
}

describe('summarize', () => {
  it('counts points per month', () => {
    const summary = summarize(build([
      '2024-12-31T00:00:00Z',
      '2025-01-05T00:00:00Z',
      '2025-01-20T00:00:00Z',
      '2026-03-01T00:00:00Z',
    ]))
    expect(summary.countByMonth.get('2025-01')).toBe(2)
    expect(summary.countByMonth.get('2024-12')).toBe(1)
    expect(summary.firstMs).toBe(Date.parse('2024-12-31T00:00:00Z'))
    expect(summary.lastMs).toBe(Date.parse('2026-03-01T00:00:00Z'))
  })

  it('defaults to the most recent complete year', () => {
    const summary = summarize(build([
      '2024-06-01T00:00:00Z',
      '2025-06-01T00:00:00Z',
      '2026-03-01T00:00:00Z',
    ]))
    expect(defaultRange(summary)).toEqual({ start: '2025-01-01', end: '2025-12-31' })
  })

  it('falls back to the full range when no complete year exists', () => {
    const summary = summarize(build(['2026-02-01T00:00:00Z', '2026-03-01T00:00:00Z']))
    expect(defaultRange(summary)).toEqual({ start: '2026-02-01', end: '2026-03-01' })
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/summary.test.ts`
Expected: FAIL

- [ ] **Step 3: 実装**

`summarize` は SoA を 1 回走査するだけ。月キーは UTC ではなく**端末のローカルタイムゾーン**で作る（Android 版が `ZoneId.systemDefault()` を使っているため）。

`PeriodPicker.tsx` は月レンジと日付指定を切り替えるだけの UI。選択中の範囲に含まれる点数をリアルタイムに出す。点数が 2 未満なら次に進ませない。

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/summary.test.ts`
Expected: PASS

- [ ] **Step 5: コミット**

```bash
git add web/src web/tests
git commit -m "Add period selection with per-month point counts"
```

---

## Task 12: プレビュー

**Files:**
- Create: `web/src/features/Preview.tsx`, `web/src/features/usePreviewLoop.ts`
- Test: `web/tests/preview-loop.test.ts`

**Interfaces:**
- Consumes: Task 6、Task 7、Task 8、Task 9
- Produces: `createPreviewLoop(options): { start(): void, stop(): void, seek(seconds: number): void }`

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/preview-loop.test.ts`:

```ts
import { describe, expect, it, vi } from 'vitest'
import { createPreviewLoop } from '../src/features/usePreviewLoop'

describe('createPreviewLoop', () => {
  it('draws once per animation frame and stops cleanly', () => {
    let callback: FrameRequestCallback | null = null
    vi.stubGlobal('requestAnimationFrame', (cb: FrameRequestCallback) => {
      callback = cb
      return 1
    })
    const cancel = vi.fn()
    vi.stubGlobal('cancelAnimationFrame', cancel)

    const draw = vi.fn()
    const loop = createPreviewLoop({ draw, durationSeconds: 30 })

    loop.start()
    callback?.(0)
    callback?.(500)

    expect(draw).toHaveBeenCalledTimes(2)
    expect(draw.mock.calls[1][0]).toBeCloseTo(0.5, 3)  // 経過秒

    loop.stop()
    expect(cancel).toHaveBeenCalled()
  })

  it('holds at the end instead of looping', () => {
    let callback: FrameRequestCallback | null = null
    vi.stubGlobal('requestAnimationFrame', (cb: FrameRequestCallback) => {
      callback = cb
      return 1
    })
    vi.stubGlobal('cancelAnimationFrame', vi.fn())

    const draw = vi.fn()
    const loop = createPreviewLoop({ draw, durationSeconds: 30 })
    loop.start()
    callback?.(0)
    callback?.(999_000)

    expect(draw.mock.calls.at(-1)?.[0]).toBe(31.5)  // 30 + アウトロ 1.5 秒で止まる
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/preview-loop.test.ts`
Expected: FAIL

- [ ] **Step 3: 実装**

- `requestAnimationFrame` の `timestamp` から経過秒を出す。**フレーム番号で数えない**（端末のリフレッシュレートで速度が変わってしまう）
- タイルは事前取得しない。`tiles.get()` が `null` を返したら背景のまま描き、非同期でロードして次のフレームから反映する
- 端まで行ったら止める。Android 版と同じく、もう一度押したら頭から再生する
- Canvas の解像度は `devicePixelRatio` を掛けて設定するが、**上限 2 倍**で止める（スマホで 3 倍にすると描画が重くなる）

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/preview-loop.test.ts`
Expected: PASS

- [ ] **Step 5: 実機で目視確認**

開発サーバーで `test-fixtures/seoul-bohol-sample.json` を読み込み、アニメーションが動くことを**画面で見る**。この段階で Android 版のプレビューと並べて、カメラの動きが同じか確認する。

- [ ] **Step 6: コミット**

```bash
git add web/src web/tests
git commit -m "Add the interactive preview loop"
```

---

## Task 13: 書き出し Worker（エンコードと mux）

**Files:**
- Create: `web/src/workers/export.worker.ts`, `web/src/core/render/exportPlan.ts`
- Test: `web/tests/export-plan.test.ts`

**移植元:** `export/Mp4Exporter.kt`（315 行）。特にタイルの事前収集（`sampleCount` と `requiredTiles`）と 3 フェーズの進捗

**Interfaces:**
- Consumes: Task 7、Task 8、Task 9
- Produces:
  - `planTiles(journey, camera, options): TileId[]`（全フレームで必要になるタイルを重複排除して返す）
  - `frameCount(journeyDurationSeconds: number, fps: number): number`
  - Worker メッセージ: `{ type: 'export', ... }` → `{ type: 'progress', phase: 'preparing-map' | 'creating-video' | 'finishing-video', fraction: number }` / `{ type: 'done', blob: Blob }` / `{ type: 'error', reason: string }`

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/export-plan.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { createPoints, finalizePoints } from '../src/core/timeline/points'
import { createJourney } from '../src/core/journey/journey'
import { createCamera, STEADY } from '../src/core/render/camera'
import { planTiles, frameCount } from '../src/core/render/exportPlan'

function fixture() {
  const mutable = createPoints(3)
  mutable.push(Date.parse('2025-06-01T00:00:00Z'), 37.5665, 126.978)
  mutable.push(Date.parse('2025-06-01T04:00:00Z'), 9.85, 124.1435)
  mutable.push(Date.parse('2025-06-01T08:00:00Z'), 35.6762, 139.6503)
  const journey = createJourney(finalizePoints(mutable))
  const camera = createCamera(journey, {
    width: 480,
    height: 480,
    journeyDurationSeconds: 30,
    movement: STEADY,
    compressionExponent: 0.85,
  })
  return { journey, camera }
}

describe('frameCount', () => {
  it('covers the journey plus the outro at 24 fps', () => {
    expect(frameCount(30, 24)).toBe(Math.ceil(31.5 * 24))
  })
})

describe('planTiles', () => {
  it('collects every tile the video will need, without duplicates', () => {
    const { journey, camera } = fixture()
    const tiles = planTiles(journey, camera, { journeyDurationSeconds: 30, width: 480, height: 480 })

    expect(tiles.length).toBeGreaterThan(0)
    const keys = tiles.map((t) => `${t.zoom}/${t.x}/${t.y}`)
    expect(new Set(keys).size).toBe(keys.length)
  })

  it('includes the tiles the first and last frames need', () => {
    const { journey, camera } = fixture()
    const tiles = planTiles(journey, camera, { journeyDurationSeconds: 30, width: 480, height: 480 })
    const keys = new Set(tiles.map((t) => `${t.zoom}/${t.x}/${t.y}`))

    for (const progress of [0, 1]) {
      const view = camera.viewportAt(progress)
      const required = camera.requiredTiles(view)
      for (const tile of required) {
        expect(keys.has(`${tile.id.zoom}/${tile.id.x}/${tile.id.y}`)).toBe(true)
      }
    }
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/export-plan.test.ts`
Expected: FAIL

- [ ] **Step 3: Mediabunny を入れて Worker を実装**

```bash
cd web && bun add mediabunny
```

Worker の流れ:

```ts
// 1. タイルを先に全部取る
const tiles = planTiles(journey, camera, options)
await tileSource.loadAll(tiles, (done, total) =>
  post({ type: 'progress', phase: 'preparing-map', fraction: done / total }))

// 2. エンコーダを用意する
const canvas = new OffscreenCanvas(VIDEO_SIZE, VIDEO_SIZE)
const ctx = canvas.getContext('2d')!
const output = new Output({ format: new Mp4OutputFormat(), target: new BufferTarget() })
const videoSource = new EncodedVideoPacketSource('avc')
output.addVideoTrack(videoSource)
await output.start()

const encoder = new VideoEncoder({
  output: (chunk, meta) => videoSource.add(chunk, meta),
  error: (e) => post({ type: 'error', reason: String(e) }),
})
encoder.configure({ codec, width: VIDEO_SIZE, height: VIDEO_SIZE, bitrate: VIDEO_BITRATE, framerate: VIDEO_FPS })

// 3. 1 フレームずつ描いて流す
const total = frameCount(durationSeconds, VIDEO_FPS)
for (let i = 0; i < total; i++) {
  if (signal.aborted) { encoder.close(); return }

  // ★ バックプレッシャ。これが無いとスマホでメモリを食い潰して落ちる
  while (encoder.encodeQueueSize > 4) {
    await new Promise<void>((resolve) => { encoder.ondequeue = () => resolve() })
  }

  const elapsed = (i / VIDEO_FPS)
  const frame = frameAtElapsedSeconds(elapsed, durationSeconds)
  drawFrame(ctx, { ...buildFrameArgs(frame) })

  const videoFrame = new VideoFrame(canvas, { timestamp: Math.round(elapsed * 1_000_000) })
  encoder.encode(videoFrame, { keyFrame: i % (VIDEO_FPS * 2) === 0 })
  videoFrame.close()   // ★ 必須。忘れると GPU メモリが解放されない

  if (i % 12 === 0) post({ type: 'progress', phase: 'creating-video', fraction: i / total })
}

// 4. 締める
await encoder.flush()
encoder.close()
await output.finalize()
post({ type: 'done', blob: new Blob([output.target.buffer!], { type: 'video/mp4' }) })
```

**Mediabunny の API 名は必ず [公式ドキュメント](https://mediabunny.dev/guide/introduction) で確認してから書くこと。** 上の擬似コードは構造を示すものであり、クラス名を鵜呑みにしない。

コーデック選択は `avc1.42001f` → 失敗したら `avc1.42E01F` の順に `VideoEncoder.isConfigSupported()` で試し、どちらも通らなければ `{ type: 'error', reason: 'codec-unsupported' }` を返す。

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/export-plan.test.ts`
Expected: PASS

- [ ] **Step 5: PC の実ブラウザで 1 本書き出す**

`seoul-bohol-sample.json` で 10 秒の動画を書き出し、**ダウンロードした MP4 を実際に再生して目で見る。** 「エラーが出なかった」は再生できることの証明にならない。

- [ ] **Step 6: コミット**

```bash
git add web/src web/tests web/package.json
git commit -m "Encode the video in a worker with WebCodecs and Mediabunny"
```

---

## Task 14: 書き出し画面と機能検出

**Files:**
- Create: `web/src/features/ExportPanel.tsx`, `web/src/core/support.ts`
- Test: `web/tests/support.test.ts`

**Interfaces:**
- Consumes: Task 13
- Produces: `checkExportSupport(): Promise<{ supported: boolean, reason?: 'no-videoencoder' | 'no-offscreencanvas' | 'codec-unsupported' }>`

- [ ] **Step 1: 失敗するテストを書く**

`web/tests/support.test.ts`:

```ts
import { describe, expect, it, vi, afterEach } from 'vitest'
import { checkExportSupport } from '../src/core/support'

afterEach(() => vi.unstubAllGlobals())

describe('checkExportSupport', () => {
  it('reports no-videoencoder when WebCodecs is missing', async () => {
    vi.stubGlobal('VideoEncoder', undefined)
    vi.stubGlobal('OffscreenCanvas', class {})
    expect(await checkExportSupport()).toEqual({ supported: false, reason: 'no-videoencoder' })
  })

  it('reports no-offscreencanvas when OffscreenCanvas is missing', async () => {
    vi.stubGlobal('VideoEncoder', { isConfigSupported: async () => ({ supported: true }) })
    vi.stubGlobal('OffscreenCanvas', undefined)
    expect(await checkExportSupport()).toEqual({ supported: false, reason: 'no-offscreencanvas' })
  })

  it('falls back to constrained baseline when the main profile is rejected', async () => {
    const tried: string[] = []
    vi.stubGlobal('OffscreenCanvas', class {})
    vi.stubGlobal('VideoEncoder', {
      isConfigSupported: async (config: { codec: string }) => {
        tried.push(config.codec)
        return { supported: config.codec === 'avc1.42E01F' }
      },
    })
    expect(await checkExportSupport()).toEqual({ supported: true })
    expect(tried).toEqual(['avc1.42001f', 'avc1.42E01F'])
  })

  it('reports codec-unsupported when no profile is accepted', async () => {
    vi.stubGlobal('OffscreenCanvas', class {})
    vi.stubGlobal('VideoEncoder', { isConfigSupported: async () => ({ supported: false }) })
    expect(await checkExportSupport()).toEqual({ supported: false, reason: 'codec-unsupported' })
  })
})
```

- [ ] **Step 2: 実行して落ちることを確認**

Run: `cd web && bun run vitest run tests/support.test.ts`
Expected: FAIL

- [ ] **Step 3: 実装**

`ExportPanel.tsx` の要件:

- タイトル入力（既定は「{年}年の旅」相当。Android 版の `TitleTemplate.kt` を参照）
- 動画長スライダー（10〜300 秒）。60 秒を超えたら所要時間と容量の注意を出す
- 3 フェーズの進捗表示と、経過から推定した残り時間
- キャンセルボタン（`AbortController` を Worker に渡す）
- 完了したら `URL.createObjectURL(blob)` を `<a download>` に渡してダウンロードさせる
- 非対応ブラウザでは書き出しボタンを無効にし、理由を文章で出す。**「Firefox Android は動画の書き出しに対応していないため、Chrome または Safari で開いてほしい」と具体的に書く。** 黙って落とさない

**文言の多言語化（日本語 / 英語のみ）:** UI 文字列は `web/src/core/messages.ts` に `{ ja: {...}, en: {...} }` の 2 言語で置き、`navigator.language` が `ja` で始まるかどうかだけで選ぶ。切り替え UI は作らない。動画に焼き込む文字（日付形式・距離単位・帰属表示）は Task 8 の `RenderText` 経由で同じ辞書から渡す。**画面と動画で別々の文言を持たない。**

- [ ] **Step 4: テストが通ることを確認**

Run: `cd web && bun run vitest run tests/support.test.ts`
Expected: PASS

- [ ] **Step 5: コミット**

```bash
git add web/src web/tests
git commit -m "Add the export panel with progress, cancel and capability detection"
```

---

## Task 15: デプロイと実機検証

**Files:**
- Create: `.github/workflows/web.yml`
- Modify: `README.md`, `README.ja.md`（Web 版へのリンクを追加）

- [ ] **Step 0: fork 側の一度きりの設定を済ませる**

fork ではデフォルトで GitHub Actions が無効になっている。**これをやらないとワークフローは黙って一度も動かない。**

```bash
# fork の Actions タブを開いて「I understand my workflows, go ahead and enable them」を押す
gh repo view konaito/google-timeline-visualizer --web
# Pages の配信元を GitHub Actions にする
gh api -X POST repos/konaito/google-timeline-visualizer/pages -f build_type=workflow
```

- [ ] **Step 1: GitHub Actions のワークフローを書く**

```yaml
name: web
on:
  push:
    branches: [main]
  pull_request:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: false
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install --frozen-lockfile
        working-directory: web
      - run: bun run vitest run
        working-directory: web
      - run: bun run build
        working-directory: web
      - uses: actions/upload-pages-artifact@v3
        with:
          path: web/dist
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

**`build` だけでは公開されない。** `upload-pages-artifact` はアーティファクトを上げるだけで、実際に配信するのは `deploy-pages`。

- [ ] **Step 2: ローカルで同じコマンドを流す**

Run: `cd web && bun install --frozen-lockfile && bun run vitest run && bun run build`
Expected: すべて成功。**CI に投げて確かめない。** ローカルで通してから push する

- [ ] **Step 3: コミットして push する**

```bash
git add .github/workflows/web.yml
git commit -m "Build and publish the web app to GitHub Pages"
git push fork feat/web-version
```

この時点ではまだ公開されない。`deploy` ジョブは fork の `main` にしか反応しない。公開は Step 8 のマージ後。

- [ ] **Step 4: PC の Chrome で実データを通す**

自分の本物の `Timeline.json`（数百 MB クラス）を読み込み、1 年分を 30 秒で書き出す。所要時間とピークメモリを記録する。

- [ ] **Step 5: iPhone の Safari で同じことをやる**

**これが本計画で最大の未検証項目。** 同じファイルを iOS Safari で読み込み、書き出しまで到達するか確かめる。落ちる場合は、どの点数で落ちるかを二分探索で特定し、その閾値を超えたら期間を狭めるよう案内する実装を入れる。

- [ ] **Step 6: Android 版と出力を並べて見比べる**

同じ期間・同じ設定で Android 版と Web 版の MP4 を作り、並べて再生する。カメラの動き、経路の色と太さ、オーバーレイの位置、日付と距離の表示を確認する。ずれていたら該当箇所の移植元を読み直す。

- [ ] **Step 7: README に Web 版を追記してコミット**

```bash
git add README.md README.ja.md
git commit -m "Document the web version in the README"
git push fork feat/web-version
```

- [ ] **Step 8: fork の main にマージして公開する**

**Step 4 から 6 の実機検証がすべて通ってから実行する。** `main` へ直接 push しない。PR を作ってマージする。

```bash
gh pr create --repo konaito/google-timeline-visualizer \
  --base main --head feat/web-version \
  --title "Add the browser-based web version" \
  --body "Ports the Android app to a browser-only web app. No server, no wasm."
gh pr merge --repo konaito/google-timeline-visualizer --squash
```

マージ後に `deploy` ジョブが走る。**公開 URL を実際にブラウザで開いて、動くことを目で確認する。** Actions が緑になったことは、ページが動くことの証明ではない。

---

## 完了条件

1. `bun run vitest run` が全件通る
2. PC Chrome と iOS Safari の両方で、実データから MP4 をダウンロードできた
3. 出力を Android 版と見比べて、見た目が一致している
4. fork の `main` にマージされ、GitHub Pages の公開 URL を開いて実際に動いた
5. ネットワークタブを見て、地図タイル以外の外向き通信が 1 本も無いことを確認した
