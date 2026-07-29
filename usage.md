我把导出逻辑彻底从「海报片场」里剥离了——它原本和 `SCENES`/`drawAt`/`LITE` 这些业务耦合在一起。剥离后只剩一个**与业务无关的核心抽象**：你给库一个 `render(ctx, t)` 回调（"第 t 秒画到 ctx 上"），库负责按 fps 遍历每一帧、编码、兜底、下载。库不关心你画的是海报、数据可视化还是游戏画面。

交付两部分：① 下面这段**关键代码速览**（带中文注释，节选自库，讲清原理）；② 一个**单文件文档站**——打开即是精美使用说明 + API 表 + 踩坑清单，并且**页面里 live 跑着一个真实导出 demo**（带实时预览、进度环、真能下载 MP4 验证）。库的完整源码内嵌在该文件里，点「复制 video-exporter.js」即可拷走接入任何项目，单一数据源、零不一致。

### 关键代码速览（节选自库，讲解用）

```js
/* ① mp4-muxer 多 CDN 动态加载 + 全局缓存，离线/失败由调用方兜底 */
let _muxerP = null;
function loadMp4Muxer(urls){
  if (_muxerP) return _muxerP;
  urls = urls || [
    'https://cdn.jsdelivr.net/npm/mp4-muxer@5.1.3/build/mp4-muxer.min.js',
    'https://unpkg.com/mp4-muxer@5.1.3/build/mp4-muxer.min.js',
    'https://cdnjs.cloudflare.com/ajax/libs/mp4-muxer/5.1.3/mp4-muxer.min.js',
    'https://cdn.bootcdn.net/ajax/libs/mp4-muxer/5.1.3/mp4-muxer.min.js'];
  return _muxerP = (async () => {
    if (typeof Mp4Muxer !== 'undefined') return Mp4Muxer;     // 已手动引入
    for (const u of urls) {                                    // 逐个 CDN 试，全失败才抛
      try { await new Promise((res, rej) => {
        const s = document.createElement('script');
        s.src = u; s.onload = res; s.onerror = rej; document.head.appendChild(s);
      }); if (typeof Mp4Muxer !== 'undefined') return Mp4Muxer; } catch (e) {}
    }
    throw new Error('mp4-muxer 加载失败（全部 CDN 不可达）');
  })();
}

/* ② WebCodecs 逐帧离线编码：时长精确 = frames/fps，不受实时性能影响 */
async _exportWC(canvas, ctx){
  const M = await loadMp4Muxer();
  const { fps, duration, bitrate, gop, yieldEvery } = this.o;
  const frames = Math.max(1, Math.ceil(duration * fps));
  const muxer = new M.Muxer({ target: new M.ArrayBufferTarget(),
    video: { codec: 'avc', width: canvas.width, height: canvas.height }, fastStart: 'in-memory' });
  const enc = new VideoEncoder({
    output: (chunk, meta) => muxer.addVideoChunk(chunk, meta),
    error:  e => { this._encError = e; } });                 // 编码异步错误存标志，循环里抛
  enc.configure({ codec: this._avcCodec, width: canvas.width, height: canvas.height,
    bitrate, framerate: fps });
  for (let f = 0; f < frames; f++){
    if (this._aborted) throw this._abortErr();               // 响应 AbortSignal
    if (this._encError) throw this._encError;
    const t = f / fps;                                       // ★ 时间由库精确控制
    try { this.o.render(ctx, t, f, frames); } catch (e) { this._log('render@'+f+' '+e.message); }
    const fr = new VideoFrame(canvas, { timestamp: Math.round(f*1e6/fps), duration: Math.round(1e6/fps) });
    try { enc.encode(fr, { keyFrame: f % gop === 0 }); } catch (e) { this._log('encode@'+f+' '+e.message); }
    fr.close();
    if (f % yieldEvery === 0){ this._progress(f/frames, {frame:f, frames, t, used:'webcodecs'});
      await new Promise(r => setTimeout(r, 0)); }            // 让出主线程，进度才更新、UI 不卡死
  }
  this._progress(1, {frame:frames, frames, t:duration, used:'webcodecs'});
  await enc.flush(); muxer.finalize(); enc.close();
  return new Blob([muxer.target.buffer], { type: 'video/mp4' });
}

/* ③ MediaRecorder 兜底：墙钟驱动，实时录制，输出 webm/mp4 */
_exportMR(canvas, ctx){
  return new Promise((resolve, reject) => {
    const mime = pickRecorderMime();                          // 探测浏览器支持的容器
    if (!mime) return reject(new Error('无可用 MediaRecorder 编码格式'));
    const mr = new MediaRecorder(canvas.captureStream(this.o.fps),
      { mimeType: mime, videoBitsPerSecond: this.o.bitrate });
    const chunks = [];
    mr.ondataavailable = e => { if (e.data.size) chunks.push(e.data); };
    mr.onerror = e => reject(e.error || new Error('MediaRecorder 错误'));
    mr.onstop = () => this._aborted
      ? reject(this._abortErr())
      : resolve(new Blob(chunks, { type: mime }));
    const wall = performance.now(); mr.start(100);
    const tick = () => {
      const t = (performance.now() - wall) / 1000;            // ★ 墙钟同步，禁被增量钳制
      if (this._aborted) { if (mr.state === 'recording') mr.stop(); return; }
      if (t >= this.o.duration){ try{ this.o.render(ctx, this.o.duration); }catch(e){}
        if (mr.state === 'recording') mr.stop(); return; }
      try { this.o.render(ctx, t); } catch (e) { this._log('render '+e.message); }
      this._progress(t/this.o.duration, { t, used:'mediarecorder' });
      requestAnimationFrame(tick);
    };
    tick();
  });
}
```

