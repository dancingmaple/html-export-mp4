# video-exporter.js

[![License: MIT](https://img.shields.io/badge/License-MIT-f4b740.svg)](LICENSE)
[![Dependencies](https://img.shields.io/badge/dependencies-none-34d3c0.svg)](#installation)
[![Engines](https://img.shields.io/badge/engines-WebCodecs%20%2B%20MediaRecorder-9d8cff.svg)](#how-it-works)
[![Format](https://img.shields.io/badge/format-UMD%20%2F%20bundler%20ESM-69727f.svg)](#installation)

> Turn **any** `<canvas>` animation into a high-quality MP4 — with one function.

`video-exporter.js` is a tiny, zero-dependency library that records a canvas animation frame by frame and encodes it to video. You only supply a `render(ctx, t)` callback that draws one frame at time `t`; the library drives the timeline, encodes, falls back gracefully, and triggers the download.

It is **decoupled from any UI or content** — quote posters, data visualizations, particle systems, game replays, generative art: if you can draw it on a canvas, you can export it.

[Live demo](#live-demo) · [Quick start](#quick-start) · [API](#api-reference) · [中文文档](README.zh-CN.md)

---

## Why this exists

Browsers give you no simple, reliable way to record a canvas animation to MP4.

- `MediaRecorder` records in **real time**, so a slow render drops frames and the output length drifts.
- `WebCodecs` is exact and offline, but wiring it to an MP4 muxer by hand (codec strings, keyframes, timestamps, even dimensions, main-thread yielding) is tedious and error-prone.

This library hides all of that behind a single call, and uses `MediaRecorder` as an automatic safety net on browsers or contexts where `WebCodecs` is unavailable.

## Features

- **One function.** `exportVideo({ render, width, height, fps, duration })` → done.
- **Dual engine.** `WebCodecs` for exact, offline, frame-accurate encoding; `MediaRecorder` as a real-time fallback. Auto-negotiated, or force either via `mode`.
- **Frame-accurate duration.** With WebCodecs, length is strictly `frames / fps` — render speed never loses frames.
- **Robust muxer loading.** `mp4-muxer` is loaded at export time across four CDNs (jsDelivr → unpkg → cdnjs → BootCDN), with global caching and retry.
- **Zero runtime dependencies.** Ships as UMD; works via `<script>`, bundler `import`/`require`, or inline.
- **Cancellation.** Pass an `AbortSignal`; abort throws a clean `AbortError`.
- **Sane defaults.** Even-dimension alignment, resolution-aware H.264 level, auto bitrate, periodic main-thread yields, per-frame `try/catch` so one bad frame never kills the whole export.
- **Progress + logs.** `onProgress(p, info)` for progress bars; `onLog(msg)` for diagnostics.

## Installation

There is no build step and no package to install. Pick one:

**1. `<script>` tag (simplest)** — download `video-exporter.js` and reference it:

```html
<script src="video-exporter.js"></script>
<!-- exposes window.exportVideo, window.VEX, window.VideoExporter -->
```

**2. Bundler (Vite / webpack / Rollup / esbuild)** — the UMD header is bundler-friendly:

```js
import { exportVideo } from './video-exporter.js';
// or: const { exportVideo } = require('./video-exporter.js');
```

**3. Inline** — paste the source into a `<script>` block for single-file projects.

> The only network dependency, `mp4-muxer`, is fetched automatically by the library the first time you export with WebCodecs. You can pre-include it globally or pass custom URLs via `muxerUrls`.

## Quick start

```js
import { exportVideo } from './video-exporter.js';

// Your animation depends only on ctx and t (seconds).
// The same function can drive a live preview AND the export.
function render(ctx, t) {
  const W = ctx.canvas.width, H = ctx.canvas.height;
  ctx.fillStyle = '#0e1014';
  ctx.fillRect(0, 0, W, H);                 // fill the frame (avoids black edges)

  ctx.fillStyle = '#f4b740';
  ctx.beginPath();
  ctx.arc(W / 2, H / 2, 60 + 20 * Math.sin(t * 3), 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#fff';
  ctx.font = '700 28px monospace';
  ctx.fillText(t.toFixed(2) + 's', 24, 40);
}

const ctrl = new AbortController();         // ctrl.abort() to cancel

const result = await exportVideo({
  width: 1080,
  height: 1920,
  fps: 30,
  duration: 4,
  render,
  onProgress: (p) => console.log((p * 100 | 0) + '%'),
  signal: ctrl.signal,
});

// result = { blob, filename, used, duration, width, height, frames }
// `download` defaults to true, so the file is already saved.
// Otherwise use result.blob yourself.
```

That's the whole contract: **draw the full frame, depend only on `ctx` and `t`, and avoid cross-origin images that taint the canvas.** Everything else is automatic.

## Live demo

Open `docs/index.html` (or the bundled documentation page) in any modern browser. The left pane is a real-time preview; the right pane lets you pick resolution / fps / duration / engine and export a real MP4 to verify which engine your browser used (`used` is shown in the status line).

> WebCodecs requires a **secure context** (`https://` or `localhost`). Opening the file via `file://` usually has no WebCodecs, so `auto` mode transparently falls back to `MediaRecorder`. Host it on any static provider for full quality.

## API reference

### `exportVideo(options) → Promise<Result>`

| Option | Type | Default | Description |
|---|---|---|---|
| `render` | `(ctx, t, frame, frames) => void` | **required** | Called once per frame. `t` is the current time in seconds (exactly `frame / fps` under WebCodecs). **Paint the whole canvas** and depend only on `ctx` and `t` so preview and export can share it. |
| `width` | `number` | **required** | Output width in px. Auto-rounded to even (H.264 requirement). |
| `height` | `number` | **required** | Output height in px. Auto-rounded to even. |
| `fps` | `number` | **required** | Frame rate, typically `24` / `30` / `60`. |
| `duration` | `number` | **required** | Total length in seconds. Total frames = `⌈duration · fps⌉`. |
| `mode` | `'auto' \| 'webcodecs' \| 'mediarecorder'` | `'auto'` | Try WebCodecs first, fall back to MediaRecorder. Force one engine if needed. |
| `bitrate` | `number` | auto (2–24 Mbps) | Target bitrate. Larger = sharper + bigger file. Auto value is derived from resolution × fps. |
| `codec` | `string` | auto | Force an H.264 codec string (e.g. `avc1.420028`). Leave unset to let the library pick a level by resolution. |
| `filename` | `string` | `'video-export'` | Download name; extension (`.mp4` / `.webm`) is appended per engine. |
| `download` | `boolean` | `true` | Set `false` to receive the blob without triggering a download. |
| `onProgress` | `(p, info) => void` | noop | `p` ∈ [0, 1]; `info` = `{ frame, frames, t, used }`. Use it to draw a progress bar. |
| `onLog` | `(msg) => void` | noop | Diagnostic messages (skipped frames, fallback reasons, etc.). |
| `signal` | `AbortSignal` | — | Abort to cancel; the promise rejects with `AbortError`. |
| `alpha` | `boolean` | `false` | Transparent background. H.264 has no alpha; enable only when targeting WebM/VP9. |
| `gop` | `number` | `fps * 2` | Keyframe interval in frames. |
| `yieldEvery` | `number` | `3` | Yield to the main thread every N frames so progress updates and the page stays responsive. |
| `muxerUrls` | `string[]` | built-in 4 CDNs | Custom `mp4-muxer` URLs. |

**`Result`** = `{ blob: Blob, filename: string, used: 'webcodecs' | 'mediarecorder', duration: number, width: number, height: number, frames: number }`

### Class & utilities

- `new VideoExporter(options).export()` — the class form behind `exportVideo`, useful if you want to inspect or reuse an instance.
- `VEX.isWebCodecsSupported()` — feature-detect before showing engine options in your UI.
- `VEX.loadMp4Muxer(urls?)` — preload / warm the muxer (cached globally).
- `VEX.pickRecorderMime()` — the container `MediaRecorder` will use on this browser.
- `VEX.defaultBitrate(w, h, fps)` — the auto-bitrate formula, if you want to display or tweak it.
- `VEX.even(n)` — round to even.

## How it works

```
render(ctx, t)            your drawing code (one frame at time t)
        │
        ▼
┌─────────────────────────────────────────────┐
│  mode = 'auto'                              │
│                                             │
│   WebCodecs? ──yes──►  offline loop:       │
│        │                for f in 0..frames: │
│        │                  t = f / fps       │  ← exact timeline
│        │                  render(ctx, t)    │
│        │                  encode(VideoFrame)│
│        │                  yield every N     │
│        │                flush → mux → .mp4  │
│        │                                    │
│        no / failed ──►  MediaRecorder:      │
│                          captureStream(fps) │
│                          wall-clock loop    │  ← real-time fallback
│                          render(ctx, t)     │
│                          stop at duration   │
│                          → .webm / .mp4     │
└─────────────────────────────────────────────┘
        │
        ▼
   download (or return blob)
```

- **WebCodecs path** renders off-screen at your exact resolution and feeds each frame to a `VideoEncoder`, then `mp4-muxer` produces an H.264 MP4 in memory. Duration is deterministic; a slow render just takes longer to encode, it never skips frames.
- **MediaRecorder path** records the canvas stream against the wall clock. It is the safety net: it works where WebCodecs does not, but a stuttering render can make the clip run short.
- A user abort never falls through to the fallback — it rejects immediately with `AbortError`.

## Caveats (read these)

- **Secure context.** WebCodecs is only available on `https://` or `localhost`. On `file://`, `auto` falls back to MediaRecorder (which may itself be limited). Deploy to any static host for full quality.
- **Cross-origin images taint the canvas.** A `drawImage` of a cross-origin image without CORS headers throws `SecurityError` / produces black frames. Use same-origin images, or images served with `crossOrigin = 'anonymous'` **and** a permissive server. The library `try/catch`s each frame, but tainting is canvas-level — keep your sources clean.
- **Fill the frame.** The library does not clear or fill the background for you (so it never clobbers a transparent intent). With `alpha: false`, unpainted pixels are black. Start `render` with a full-canvas `fillRect`.
- **Safari / older browsers.** `VideoEncoder` support arrived late on Safari; unsupported browsers auto-use MediaRecorder. Use `VEX.isWebCodecsSupported()` to gate UI hints.
- **Duration accuracy.** WebCodecs = exact. MediaRecorder = wall-clock (may run short under load). Prefer WebCodecs when frame accuracy matters.
- **H.264 = even dimensions, no alpha.** Width/height are auto-rounded; for transparency use WebM/VP9 with `alpha: true`.

## Browser support

| Engine | Requirement |
|---|---|
| WebCodecs (MP4) | Chromium-based browsers in a secure context; recent Safari |
| MediaRecorder (fallback) | Most evergreen browsers; container depends on the browser (MP4 or WebM) |

The library degrades gracefully: if neither path is viable, the promise rejects with a clear message rather than hanging.

## Project layout

```
video-exporter.js     the library (UMD, zero deps)
docs/index.html       documentation site + live export demo
LICENSE               MIT
README.md             this file (English)
README.zh-CN.md       中文文档
```

## License

[MIT](LICENSE) © your-name

`mp4-muxer` (used internally for the WebCodecs path) is © its authors under the MIT License.

---

Made for canvas animations that deserve a real video file.
