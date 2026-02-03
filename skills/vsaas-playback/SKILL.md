# VSaaS Portal - Playback

回放控制模組，支援時間跳轉、速度控制、單幀前進等功能。

## 模組概述

### 功能
- 時間跳轉（seek）
- 播放速度控制
- 暫停/恢復
- 單幀前進
- 時間範圍播放
- 快照模式

---

## 檔案結構

```
packages/lib-rtsp-player/src/
├── Liveview.js                  # 即時串流基類
├── Playback.js                  # 回放類（繼承 Liveview）
└── RTSPPlayer.js                # 統一入口

packages/app-vsaas-portal/src/
├── components/Player/
│   └── PlayerWrapper.vue        # 播放器容器
└── pages/archive/               # 錄影回放頁面
```

---

## 類別繼承

```
RTSPPlayer (統一入口)
    │
    ├── Liveview (即時串流)
    │     │
    │     ├── RTSPHandler
    │     └── StreamProcessor
    │
    └── Playback (回放，繼承 Liveview)
          │
          ├── setRange()
          ├── seek()
          ├── setPlaybackRate()
          └── nextFrame()
```

---

## Liveview（基類）

```javascript
// packages/lib-rtsp-player/src/Liveview.js

class Liveview {
  constructor(reference, options) {
    // 決定串流策略
    const { streamPipelineMode, rtspHandlerUseWorker } = 
      decideRtspEntryStrategy(options.feature);

    // 建立 RTSP Handler
    this.rtspHandler = new RTSPHandler({ 
      ...options, 
      useWorker: rtspHandlerUseWorker 
    }, reference);

    // 建立 Stream Processor（MSE 或 WebCodecs）
    const useWebCodecs = streamPipelineMode.includes('WebCodecs');
    this.streamProcessor = useWebCodecs
      ? new WebCodecsStreamProcessor({ ... })
      : new StreamProcessor({ ... });

    this.isPaused = false;
    this.bindEvents();
  }

  get streamPlayer() {
    return this.streamProcessor?.streamPlayer;
  }

  get isPlaying() {
    return this.streamProcessor?.isPlaying;
  }

  // 播放
  play() {
    if (!this.hasPermission) {
      return Promise.reject(new Error('No permission'));
    }
    return this.rtspHandler.start()
      .then(() => this.streamProcessor.play());
  }

  // 停止
  stop() {
    return this.rtspHandler.stop();
  }

  // 設定 URL
  setUrl(url) {
    this.url = url;
    this.rtspHandler.setUrl(url);
  }
}
```

---

## Playback（回放類）

```javascript
// packages/lib-rtsp-player/src/Playback.js

class Playback extends Liveview {
  constructor(reference, options) {
    const snapshotMode = !!options.snapshotMode;
    super(reference, {
      snapshotMode,
      ssmuxType: snapshotMode ? 2 : 1,  // 快照模式使用不同的 ssmux 類型
      ...options,
    });

    this.snapshotMode = snapshotMode;
  }

  get hasPermission() {
    return this.permission.canViewPlayback ?? true;
  }

  get playbackRate() {
    return this.streamPlayer?.playbackRate;
  }
}
```

### 設定時間範圍

```javascript
setRange(timestamp) {
  this.rtspHandler.setRange(timestamp);
}
```

### 播放

```javascript
play({ timestamp, url } = {}) {
  const { isPaused, snapshotMode } = this;

  // 如果已暫停，恢復播放
  if (isPaused) {
    return this.resume();
  }

  if (!snapshotMode) {
    // 驗證參數
    if (!timestamp || !url) {
      return Promise.reject(new Error('Playback requires both timestamp and url parameters'));
    }

    if (!url.includes('&start=') || !url.includes('&end=')) {
      return Promise.reject(new Error('Playback URL missing time range parameters'));
    }

    this.setUrl(url);
    this.setRange(timestamp);
    this.streamProcessor.setSeek(timestamp);
  }

  return super.play();
}
```

### 暫停

```javascript
pause({ isNextFrame } = { isNextFrame: false }) {
  if (this.isPaused || this.$_isPausing) {
    return Promise.resolve();
  }

  const { timestamp } = this;
  this.$_isPausing = true;

  return this.streamProcessor.pause(timestamp)
    .then(() => {
      this.isPaused = true;
      if (!isNextFrame) {
        this.rtspHandler.pause(timestamp);
      }
    })
    .finally(() => {
      this.$_isPausing = false;
    });
}
```

### 恢復

```javascript
resume() {
  return this.rtspHandler.resume(this.playbackRate)
    .then(() => {
      this.streamProcessor.resume();
    });
}
```

### 跳轉

```javascript
seek({ timestamp, url } = {}) {
  const { streamProcessor, rtspHandler } = this;

  if (!timestamp || !url) {
    return Promise.reject(new Error('Seek requires both timestamp and url parameters'));
  }

  return streamProcessor.stop()
    .then(() => {
      streamProcessor.setSeek(timestamp);
      this.setUrl(url);
      this.setRange(timestamp);
      return rtspHandler.seek();
    });
}
```

### 移動

```javascript
move(sec) {
  const { timestamp } = this;
  if (!timestamp) return Promise.reject();

  const newTime = timestamp + sec * 1000;
  return this.seek({ timestamp: newTime });
}
```

