# VSaaS Portal - Streaming Workers

Web Workers 架構，將解碼與媒體處理移到背景執行緒，減少主執行緒負擔。

## 模組概述

### Workers 類型

| Worker | 職責 |
|--------|------|
| `rtp-transformer.worker` | RTP 封包解析、NALU 重組 |
| `media-source.worker` | MediaSource 管理、SourceBuffer 操作 |
| `web-codecs-decoder.worker` | WebCodecs 解碼 |

---

## 檔案結構

```
packages/lib-rtsp-player/src/workers/
├── workerEventType.js           # 事件類型常數
├── rtp-transformer.worker.js    # RTP 轉換 Worker
├── RtpTransformerWorkerProxy.js # RTP Worker 代理
├── media-source.worker.js       # MediaSource Worker
├── MediaSourceWorkerProxy.js    # MediaSource Worker 代理
├── web-codecs-decoder.worker.js # WebCodecs Worker
└── WebCodecsWorkerProxy.js      # WebCodecs Worker 代理
```

---

## 架構圖

```
Main Thread                          Worker Thread
┌────────────────────────┐          ┌────────────────────────┐
│                        │          │                        │
│   WorkerProxy          │ ←─────→  │   Worker               │
│   (postMessage)        │          │   (addEventListener)   │
│                        │          │                        │
└────────────────────────┘          └────────────────────────┘
         ↑                                    │
         │ EventEmitter                       │ postMessage
         │                                    ▼
┌────────────────────────┐          ┌────────────────────────┐
│   Player / Controller  │          │   MediaSource          │
│                        │          │   VideoDecoder         │
└────────────────────────┘          └────────────────────────┘
```

---

## Worker 事件類型

```javascript
// packages/lib-rtsp-player/src/workers/workerEventType.js

export default Object.freeze({
  // 錯誤
  ERROR: 'worker-error',

  // → 發送到 Worker
  INIT: 'worker-init',
  DESTROY: 'worker-destroy',
  FEED: 'worker-feed',
  APPEND_PACKET: 'worker-append-packet',
  SET_CODEC: 'worker-set-codec',
  UPDATE_BUFFER_STRATEGY: 'worker-update-buffer-strategy',
  UPDATE_CURRENT_TIME: 'worker-update-current-time',
  ATTACH_TARGET: 'worker-attach-target',

  // ← Worker 發送
  INITIALIZED: 'worker-initialized',
  DESTROYED: 'worker-destroyed',
  TRANSFORMED: 'worker-transformed',
  MEDIA_SOURCE_CREATED: 'worker-media-source-created',
  SOURCE_OPEN: 'worker-source-open',
  PLAYBACK_READY: 'worker-playback-ready',
  SEEK_BUFFERED_END: 'worker-seek-buffered-end',
  STEP_DONE: 'worker-step-done',
});
```

---

## RTP Transformer Worker

將 RTP 封包解析為 NALU 或音訊資料。

### Worker 端

