---
name: vsaas-streaming-architecture
description: Use when working with VSaaS Portal streaming architecture, understanding video/audio flow, WebRTC vs RTSP, or debugging streaming issues. Triggers on keywords like "streaming", "串流", "播放", "video", "audio", "WebRTC", "RTSP".
version: 1.0.0
---

# VSaaS Portal - Streaming 架構概覽

串流系統架構說明，包含套件依賴、資料流、Liveview vs Playback 模式。

---

## 核心套件

| 套件 | 職責 | 說明 |
|------|------|------|
| `lib-rtsp-player` | RTSP 播放器 | 封包解析、remuxing、播放控制 |
| `lib-webrtc-odyssey` | WebRTC 連線 | P2P 連線、Data Channel、Signaling |
| `lib-medama` | 舊版播放器 | WASM 解碼（legacy） |
| `lib-device-plugin-odyssey` | 新版 P2P 外掛 | TypeScript 重構版 |

---

## 資料流架構

```
┌─────────────────────────────────────────────────────────────────┐
│                          Device (Camera/NVR)                     │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                    WebRTC Data Channel (Binary)
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  lib-webrtc-odyssey                                              │
│  ┌─────────────────┐    ┌──────────────────┐                    │
│  │ RTCPeerClient   │───▶│ DataChannelWrapper│                    │
│  └─────────────────┘    └──────────────────┘                    │
│           │                      │                               │
│           ▼                      │                               │
│  ┌─────────────────┐             │                               │
│  │ IoTSignaling    │             │                               │
│  └─────────────────┘             │                               │
└──────────────────────────────────┼──────────────────────────────┘
                                   │
                          RTP Packets (Binary)
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  lib-rtsp-player                                                 │
│  ┌─────────────────┐                                            │
│  │ RTSPHandler     │◀── SETUP / PLAY / TEARDOWN                 │
│  └────────┬────────┘                                            │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────┐    ┌─────────────────┐                     │
│  │ RtpTransformer  │───▶│ NaluParser      │                     │
│  └─────────────────┘    └────────┬────────┘                     │
│                                  │                               │
│                                  ▼                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Remuxer                                                  │    │
│  │ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │    │
│  │ │H264Remuxer  │ │H265Remuxer  │ │AudioRemuxer │         │    │
│  │ └─────────────┘ └─────────────┘ └─────────────┘         │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │                                    │
│              ┌──────────────┴──────────────┐                    │
│              ▼                             ▼                     │
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │ MSE Path            │    │ WebCodecs Path      │             │
│  │ MediaSourceController│    │ WebCodecsController │             │
│  │ SourceBuffer        │    │ VideoDecoder        │             │
│  │ Video Element       │    │ Canvas              │             │
│  └─────────────────────┘    └─────────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 播放模式

### Liveview（即時串流）

```javascript
// 使用方式
const player = new RTSPPlayer({ mode: 'liveview' });
player.play();
```

**特點：**
- 即時影像，低延遲
- 單向串流（Device → Browser）
- 支援雙向音訊

### Playback（回放）

```javascript
// 使用方式
const player = new RTSPPlayer({ mode: 'playback' });
player.play({ timestamp: 1234567890 });
player.seek({ timestamp: 1234567900 });
```

**特點：**
- 歷史影像回放
- 支援 seek、暫停、播放速率
- 時間範圍控制（npt range）

---

## 解碼策略

### MSE（MediaSource Extensions）

```
RTP → Remuxer → MP4 Boxes → SourceBuffer → Video Element
```

**優點：**
- 瀏覽器原生支援
- 穩定性高

**缺點：**
- 延遲較高（~500ms+）

### WebCodecs

```
RTP → Remuxer → VideoDecoder → VideoFrame → Canvas
```

**優點：**
- 低延遲（~100ms）
- 硬體加速

**缺點：**
- 需要較新瀏覽器
- 需要 Canvas 渲染

### 策略選擇

```javascript
// StrategyHelper.js
function chooseStrategy() {
  if (isWebCodecsSupported() && isLowLatencyRequired()) {
    return 'webcodecs';
  }
  return 'mse';
}
```

---

## 支援的編碼格式

### 影片

| 格式 | 說明 | 檔案 |
|------|------|------|
| H.264 (AVC) | 主流格式 | `H264Remuxer.js`, `NaluParser.js` |
| H.265 (HEVC) | 高效編碼 | `H265Remuxer.js`, `NaluH265Parser.js` |

### 音訊

| 格式 | 說明 | 檔案 |
|------|------|------|
| G.711 μ-law | PCM 編碼 | `AudioRemuxer.js` |
| G.726 | ADPCM 編碼 | `AudioRemuxer.js` |
| AAC | 進階編碼 | `AACRemuxer.js` |

---

## 關鍵檔案路徑

```
# lib-rtsp-player
packages/lib-rtsp-player/src/
├── RTSPPlayer.js              # 主入口（Facade）
├── Liveview.js                # 即時串流模式
├── Playback.js                # 回放模式
├── handler/
│   ├── RTSPHandler.js         # RTSP 協議處理
│   └── StreamProcessor.js     # 串流處理管線
├── parser/
│   ├── NaluParser.js          # H.264 NALU 解析
│   └── NaluH265Parser.js      # H.265 NALU 解析
├── remuxer/
│   ├── Remuxer.js             # Remuxer 控制器
│   ├── H264Remuxer.js         # H.264 remuxing
│   ├── H265Remuxer.js         # H.265 remuxing
│   └── AudioRemuxer.js        # 音訊 remuxing
├── player/
│   ├── VideoPlayer.js         # MSE 影片播放
│   └── WebCodecsVideoPlayer.js # WebCodecs 影片播放
└── media/
    ├── MediaSourceController.js  # MSE 控制
    └── WebCodecsController.js    # WebCodecs 控制

# lib-webrtc-odyssey
packages/lib-webrtc-odyssey/src/lib/
├── RTCPeerClient.js           # WebRTC 連線
├── DataChannelWrapper.js      # Data Channel 封裝
└── IoTSignaling.js            # AWS IoT Signaling

# VSaaS Portal 整合
packages/app-vsaas-portal/src/
├── models/IoT/Odyssey/
│   └── OdysseyConnection.js   # 連線管理
└── components/Player/
    └── PlayerWrapper.vue      # 播放器元件
```

---

## 常見問題排查

### 1. 連線失敗
- 檢查 `RTCPeerClient` 連線狀態
- 確認 ICE Server 配置
- 查看 Signaling 是否成功

### 2. 畫面黑屏
- 檢查 RTP 封包是否收到
- 確認 SPS/PPS 是否正確解析
- 查看 SourceBuffer 錯誤

### 3. 延遲過高
- 考慮切換到 WebCodecs
- 檢查 buffer 設定
- 確認網路品質

### 4. 音訊問題
- 檢查 AudioContext 狀態
- 確認音訊格式（G.711 / AAC）
- 查看音訊同步邏輯

---

## 相關 Skills

| Skill | 說明 |
|-------|------|
| `vsaas-webrtc-connection` | WebRTC 連線細節 |
| `vsaas-rtsp-protocol` | RTSP 協議細節 |
| `vsaas-odyssey-connection` | Portal 連線管理 |

---

## 更新日誌

### v1.0.0 (2025-02-01)
- 初始版本
- 架構概覽、資料流圖
- 播放模式說明
- 解碼策略說明
