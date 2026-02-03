# VSaaS Portal - MSE Player

MediaSource Extensions (MSE) 播放模組，處理 fMP4 封包的緩衝與播放。

## 模組概述

### 職責
- 管理 MediaSource 與 SourceBuffer 生命週期
- 緩衝區管理（append / remove）
- 動態播放速率調整（追趕即時串流）
- Liveview / Playback 模式支援
- 幀率計算與通知事件

---

## 檔案結構

```
packages/lib-rtsp-player/src/
├── media/
│   └── MediaSourceController.js     # MediaSource/SourceBuffer 控制器
├── player/
│   ├── VideoPlayer.js               # MSE 影片播放器
│   ├── AudioPlayer.js               # PCM 音訊播放器
│   ├── AACPlayer.js                 # AAC 音訊播放器
│   ├── WebCodecsVideoPlayer.js      # WebCodecs 影片播放器
│   └── WebCodecsAudioPlayer.js      # WebCodecs 音訊播放器
├── handler/
│   ├── StreamProcessor.js           # 串流管線協調器
│   └── NotifyHandler.js             # 通知事件管理
├── workers/
│   ├── MediaSourceWorkerProxy.js    # Worker 代理
│   └── media-source.worker.js       # MediaSource Worker
└── utils/
    ├── FrameRateCalculator.js       # 幀率計算
    └── StrategyHelper.js            # 策略選擇輔助
```

---

## 架構圖

```
Remuxer (H264/H265)
     │
     ▼
StreamProcessor
     │
     ├─── inputPacket() ──→ SSMUX (WASM)
     │                         │
     │                         ├─ processInfo() → streamInfo
     │                         ├─ processVideoEvent() → fMP4 packet
     │                         └─ processAudioEvent() → PCM data
     │
     ▼
┌─────────────────────────────────────────────────────┐
│ VideoPlayer                                          │
│   ├── createVideo() → <video> element              │
│   ├── MediaSourceController                         │
│   │     ├── MediaSource                             │
│   │     └── SourceBuffer                            │
│   └── appendPacket() → SourceBuffer.appendBuffer() │
└─────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────┐
│ <video> element                                      │
│   ├── src = MediaSource URL                         │
│   └── 播放 fMP4 封包                                │
└─────────────────────────────────────────────────────┘
```

---

## 核心類別

### MediaSourceController

管理 MediaSource 和 SourceBuffer 的生命週期與緩衝策略。

```javascript
// packages/lib-rtsp-player/src/media/MediaSourceController.js

class MediaSourceController extends EventEmitter {
  constructor(video, mode = 'liveview', meta = {}) {
    this.video = video;
    this.codec = meta.codec;
    this.mediaCodec = meta.mediaCodec;  // RFC 6381 格式
    
    this.mediaSource = null;
    this.sourceBuffer = null;
    this.queue = [];                    // 待 append 的封包佇列
    
    // 緩衝區管理參數
    this.KEEP_PASSED_LENGTH = 20;        // 保留的已播放秒數
    this.SOURCEBUFFER_LOWER_LENGTH = 0.8;
    this.SOURCEBUFFER_UPPER_LENGTH = 1.8;
    this.SOURCEBUFFER_LIMITED_LENGTH = 3.6;
  }
}
```

#### 關鍵屬性

| 屬性 | 說明 |
|------|------|
| `currentTime` | 目前播放時間（秒） |
| `bufferedLength` | currentTime 之後的緩衝長度 |
| `bufferedEnd` | SourceBuffer 緩衝區結束時間 |
| `canAppendBuffer` | 是否可以 append 資料 |

#### 關鍵方法

| 方法 | 說明 |
|------|------|
| `createMediaSource()` | 建立 MediaSource 並綁定到 video |
| `setCodec(codec, mediaCodec)` | 切換 codec，重建 SourceBuffer |
| `appendPacket(packet)` | 添加封包到 SourceBuffer |
| `updateBufferStrategy()` | 執行緩衝策略（速率調整、清理） |
| `seekBufferedEnd()` | 跳到緩衝區末端（追趕即時串流） |

---

### VideoPlayer

整合 MediaSourceController 與 `<video>` 元素的播放控制。

```javascript
// packages/lib-rtsp-player/src/player/VideoPlayer.js

class VideoPlayer extends EventEmitter {
  constructor({ mode, codec, mediaCodec, pipelineMode }) {
    this.isLiveview = mode === 'liveview';
    this.isPaused = false;
    this.isPlaying = false;
    
    this.video = null;
    this.fps = 0;
    
    this.createVideo();
    this.$initMediaController(codec, mediaCodec, mode);
    
    this.notifyHandler = new NotifyHandler();
    this.frameRateCalculator = new FrameRateCalculator();
  }
}
```

#### 事件監聽

