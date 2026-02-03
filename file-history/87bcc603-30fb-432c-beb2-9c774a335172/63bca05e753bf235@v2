---
name: vsaas-streaming
description: Use when working with WebRTC connection, live streaming, playback, video player, RTSP, data channels, or media playback. Triggers on keywords like "WebRTC", "streaming", "live", "playback", "RTSP", "data channel", "OdysseyConnection", "串流", "即時影像", "回放".
version: 1.0.0
---

# VSaaS Portal - WebRTC 連線與串流播放指南

此指南詳細說明 VSaaS Portal 的 WebRTC 連線機制、Live 即時串流和 Playback 回放功能的實現架構。

## 系統架構概覽

```
┌─────────────────────────────────────────────────────────────┐
│                   Vue 3 UI 層 (表現層)                      │
│  DisplayStreamingBlock → PluginlessWrapper (組件層)         │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│              Device Service 層 (業務邏輯層)                 │
│  Streaming Service → VideoStreamService                     │
│  Connection (OdysseyConnectionManager)                      │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│           Streaming Handler 層 (流程控制層)                 │
│  StreamingHandler → VSaaSStreamingHandler                   │
│  TimeHandler, ErrorHandler, RtspHandler                    │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│            連線管理層 (WebRTC & Data Channel)               │
│  OdysseyConnection → OdysseyConnectionManager               │
│  OdysseyConnectPager → RTCPeerClient                        │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│          信令層 (MQTT 和 WebSocket 信令)                    │
│  IoTSignaling ↔ WebsocketSignaling                          │
│  信令交換：Offer/Answer/ICE Candidates                     │
└──────────────┬──────────────────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────────────────┐
│            低層 WebRTC & 媒體層                             │
│  RTCPeerConnection, RTCDataChannel                          │
│  RTSPPlayer, HLSPlayer (媒體播放器)                        │
│  AWS IoT MQTT Client                                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. WebRTC 連線機制

### 1.1 OdysseyConnection 初始化

**核心類位置**：`src/models/IoT/Odyssey/OdysseyConnection.js`

**初始化流程**：

```javascript
// 1. 應用啟動時初始化 IoT 連接
await OdysseyConnection.initialize()

// 2. 獲取信令必要信息
static async getSignalingEssentials() {
  // 獲取 ICE Servers 列表
  // 獲取 MQTT 客戶端認證信息
  // 設置 ICE Servers 自動更新計時器
  return { iceServers, mqttCred, expires }
}

// 3. 創建 IoT MQTT 連接
static async createIoTConnection(clearSignaling = false) {
  // 使用 AWS IoT Core 認證建立 MQTT 連接
  // 更新 IoTSignaling.mqttClient
  // 註冊重連事件
}
```

### 1.2 RTCPeerClient 連線流程

**核心類位置**：`packages/lib-webrtc-odyssey/src/lib/RTCPeerClient.js`

```javascript
// 步驟 1: 初始化 RTCPeerClient
const peerClient = new RTCPeerClient({
  Signaling: IoTSignaling
})

// 步驟 2: 執行連線
peerClient.connect({ thingName, derivant })
  .then(res => this.createPeerConnection(res))

// 步驟 3: 建立 RTCPeerConnection
const config = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    // TURN servers...
  ],
  bundlePolicy: 'max-bundle',
  rtcpMuxPolicy: 'require'
}
this.peerConnection = new RTCPeerConnection(config)

// 步驟 4: 創建 Data Channels
channels.map(label => this.createDataChannel(label))

// 步驟 5: 建立本地 Offer 並發送
peerConnection.createOffer().then(desc => {
  peerConnection.setLocalDescription(desc)
  signaling.addOffer(desc)
})
```

### 1.3 MQTT 信令交換

**IoTSignaling 流程**：

```javascript
// 1. 訂閱信令主題
subscribeSignaling() {
  const topicAnswer = `vsaas/clients/${clientId}/rtcsess/${sessionId}/answer`
  const topicIcecandidate = `vsaas/clients/${clientId}/rtcsess/${sessionId}/icecand`

  mqttClient.subscribe(topicAnswer)
  mqttClient.subscribe(topicIcecandidate)
}

