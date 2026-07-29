# video-exporter.js

[![License: MIT](https://img.shields.io/badge/许可证-MIT-f4b740.svg)](LICENSE)
[![Dependencies](https://img.shields.io/badge/依赖-无-34d3c0.svg)](#安装)
[![Engines](https://img.shields.io/badge/引擎-WebCodecs%20%2B%20MediaRecorder-9d8cff.svg)](#工作原理)
[![Format](https://img.shields.io/badge/格式-UMD%20%2F%20打包器%20ESM-69727f.svg)](#安装)

> 把**任意** `<canvas>` 动画导出成高清 MP4——一个函数搞定。

`video-exporter.js` 是一个零依赖的小库：逐帧录制 canvas 动画并编码成视频。你只需给一个 `render(ctx, t)` 回调，画出第 `t` 秒那一帧；剩下的时间轴驱动、编码、兜底、触发下载，全交给库。

它**与任何 UI、任何内容都解耦**——金句海报、数据可视化、粒子、游戏回放、生成艺术，凡是能画到 canvas 上的，都能导出。

[在线演示](#在线演示) · [快速开始](#快速开始) · [API 参考](#api-参考) · [English](README.md)

---

## 为什么要有它

浏览器没有一条简单可靠的路，把 canvas 动画录成 MP4。

- `MediaRecorder` 是**实时**录制，渲染一慢就丢帧，成片时长也会漂。
- `WebCodecs` 精确、离线，但自己接 MP4 封装器很繁琐——codec 字符串、关键帧、时间戳、偶数宽高、让出主线程，处处是坑。

这个库把这一切收进一次调用，并在 WebCodecs 不可用的浏览器或环境里，自动用 `MediaRecorder` 兜底。

## 特性

- **一个函数。** `exportVideo({ render, width, height, fps, duration })`，完事。
- **双引擎。** `WebCodecs` 逐帧离线、帧级精确；`MediaRecorder` 实时兜底。自动协商，也能用 `mode` 强制指定。
- **时长精确。** 走 WebCodecs 时，时长严格等于 `帧数 / fps`——渲染再慢也不丢帧。
- **封装器加载稳。** `mp4-muxer` 在导出时按四个 CDN 加载（jsDelivr → unpkg → cdnjs → BootCDN），全局缓存、失败可重试。
- **零运行时依赖。** 以 UMD 形式发布，`<script>` 引入、打包器 `import`/`require`、内联都行。
- **可取消。** 传一个 `AbortSignal`，取消时干净地抛 `AbortError`。
- **默认值省心。** 偶数宽高自动对齐、按分辨率选 H.264 level、按像素算码率、定期让出主线程、单帧 `try/catch`——一帧画炸不会拖垮整段导出。
- **进度与日志。** `onProgress(p, info)` 画进度条，`onLog(msg)` 看诊断信息。

## 安装

没有构建步骤，也没有要装的包。三选一：

**1. `<script>` 标签（最省事）** —— 下载 `video-exporter.js` 直接引用：

```html
<script src="video-exporter.js"></script>
<!-- 暴露 window.exportVideo、window.VEX、window.VideoExporter -->
```

**2. 打包器（Vite / webpack / Rollup / esbuild）** —— UMD 头对打包器友好：

```js
import { exportVideo } from './video-exporter.js';
// 或：const { exportVideo } = require('./video-exporter.js');
```

**3. 内联** —— 单文件项目可直接把源码贴进 `<script>`。

> 唯一的网络依赖 `mp4-muxer`，会在你首次用 WebCodecs 导出时由库自动拉取。你也可以预先全局引入，或用 `muxerUrls` 传自定义地址。

## 快速开始

```js
import { exportVideo } from './video-exporter.js';

// 你的动画只依赖 ctx 和 t（秒）。
// 同一个函数既能驱动实时预览，也能驱动导出。
function render(ctx, t) {
  const W = ctx.canvas.width, H = ctx.canvas.height;
  ctx.fillStyle = '#0e1014';
  ctx.fillRect(0, 0, W, H);                 // 铺满画布，避免黑边

  ctx.fillStyle = '#f4b740';
  ctx.beginPath();
  ctx.arc(W / 2, H / 2, 60 + 20 * Math.sin(t * 3), 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#fff';
  ctx.font = '700 28px monospace';
  ctx.fillText(t.toFixed(2) + 's', 24, 40);
}

const ctrl = new AbortController();         // 想取消就 ctrl.abort()

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
// download 默认 true，文件已经存好了；
// 否则自己用 result.blob 处理。
```

整个契约就三句话：**画满整帧、只依赖 `ctx` 和 `t`、别画会污染画布的跨域图。** 其余全自动。

## 在线演示

用现代浏览器打开 `docs/index.html`（或随附的文档页）。左栏是实时预览，右栏可调分辨率 / 帧率 / 时长 / 引擎，并真导出一个 MP4，验证你的浏览器走了哪个引擎（状态行会写明 `used`）。

> WebCodecs 需要**安全上下文**（`https://` 或 `localhost`）。用 `file://` 双击打开通常没有 WebCodecs，此时 `auto` 模式会自动回退到 `MediaRecorder`。部署到任意静态托管即可满血。

## API 参考

### `exportVideo(options) → Promise<Result>`

| 选项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `render` | `(ctx, t, frame, frames) => void` | **必填** | 每帧调用一次。`t` 为当前秒数（WebCodecs 下精确等于 `frame / fps`）。**请画满整个画布**，且只依赖 `ctx` 与 `t`，以便预览与导出共用。 |
| `width` | `number` | **必填** | 输出宽（像素），自动对齐到偶数（H.264 硬性要求）。 |
| `height` | `number` | **必填** | 输出高（像素），自动对齐到偶数。 |
| `fps` | `number` | **必填** | 帧率，常用 `24` / `30` / `60`。 |
| `duration` | `number` | **必填** | 总时长（秒）。总帧数 = `⌈duration · fps⌉`。 |
| `mode` | `'auto' \| 'webcodecs' \| 'mediarecorder'` | `'auto'` | 先试 WebCodecs，失败回退 MediaRecorder；也可强制指定其一。 |
| `bitrate` | `number` | 自动（2–24 Mbps） | 目标码率，越大越清晰、文件越大。默认按 分辨率×帧率 算。 |
| `codec` | `string` | 自动 | 强制 H.264 codec 串（如 `avc1.420028`）。不填则由库按分辨率选 level。 |
| `filename` | `string` | `'video-export'` | 下载文件名；扩展名（`.mp4` / `.webm`）按引擎自动加。 |
| `download` | `boolean` | `true` | 设 `false` 则只返回 blob，不触发下载。 |
| `onProgress` | `(p, info) => void` | 空函数 | `p` ∈ [0, 1]；`info` = `{ frame, frames, t, used }`，用来画进度条。 |
| `onLog` | `(msg) => void` | 空函数 | 诊断信息（跳过的帧、回退原因等）。 |
| `signal` | `AbortSignal` | — | 取消导出，Promise 以 `AbortError` 拒绝。 |
| `alpha` | `boolean` | `false` | 透明背景。H.264 不含透明；只有目标为 WebM/VP9 时才开。 |
| `gop` | `number` | `fps * 2` | 关键帧间隔（帧数）。 |
| `yieldEvery` | `number` | `3` | 每 N 帧让出一次主线程，进度才刷新、页面不卡死。 |
| `muxerUrls` | `string[]` | 内置四 CDN | 自定义 `mp4-muxer` 地址。 |

**`Result`** = `{ blob: Blob, filename: string, used: 'webcodecs' | 'mediarecorder', duration: number, width: number, height: number, frames: number }`

### 类与工具函数

- `new VideoExporter(options).export()` —— `exportVideo` 背后的类形式，想检查或复用实例时用。
- `VEX.isWebCodecsSupported()` —— 在 UI 里展示引擎选项前先做能力检测。
- `VEX.loadMp4Muxer(urls?)` —— 预热 / 预加载封装器（全局缓存）。
- `VEX.pickRecorderMime()` —— 当前浏览器下 MediaRecorder 会用的容器格式。
- `VEX.defaultBitrate(w, h, fps)` —— 自动码率公式，想展示或微调时用。
- `VEX.even(n)` —— 对齐到偶数。

## 工作原理

```
render(ctx, t)            你的绘制代码（画出第 t 秒那一帧）
        │
        ▼
┌─────────────────────────────────────────────┐
│  mode = 'auto'                              │
│                                             │
│   有 WebCodecs? ──是──►  离线循环：         │
│        │                for f in 0..frames: │
│        │                  t = f / fps       │  ← 精确时间轴
│        │                  render(ctx, t)    │
│        │                  encode(VideoFrame)│
│        │                  每 N 帧让出主线程 │
│        │                flush → 封装 → .mp4 │
│        │                                    │
│        否 / 失败 ──►  MediaRecorder：       │
│                          captureStream(fps) │
│                          墙钟循环           │  ← 实时兜底
│                          render(ctx, t)     │
│                          撞线即 stop        │
│                          → .webm / .mp4     │
└─────────────────────────────────────────────┘
        │
        ▼
   下载（或返回 blob）
```

- **WebCodecs 路径**在你指定的分辨率下离屏渲染，逐帧喂给 `VideoEncoder`，再由 `mp4-muxer` 在内存里产出 H.264 MP4。时长是确定的：渲染慢只会让编码更久，绝不跳帧。
- **MediaRecorder 路径**按墙钟录制画布流，是兜底方案——WebCodecs 没有的地方它能顶上，但渲染卡顿会让成片偏短。
- 用户取消**不会**穿透到兜底，而是立刻以 `AbortError` 拒绝。

## 注意事项（务必看）

- **安全上下文。** WebCodecs 只在 `https://` 或 `localhost` 可用。`file://` 下 `auto` 会回退 MediaRecorder（它本身也可能受限）。部署到任意静态托管才能满血。
- **跨域图会污染画布。** 用 `drawImage` 画一张没有 CORS 头的跨域图，会抛 `SecurityError` 或出黑帧。请用同源图，或带 `crossOrigin = 'anonymous'` **且**服务端放行的图。库对每帧 `try/catch`，但污染是画布级的——源头要干净。
- **画满整帧。** 库不会替你清屏或填底（免得盖掉透明意图）。`alpha: false` 时没画的区域是黑边，请在 `render` 开头 `fillRect` 铺满。
- **Safari / 老浏览器。** Safari 的 `VideoEncoder` 支持来得晚；不支持的浏览器自动走 MediaRecorder。用 `VEX.isWebCodecsSupported()` 给 UI 做提示。
- **时长精度。** WebCodecs 精确，MediaRecorder 走墙钟（负载高时可能偏短）。要帧级精确，优先 WebCodecs。
- **H.264 = 偶数宽高、无透明。** 宽高自动对齐；要透明背景请用 WebM/VP9 并开 `alpha: true`。

## 浏览器支持

| 引擎 | 条件 |
|---|---|
| WebCodecs（MP4） | 安全上下文下的 Chromium 系浏览器；较新的 Safari |
| MediaRecorder（兜底） | 多数常青浏览器；容器格式随浏览器而定（MP4 或 WebM） |

库会优雅降级：两条路都走不通时，Promise 给出明确报错，而不是卡死。

## 目录结构

```
video-exporter.js     库本体（UMD，零依赖）
docs/index.html       文档站 + 可运行导出演示
LICENSE               MIT
README.md             英文文档
README.zh-CN.md       本文件（中文）
```

## 许可证

[MIT](LICENSE) © 你的名字

库内部 WebCodecs 路径用到的 `mp4-muxer`，版权归其作者所有，采用 MIT 许可证。

---

为那些值得一个真正视频文件的 canvas 动画而做。