| 事件 | 處理方法 | 說明 |
|------|----------|------|
| `play` | `onVideoPlay()` | 開始播放 |
| `pause` | `onVideoPause()` | 暫停播放 |
| `timeupdate` | `onVideoTimeupdate()` | 時間更新 |
| `loadeddata` | `onVideoLoadeddata()` | 資料載入完成 |
| `error` | `onVideoError()` | 播放錯誤 |
| `resize` | `onVideoResize()` | 尺寸變化 |

#### 關鍵方法

| 方法 | 說明 |
|------|------|
| `play(instantPlay)` | 開始播放 |
| `pause()` | 暫停並記錄位置 |
| `resume()` | 從暫停位置恢復 |
| `stop()` | 停止但不釋放資源 |
| `nextFrame()` | 前進一幀 |
| `cutover()` | 重置並跳到緩衝區末端 |
| `destroy()` | 完全釋放資源 |

---

### StreamProcessor

串流管線協調器，管理 SSMUX、Remuxer、Player 之間的協調。

```javascript
// packages/lib-rtsp-player/src/handler/StreamProcessor.js

class StreamProcessor extends EventEmitter {
  constructor({ mode, ssmuxType, rtspClient, ...playerOptions }) {
    this.mode = mode;
    
    // 建立 remuxer
    this.remuxer = new Remuxer(this, rtspClient);
    
    // 建立 SSMUX (WASM 解碼器)
    this.createStreamProcessor(ssmuxType);
    
    // 播放器
    this.streamPlayer = null;  // VideoPlayer
    this.audioPlayer = null;   // AudioPlayer
  }
}
```

#### SSMUX 回呼

| 回呼 | 說明 |
|------|------|
| `processInfo(info)` | 接收串流資訊（時間戳、fisheye 等） |
| `processVideoEvent(packet)` | 接收 fMP4 影片封包 |
| `processAudioEvent(packet)` | 接收 PCM 音訊封包 |
| `processM4AEvent(packet)` | 接收 M4A 音訊封包 |

---

## MediaSource 生命週期

```
1. createMediaSource()
   │
   ├── new MediaSource() 或 new ManagedMediaSource()
   │
   ├── URL.createObjectURL(mediaSource)
   │
   └── video.src = objectURL
          │
          ▼
2. 'sourceopen' event
   │
   ├── mediaSource.addSourceBuffer(mimeType)
   │
   └── 開始接收封包
          │
          ▼
3. appendPacket(packet)
   │
   ├── canAppendBuffer? → sourceBuffer.appendBuffer(packet)
   │
   └── 否則加入 queue
          │
          ▼
4. 'updateend' event
   │
   ├── 處理 queue 中的封包
   │
   ├── 檢查 bufferedLength
   │
   └── 必要時觸發 playbackReady / seekBufferedEnd
          │
          ▼
5. destroy()
   │
   ├── sourceBuffer.abort()
   │
   ├── URL.revokeObjectURL()
   │
   └── 清理資源
```

---

## 緩衝區策略

### 動態播放速率

```javascript
adjustPlaybackRate() {
  if (!this.isLiveview) return;
  
  const { bufferedLength } = this;
  
  // 緩衝區過長 → 加速播放
  if (bufferedLength > SOURCEBUFFER_UPPER_LENGTH) {
    video.playbackRate = 1.5;
  }
  // 緩衝區恢復正常 → 正常速度
  else if (bufferedLength < SOURCEBUFFER_LOWER_LENGTH) {
    video.playbackRate = 1;
  }
}
```

### 緩衝區清理

```javascript
cutSourceBufferHead() {
  const { currentTime, sourceBufferStart, KEEP_PASSED_LENGTH } = this;
  
  // 已播放超過 40 秒 → 清理前 20 秒
  if (currentTime - sourceBufferStart > KEEP_PASSED_LENGTH * 2) {
    sourceBuffer.remove(sourceBufferStart, sourceBufferStart + KEEP_PASSED_LENGTH);
    this.sourceBufferStart += KEEP_PASSED_LENGTH;
  }
}
```

### 追趕即時串流

```javascript
// 緩衝區超過限制 → 跳到最新
if (isLiveview && bufferedLength > SOURCEBUFFER_LIMITED_LENGTH) {
  this.seekBufferedEnd();
}
```

---

## Codec 字串格式

| Codec | MIME Type | 範例 |
|-------|-----------|------|
| H.264 | `video/mp4; codecs="avc1.XXYYZZ"` | `avc1.640028` |
| H.265 | `video/mp4; codecs="hvc1.X.Y.LZ"` | `hvc1.1.6.L93` |

```javascript
const mimeType = `video/mp4; codecs="${this.mediaCodec}"`;
const sourceBuffer = mediaSource.addSourceBuffer(mimeType);
```

---

## ManagedMediaSource

iOS Safari 使用 `ManagedMediaSource` 取代 `MediaSource`。