```javascript
// packages/lib-rtsp-player/src/workers/rtp-transformer.worker.js

const transformers = new Map();

addEventListener('message', (event) => {
  const { type, data } = event.data;
  
  switch (type) {
    case WORKER_EVENT_TYPES.INIT: {
      const { id, codec, trackType } = data;
      
      if (trackType === TRACK_TYPE.APPLICATION) {
        transformers.set(id, {
          trackType,
          processor: new ApplicationTrackProcessor()
        });
      } else {
        transformers.set(id, {
          trackType,
          codec,
          assembler: new NaluAssembler()
        });
      }
      
      postMessage({ type: WORKER_EVENT_TYPES.INITIALIZED, id });
      break;
    }

    case WORKER_EVENT_TYPES.FEED: {
      const { id, rtp: rtpData } = data;
      const transformer = transformers.get(id);
      const rtp = RTP.deserialize(rtpData);

      if (transformer.trackType === TRACK_TYPE.VIDEO) {
        const nalu = transformer.assembler.onNALUFragment(rtp);
        if (nalu) {
          postMessage({
            type: WORKER_EVENT_TYPES.TRANSFORMED,
            id,
            trackType: TRACK_TYPE.VIDEO,
            data: nalu.serialize()
          });
        }
      } else if (transformer.trackType === TRACK_TYPE.AUDIO) {
        postMessage({
          type: WORKER_EVENT_TYPES.TRANSFORMED,
          id,
          trackType: TRACK_TYPE.AUDIO,
          data: {
            dts: rtp.getTimestampMS(),
            data: rtp.getPayload(),
            applicationData: rtp.getApplicationData()
          }
        });
      }
      break;
    }

    case WORKER_EVENT_TYPES.DESTROY: {
      const { id } = data;
      transformers.delete(id);
      postMessage({ type: WORKER_EVENT_TYPES.DESTROYED, id });
      break;
    }
  }
});
```

### Proxy 端

```javascript
// packages/lib-rtsp-player/src/workers/RtpTransformerWorkerProxy.js

class RtpTransformerWorkerProxy extends EventEmitter {
  constructor(id, codec, trackType) {
    super();
    this.id = id;
    this.worker = new RtpTransformerWorker();
    this.worker.addEventListener('message', this.handleMessage.bind(this));
    
    this.worker.postMessage({
      type: WORKER_EVENT_TYPES.INIT,
      data: { id, codec, trackType }
    });
  }

  feed(rtp) {
    this.worker.postMessage({
      type: WORKER_EVENT_TYPES.FEED,
      data: { id: this.id, rtp: rtp.serialize() }
    }, [rtp.serialize().buffer]);  // Transferable
  }

  handleMessage(event) {
    const { type, id, data, trackType } = event.data;
    if (id !== this.id) return;

    switch (type) {
      case WORKER_EVENT_TYPES.TRANSFORMED:
        this.emit('transformed', { trackType, data });
        break;
      case WORKER_EVENT_TYPES.ERROR:
        this.emit('error', data);
        break;
    }
  }

  destroy() {
    this.worker.postMessage({
      type: WORKER_EVENT_TYPES.DESTROY,
      data: { id: this.id }
    });
    this.worker.terminate();
  }
}
```

---

## MediaSource Worker

在 Worker 中管理 MediaSource，使用 `MediaSource.handle` 傳遞給主執行緒。

### Worker 端

```javascript
// packages/lib-rtsp-player/src/workers/media-source.worker.js

const mediaSources = new Map();

function initMediaSource(id, codec, mediaCodec, mode) {
  const mediaSource = new MediaSource();
  const isLiveview = mode === 'liveview';

  mediaSource.addEventListener('sourceopen', () => 
    onSourceOpen(id, mediaSource, codec, mediaCodec)
  );

  mediaSources.set(id, {
    mediaSource,
    sourceBuffer: null,
    queue: [],
    sourceBufferStart: 0,
    isPlaybackReady: false,
    playbackOffset: 0,
    isLiveview,
    codec,
    currentTime: 0,
    KEEP_PASSED_LENGTH: 20,
    SOURCEBUFFER_LIMITED_LENGTH: 3.6,
  });

  // 傳送 MediaSource handle 到主執行緒
  const { handle } = mediaSource;
  postMessage({
    type: WORKER_EVENT_TYPES.MEDIA_SOURCE_CREATED,
    id,
    handle
  }, [handle]);  // Transferable
}

function onSourceOpen(id, mediaSource, codec, mediaCodec) {
  const state = mediaSources.get(id);
  
  mediaSource.duration = Infinity;
  const mimeType = `video/mp4; codecs="${mediaCodec}"`;
  const sourceBuffer = mediaSource.addSourceBuffer(mimeType);
  
  sourceBuffer.addEventListener('updateend', () => onUpdateEnd(id));
  state.sourceBuffer = sourceBuffer;

  postMessage({ type: WORKER_EVENT_TYPES.SOURCE_OPEN, id });
}

function appendPacket(id, packet, timestamp) {
  const state = mediaSources.get(id);
  if (!state) return;

  if (state.queue.length === 0 && canAppendBuffer(state)) {
    tryAppendBuffer(id, packet);
  } else {
    state.queue.push(packet);
  }
}

function onUpdateEnd(id) {
  const state = mediaSources.get(id);
  
  // 檢查是否需要跳到緩衝區末端
  const bufferedEnd = getBufferedEnd(state);
  const bufferedLength = bufferedEnd - state.currentTime;
  
  if (state.isLiveview && bufferedLength > state.SOURCEBUFFER_LIMITED_LENGTH) {
    seekBufferedEnd(id);
  }

  // 處理佇列中的封包
  if (state.queue.length && canAppendBuffer(state)) {
    tryAppendBuffer(id, state.queue.shift());
  }
}
```