### 播放速度

```javascript
setPlaybackRate(rate) {
  if (this.playbackRate === rate) {
    return Promise.resolve();
  }

  const { streamProcessor, rtspHandler, timestamp } = this;

  this.setRange(timestamp);
  return streamProcessor.stop()
    .then(() => rtspHandler.change(rate))
    .then(() => {
      streamProcessor.setPlaybackRate(rate);
      streamProcessor.resume();
    });
}
```

### 單幀前進

```javascript
nextFrame() {
  return this.rtspHandler.resume(this.playbackRate)
    .then(() => this.streamProcessor.nextFrame());
}
```

---

## URL 格式

回放 URL 必須包含時間範圍參數：

```
rtsp://server/stream?start=1704067200000&end=1704153600000
```

| 參數 | 說明 |
|------|------|
| `start` | 開始時間（毫秒時間戳） |
| `end` | 結束時間（毫秒時間戳） |

---

## 事件

### 從 StreamProcessor

| 事件 | 說明 |
|------|------|
| `play` | 開始播放 |
| `pause` | 暫停 |
| `notify` | 時間戳通知 |
| `error` | 錯誤 |

### 從 RTSPHandler

| 事件 | 說明 |
|------|------|
| `rtsp-error` | RTSP 錯誤 |
| `info` | RTSP 資訊 |

---

## 快照模式

用於擷取特定時間點的畫面。

```javascript
const playback = new Playback(reference, {
  snapshotMode: true,
  // ...
});

// 快照模式會：
// 1. 使用 ssmuxType = 2
// 2. 跳過音訊處理
// 3. 不需要 URL 時間參數
```

---

## 播放速度控制

支援的速度：

| 速度 | 說明 |
|------|------|
| 0.5 | 半速 |
| 1.0 | 正常速度 |
| 2.0 | 2 倍速 |
| 4.0 | 4 倍速 |
| 8.0 | 8 倍速 |

注意：非 1 倍速時會靜音。

```javascript
set playbackRate(rate) {
  this.currentPlaybackRate = rate;
  
  // 非正常速度靜音
  if (this.gainNode) {
    this.gainNode.gain.value = this.muted ? 0 : this.volume;
  }
}

get muted() {
  return this.currentMuted || this.playbackRate !== 1;
}
```

---

## Stream Processor 整合

### setSeek

```javascript
// StreamProcessor.js

setSeek(timestamp = 0) {
  const seekMode = timestamp <= this.timestamp 
    ? PLAYER.SEEK_MODE.BACKWARD 
    : PLAYER.SEEK_MODE.FORWARD;
    
  this.$_filterTime = seekMode === PLAYER.SEEK_MODE.BACKWARD 
    ? this.timestamp 
    : timestamp;
  this.$_seekMode = seekMode;
  
  this.setPlaybackRate(1);
}
```

### 封包過濾

```javascript
get shouldFilterVideoEvent() {
  return this.supportStreamFilter && 
         this.$_filterTime !== 0 && 
         this.streamPlayer.playbackRate === 1;
}

checkShouldSkipVideoEvent(timestamp) {
  const { $_filterTime, $_seekMode } = this;

  // 前進模式：跳過時間戳小於目標的封包
  if ($_seekMode === SEEK_MODE.FORWARD && timestamp < $_filterTime) {
    return true;
  }
  
  // 後退模式：跳過時間戳大於目標的封包
  if ($_seekMode === SEEK_MODE.BACKWARD && timestamp > $_filterTime) {
    return true;
  }

  return false;
}
```

---

## Timeline 整合

```javascript
// PlayerWrapper.vue

updatePlayTime(localTime) {
  const { timestamp, tzoffs } = localTime.useClientTimezone();
  const isFutureTimestamp = timestamp >= new Date().getTime();
  
  // 未來時間戳重置為即時
  const newTimestamp = isFutureTimestamp ? 0 : timestamp;
  
  const { stream } = this.activeDeviceView.services;
  stream.resetPlayTime({ timestamp: newTimestamp, tzoffs: newTzoffs });
}
```

---

## 常見問題

### 1. 跳轉後無畫面
- 確認 URL 包含正確的時間範圍
- 檢查 `setSeek` 是否正確設定
- 確認該時間點有錄影資料

### 2. 播放速度無效
- 確認 RTSP 伺服器支援速度控制
- 檢查 `rtspHandler.change(rate)` 回應

### 3. 暫停/恢復問題
- 確認 `isPaused` 狀態正確
- 檢查 `$_isPausing` 鎖定
- 確認 RTSP PAUSE/PLAY 命令成功

### 4. 單幀前進卡住
- 確認 WebCodecs step mode 運作正常
- 檢查 frameBudget 邏輯
- 確認解碼器狀態

---

## 權限控制

```javascript
get hasPermission() {
  return this.permission.canViewPlayback ?? true;
}

play({ timestamp, url }) {
  if (!this.hasPermission) {
    return Promise.reject(new Error('No playback permission'));
  }
  // ...
}
```

---

## 相關模組

- `vsaas-streaming-architecture` - 串流架構
- `vsaas-rtsp-protocol` - RTSP 協議
- `vsaas-player-components` - 播放器元件
- `vsaas-archive` - 錄影管理