```javascript
createMediaSource() {
  const isManagedMediaSourceSupported = window.ManagedMediaSource !== undefined;
  
  if (isManagedMediaSourceSupported) {
    mediaSource = new window.ManagedMediaSource();
  } else if (window.MediaSource) {
    mediaSource = new MediaSource();
  }
  
  // ManagedMediaSource 使用 srcObject 綁定
  if (isManagedMediaSourceSupported) {
    video.srcObject = handle;  // MediaSourceHandle
    video.disableRemotePlayback = true;
  } else {
    video.src = URL.createObjectURL(mediaSource);
  }
}
```

---

## Worker 模式

### MediaSourceWorkerProxy

在 Web Worker 中執行 MediaSource 操作，減少主執行緒負擔。

```javascript
// packages/lib-rtsp-player/src/workers/MediaSourceWorkerProxy.js

class MediaSourceWorkerProxy extends EventEmitter {
  constructor(video, codec, mediaCodec, mode) {
    this.worker = new MediaSourceWorker();
    this.worker.addEventListener('message', this.handleWorkerMessage.bind(this));
    this.initMediaSource();
  }

  appendPacket(packet) {
    // 透過 postMessage 傳送到 Worker
    this.worker.postMessage({
      type: WORKER_EVENT_TYPES.APPEND_PACKET,
      data: { id: this.id, packet }
    }, [packet.buffer]);  // 使用 Transferable
  }
}
```

### Worker 事件類型

| 事件 | 說明 |
|------|------|
| `INIT` | 初始化 MediaSource |
| `MEDIA_SOURCE_CREATED` | MediaSource 建立完成 |
| `SOURCE_OPEN` | SourceBuffer 就緒 |
| `APPEND_PACKET` | 添加封包 |
| `PLAYBACK_READY` | 可開始播放 |
| `SEEK_BUFFERED_END` | 跳到緩衝區末端 |
| `ERROR` | 錯誤 |
| `DESTROY` | 銷毀 |

---

## 通知事件處理

### NotifyHandler

管理串流中的通知事件（時間戳、metadata 等）。

```javascript
// 添加通知
notifyHandler.appendNotify(info);

// 取得已過期的通知（根據播放時間）
const expired = notifyHandler.getExpiredNotify(currentTime);
expired.forEach((notify) => {
  this.emit(PLAYER_EVENT_NAMES.NOTIFY, notify);
});
```

### 幀率計算

```javascript
this.frameRateCalculator.start((fps) => {
  this.fps = fps;
});

// 推送時間戳來計算 fps
this.frameRateCalculator.pushTimestamp(notify.timestamp.stream);
```

---

## requestVideoFrameCallback

使用 `requestVideoFrameCallback` 取得精確的渲染時間。

```javascript
setupRVFC() {
  if (!this.video?.requestVideoFrameCallback) return;
  
  const tick = (now, metadata) => {
    // metadata.mediaTime: 當前渲染幀的媒體時間
    this.lastRenderedMediaTimeMs = Math.floor(metadata.mediaTime * 1000);
    this.video.requestVideoFrameCallback(tick);
  };
  
  this.video.requestVideoFrameCallback(tick);
}

getReferenceMediaTimeMs() {
  return (this.lastRenderedMediaTimeMs !== null)
    ? this.lastRenderedMediaTimeMs
    : Math.floor((this.player.currentTime || 0) * 1000);
}
```

---

## 錯誤處理

```javascript
handleError(err) {
  this.stop();
  const error = err || new Error(this.video?.error?.message || 'video error');
  this.emit(PLAYER_EVENT_NAMES.ERROR, error);
}

// 解碼錯誤恢復（Playback 模式）
if (error.message.includes('PIPELINE_ERROR_DECODE')) {
  this.handleDecodeError();
  return;
}
```

---

## 常見問題

### 1. 播放卡頓
- 檢查 `bufferedLength` 是否充足
- 確認網路連線穩定
- 檢查 CPU 負載

### 2. 延遲過大
- 確認 `isLiveview` 模式正確設定
- 檢查 `adjustPlaybackRate` 是否生效
- 考慮降低 `SOURCEBUFFER_LIMITED_LENGTH`

### 3. 記憶體增長
- 確認 `cutSourceBufferHead` 有執行
- 檢查 `destroy()` 是否正確清理
- 確認 `URL.revokeObjectURL()` 有呼叫

### 4. Codec 不支援
```javascript
// 檢查瀏覽器支援
MediaSource.isTypeSupported(`video/mp4; codecs="${mediaCodec}"`);
```

### 5. iOS Safari 問題
- 使用 `ManagedMediaSource`
- 確認 `video.disableRemotePlayback = true`
- 使用 `srcObject` 而非 `src`

---

## 相關模組

- `vsaas-video-remuxing` - 影片 Remuxing
- `vsaas-streaming-architecture` - 整體串流架構
- `vsaas-webcodecs-player` - WebCodecs 播放器（替代方案）
