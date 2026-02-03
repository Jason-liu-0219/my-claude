---
name: vsaas-rtp-packet
description: Use when working with RTP packet parsing, NALU fragmentation/reassembly, or debugging packet-level streaming issues. Triggers on keywords like "RTP", "packet", "NALU", "FU-A", "fragmentation", "封包".
version: 1.0.0
---

# VSaaS Portal - RTP Packet Handling

RTP 封包解析、NALU 分片重組、封包處理流程。

---

## RTP Header 結構

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
```

### 解析程式碼

```javascript
// RtpNaluTransformer.js
function parseRtpHeader(buffer) {
  const view = new DataView(buffer);
  
  const firstByte = view.getUint8(0);
  const secondByte = view.getUint8(1);
  
  return {
    version: (firstByte >> 6) & 0x03,        // V: 2 bits
    padding: (firstByte >> 5) & 0x01,        // P: 1 bit
    extension: (firstByte >> 4) & 0x01,      // X: 1 bit
    csrcCount: firstByte & 0x0f,             // CC: 4 bits
    marker: (secondByte >> 7) & 0x01,        // M: 1 bit (幀結束)
    payloadType: secondByte & 0x7f,          // PT: 7 bits
    sequenceNumber: view.getUint16(2),       // 16 bits
    timestamp: view.getUint32(4),            // 32 bits
    ssrc: view.getUint32(8),                 // 32 bits
  };
}

const RTP_HEADER_SIZE = 12;
```

---

## H.264 NALU 類型

### NALU Header（1 byte）

```
+---------------+
|F|NRI|  Type   |
+---------------+
 1  2    5 bits
```

### 常見類型

| Type | 名稱 | 說明 |
|------|------|------|
| 1 | Non-IDR | P-frame / B-frame |
| 5 | IDR | I-frame（關鍵幀）|
| 6 | SEI | 補充增強資訊 |
| 7 | SPS | Sequence Parameter Set |
| 8 | PPS | Picture Parameter Set |
| 24 | STAP-A | 聚合封包 |
| 28 | FU-A | 分片封包 |

```javascript
// NaluType.js
const NALU_TYPE = {
  NON_IDR: 1,
  IDR: 5,
  SEI: 6,
  SPS: 7,
  PPS: 8,
  STAP_A: 24,
  FU_A: 28,
};
```

---

## FU-A 分片處理

大型 NALU（如 IDR frame）會被分成多個 RTP 封包。

### FU-A 結構

```
RTP Payload:
+---------------+---------------+
| FU Indicator  | FU Header     | FU Payload...
+---------------+---------------+
     1 byte         1 byte

FU Indicator: [F|NRI|Type=28]
FU Header:    [S|E|R|Type]
```

### FU Header 欄位

| 欄位 | Bit | 說明 |
|------|-----|------|
| S | 1 | Start - 第一個分片 |
| E | 1 | End - 最後一個分片 |
| R | 1 | Reserved |
| Type | 5 | 原始 NALU 類型 |

### 重組邏輯

```javascript
// NaluParser.js
class NaluParser {
  constructor() {
    this.fuBuffer = null;
    this.fuStarted = false;
  }

  parseFuA(payload, rtp) {
    const fuIndicator = payload[0];
    const fuHeader = payload[1];
    const fuPayload = payload.slice(2);
    
    const startBit = (fuHeader >> 7) & 0x01;
    const endBit = (fuHeader >> 6) & 0x01;
    const naluType = fuHeader & 0x1f;
    
    if (startBit) {
      // 第一個分片：建立 NALU header
      const naluHeader = (fuIndicator & 0xe0) | naluType;
      this.fuBuffer = new Uint8Array([naluHeader, ...fuPayload]);
      this.fuStarted = true;
      
    } else if (this.fuStarted) {
      // 中間/最後分片：附加到 buffer
      this.fuBuffer = concatBuffers(this.fuBuffer, fuPayload);
      
      if (endBit) {
        // 最後一個分片：發送完整 NALU
        this.emit('nalu', {
          type: naluType,
          data: this.fuBuffer,
          timestamp: rtp.timestamp,
        });
        this.fuBuffer = null;
        this.fuStarted = false;
      }
    }
  }
}
```

---

## STAP-A 聚合處理

多個小型 NALU（如 SPS + PPS）合併成一個 RTP 封包。

### STAP-A 結構

```
+---------------+
| STAP Header   |  (1 byte, Type=24)
+---------------+
| NALU 1 Size   |  (2 bytes, big-endian)
+---------------+
| NALU 1 Data   |
+---------------+
| NALU 2 Size   |  (2 bytes)
+---------------+
| NALU 2 Data   |
+---------------+
```

### 解析邏輯

```javascript
// NaluParser.js
parseStapA(payload) {
  let offset = 1; // Skip STAP header
  
  while (offset < payload.length) {
    // 讀取 NALU 大小（2 bytes）
    const naluSize = (payload[offset] << 8) | payload[offset + 1];
    offset += 2;
    
    // 讀取 NALU 資料
    const naluData = payload.slice(offset, offset + naluSize);
    offset += naluSize;
    
    const naluType = naluData[0] & 0x1f;
    
    this.emit('nalu', {
      type: naluType,
      data: naluData,
    });
  }
}
```

---

## H.265 NALU 差異

### NALU Header（2 bytes）

```
+---------------+---------------+
|F|   Type   |  LayerId  | TID |
+---------------+---------------+
 1    6 bits     6 bits   3 bits
