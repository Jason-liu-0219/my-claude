# VSaaS Portal - Video Remuxing

影片 Remuxing 模組負責將 RTP 封包中的 NALU 轉換為播放器可用的格式（MP4 box 或 WebCodecs）。

## 模組概述

### 職責
- 解析 H.264/H.265 NALU 結構
- 提取 SPS/PPS/VPS 參數集
- 計算影片解析度與 codec 資訊
- 組裝成 SSMUX 可接受的封包格式
- GOP (Group of Pictures) 管理

---

## 檔案結構

```
packages/lib-rtsp-player/src/
├── remuxer/
│   ├── Remuxer.js              # 中央控制器，分派封包到對應 remuxer
│   ├── AbstractRemuxer.js       # 影片 remuxer 基類
│   ├── H264Remuxer.js          # H.264 remuxer
│   ├── H265Remuxer.js          # H.265/HEVC remuxer
│   ├── H264Parser.js           # H.264 NALU 解析器
│   ├── H265Parser.js           # H.265 NALU 解析器
│   ├── AbstractAudioRemuxer.js  # 音訊 remuxer 基類
│   ├── AudioRemuxer.js         # G.711/G.726 remuxer
│   └── AACRemuxer.js           # AAC remuxer
├── constants/
│   └── NaluType.js             # NALU 類型常數
└── utils/
    ├── H264Tools.js            # H.264 SPS 解析工具
    ├── H265Tools.js            # H.265 SPS 解析工具
    └── ExpGolombDecoder.js     # Exp-Golomb 編碼解碼器
```

---

## 類別繼承

```
Remuxer (中央控制器)
    │
    ├── AbstractRemuxer (影片基類)
    │   ├── H264Remuxer
    │   └── H265Remuxer
    │
    └── AbstractAudioRemuxer (音訊基類)
        ├── AudioRemuxer (G.711)
        └── AACRemuxer
```

---

## 核心類別

### Remuxer（中央控制器）

```javascript
// packages/lib-rtsp-player/src/remuxer/Remuxer.js

class Remuxer extends EventEmitter {
  constructor(streamProcessor, rtspHandler) {
    this.streamProcessor = streamProcessor;
    this.rtspHandler = rtspHandler;
    this.tracks = {};

    // 支援的影片 codec
    this.videoCodecMap = {
      H264: H264Remuxer,
      H265: H265Remuxer,
    };

    // 支援的音訊 codec
    this.audioCodecMap = {
      AAC: AACRemuxer,
      G711: AudioRemuxer,
      G711A: AudioRemuxer,
      G726: AudioRemuxer
    };

    // 監聽 RTSP 事件
    rtspHandler.on('tracks', this.processTracks);
    rtspHandler.on('media-packet', this.processPacket);
  }

  processTracks(tracks) {
    // 為每個 track 建立對應的 remuxer
  }

  processPacket(type, trunk) {
    // 將封包分派到對應的 remuxer
    this.tracks[type].remux(trunk, trunk.applicationData);
  }
}
```

### AbstractRemuxer（影片基類）

```javascript
// packages/lib-rtsp-player/src/remuxer/AbstractRemuxer.js

class AbstractRemuxer {
  constructor({ streamProcessor, track }) {
    this.track = {
      type: 'video',
      fragmented: true,
      width: 0,
      height: 0,
      timescale: 90000,  // 預設 90kHz
      ...track,
    };

    this.lastGopDTS = -99999999999999;
    this.lastGopPackets = [];
    this.queue = [];
    this.isReady = true;
  }

  get readyToDecode() {
    return this.track.width !== 0 && 
           this.track.height !== 0 && 
           this.track.codec;
  }

  remux(trunk, applicationData = []) {
    // 1. 解析 NALU
    const isFirstFound = this.packetParser.parseNAL(trunk);
    if (!isFirstFound || !this.readyToDecode) return;

    // 2. 設定 codec
    this.streamProcessor.setCodec(this.codec, this.track.codec);

    // 3. 組裝封包
    const data = this.composePacket(trunk, applicationData);
    const packet = {
      data,
      applicationData,
      meta: {
        isKeyframe: !!trunk.isKeyframe,
        codec: this.codec,
        mediaCodec: this.track.codec,
        width: this.track.width,
        height: this.track.height,
        dts: trunk.dts,
      },
      type: 'video',
    };

    // 4. 儲存 GOP 並送出
    this.storeLastGOP(trunk, packet);
    this.queue.push(packet);
    
    while (this.queue.length) {
      this.streamProcessor.inputPacket(this.queue.shift());
    }
  }

  composePacket(trunk, applicationData) {
    // MSE 模式：加入寬高欄位
    // WebCodecs 模式：只有 frameInfo + bitstream
  }

  recover() {
    // 重送 last GOP 給新的 decoder
  }
}
```

---

## H.264 NALU 類型

