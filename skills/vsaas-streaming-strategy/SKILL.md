# VSaaS Portal - Streaming Strategy

串流策略選擇模組，根據瀏覽器能力決定最佳的串流管線。

## 模組概述

### 策略類型

| 模式 | 說明 | 優點 | 缺點 |
|------|------|------|------|
| WebCodecs+Worker | WebCodecs 在 Worker | 最低延遲、最佳效能 | 瀏覽器支援有限 |
| WebCodecs | WebCodecs 主執行緒 | 低延遲、精確幀控制 | 需要 Chrome/Edge |
| MSE+Worker | MSE 在 Worker | 廣泛支援、減少阻塞 | 需要 fMP4 封裝 |
| MSE | MSE 主執行緒 | 最廣泛支援 | 較高延遲 |

---

## 檔案結構

```
packages/lib-rtsp-player/src/utils/
├── StrategyHelper.js            # 策略選擇邏輯
├── WebCodecsHelper.js           # WebCodecs 輔助
└── ...

packages/lib-rtsp-player/src/
├── Liveview.js                  # 套用策略的入口
├── handler/
│   ├── StreamProcessor.js       # MSE 處理器
│   └── WebCodecsStreamProcessor.js  # WebCodecs 處理器
└── ...
```

---

## 策略選擇函式

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
  
  // 檢查全域覆蓋（用於 QA 測試）
  const w = getWindow();
  const useWebCodecsOverride = w?.useWebCodecs ?? feature.useWebCodecs;
  
  // 決定使用 WebCodecs 或 MSE
  const isWebCodecsSupported = useWebCodecsOverride && 
    hasWebCodecsAPI({ requireAudio: !feature.allowMute });

  let mode;
  if (isWebCodecsSupported) {
    mode = isWorkerSupported 
      ? STREAMING_PROCESSOR_MODE.WEB_CODECS_WORKER 
      : STREAMING_PROCESSOR_MODE.WEB_CODECS;
  } else if (!isWorkerSupported || hasManagedMediaSource()) {
    mode = STREAMING_PROCESSOR_MODE.MSE;
  } else {
    mode = STREAMING_PROCESSOR_MODE.MSE_WORKER;
  }

  return {
    streamPipelineMode: mode,
    rtsphandlerUseWorker: isWorkerSupported,
  };
}
```

---

## 環境檢測

### Worker 支援

```javascript
function hasWorkerSupport() {
  const g = getGlobal();
  return !!g && typeof g.Worker !== 'undefined';
}
```

### WebCodecs 支援

```javascript
function hasWebCodecsAPI({ requireAudio = true } = {}) {
  const w = getWindow() ?? getGlobal();
  if (!w) return false;

  const hasVideoDecode = 
    typeof w.VideoDecoder !== 'undefined' &&
    typeof w.EncodedVideoChunk !== 'undefined' &&
    typeof w.VideoFrame !== 'undefined';

  const hasAudioDecode = !requireAudio || (
    typeof w.AudioDecoder !== 'undefined' &&
    typeof w.EncodedAudioChunk !== 'undefined'
  );

  return hasVideoDecode && hasAudioDecode;
}
```

### ManagedMediaSource 支援

```javascript
function hasManagedMediaSource() {
  const w = getWindow();
  return !!w && typeof w.ManagedMediaSource !== 'undefined';
}
```

---

## 策略套用

### Liveview 初始化

```javascript
// packages/lib-rtsp-player/src/Liveview.js

class Liveview {
  constructor(reference, options) {
    // 決定策略
    const {
      streamPipelineMode,
      rtspHandlerUseWorker,
    } = decideRtspEntryStrategy(options.feature);

    // 建立 RTSP Handler
    this.rtspHandler = new RTSPHandler({ 
      ...options, 
      useWorker: rtspHandlerUseWorker 
    }, reference);

    // 根據策略選擇 StreamProcessor
    const useWebCodecs = 
      streamPipelineMode === STREAMING_PROCESSOR_MODE.WEB_CODECS_WORKER ||
      streamPipelineMode === STREAMING_PROCESSOR_MODE.WEB_CODECS;

    this.streamProcessor = useWebCodecs
      ? new WebCodecsStreamProcessor({ 
          rtspClient: this.rtspHandler, 
          pipelineMode: streamPipelineMode,
          ...options 
        })
      : new StreamProcessor({ 
          rtspClient: this.rtspHandler, 
          pipelineMode: streamPipelineMode,
          ...options 
        });
  }
}
```

### Feature Options

```javascript
// DisplayStreamingBlock.vue

provide() {
  return {
    featureOptions: {
      allowMute: false,        // 是否允許靜音（影響音訊解碼需求）
      useWorker: true,         // 是否使用 Worker
      useWebCodecs: this.isWebCodecsSupported,  // 是否使用 WebCodecs
    },
  };
}
```

---

## 全域覆蓋

可透過瀏覽器 Console 覆蓋策略（用於 QA 測試）：

```javascript
// 強制禁用 WebCodecs
window.useWebCodecs = false;

// 強制啟用 WebCodecs（如果支援）
window.useWebCodecs = true;