// 2. 發送 Offer
addOffer(desc) {
  const topic = `vsaas/things/${thingName}/rtcsess/${sessionId}/offer`
  mqttClient.publish(topic, JSON.stringify({ sdp: desc }))
}

// 3. 接收 Answer
this.$onmessage = (topic, payload) => {
  const message = JSON.parse(payload.toString())
  if (topicAnswer === topic && message.status === 0) {
    this.emit('answer', message.sdp)
  }
}
```

### 1.4 ICE Candidate 處理

```javascript
// 發送本地 ICE Candidate
peerConnection.onicecandidate = ({ candidate }) => {
  signaling.addIceCandidate(candidate)
}

// 接收遠端 ICE Candidate（帶緩衝機制）
addIceCandidate(desc) {
  // 如果未設置遠端 SDP，暫存在緩衝區
  if (!isSetRemoteDescriptionSuccess) {
    this.$iceBuffer = this.$iceBuffer || []
    this.$iceBuffer.push(desc)
    return
  }

  // 設置遠端 SDP 後，添加所有緩衝的 ICE Candidates
  peerConnection.addIceCandidate(desc)
}
```

### 1.5 Data Channel 類型

| 通道類型 | 標籤 | 用途 |
|---------|------|------|
| 控制通道 | `rsvd-ctrl` | 會話管理、端點查詢 |
| RTSP 通道 | `rtsp-*` | 視頻串流傳輸 |
| HTTP 通道 | `http` | API 請求代理 |
| VCA 通道 | `vca` | 視頻分析元數據 |
| 音頻回傳 | `audio-back` | 雙向語音 |
| 警報通道 | `volalarm` | 警報信號 |

---

## 2. Live 播放流程

### 2.1 組件層次結構

```
DisplayStreamingBlock (高階組件)
  ├─ DisplayStatusBar (狀態欄)
  ├─ PluginlessWrapper (媒體渲染層)
  │   ├─ video_wrapper (視頻元素容器)
  │   ├─ canvas_wrapper (魚眼畫布容器)
  │   └─ thumbnail (加載時的縮圖)
  └─ ViewcellHud (連線信息提示)
```

### 2.2 Live 模式初始化

**DisplayStreamingBlock 組件**：

位置：`packages/advanced-vue-components/components/DisplayStreamingBlock.vue`

```javascript
// 組件建立時安裝 Streaming 服務
created() {
  if (this.device?.install && wizardFinished) {
    this.device?.install([
      new Streaming({
        muted: this.defalutMute,
        volume: this.device?.services.ui?.state.volume,
        dewarpType: this.dewarpType,
        streamIndex: this.playbackIndex ?? this.device?.services.ui?.state.streamIndex,

        // Live 視圖標誌
        isActive: this.isActiveView,
        mode: 'liveview',

        enableUserIdle: this.enableUserIdle,
        enableAuditLog: this.enableAuditLog,
        enableControlConnection: this.hasPermission('device/playback', this.device.deviceId),
      }, this.device, VsaasStreamingHandler)
    ])
  }
}
```

**PluginlessWrapper 自動播放**：

位置：`packages/advanced-vue-components/components/PluginlessWrapper.vue`

```javascript
mounted() {
  this.bindStreamEvents()
  this.installStreamingInterface()

  if (this.autoPlay) {
    this.device.services.stream.play(this.isReplayRotation)
  }
}

bindStreamEvents() {
  this.device.services.stream.on('appendVideoPlayer', this.appendVideoPlayerToWrapper)
  this.device.services.stream.on('appendAudioPlayer', this.appendAudioPlayerToWrapper)
  this.device.services.stream.on('appendimage', this.appendImageToWrapper)
  this.device.services.stream.on('removeplayer', this.removePlayerFromWrapper)
}
```

### 2.3 StreamingHandler 播放流程

位置：`core/device/streaming/streamingHandler.js`

```javascript
play(instantPlay = false) {
  if (instantPlay) {
    return this.$play(instantPlay)
  }
  return this.tryToPlayLooped()
}

