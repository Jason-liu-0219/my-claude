# VSaaS Portal - WebCodecs Player

WebCodecs API 播放模組，使用硬體加速解碼，提供更低延遲的播放體驗。

## 模組概述

### 職責
- 使用 WebCodecs VideoDecoder 進行硬體解碼
- 管理 MediaStreamTrackGenerator 或 Canvas 渲染
- 支援單幀前進（step mode）
- 支援暫停/恢復解碼管線

### 相較 MSE 的優勢
- 更低延遲（無需 fMP4 容器封裝）
- 更精確的幀控制
- 硬體加速解碼
- 更好的 H.265 支援

---

## 檔案結構

```
packages/lib-rtsp-player/src/
├── media/
│   ├── WebCodecsController.js       # WebCodecs 解碼控制器
│   └── WebCodecsAudioController.js  # WebCodecs 音訊解碼
├── player/
│   ├── WebCodecsVideoPlayer.js      # WebCodecs 影片播放器
│   └── WebCodecsAudioPlayer.js      # WebCodecs 音訊播放器
├── handler/
│   └── WebCodecsStreamProcessor.js  # WebCodecs 串流管線
├── workers/
│   ├── WebCodecsWorkerProxy.js      # Worker 代理
│   └── web-codecs-decoder.worker.js # 解碼 Worker
└── utils/
    ├── WebCodecsHelper.js           # 輔助函式
    ├── StrategyHelper.js            # 策略選擇
    └── StreamClock.js               # 串流時鐘
```

---

## 架構圖

```
Remuxer (H264/H265)
     │
     ▼
WebCodecsStreamProcessor
     │
     ├─── 不需要 SSMUX！直接處理 NALU
     │
     ▼
┌─────────────────────────────────────────────────────┐
│ WebCodecsVideoPlayer                                 │
│   ├── WebCodecsController                           │
│   │     ├── VideoDecoder (WebCodecs API)            │
│   │     └── MediaStreamTrackGenerator               │
│   └── StreamClock (時間同步)                        │
└─────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────┐
│ <video> element                                      │
│   ├── srcObject = MediaStream                       │
│   └── 播放解碼後的 VideoFrame                       │
└─────────────────────────────────────────────────────┘
```

---

## 核心類別

### WebCodecsController

管理 VideoDecoder 和渲染管線。

```javascript
// packages/lib-rtsp-player/src/media/WebCodecsController.js

class WebCodecsController extends EventEmitter {
  constructor(video, codec, mediaCodec, useCanvasRenderer) {
    this.video = video;
    this.codec = codec;
    this.mediaCodec = mediaCodec;
    
    this.decoder = null;           // VideoDecoder
    this.generator = null;         // MediaStreamTrackGenerator
    this.writable = null;          // WritableStreamDefaultWriter
    
    // 狀態
    this.waitingForKeyframe = true;
    this.isCodecReady = false;
    this.isPlaybackReady = false;
    
    // 暫停/單步控制
    this.isPaused = false;
    this.queue = [];               // 待解碼的 chunks
    this.frameBudget = Infinity;   // Infinity=連續, N>0=單步模式
  }
}
```

#### 解碼管線

```javascript
async createWebCodecsPipeline() {
  // 1. 建立 VideoDecoder
  this.decoder = new VideoDecoder({
    output: this.handleVideoFrame.bind(this),
    error: (e) => this.emit('error', e)
  });

  // 2. 建立 MediaStreamTrackGenerator（或 Canvas）
  if (this.useCanvasRenderer) {
    this.setupCanvasRenderer();
  } else {
    this.generator = new MediaStreamTrackGenerator({ kind: 'video' });
    this.writable = this.generator.writable.getWriter();
    
    const stream = new MediaStream([this.generator]);
    this.video.srcObject = stream;
  }
}
```

#### 幀處理

```javascript
handleVideoFrame(frame) {
  // 跳過 preroll 幀
  if (this.chunkPreRoll.get(frame.timestamp)) {
    frame.close();
    return;
  }

  // 渲染到 generator 或 canvas
  if (this.useCanvasRenderer) {
    this.ctx2d.drawImage(frame, 0, 0);
  } else {
    this.writable.write(frame);
  }
  
  this.emit('presented-ts', frame.timestamp / 1000);
  frame.close();  // 必須關閉以釋放資源

  // 單步模式：計算剩餘 budget
  if (this.frameBudget !== Infinity) {
    this.frameBudget--;
    if (this.frameBudget === 0) {
      this.onStepDone?.();
    }
  }
}
```

---

### WebCodecsVideoPlayer