// 重新載入頁面生效
location.reload();
```

---

## 瀏覽器支援矩陣

| 瀏覽器 | WebCodecs | MSE | Worker | ManagedMSE |
|--------|-----------|-----|--------|------------|
| Chrome 94+ | ✅ | ✅ | ✅ | ❌ |
| Chrome 105+ | ✅ | ✅ | ✅ | ❌ |
| Edge 94+ | ✅ | ✅ | ✅ | ❌ |
| Firefox | ❌ | ✅ | ✅ | ❌ |
| Safari 17+ | ❌ | ✅ | ✅ | ✅ |
| Safari (iOS) | ❌ | ✅ | ✅ | ✅ |

---

## 策略決策流程

```
開始
  │
  ▼
useWebCodecs 且 hasWebCodecsAPI()?
  │
  ├── 是 ───→ useWorker 且 hasWorkerSupport()?
  │              │
  │              ├── 是 ───→ WebCodecs+Worker
  │              │
  │              └── 否 ───→ WebCodecs
  │
  └── 否 ───→ hasManagedMediaSource()?
                 │
                 ├── 是 ───→ MSE (iOS Safari)
                 │
                 └── 否 ───→ useWorker 且 hasWorkerSupport()?
                               │
                               ├── 是 ───→ MSE+Worker
                               │
                               └── 否 ───→ MSE
```

---

## StreamProcessor vs WebCodecsStreamProcessor

### StreamProcessor (MSE)

```javascript
// packages/lib-rtsp-player/src/handler/StreamProcessor.js

class StreamProcessor extends EventEmitter {
  constructor({ mode, ssmuxType, rtspClient, ...playerOptions }) {
    // 使用 SSMUX 將 NALU 轉為 fMP4
    this.createStreamProcessor(ssmuxType);
    
    // 建立 VideoPlayer（使用 MediaSource）
    this.streamPlayer = new VideoPlayer(options);
  }

  inputPacket(packet) {
    // 送入 SSMUX 處理
    this.ssmux.inputPacketV1(buf, arr.length);
  }

  // SSMUX 回呼
  processVideoEvent = (fmp4Packet) => {
    // fMP4 封包送到 VideoPlayer
    this.streamPlayer.appendPacket({ packet: fmp4Packet });
  }
}
```

### WebCodecsStreamProcessor

```javascript
// packages/lib-rtsp-player/src/handler/WebCodecsStreamProcessor.js

class WebCodecsStreamProcessor extends EventEmitter {
  constructor({ mode, rtspClient, pipelineMode, ...playerOptions }) {
    // 不需要 SSMUX
    
    // 建立 WebCodecsVideoPlayer（使用 VideoDecoder）
    this.streamPlayer = new WebCodecsVideoPlayer(options);
  }

  inputPacket(packet) {
    // 直接將 NALU 送到 WebCodecsVideoPlayer
    this.streamPlayer.appendPacket({
      data: packet.data,
      meta: packet.meta,
      info: this.streamInfo,
    });
  }
}
```

---

## 效能比較

| 指標 | WebCodecs | MSE |
|------|-----------|-----|
| 延遲 | ~50-100ms | ~200-500ms |
| CPU 使用 | 較低（硬體解碼） | 較高 |
| 記憶體 | 較低 | 較高（緩衝區） |
| 幀控制 | 精確 | 有限 |
| H.265 支援 | 較好 | 瀏覽器依賴 |

---

## Codec 支援檢測

```javascript
// packages/lib-rtsp-player/src/utils/WebCodecsHelper.js

export async function getVideoDecoderSupportedConfig(config) {
  try {
    const { supported } = await VideoDecoder.isConfigSupported(config);
    return supported ? config : null;
  } catch {
    return null;
  }
}

// MSE 方式
export function isMseCodecSupported(mimeType) {
  return MediaSource.isTypeSupported(mimeType);
}
```

---

## 降級處理

```javascript
// 如果 WebCodecs 解碼失敗，降級到 MSE
handleError(error) {
  if (error.message.includes('decoder') && this.pipelineMode.includes('WebCodecs')) {
    // 重新初始化為 MSE 模式
    this.reinitWithMSE();
  }
}
```

---

## 常見問題

### 1. 選錯策略
- 確認 feature options 正確傳入
- 檢查全域覆蓋變數
- 確認瀏覽器版本

### 2. WebCodecs 不工作
- 確認 Chrome/Edge 版本 >= 94
- 檢查 codec 是否支援
- 確認 VideoDecoder.isConfigSupported

### 3. iOS Safari 問題
- 確認使用 ManagedMediaSource
- 檢查 disableRemotePlayback
- 使用 srcObject 而非 src

### 4. Worker 不工作
- 確認 Vite 設定正確
- 檢查 CSP 是否阻擋
- 確認 Worker 檔案可載入

---

## 相關模組

- `vsaas-streaming-architecture` - 整體架構
- `vsaas-mse-player` - MSE 播放器
- `vsaas-webcodecs-player` - WebCodecs 播放器
- `vsaas-streaming-workers` - Worker 架構
