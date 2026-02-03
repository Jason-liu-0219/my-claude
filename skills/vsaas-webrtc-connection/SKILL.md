---
name: vsaas-webrtc-connection
description: Use when working with WebRTC connections, Data Channels, Signaling, or P2P communication issues. Triggers on keywords like "WebRTC", "RTCPeerClient", "DataChannel", "Signaling", "ICE", "P2P".
version: 1.0.0
---

# VSaaS Portal - WebRTC Connection

WebRTC 連線機制，包含 RTCPeerClient、Data Channel、Signaling。

---

## 核心元件

### RTCPeerClient

**位置**: `packages/lib-webrtc-odyssey/src/lib/RTCPeerClient.js`

WebRTC 連線的主控類，管理 peer connection 生命週期。

```javascript
// 建立連線
const client = new RTCPeerClient({
  thingName: 'device-xxx',
  derivant: 'main',
  iceServers: [...],
  signalingClient: signalingInstance,
});

// 連線事件
client.on('connected', () => {});
client.on('disconnected', () => {});
client.on('datachannel', (channel) => {});

// 建立連線
await client.connect();
```

### DataChannelWrapper

**位置**: `packages/lib-webrtc-odyssey/src/lib/DataChannelWrapper.js`

封裝 RTCDataChannel，提供統一的事件介面。

```javascript
const channel = new DataChannelWrapper(rtcDataChannel);

channel.on('message', (data) => {
  // data: ArrayBuffer (RTP packets)
});

channel.on('open', () => {});
channel.on('close', () => {});
channel.on('error', (error) => {});

// 傳送資料（雙向音訊）
channel.send(buffer);
```

### Signaling

**位置**: `packages/lib-webrtc-odyssey/src/lib/IoTSignaling.js`

使用 AWS IoT 進行 WebRTC signaling。

```javascript
const signaling = new IoTSignaling({
  endpoint: 'xxx.iot.region.amazonaws.com',
  credentials: {...},
});

// Signaling 事件
signaling.on('offer', (sdp) => {});
signaling.on('answer', (sdp) => {});
signaling.on('candidate', (candidate) => {});

// 發送 signaling 訊息
signaling.sendOffer(sdp);
signaling.sendAnswer(sdp);
signaling.sendCandidate(candidate);
```

---

## 連線流程

```
┌─────────────┐                    ┌─────────────┐
│   Browser   │                    │   Device    │
└──────┬──────┘                    └──────┬──────┘
       │                                  │
       │  1. Connect to Signaling (IoT)   │
       │─────────────────────────────────▶│
       │                                  │
       │  2. Send Offer (SDP)             │
       │─────────────────────────────────▶│
       │                                  │
       │  3. Receive Answer (SDP)         │
       │◀─────────────────────────────────│
       │                                  │
       │  4. Exchange ICE Candidates      │
       │◀────────────────────────────────▶│
       │                                  │
       │  5. P2P Connection Established   │
       │◀═══════════════════════════════▶│
       │                                  │
       │  6. Data Channel Open            │
       │◀═══════════════════════════════▶│
       │                                  │
       │  7. RTP Packets via DataChannel  │
       │◀─────────────────────────────────│
       │                                  │
```

---

## Data Channel 類型

### 影片串流 Channel

```javascript
// Label 格式
const label = `rtsp-${thingName}-${derivant}-${channelId}`;

// 接收 RTP 封包
channel.on('message', (buffer) => {
  rtspHandler.processPacket(buffer);
});
```

### 控制 Channel

```javascript
// PTZ 控制、設定等
const controlChannel = connection.getControlChannel();
controlChannel.send(JSON.stringify({ type: 'ptz', action: 'left' }));
```

### 雙向音訊 Channel

```javascript
// 發送麥克風音訊（G.711）
const audioChannel = connection.getAudioChannel();
audioChannel.send(g711Buffer);
```

---

## ICE Server 配置

```javascript
const iceServers = [
  {
    urls: 'stun:stun.example.com:3478',
  },
  {
    urls: 'turn:turn.example.com:3478',
    username: 'user',
    credential: 'pass',
  },
];

// 動態更新 ICE Servers
client.updateIceServers(newIceServers);
```

---

## 連線狀態

```javascript
// RTCPeerConnection 狀態
client.on('connectionstatechange', (state) => {
  // 'new' | 'connecting' | 'connected' | 'disconnected' | 'failed' | 'closed'
});

// ICE 連線狀態
client.on('iceconnectionstatechange', (state) => {
  // 'new' | 'checking' | 'connected' | 'completed' | 'failed' | 'disconnected' | 'closed'
});

// ICE 收集狀態
client.on('icegatheringstatechange', (state) => {
  // 'new' | 'gathering' | 'complete'
});
```

---

## 關鍵檔案

```
packages/lib-webrtc-odyssey/src/
├── lib/
│   ├── RTCPeerClient.js          # WebRTC 連線主控
│   ├── DataChannelWrapper.js     # Data Channel 封裝
│   ├── IoTSignaling.js           # AWS IoT Signaling
│   ├── WebsocketSignaling.js     # WebSocket Signaling（備用）
│   └── MockRTCPeerClient.js      # 測試用 Mock
├── utility/
│   ├── genChannelLabel.js        # 產生 channel label
│   ├── getConnectionInfo.js      # 連線資訊
│   └── peerConnectionStatsUtility.js  # 連線統計
└── graphql/
    └── mutations.js              # GraphQL 請求
```

---

## 常見問題排查

### 1. 連線失敗 - ICE Failed

```javascript
// 檢查 ICE 狀態
client.on('iceconnectionstatechange', (state) => {
  if (state === 'failed') {
    console.error('ICE failed, check TURN server');
  }
});
```

**可能原因：**
- TURN server 不可用
- 防火牆阻擋 UDP
- NAT 穿透失敗

### 2. Signaling 失敗

```javascript
// 檢查 IoT 連線
signaling.on('error', (error) => {
  console.error('Signaling error:', error);
});
```

**可能原因：**
- IoT endpoint 錯誤
- Credentials 過期
- 網路問題

### 3. Data Channel 沒開啟

```javascript
// 確認 channel 狀態
channel.on('open', () => {
  console.log('Channel opened');
});

channel.on('error', (error) => {
  console.error('Channel error:', error);
});
```

**可能原因：**
- 連線尚未建立
- Device 端未創建 channel
- Label 不匹配

### 4. 封包遺失

```javascript
// 檢查連線統計
const stats = await client.getStats();
console.log('Packets lost:', stats.packetsLost);
console.log('Jitter:', stats.jitter);
```

---

## 除錯工具

### Chrome WebRTC Internals

```
chrome://webrtc-internals/
```

可查看：
- ICE candidates
- SDP offer/answer
- 連線統計
- Data channel 狀態

### 程式碼除錯

```javascript
// 開啟 debug 模式
RTCPeerClient.setDebug(true);

// 監聽所有事件
client.on('*', (event, data) => {
  console.log('Event:', event, data);
});
```

---

## 相關 Skills

| Skill | 說明 |
|-------|------|
| `vsaas-streaming-architecture` | 整體架構 |
| `vsaas-odyssey-connection` | Portal 連線管理 |
| `vsaas-rtsp-protocol` | RTSP 協議 |

---

## 更新日誌

### v1.0.0 (2025-02-01)
- 初始版本
- RTCPeerClient、DataChannelWrapper、Signaling 說明
- 連線流程圖
- 常見問題排查