$play(instantPlay = false) {
  this.reference.updateViewStatus(STREAMING_STATUS.LOADING)

  return this.createStreamingInterface()
    .then(streamingInterface => {
      const { duration } = this.reference.state
      const { timestamp, tzoffs } = this.getPlayTimeInfo()

      // 構建 RTSP URL
      const url = this.buildRTSPUrl({ timestamp, tzoffs, duration })

      return streamingInterface.play({
        instantPlay,
        timestamp,
        tzoffs,
        duration,
        url
      })
    })
}
```

### 2.4 Live RTSP URL 格式

位置：`core/device/streaming/vsaasStreamingHanlder.js`

```javascript
// 普通相機 (DEVICE_TYPE.CAMERA)
`rtsp://localhost/live_stream=${streamIndex}_channel=${channel}`

// NVR 通道 (DEVICE_TYPE.NVR_CHANNEL)
`rtsp://localhost/live?stream=${streamIndex}&channel=${channel}`

// VSS 通道 (DEVICE_TYPE.VSS_CHANNEL)
`rtsp://localhost/Media/VortexLive/Normal?channel=${channel}&stream=${streamIndex}`

// 參數說明：
// - streamIndex: 串流品質索引 (SD=1, HD=2 等)
// - channel: 攝像機或 NVR 通道編號
// - localhost: 由 WebRTC Data Channel 代理
```

### 2.5 Live 數據流圖

```
[攝像機] ─────> [WebRTC Data Channel] ─────> [RTSPPlayer] ─────> [video 元素]
                      │
                      ├─ RTSP 包傳輸
                      ├─ 控制指令
                      └─ VCA 元數據
```

---

## 3. Playback 播放流程

### 3.1 Archive 播放組件

位置：`src/pages/archive/management/components/ArchivePlayerPanel.vue`

```javascript
watch: {
  player() {
    if (!this.player) return
    this.createVideoDom()
    this.tryToPlay()
  }
}

tryToPlay() {
  this.streamingStatus = STREAMING_STATUS.LOADING

  this.player.on('play', this.handlePlay)
  this.player.on('stop', this.handleStop)
  this.player.on('notify', this.handleTimeUpdate)
  this.player.on('error', this.handleError)

  this.player.play(this.url)
}
```

### 3.2 Playback RTSP URL 格式

```javascript
// 普通相機 (DEVICE_TYPE.CAMERA)
`rtsp://localhost/playback?channel=${channel}&fusion=${fusion}`
// fusion: 是否融合 SD+HD，預設為 true

// NVR 通道
`rtsp://localhost/playback?channel=${channel}&stream=${playbackIndex}`

// VSS 通道
`rtsp://localhost/Media/VortexDatabase/Normal?channel=${channel}&stream=${playbackIndex}`

// 附加時間參數
`${url}?start=${startTimeInMs}&stop=${endTimeInMs}`
```

### 3.3 時間軸控制

位置：`src/pages/archive/management/components/ArchivePlayerTimeline.vue`

```javascript
methods: {
  seek(evt) {
    if (!isNaN(evt)) {
      // 從滑塊輸入
      this.$emit('seekPlayTime', evt)
    } else {
      // 從點擊時間軸
      const offset = evt.clientX - this.$refs.timeline.getBoundingClientRect().x
      const currentTime = this.getPlayTime(offset)
      this.$emit('seekPlayTime', currentTime)
    }
  },

  getPlayTime(offset) {
    const percent = offset / this.$refs.timeline.clientWidth
    return percent * this.duration
  }
}
```

### 3.4 回放尋點流程

```javascript
// StreamingHandler.seek()
seek() {
  return this.tryToPlayLooped(this.$seek.bind(this))
}

$seek() {
  this.reference.updateViewStatus(STREAMING_STATUS.SEEKING)
  const { duration } = this.reference.state
  const { timestamp, tzoffs } = this.getPlayTimeInfo()

  const url = this.buildRTSPUrl({ timestamp, tzoffs, duration })

  return this.streamingInterface.seek({
    timestamp,
    tzoffs,
    duration,
    url
  })
}
```

### 3.5 時間更新處理

位置：`core/device/streaming/timeHandler.js`

```javascript
handleNotify(notify) {
  // 更新經過的時間
  this.reference.state.elapsedTime =
    this.reference.streamingInterface?.getPlayerCurrentTime() * 1000

  // 更新攝像機時間
  if (notify?.timestamp?.stream) {
    this.reference.state.cameraTime = new LocalTime({
      timestamp: notify.timestamp.stream,
      tzoffs: notify.timestamp.tzoffs ?? LocalTime.getBrowserTimezoneOffset()
    })
  }
}