整合 WebCodecsController 的播放器。

```javascript
// packages/lib-rtsp-player/src/player/WebCodecsVideoPlayer.js

class WebCodecsVideoPlayer extends EventEmitter {
  constructor({ mode, codec, mediaCodec, pipelineMode }) {
    this.mediaController = null;
    this.streamClock = new StreamClock();
    
    this.createVideo();
    this.initMediaController(codec, mediaCodec, mode);
    
    // 綁定時鐘 tick 到 timeupdate
    this.streamClock.on('tick', (nowMs) => this.pumpTimeupdate(nowMs));
    this.streamClock.bindController(this.mediaController);
  }
}
```

#### 時間同步

```javascript
// WebCodecs 沒有 video.currentTime，使用 StreamClock
get currentTime() {
  return (this.streamClock.getNowMs() ?? 0) / 1000;
}
```

---

## 策略選擇

### decideRtspEntryStrategy

```javascript
// packages/lib-rtsp-player/src/utils/StrategyHelper.js

export const STREAMING_PROCESSOR_MODE = {
  WEB_CODECS_WORKER: 'WebCodecs+Worker',
  WEB_CODECS: 'WebCodecs',
  MSE_WORKER: 'MSE+Worker',
  MSE: 'MSE',
};

export function decideRtspEntryStrategy(feature = {}) {
  const isWorkerSupported = feature.useWorker && hasWorkerSupport();
  const isWebCodecsSupported = feature.useWebCodecs && hasWebCodecsAPI();

  if (isWebCodecsSupported) {
    return isWorkerSupported 
      ? STREAMING_PROCESSOR_MODE.WEB_CODECS_WORKER 
      : STREAMING_PROCESSOR_MODE.WEB_CODECS;
  }
  
  if (hasManagedMediaSource()) {
    return STREAMING_PROCESSOR_MODE.MSE;  // iOS Safari
  }
  
  return isWorkerSupported 
    ? STREAMING_PROCESSOR_MODE.MSE_WORKER 
    : STREAMING_PROCESSOR_MODE.MSE;
}
```

### 環境檢測

```javascript
function hasWebCodecsAPI({ requireAudio = true } = {}) {
  const hasVideoDecode = 
    typeof VideoDecoder !== 'undefined' &&
    typeof EncodedVideoChunk !== 'undefined' &&
    typeof VideoFrame !== 'undefined';

  const hasAudioDecode = !requireAudio || (
    typeof AudioDecoder !== 'undefined' &&
    typeof EncodedAudioChunk !== 'undefined'
  );

  return hasVideoDecode && hasAudioDecode;
}
```

---

## 單步模式（Step Mode）

### 控制 API

```javascript
// 暫停解碼輸入
pauseDecoder() {
  this.isPaused = true;
}

// 恢復連續解碼
resumeDecoder() {
  this.isPaused = false;
  this.frameBudget = Infinity;
  this.drainQueue(false);
}

// 解碼一幀
decodeOneFrame() {
  this.isPaused = true;
  this.frameBudget = 1;
  return new Promise((resolve) => {
    this.onStepDone = resolve;
    this.drainQueue(true);
  });
}
```

### 佇列排空

```javascript
drainQueue(stepMode = false) {
  while (this.queue.length && (!this.isPaused || this.frameBudget > 0)) {
    this.decoder.decode(this.queue.shift());
    if (stepMode) break;  // 單步模式只處理一個
  }
}
```

---

## Canvas 渲染

當 `MediaStreamTrackGenerator` 不支援時使用 Canvas。

```javascript
setupCanvasRenderer() {
  this.canvas = document.createElement('canvas');
  this.ctx2d = this.canvas.getContext('2d', { alpha: false });
  
  // 嘗試使用 captureStream
  if (typeof this.canvas.captureStream === 'function') {
    this.canvasStream = this.canvas.captureStream();
    this.video.srcObject = this.canvasStream;
  } else {
    // 直接顯示 canvas，隱藏 video
    const parent = this.video.parentNode;
    parent.insertBefore(this.canvas, this.video);
    this.video.style.display = 'none';
  }
}
```

---

## Decoder 配置

```javascript
async tryConfigureDecoder({ height, width }) {
  const config = await getVideoDecoderSupportedConfig({
    codec: this.mediaCodec,
    codedWidth: width,
    codedHeight: height,
  });
  
  if (!config) {
    throw new Error(`codec: ${this.codec} is not supported`);
  }

  this.decoder.configure(config);
  this.isCodecReady = true;
}
```

### getVideoDecoderSupportedConfig

