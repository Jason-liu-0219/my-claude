---
name: vsaas-rtsp-protocol
description: Use when working with RTSP protocol, RTP packets, streaming commands, or playback control. Triggers on keywords like "RTSP", "RTP", "SETUP", "PLAY", "TEARDOWN", "封包", "packet".
version: 1.0.0
---

# VSaaS Portal - RTSP Protocol

RTSP 協議處理，包含命令控制、RTP 封包接收、時間控制。

---

## RTSP 命令

### SETUP

建立串流 session。

```javascript
// RTSPHandler.js
async setup(url) {
  const response = await this.sendCommand('SETUP', url, {
    Transport: 'RTP/AVP/TCP;unicast',
  });
  this.sessionId = response.headers['Session'];
}
```

### PLAY

開始播放串流。

```javascript
// Liveview 模式
async play() {
  await this.sendCommand('PLAY', this.url, {
    Range: 'npt=now-',
  });
}

// Playback 模式（指定時間）
async play({ timestamp }) {
  await this.sendCommand('PLAY', this.url, {
    Range: `npt=${timestamp}-`,
  });
}
```

### PAUSE

暫停播放。

```javascript
async pause() {
  await this.sendCommand('PAUSE', this.url);
}
```

### TEARDOWN

結束串流 session。

```javascript
async teardown() {
  await this.sendCommand('TEARDOWN', this.url);
  this.sessionId = null;
}
```

---

## RTP 封包結構

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       Sequence Number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             SSRC                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            Payload                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 欄位說明

| 欄位 | 位元 | 說明 |
|------|------|------|
| V | 2 | 版本（固定 2） |
| P | 1 | Padding |
| X | 1 | Extension |
| CC | 4 | CSRC count |
| M | 1 | Marker（幀結束標記） |
| PT | 7 | Payload Type |
| Sequence | 16 | 序號 |
| Timestamp | 32 | 時間戳 |
| SSRC | 32 | 同步來源識別 |

### Payload Type

| PT | 格式 | 說明 |
|----|------|------|
| 96 | H.264 | 動態 PT |
| 97 | H.265 | 動態 PT |
| 0 | PCMU | G.711 μ-law |
| 8 | PCMA | G.711 A-law |

---

## NALU 封裝（H.264）

### 單一 NALU

```
RTP Payload:
+---------------+
| NALU Header   | (1 byte)
+---------------+
| NALU Payload  |
+---------------+
```

### FU-A（分片）

大型 NALU 會被分成多個 RTP 封包。

```
RTP Payload:
+---------------+
| FU Indicator  | (1 byte)
+---------------+
| FU Header     | (1 byte)
+---------------+
| FU Payload    |
+---------------+

FU Indicator: [F|NRI|Type=28]
FU Header:    [S|E|R|Type]
  S = Start bit (第一片)
  E = End bit (最後一片)
```

### STAP-A（聚合）

多個小型 NALU 合併成一個 RTP 封包。

```
RTP Payload:
+---------------+
| STAP Header   | (1 byte, Type=24)
+---------------+
| NALU 1 Size   | (2 bytes)
| NALU 1        |
+---------------+
| NALU 2 Size   | (2 bytes)
| NALU 2        |
+---------------+
```

---

## 封包處理流程

```javascript
// RTSPHandler.js
processPacket(buffer) {
  // 1. 解析 RTP header
  const rtp = parseRtpHeader(buffer);
  
  // 2. 根據 payload type 路由
  switch (rtp.payloadType) {
    case 96: // H.264
      this.processH264(rtp, buffer);
      break;
    case 97: // H.265
      this.processH265(rtp, buffer);
      break;
    case 0:  // G.711 μ-law
    case 8:  // G.711 A-law
      this.processAudio(rtp, buffer);
      break;
  }
}

// NaluParser.js
processH264(rtp, buffer) {
  const payload = buffer.slice(RTP_HEADER_SIZE);
  const naluType = payload[0] & 0x1f;
  
  switch (naluType) {
    case 1-23: // 單一 NALU
      this.emitNalu(payload);
      break;
    case 24: // STAP-A
      this.parseStapA(payload);
      break;
    case 28: // FU-A
      this.parseFuA(payload, rtp);
      break;
  }
}
```

---

## 時間控制（Playback）

### NPT Range

```
Range: npt=<start>-<end>

# 從現在開始（Liveview）
Range: npt=now-

# 從指定時間開始
Range: npt=1234567890-

# 指定時間範圍
Range: npt=1234567890-1234567900
```

### Seek 實現

```javascript
// Playback.js
async seek(timestamp) {
  // 1. 停止當前串流
  await this.teardown();
  
  // 2. 清除 buffer
  this.streamProcessor.flush();
  
  // 3. 從新時間點開始播放
  await this.setup();
  await this.play({ timestamp });
}
```

### 播放速率

```javascript
// 設定播放速率
setPlaybackRate(rate) {
  this.sendCommand('PLAY', this.url, {
    Scale: rate.toString(), // 0.5, 1, 2, 4, ...
  });
}
```

---

## 關鍵檔案

```
packages/lib-rtsp-player/src/
├── RTSPPlayer.js              # 主入口
├── Liveview.js                # 即時串流
├── Playback.js                # 回放（extends Liveview）
├── handler/
│   ├── RTSPHandler.js         # RTSP 協議處理
│   └── StreamProcessor.js     # 串流處理管線
├── parser/
│   ├── NaluParser.js          # H.264 NALU 解析
│   ├── NaluH265Parser.js      # H.265 NALU 解析
│   ├── NaluAssembler.js       # NALU 組裝
│   └── RtpNaluTransformer.js  # RTP → NALU 轉換
└── constants/
    ├── NaluType.js            # NALU 類型常數
    └── PayloadType.js         # Payload Type 常數
```

---

## 常見問題排查

### 1. 畫面破碎

**可能原因：**
- NALU 分片重組錯誤
- 封包遺失
- SPS/PPS 未正確解析

```javascript
// 檢查 NALU 完整性
naluParser.on('nalu', (nalu) => {
  console.log('NALU type:', nalu.type);
  console.log('NALU size:', nalu.data.length);
});
```

### 2. 播放不流暢

**可能原因：**
- 網路抖動
- Buffer 不足
- Timestamp 不連續

```javascript
// 檢查 timestamp
rtspHandler.on('packet', (rtp) => {
  console.log('Timestamp:', rtp.timestamp);
  console.log('Sequence:', rtp.sequence);
});
```

### 3. Seek 失敗

**可能原因：**
- TEARDOWN 未完成
- 新連線建立失敗
- 時間戳格式錯誤

```javascript
// 確保 teardown 完成
await this.teardown();
// 等待一下再重新建立
await new Promise(r => setTimeout(r, 100));
await this.setup();
```

---

## 相關 Skills

| Skill | 說明 |
|-------|------|
| `vsaas-streaming-architecture` | 整體架構 |
| `vsaas-webrtc-connection` | WebRTC 連線 |
| `vsaas-video-remuxing` | 影片 remuxing |

---

## 更新日誌

### v1.0.0 (2025-02-01)
- 初始版本
- RTSP 命令說明
- RTP 封包結構
- NALU 封裝說明
- 時間控制
