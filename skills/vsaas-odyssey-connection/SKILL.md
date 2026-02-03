---
name: vsaas-odyssey-connection
description: Use when working with OdysseyConnection in VSaaS Portal, WebRTC connection management, IoT signaling, or device streaming connections. Triggers on keywords like "OdysseyConnection", "連線管理", "IoT", "signaling", "ICE servers".
version: 1.0.0
---

# VSaaS Portal - Odyssey Connection

Portal 層級的 WebRTC 連線管理，包含連線池、IoT signaling、ICE server 更新。

---

## OdysseyConnection 類別

**位置**: `packages/app-vsaas-portal/src/models/IoT/Odyssey/OdysseyConnection.js`

靜態類別，管理所有 WebRTC 連線的生命週期。

---

## 核心功能

### 初始化

```javascript
// 應用程式啟動時呼叫
await OdysseyConnection.initialize();
```

**流程：**
1. 取得 signaling essentials（ICE servers、MQTT credentials）
2. 建立 IoT MQTT 連線
3. 註冊重連函數

### 建立連線

```javascript
// 連線到設備
const peerClient = await OdysseyConnection.connect({
  thingName: 'device-xxx',
  derivant: 'main', // 或 'none'
});

// peerClient 是 RTCPeerClient 實例
peerClient.on('datachannel', (channel) => {
  // 開始接收串流
});
```

### 關閉連線

```javascript
// 關閉特定連線
OdysseyConnection.close({
  thingName: 'device-xxx',
  derivant: 'main',
});
```

---

## 連線池管理

```javascript
// 內部結構
static connections = new Map();  // connectionKey → { peerClient, closeHandler }
static promiseQueue = new Map(); // connectionKey → Promise

// connectionKey 格式
// useSeparateDerivantConnection = false: thingName
// useSeparateDerivantConnection = true:  thingName-derivant
```

### 連線復用

```javascript
// 如果連線已存在且連接中，直接返回
if (connections.has(connectionKey)) {
  const { peerClient } = connections.get(connectionKey);
  if (peerClient.connected) {
    return Promise.resolve(peerClient);
  }
}
```

### 防止重複連線

```javascript
// 使用 promiseQueue 避免同時多次連線
if (this.promiseQueue.has(connectionKey)) {
  return this.promiseQueue.get(connectionKey);
}
```

---

## Signaling Essentials

### 取得 Essentials

```javascript
static getSignalingEssentials = async () => {
  const response = await ApiService.requestToIoTServer({
    apiName: `clients/${clientId}/signalingEssentials`,
    method: 'GET',
  });
  
  // 回傳內容
  // {
  //   iceServers: [...],      // STUN/TURN servers
  //   mqttCred: {...},        // MQTT 憑證
  //   expires: '2025-01-01T00:00:00Z'
  // }
  
  return response;
};
```

### 自動更新 ICE Servers

```javascript
static scheduleUpdateIceServers() {
  const timeout = this.expiredTime - Date.now();
  
  this.scheduleUpdateEssentialsTimeout = setTimeout(() => {
    this.getSignalingEssentials().then(({ iceServers }) => {
      IoTSignaling.updateIceServers(iceServers);
      this.scheduleUpdateIceServers(); // 遞迴排程
    });
  }, timeout);
}
```

---

## IoT 連線

### 建立 MQTT 連線

```javascript
static async createIoTConnection(clearSignaling = false) {
  const { iceServers, mqttCred } = await this.getSignalingEssentials();
  
  IoTSignaling.updateIceServers(iceServers);
  
  const iotConnectFactory = new IoTClient({ clientId });
  const mqttClient = await iotConnectFactory.getIoTConnect(mqttCred);
  
  IoTSignaling.updateMqttClient(mqttClient);
  ConnectionManager.register(clientId, mqttClient);
  
  return mqttClient;
}
```

### 重連機制

```javascript
// 註冊重連函數（bfcache 或網路中斷後）
ConnectionManager.registReconnectFn('OdysseyConnection', async () => {
  await OdysseyConnection.createIoTConnection(true);
});
```