```

### 額外類型

| Type | 名稱 | 說明 |
|------|------|------|
| 32 | VPS | Video Parameter Set |
| 33 | SPS | Sequence Parameter Set |
| 34 | PPS | Picture Parameter Set |
| 49 | FU | 分片單元 |

```javascript
// NaluH265Parser.js
function parseH265NaluType(byte1, byte2) {
  return (byte1 >> 1) & 0x3f;
}
```

---

## 封包處理流程

```
DataChannel (binary)
       │
       ▼
┌─────────────────────┐
│ RtpNaluTransformer  │
│ - parseRtpHeader()  │
│ - route by PT       │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ NaluParser          │
│ - Single NALU       │
│ - STAP-A            │
│ - FU-A              │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ NaluAssembler       │
│ - Add start code    │
│ - Emit complete AU  │
└──────────┬──────────┘
           │
           ▼
     Remuxer
```

---

## Web Worker 處理

為避免 UI blocking，封包處理可在 Worker 中執行。

```javascript
// rtp-transformer.worker.js
self.onmessage = (event) => {
  const { type, data } = event.data;
  
  if (type === 'packet') {
    const nalu = processRtpPacket(data);
    if (nalu) {
      self.postMessage({ type: 'nalu', nalu });
    }
  }
};

// RtpTransformerWorkerProxy.js
class RtpTransformerWorkerProxy {
  constructor() {
    this.worker = new Worker('rtp-transformer.worker.js');
    this.worker.onmessage = this.handleMessage.bind(this);
  }
  
  processPacket(buffer) {
    this.worker.postMessage({ type: 'packet', data: buffer });
  }
}
```

---

## 關鍵檔案

```
packages/lib-rtsp-player/src/
├── parser/
│   ├── NaluParser.js            # H.264 NALU 解析
│   ├── NaluH265Parser.js        # H.265 NALU 解析
│   ├── NaluAssembler.js         # NALU 組裝（加 start code）
│   └── RtpNaluTransformer.js    # RTP → NALU 轉換
├── workers/
│   ├── rtp-transformer.worker.js
│   └── RtpTransformerWorkerProxy.js
├── model/
│   ├── NALU.js                  # NALU 資料模型
│   └── NALU_H265.js             # H.265 NALU 模型
└── constants/
    ├── NaluType.js              # NALU 類型常數
    └── PayloadType.js           # Payload Type 常數
```

---

## 常見問題排查

### 1. 封包遺失

```javascript
// 檢查 sequence number 連續性
let lastSeq = null;
rtpHandler.on('packet', (rtp) => {
  if (lastSeq !== null) {
    const expected = (lastSeq + 1) & 0xffff;
    if (rtp.sequenceNumber !== expected) {
      console.warn('Packet lost:', expected, '→', rtp.sequenceNumber);
    }
  }
  lastSeq = rtp.sequenceNumber;
});
```

### 2. FU-A 重組失敗

```javascript
// 檢查 FU 狀態
naluParser.on('fu-error', (error) => {
  console.error('FU reassembly error:', error);
  // 可能原因：中間封包遺失
});
```

### 3. 畫面花屏

**可能原因：**
- SPS/PPS 未正確解析
- IDR frame 不完整
- NALU start code 錯誤

```javascript
// 確認收到 SPS/PPS
naluParser.on('nalu', (nalu) => {
  if (nalu.type === 7) console.log('Got SPS');
  if (nalu.type === 8) console.log('Got PPS');
});
```

---

## 相關 Skills

| Skill | 說明 |
|-------|------|
| `vsaas-rtsp-protocol` | RTSP 協議層 |
| `vsaas-video-remuxing` | 影片 remuxing |
| `vsaas-streaming-architecture` | 整體架構 |

---

## 更新日誌

### v1.0.0 (2025-02-01)
- 初始版本
- RTP header 解析
- NALU 類型說明
- FU-A / STAP-A 處理
- Web Worker 模式