```javascript
// packages/lib-rtsp-player/src/constants/NaluType.js

const NALU_TYPE = {
  NDR: 1,       // P-frame (Non-IDR)
  IDR: 5,       // I-frame (Keyframe)
  SEI: 6,       // Supplemental Enhancement Information
  SPS: 7,       // Sequence Parameter Set
  PPS: 8,       // Picture Parameter Set
  DELIMITER: 9, // Access Unit Delimiter
  EOSEQ: 10,    // End of Sequence
  EOSTR: 11,    // End of Stream
  STAP_A: 24,   // Single-Time Aggregation Packet
  FU_A: 28,     // Fragmentation Unit
};
```

---

## H.265/HEVC NALU 類型

```javascript
const NALU_TYPE_HEVC = {
  NDR: 1,       // P-frame
  IDR: 19,      // I-frame (Keyframe)
  VPS: 32,      // Video Parameter Set
  SPS: 33,      // Sequence Parameter Set
  PPS: 34,      // Picture Parameter Set
  DELIMITER: 35,
  EOSEQ: 36,
  EOSTR: 37,
  SEI: 39,
  FU_A: 49,     // Fragmentation Unit
};
```

---

## SPS 解析

### H.264 SPS

```javascript
// packages/lib-rtsp-player/src/utils/H264Tools.js

export function readSPS(data) {
  const decoder = new ExpGolombDecoder(data);

  // 1. 解析 Profile & Level
  const profileIdc = decoder.readUByte();
  const profileCompat = decoder.readUByte();
  const levelIdc = decoder.readUByte();

  // 2. 產生 codec 字串 (avc1.XXYYZZ)
  const codec = `avc1.${profileIdc.toString(16)}${profileCompat.toString(16)}${levelIdc.toString(16)}`;

  // 3. 處理 High Profile 特殊欄位
  if ([100, 110, 122, 244, ...].includes(profileIdc)) {
    // chroma, bit depth, scaling list 等
  }

  // 4. 解析尺寸
  const picWidthInMbsMinus1 = decoder.readUEG();
  const picHeightInMapUnitsMinus1 = decoder.readUEG();

  // 5. 處理裁切
  // frameCropLeftOffset, frameCropRightOffset, ...

  // 6. VUI 參數（長寬比）
  // sarScale = sarRatio[0] / sarRatio[1]

  // 7. 計算最終尺寸
  const width = ((picWidthInMbsMinus1 + 1) * 16 - crops) * sarScale;
  const height = (2 - frameMbsOnlyFlag) * (picHeightInMapUnitsMinus1 + 1) * 16 - crops;

  return { width, height, codec };
}
```

### Codec 字串格式

| Codec | 格式 | 範例 |
|-------|------|------|
| H.264 | `avc1.XXYYZZ` | `avc1.640028` (High Profile, Level 4.0) |
| H.265 | `hvc1.X.Y.LZ` | `hvc1.1.6.L93` |

---

## Remuxer 輸出格式

### MSE 模式封包結構

```
┌─────────────────────────────────────────────────────┐
│ extraLength (4 bytes, Big Endian)                   │
├─────────────────────────────────────────────────────┤
│ applicationData (metadata from RTP)                 │
├─────────────────────────────────────────────────────┤
│ widthField (6 bytes: ID=91, Len=4, Value)          │
├─────────────────────────────────────────────────────┤
│ heightField (6 bytes: ID=92, Len=4, Value)         │
├─────────────────────────────────────────────────────┤
│ frameInfo (SPS+PPS for IDR, null for non-IDR)      │
├─────────────────────────────────────────────────────┤
│ bitstream (NALU data)                               │
├─────────────────────────────────────────────────────┤
│ padding (if needed to match expected size)          │
└─────────────────────────────────────────────────────┘
```

### WebCodecs 模式封包結構

```
┌─────────────────────────────────────────────────────┐
│ frameInfo (SPS+PPS for IDR, null for non-IDR)      │
├─────────────────────────────────────────────────────┤
│ bitstream (NALU data)                               │
└─────────────────────────────────────────────────────┘
```

---

## H.264 Remuxer

```javascript
// packages/lib-rtsp-player/src/remuxer/H264Remuxer.js

class H264Remuxer extends AbstractRemuxer {
  constructor(options) {
    super(options);
    this.codec = 'H264';
    this.packetParser = new H264Parser(this.track);
  }

  // IDR frame 需要附加 SPS + PPS
  composeFrameInfo(trunk) {
    if (trunk.type !== NALU_TYPE.IDR) return null;
    
    const { sps, pps } = this.track;
    const frameInfo = new Uint8Array(sps.length + pps.length);
    frameInfo.set(sps, 0);
    frameInfo.set(pps, sps.length);
    return frameInfo;
  }

  // GOP 管理
  storeLastGOP(trunk, packet) {
    if (trunk.type === NALU_TYPE.IDR) {
      this.lastGopPackets = [packet];  // 新 GOP
    } else {
      this.lastGopPackets.push(packet);
    }
  }
}
```