// 播放結束判斷
get isPlayEnd() {
  const { duration, elapsedTime, startingPointBySec } = this.reference.state
  return !!(duration && (elapsedTime >= duration + 1000 * startingPointBySec))
}
```

---

## 4. OdysseyConnectionManager

### 4.1 服務層架構

位置：`src/models/IoT/Odyssey/OdysseyConnectionManager.js`

```javascript
class OdysseyConnectionManager extends VDSI {
  // 連線狀態
  isConnected: boolean
  isRelay: boolean  // 是否使用 TURN 中繼

  // Data Channels
  httpChannel
  rtspChannel
  vcaChannel
  volalarmChannel
  audioBackChannel

  // 請求連線
  async requestConnection()           // HTTP 通道
  async requestStreamingConnection()  // RTSP 通道
  async requestVcaConnection()        // VCA 通道
  async requestVolalarmConnection()   // 警報通道
  async requestAudioBackChannelConnection() // 音頻通道

  // 釋放連線
  async releaseConnection()
  async releaseStreamingConnection()
  async releaseVcaConnection()

  // 連線信息
  async requestConnectionInfo()  // 獲取 P2P/RELAY 模式
}
```

### 4.2 連線請求流程

```
requestStreamingConnection()
  │
  ├─ 檢查現有 RTSP 通道是否可用
  │   ├─ 可用 → 直接返回
  │   └─ 不可用 → 繼續
  │
  ├─ 有 pager？
  │   └─ 等待 pager 的 'connected' 事件
  │
  └─ 無 pager？
      └─ createConnection(thingName, derivant)
          ├─ 創建 OdysseyConnectPager
          ├─ 等待連線完成
          └─ peerClient.getRtspChannel()
```

---

## 5. 串流狀態管理

### 5.1 STREAMING_STATUS 枚舉

```javascript
const STREAMING_STATUS = {
  LOADING: 'LOADING',      // 加載中
  PLAYING: 'PLAYING',      // 播放中
  PAUSED: 'PAUSED',        // 已暫停
  SEEKING: 'SEEKING',      // 尋點中
  STOPPED: 'STOPPED',      // 已停止
  STOPPING: 'STOPPING',    // 停止中
  BUFFERING: 'BUFFERING',  // 緩衝中
  ERROR: 'ERROR',          // 錯誤
  NOT_COMPATIBLE: 'NOT_COMPATIBLE',  // 不支援編解碼器
  NO_RECORDING: 'NO_RECORDING'       // 無錄影
}
```

### 5.2 Device Stream State

```javascript
device.services.stream.state = {
  status: STREAMING_STATUS,
  streamIndex: number,      // Live 串流品質
  playbackIndex: number,    // Playback 串流品質
  seekable: boolean,
  isFisheye: boolean,
  dewarpType: number,
  volume: number,
  muted: boolean,
  cameraTime: LocalTime,    // 攝像機當前時間
  playTime: LocalTime,      // 播放點
  elapsedTime: number,      // 已播放時長 (ms)
  duration: number,         // 總時長 (ms)
  currentEvent: Object      // 事件播放模式
}
```

---

## 6. 關鍵文件位置

### WebRTC 連線

| 文件 | 用途 |
|------|------|
| `src/models/IoT/Odyssey/OdysseyConnection.js` | 連線管理和生命週期 |
| `src/models/IoT/Odyssey/OdysseyConnectionManager.js` | Data Channel 管理 |
| `src/models/IoT/Odyssey/OdysseyConnectPager.js` | 連線重試和超時 |
| `packages/lib-webrtc-odyssey/src/lib/RTCPeerClient.js` | RTCPeerConnection |
| `packages/lib-webrtc-odyssey/src/lib/IoTSignaling.js` | MQTT 信令 |
| `packages/lib-webrtc-odyssey/src/lib/DataChannelWrapper.js` | Data Channel 包裝 |

### 串流播放

| 文件 | 用途 |
|------|------|
| `packages/advanced-vue-components/components/DisplayStreamingBlock.vue` | 串流容器組件 |
| `packages/advanced-vue-components/components/PluginlessWrapper.vue` | 媒體渲染組件 |
| `core/device/streaming/streamingHandler.js` | 播放流程控制 |
| `core/device/streaming/vsaasStreamingHanlder.js` | VSaaS 特定處理 |
| `core/device/streaming/timeHandler.js` | 時間和進度管理 |

### Archive 回放

| 文件 | 用途 |
|------|------|
| `src/pages/archive/management/components/ArchivePlayerPanel.vue` | 回放播放器 |
| `src/pages/archive/management/components/ArchivePlayerTimeline.vue` | 時間軸控制 |
| `src/pages/archive/management/components/ArchivePlayerControl.vue` | 播放控制 |

---

## 7. 常見開發任務

### 7.1 監聽串流狀態變化

```javascript
// 在組件中
watch(() => device.services.stream.state.status, (newStatus) => {
  console.log('串流狀態變更:', newStatus)
})