主入口 `exportVideo(opts)` 只做一件事：`new VideoExporter(opts).export()`，内部按 `mode`（`auto`/`webcodecs`/`mediarecorder`）先试 WebCodecs，失败且非用户取消时回退 MediaRecorder，最后按 `download` 触发下载并返回 `{blob, filename, used, duration, frames}`。完整实现（含偶数尺寸对齐、按分辨率自动选 H.264 level、按像素自动算码率、AbortSignal 取消、单帧 try/catch 防一帧炸全片）都在下面文件的 `video-exporter.js` 里。

### 完整工具库 + 文档站 + 可运行 demo（单文件，打开即用）

把下面整段存为 `video-exporter-doc.html` 双击打开：阅读文档、点「复制 video-exporter.js」拿走库、在页面里直接跑 demo 验证导出。



**怎么用这份交付**

- **拿库**：打开上面的 HTML，滚到「00:05 三种接入方式」，点 **复制 video-exporter.js** → 存成 `video-exporter.js` → 在你的项目里 `<script src>` 或 `import { exportVideo }` 即可。库是 UMD、零依赖，唯一外部依赖 `mp4-muxer` 由库在导出时按四 CDN 自动加载（也支持你预先全局引入或传 `muxerUrls`）。
- **看原理**：页面里 `render` 的最小实现、双引擎编码循环、多 CDN 加载都已用中文注释标在速览和源码里。
- **验证**：页面「00:02 Live Demo」左边实时预览、右边调参点导出，会真下载一个 mp4——证明库在你当前浏览器/协议下走的是 WebCodecs 还是回退到了 MediaRecorder（状态行会写明 `used`）。

**接入时唯一要记住的契约**：`render(ctx, t)` 里**画满画布**、**只依赖 `ctx` 和 `t`**、**别画跨域污染图**。满足这三条，库剩下的（偶数对齐、level 选择、码率、关键帧、让出主线程、单帧容错、取消、兜底、下载）全部自动。需要把现有「可见 canvas」直接录下来而不是用回调？把 `render` 写成 `(ctx,t)=>ctx.drawImage(yourCanvas,0,0)` 即可——库用的是它自己建的离屏画布，不会动你的 DOM。