```javascript
// packages/lib-rtsp-player/src/utils/WebCodecsHelper.js

export async function getVideoDecoderSupportedConfig(config) {
  const { supported } = await VideoDecoder.isConfigSupported(config);
  return supported ? config : null;
}
```

---

## EncodedVideoChunk

```javascript
appendPacket(packet) {
  const { data, timestamp, isKeyframe } = packet;
  
  // 等待 keyframe
  if (!isKeyframe && this.waitingForKeyframe) {
    return;
  }

  const chunk = new EncodedVideoChunk({
    type: isKeyframe ? 'key' : 'delta',
    timestamp: timestamp * 1000,  // 轉換為微秒
    data,
  });
  
  this.queue.push(chunk);
  this.drainQueue();
}
```

---

## Worker 模式

### WebCodecsWorkerProxy

```javascript
// packages/lib-rtsp-player/src/workers/WebCodecsWorkerProxy.js

class WebCodecsWorkerProxy extends EventEmitter {
  constructor(video, codec, mediaCodec, useCanvasRenderer) {
    this.worker = new WebCodecsDecoderWorker();
    this.worker.onmessage = this.handleMessage.bind(this);
    
    // 使用 OffscreenCanvas 進行渲染
    const offscreen = video.transferControlToOffscreen();
    this.worker.postMessage({ 
      type: 'init', 
      canvas: offscreen 
    }, [offscreen]);
  }

  appendPacket(packet) {
    this.worker.postMessage({
      type: 'decode',
      data: packet.data,
      timestamp: packet.timestamp,
      isKeyframe: packet.isKeyframe
    }, [packet.data.buffer]);  // Transferable
  }
}
```

---

## StreamClock

由於 WebCodecs 沒有內建的 `currentTime`，需要自己維護時鐘。

```javascript
// packages/lib-rtsp-player/src/utils/StreamClock.js

class StreamClock extends EventEmitter {
  constructor() {
    this.baseTime = null;      // 串流起始時間
    this.lastPresentedTs = 0;  // 最後渲染的時間戳
  }

  getNowMs() {
    return this.lastPresentedTs;
  }

  bindController(controller) {
    controller.on('presented-ts', (ts) => {
      this.lastPresentedTs = ts;
      this.emit('tick', ts);
    });
  }

  reset() {
    this.baseTime = null;
    this.lastPresentedTs = 0;
  }
}
```

---

## 錯誤處理

```javascript
this.decoder = new VideoDecoder({
  output: this.handleVideoFrame.bind(this),
  error: (e) => {
    Log.error('[WebCodec] decode error:', e);
    this.emit('error', e);
  }
});
```

### 常見錯誤
- `NotSupportedError`: Codec 不支援
- `DataError`: 資料格式錯誤
- `AbortError`: 解碼中斷

---

## 資源釋放

```javascript
destroy() {
  // 1. 關閉 decoder
  if (this.decoder) {
    this.decoder.close();
    this.decoder = null;
  }

  // 2. 關閉 writable
  if (this.writable) {
    this.writable.close();
    this.writable = null;
  }

  // 3. 停止 generator
  if (this.generator) {
    this.generator.stop();
    this.generator = null;
  }

  // 4. 清理 video
  this.video.srcObject = null;
}
```

---

## MSE vs WebCodecs 比較

| 特性 | MSE | WebCodecs |
|------|-----|-----------|
| 延遲 | 較高（需要緩衝） | 較低 |
| 幀控制 | 無 | 精確控制 |
| H.265 支援 | 瀏覽器依賴 | 較好 |
| 封裝格式 | 需要 fMP4 | 原始 NALU |
| 瀏覽器支援 | 廣泛 | Chrome/Edge |
| 音訊同步 | 自動 | 需要手動 |

---

## 常見問題

### 1. 畫面不顯示
- 確認 `waitingForKeyframe` 狀態
- 檢查 decoder 是否已 configure
- 確認 `frame.close()` 有呼叫

### 2. 記憶體洩漏
- 確保每個 VideoFrame 都有 `close()`
- 檢查 EncodedVideoChunk 是否堆積

### 3. 時間不同步
- 檢查 StreamClock 綁定
- 確認 timestamp 單位（微秒 vs 毫秒）

### 4. 單步模式卡住
- 確認 frameBudget 邏輯
- 檢查 drainQueue 是否被呼叫

---

## 相關模組

- `vsaas-mse-player` - MSE 播放器
- `vsaas-video-remuxing` - 影片 Remuxing
- `vsaas-streaming-strategy` - 策略選擇