---

## H.265 Remuxer

```javascript
// packages/lib-rtsp-player/src/remuxer/H265Remuxer.js

class H265Remuxer extends AbstractRemuxer {
  constructor(options) {
    super(options);
    this.codec = 'H265';
    this.packetParser = new H265Parser(this.track);
  }

  // IDR frame 需要附加 VPS + SPS + PPS
  composeFrameInfo(trunk) {
    if (!trunk.isKeyFrameOrSEI) return null;
    
    const { vps, sps, pps } = this.track;
    const frameInfo = new Uint8Array(vps.length + sps.length + pps.length);
    frameInfo.set(vps, 0);
    frameInfo.set(sps, vps.length);
    frameInfo.set(pps, vps.length + sps.length);
    return frameInfo;
  }
}
```

---

## H.264 Parser

```javascript
// packages/lib-rtsp-player/src/remuxer/H264Parser.js

class H264Parser {
  constructor(track) {
    this.firstFound = false;  // 是否已收到 keyframe
    this.track = track;       // 存放 width, height, codec, sps, pps
  }

  parseNAL(nal) {
    switch (nal.type) {
      case NALU_TYPE.SPS:
        // 解析 SPS，更新 track.width, height, codec
        const { width, height, codec, sps } = this.parseSPS(nal.getData());
        this.track = { ...this.track, width, height, codec, sps };
        return false;  // SPS 本身不需要送到 decoder

      case NALU_TYPE.PPS:
        this.track.pps = this.parsePPS(nal.getData());
        return false;

      case NALU_TYPE.IDR:
      case NALU_TYPE.NDR:
        if (nal.isKeyframe && !this.firstFound) {
          this.firstFound = true;
        }
        return this.firstFound;  // 只有收到 keyframe 後才開始解碼

      default:
        return nal.isReferencePicture;
    }
  }
}
```

---

## 資料流程圖

```
RTP Packet (from RTSPHandler)
     │
     ▼
┌─────────────┐
│  Remuxer    │ ← 根據 track type 分派
└─────────────┘
     │
     ▼
┌─────────────────┐
│ H264Remuxer     │
│ or H265Remuxer  │
└─────────────────┘
     │
     ├─── parseNAL() ───→ H264Parser / H265Parser
     │                         │
     │                         ▼
     │                    更新 track.sps/pps/width/height
     │
     ├─── composeFrameInfo() ───→ 組裝 SPS+PPS (IDR only)
     │
     ├─── composePacket() ───→ 組裝完整封包
     │
     ├─── storeLastGOP() ───→ 儲存 GOP 供恢復用
     │
     ▼
streamProcessor.inputPacket()
     │
     ▼
SSMUX / WebCodecs
```

---

## GOP 恢復機制

```javascript
// 當需要恢復播放時（例如 resolution change）
remuxer.recover() {
  this.isReady = false;
  this.streamProcessor.setCodec(this.codec);
  
  // 重送 last GOP
  while (this.lastGopPackets.length) {
    const packet = this.lastGopPackets.shift();
    this.streamProcessor.inputPacket(packet);
    this.isReady = this.lastGopPackets.length === 0;
  }
}
```

---

## 音訊 Remuxing

### G.711 / G.726

```javascript
// packages/lib-rtsp-player/src/remuxer/AudioRemuxer.js

class AudioRemuxer extends AbstractAudioRemuxer {
  remux(trunk, applicationData) {
    // 直接轉發 bitstream
    const bitstream = trunk.data;
    super.remux({ ...trunk, data: bitstream }, applicationData);
  }
}
```

### AAC

```javascript
// packages/lib-rtsp-player/src/remuxer/AACRemuxer.js

class AACRemuxer extends AbstractAudioRemuxer {
  remux(trunk, applicationData) {
    // 跳過前 4 bytes (AU header)
    const bitstream = trunk.data?.subarray(4);
    super.remux({ ...trunk, data: bitstream }, applicationData);
  }
}
```

---

## 常見問題

### 1. 畫面不顯示
- 檢查是否收到 SPS/PPS
- 確認 `firstFound` 狀態
- 檢查 `readyToDecode` 條件

### 2. 解析度錯誤
- 檢查 SPS 解析是否正確
- 確認裁切 (crop) 計算
- 檢查長寬比 (SAR) 設定

### 3. 播放卡頓
- 檢查 GOP 完整性
- 確認時間戳 (DTS) 連續性
- 檢查 queue 是否堆積

### 4. H.265 不播放
- 確認瀏覽器支援 HEVC
- 檢查 VPS 是否正確
- WebCodecs 通常有較好的 H.265 支援

---

## 相關模組

- `vsaas-rtp-packet` - RTP 封包解析與 NALU 重組
- `vsaas-mse-player` - MSE 播放器
- `vsaas-streaming-architecture` - 整體串流架構