### Proxy 端

```javascript
// packages/lib-rtsp-player/src/workers/MediaSourceWorkerProxy.js

class MediaSourceWorkerProxy extends EventEmitter {
  constructor(video, codec, mediaCodec, mode) {
    super();
    this.video = video;
    this.id = uuidv4();
    this.worker = new MediaSourceWorker();
    this.worker.addEventListener('message', this.handleWorkerMessage.bind(this));
    
    this.initMediaSource();
  }

  initMediaSource() {
    this.worker.postMessage({
      type: WORKER_EVENT_TYPES.INIT,
      data: {
        id: this.id,
        codec: this.codec,
        mediaCodec: this.mediaCodec,
        mode: this.mode,
      }
    });
  }

  handleWorkerMessage(event) {
    const { type, id, handle, bufferedEnd, error } = event.data;
    if (id !== this.id) return;

    switch (type) {
      case WORKER_EVENT_TYPES.MEDIA_SOURCE_CREATED:
        // 使用 handle 綁定到 video
        this.video.srcObject = handle;
        this.video.disableRemotePlayback = true;
        break;

      case WORKER_EVENT_TYPES.PLAYBACK_READY:
        this.emit('playbackReady');
        break;

      case WORKER_EVENT_TYPES.SEEK_BUFFERED_END:
        this.video.currentTime = bufferedEnd;
        this.emit('seekBufferedEnd');
        break;

      case WORKER_EVENT_TYPES.ERROR:
        this.emit('error', new Error(error));
        break;
    }
  }

  appendPacket(packet) {
    this.worker.postMessage({
      type: WORKER_EVENT_TYPES.APPEND_PACKET,
      data: { id: this.id, packet }
    }, [packet.buffer]);  // Transferable
  }

  updateCurrentTime(currentTime) {
    this.worker.postMessage({
      type: WORKER_EVENT_TYPES.UPDATE_CURRENT_TIME,
      data: { id: this.id, currentTime }
    });
  }

  destroy() {
    this.worker.postMessage({
      type: WORKER_EVENT_TYPES.DESTROY,
      data: { id: this.id }
    });
    this.worker.terminate();
  }
}
```

---

## WebCodecs Worker

在 Worker 中使用 VideoDecoder，透過 OffscreenCanvas 渲染。