// 或使用事件
device.services.stream.on('statuschange', (status) => {
  console.log('狀態:', status)
})
```

### 7.2 手動控制播放

```javascript
// 播放
await device.services.stream.play()

// 暫停
await device.services.stream.pause()

// 繼續
await device.services.stream.resume()

// 停止
await device.services.stream.stop()

// 尋點（回放模式）
device.services.stream.state.playTime = new LocalTime({
  timestamp: targetTimestamp,
  tzoffs: timezoneOffset
})
await device.services.stream.seek()
```

### 7.3 切換串流品質

```javascript
// 切換 Live 串流品質
device.services.ui.state.streamIndex = 2  // HD

// 切換 Playback 串流品質
device.services.ui.state.playbackIndex = 1  // SD
```

### 7.4 除錯連線問題

```javascript
// 檢查連線狀態
console.log('連線狀態:', device.services.connection.isConnected)
console.log('是否中繼:', device.services.connection.isRelay)

// 檢查 Data Channel
console.log('RTSP 通道:', device.services.connection.rtspChannel)
console.log('HTTP 通道:', device.services.connection.httpChannel)

// 監聽連線事件
device.services.connection.on('connected', () => {
  console.log('連線建立')
})

device.services.connection.on('disconnected', () => {
  console.log('連線斷開')
})
```

---

## 8. Live vs Playback 差異

| 特性 | Live | Playback |
|------|------|----------|
| URL 格式 | `/live_stream=...` | `/playback?start=...&stop=...` |
| 時間參數 | 無 | 有（start, stop） |
| 尋點功能 | 不支援 | 支援 |
| 串流品質 | streamIndex | playbackIndex |
| 時間軸 | 無 | 有 |
| 播放速度 | 固定 | 可調整 |
| VCA 覆蓋 | 即時 | 歷史事件 |

---

## 9. 性能優化要點

1. **ICE Candidate 緩衝**：在 Remote SDP 設置前暫存，避免丟失
2. **Data Channel 復用**：多個播放器共享 PeerConnection
3. **H.265 硬體解碼**：支援 Web Codecs API
4. **自動重連**：連線斷開時自動重建
5. **TURN 中繼**：P2P 失敗時自動切換 RELAY 模式

---

## 10. 常見問題排查

| 問題 | 可能原因 | 檢查方式 |
|-----|--------|--------|
| 無法連接 | MQTT 連接失敗 | 檢查 AWS IoT Core 認證 |
| 播放無聲音 | 預設靜音 | 檢查 `defalutMute` 設定 |
| H.265 不支援 | 瀏覽器不支援 | 檢查 `isWebCodecsSupported` |
| 尋點失敗 | 無該時段錄影 | 檢查錄影時段 |
| 連線中斷 | 網路問題 | 檢查重連機制 |
| 畫面卡頓 | 頻寬不足 | 降低串流品質 |

---

## 11. 相關常數

```javascript
// 設備類型
DEVICE_TYPE = {
  CAMERA: 'camera',
  NVR_CHANNEL: 'nvr_channel',
  VSS_CHANNEL: 'vss_channel'
}

// 串流品質
STREAM_INDEX = {
  SD: 1,
  HD: 2,
  FHD: 3
}

// 魚眼去畸變類型
DEWARP_TYPE = {
  ORIGINAL: 0,
  PTZ: 1,
  PANORAMA: 2,
  DOUBLE_PANORAMA: 3
}
```