---

## 建立 Peer 連線

```javascript
static async createPeerClient({ thingName, derivant }) {
  // 1. 確保 IoT 連線存在
  await this.createIoTConnection();

  // 2. 建立 RTCPeerClient
  const peerClient = new RTCPeerClient({
    Signaling: IoTSignaling,
  });

  // 3. 儲存到連線池
  const connectionKey = this.getConnectionKey({ thingName, derivant });
  this.connections.set(connectionKey, { peerClient, closeHandler });

  // 4. 監聽關閉事件
  peerClient.on('close', closeHandler);

  // 5. 連線到設備
  await peerClient.connect({ thingName, derivant });
  
  return peerClient;
}
```

---

## 環境變數

```javascript
// IoT 設定
IoTClient.region = import.meta.env.VITE_AMPLIFY_BACKEND_REGION;
IoTClient.host = import.meta.env.VITE_AWS_IOT_CORE_DATA_PLANE_ENDPOINT;
```

---

## 共享設備支援

```javascript
// 共享設備使用不同的 API 路徑
if (Context.provide('sharedDeviceViewerToken')) {
  return ApiService.requestToIoTServerWithSharedDeviceViewerToken({
    apiName: `shared-devices/things/${thingName}/rtcsess/${sessionId}/preoffer`,
    method: 'POST',
    body: { clientId, derivant },
  });
}
```

---

## 關鍵檔案

```
packages/app-vsaas-portal/src/
├── models/IoT/Odyssey/
│   ├── OdysseyConnection.js       # 主要連線管理
│   ├── OdysseyConnectionManager.js # 連線池管理（進階）
│   └── OdysseyConnectPager.js     # 連線分頁器
└── singletons/
    └── ConnectionManager.js       # 全域連線管理器
```

---

## 使用範例

### 在 Vue 組件中使用

```javascript
// PlayerWrapper.vue
import OdysseyConnection from '@/models/IoT/Odyssey/OdysseyConnection';

async function startStreaming(device) {
  try {
    const peerClient = await OdysseyConnection.connect({
      thingName: device.thingName,
      derivant: device.derivant,
    });
    
    // 取得 data channel
    const channel = peerClient.getDataChannel('rtsp');
    
    // 開始 RTSP 串流
    rtspPlayer.setDataChannel(channel);
    await rtspPlayer.play();
    
  } catch (error) {
    console.error('Connection failed:', error);
  }
}

// 組件銷毀時關閉連線
onUnmounted(() => {
  OdysseyConnection.close({
    thingName: device.thingName,
    derivant: device.derivant,
  });
});
```

---

## 常見問題排查

### 1. 連線失敗 - IoT 連線問題

```javascript
// 檢查 MQTT 連線狀態
console.log('MQTT connected:', IoTSignaling.mqttClient?.connected);
```

**可能原因：**
- MQTT credentials 過期
- 網路問題
- IoT endpoint 錯誤

### 2. ICE Servers 過期

```javascript
// 檢查過期時間
console.log('ICE expires:', new Date(OdysseyConnection.expiredTime));
```

**解決方式：**
- 確認 `scheduleUpdateIceServers` 有正常執行
- 手動觸發更新

### 3. 連線池問題

```javascript
// 查看現有連線
console.log('Connections:', OdysseyConnection.connections);
console.log('Promise queue:', OdysseyConnection.promiseQueue);
```

### 4. 重連後連線失敗

**可能原因：**
- 舊的 signaling 未清除
- MQTT client 狀態不一致

```javascript
// 強制重建連線
await OdysseyConnection.createIoTConnection(true); // clearSignaling = true
```

---

## 相關 Skills

| Skill | 說明 |
|-------|------|
| `vsaas-streaming-architecture` | 整體架構 |
| `vsaas-webrtc-connection` | WebRTC 底層 |
| `vsaas-rtsp-protocol` | RTSP 協議 |

---

## 更新日誌

### v1.0.0 (2025-02-01)
- 初始版本
- OdysseyConnection 類別說明
- 連線池管理
- Signaling essentials
- 使用範例