```javascript
// packages/lib-rtsp-player/src/workers/web-codecs-decoder.worker.js

let decoder = null;
let canvas = null;
let ctx2d = null;

addEventListener('message', async (event) => {
  const { type, data } = event.data;

  switch (type) {
    case WORKER_EVENT_TYPES.INIT: {
      const { codec, mediaCodec } = data;
      
      decoder = new VideoDecoder({
        output: handleFrame,
        error: (e) => postMessage({ type: WORKER_EVENT_TYPES.ERROR, error: e.message })
      });
      
      const config = await VideoDecoder.isConfigSupported({
        codec: mediaCodec,
        // ...
      });
      decoder.configure(config);
      break;
    }

    case WORKER_EVENT_TYPES.ATTACH_TARGET: {
      // 接收 OffscreenCanvas
      canvas = data.canvas;
      ctx2d = canvas.getContext('2d', { alpha: false });
      break;
    }

    case WORKER_EVENT_TYPES.FEED: {
      const { buffer, timestamp, isKeyframe } = data;
      
      const chunk = new EncodedVideoChunk({
        type: isKeyframe ? 'key' : 'delta',
        timestamp,
        data: buffer,
      });
      
      decoder.decode(chunk);
      break;
    }

    case WORKER_EVENT_TYPES.DESTROY: {
      decoder?.close();
      decoder = null;
      break;
    }
  }
});

function handleFrame(frame) {
  // 繪製到 OffscreenCanvas
  ctx2d.drawImage(frame, 0, 0, canvas.width, canvas.height);
  frame.close();
  
  postMessage({ type: WORKER_EVENT_TYPES.PLAYBACK_READY });
}
```

---

## Transferable Objects

使用 Transferable 避免資料複製：

```javascript
// ArrayBuffer
worker.postMessage({ data: packet }, [packet.buffer]);

// OffscreenCanvas
const offscreen = canvas.transferControlToOffscreen();
worker.postMessage({ canvas: offscreen }, [offscreen]);

// MediaSourceHandle
const { handle } = mediaSource;
postMessage({ handle }, [handle]);
```

---

## 策略選擇

```javascript
// 決定是否使用 Worker

const { streamPipelineMode, rtspHandlerUseWorker } = decideRtspEntryStrategy({
  useWorker: true,
  useWebCodecs: true,
});

// 可能的模式：
// - 'WebCodecs+Worker'
// - 'WebCodecs'
// - 'MSE+Worker'
// - 'MSE'
```

---

## Worker 載入

使用 Vite 的 Worker 語法：

```javascript
// 內聯 Worker
import MediaSourceWorker from './media-source.worker.js?worker&inline';

// URL 方式
import workletUrl from './PCMUToG711AudioWorkletProcessor.js?worker&url';
```

---

## 錯誤處理

```javascript
// Worker 端
try {
  // ...
} catch (error) {
  postMessage({
    type: WORKER_EVENT_TYPES.ERROR,
    id,
    error: error.message
  });
}

// Proxy 端
handleWorkerMessage(event) {
  if (event.data.type === WORKER_EVENT_TYPES.ERROR) {
    this.emit('error', new Error(event.data.error));
  }
}
```

---

## 常見問題

### 1. Worker 無法載入
- 確認 Vite 設定支援 Worker
- 檢查路徑是否正確
- 確認沒有 CORS 問題

### 2. Transferable 失敗
- 確認物件支援 Transfer
- 檢查是否已被 transfer
- ArrayBuffer 只能 transfer 一次

### 3. MediaSource handle 不支援
- 需要 Chrome 105+ 或支援的瀏覽器
- 降級使用主執行緒 MediaSource

### 4. Worker 記憶體洩漏
- 確認 `terminate()` 有呼叫
- 檢查 Map/Set 有正確清理
- 確認 VideoFrame/AudioBuffer 有 close

---

## 效能考量

| 操作 | 主執行緒 | Worker |
|------|---------|--------|
| RTP 解析 | 阻塞 UI | 背景執行 |
| NALU 重組 | 阻塞 UI | 背景執行 |
| VideoDecoder | 阻塞 UI | 背景執行 |
| SourceBuffer | 可能阻塞 | 背景執行 |
| 渲染 | 必須主執行緒 | OffscreenCanvas |

---

## 相關模組

- `vsaas-streaming-strategy` - 策略選擇
- `vsaas-mse-player` - MSE 播放器
- `vsaas-webcodecs-player` - WebCodecs 播放器
