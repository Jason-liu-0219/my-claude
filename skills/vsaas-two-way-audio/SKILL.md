# VSaaS Portal - Two-Way Audio

雙向音訊模組，支援麥克風串流與設備對講功能。

## 模組概述

### 功能
- 麥克風擷取與串流
- PCM 轉 G.711 編碼
- RTSP Audio Back Channel
- WebRTC Data Channel 傳輸

---

## 檔案結構

```
packages/app-vsaas-portal/src/models/Audio/
├── TwoWayAudioService.js              # 雙向音訊服務
├── MicrophoneStreamer.js              # 麥克風串流
└── PCMUToG711AudioWorkletProcessor.js # Audio Worklet (PCM→G.711)

packages/app-vsaas-portal/src/components/Speakers/
├── SpeakerTalkDownController.vue      # 對講控制元件
├── SpeakerInstantAudioController.vue  # 即時音訊控制
└── composables/
    └── useTwoWayAudio.js              # Composable

packages/lib-rtsp-protocol/
└── AudioBackChannelRTP.js             # RTP 封包產生
```

---

## 架構圖

```
┌─────────────────────────────────────────────────────┐
│ Browser                                              │
│                                                      │
│  ┌───────────────┐    ┌──────────────────────┐     │
│  │ Microphone    │───→│ MicrophoneStreamer   │     │
│  │ (getUserMedia)│    │                      │     │
│  └───────────────┘    │  ┌─────────────────┐ │     │
│                       │  │ AudioWorklet    │ │     │
│                       │  │ (PCM → G.711)   │ │     │
│                       │  └────────┬────────┘ │     │
│                       └───────────┼──────────┘     │
│                                   │                 │
│                                   ▼                 │
│                       ┌──────────────────────┐     │
│                       │ TwoWayAudioService   │     │
│                       │                      │     │
│                       │  - RTSP Protocol     │     │
│                       │  - AudioBackChannel  │     │
│                       └───────────┬──────────┘     │
│                                   │                 │
└───────────────────────────────────┼─────────────────┘
                                    │
                                    ▼ (WebRTC Data Channel)
                              ┌───────────┐
                              │  Device   │
                              │  Speaker  │
                              └───────────┘
```

---

## MicrophoneStreamer

麥克風擷取與串流處理。

```javascript
// packages/app-vsaas-portal/src/models/Audio/MicrophoneStreamer.js

class MicrophoneStreamer {
  constructor(options = {}) {
    this.audioContext = null;
    this.microphoneSource = null;
    this.workletNode = null;
    this.stream = null;
    this.isStreaming = false;
    
    this.options = {
      sampleRate: 8000,      // G.711 標準
      channelCount: 1,       // 單聲道
      bufferSize: 1200,
      ...options,
    };

    this.onPacketReceived = options.onPacketReceived || (() => {});
    this.onError = options.onError || (() => {});
  }
}
```

### 初始化

```javascript
async initialize() {
  // 1. 建立 AudioContext
  this.audioContext = new AudioContext({
    sampleRate: this.options.sampleRate,  // 8000 Hz
  });

  // 2. 載入 Audio Worklet
  await this.audioContext.audioWorklet.addModule(workletUrl);

  // 3. 建立 Worklet Node
  this.workletNode = new AudioWorkletNode(
    this.audioContext, 
    'pcmu-to-g711-worklet-processor',
    {
      numberOfInputs: 1,
      numberOfOutputs: 0,
      channelCount: this.options.channelCount,
      channelCountMode: 'explicit',
    }
  );

  // 4. 監聽封包
  this.workletNode.port.onmessage = (event) => {
    if (event.data.type === 'receivePacket') {
      this.onPacketReceived(event.data);
    }
  };
}
```

### 開始串流

```javascript
async startStreaming() {
  if (this.isStreaming) return;

  // 1. 取得麥克風權限
  this.stream = await navigator.mediaDevices.getUserMedia({
    audio: {
      sampleRate: this.options.sampleRate,
      channelCount: this.options.channelCount,
      echoCancellation: true,
      noiseSuppression: true,
      autoGainControl: true,
    },
    video: false,
  });

  // 2. 確保 AudioContext 運行
  if (this.audioContext.state === 'suspended') {
    await this.audioContext.resume();
  }

  // 3. 連接麥克風到 Worklet
  this.microphoneSource = this.audioContext.createMediaStreamSource(this.stream);
  this.microphoneSource.connect(this.workletNode);

  this.isStreaming = true;
}
```

### 停止串流

```javascript
stopStreaming() {
  if (!this.isStreaming) return;

  // 1. 斷開麥克風
  if (this.microphoneSource) {
    this.microphoneSource.disconnect();
    this.microphoneSource = null;
  }

  // 2. 停止 media tracks
  if (this.stream) {
    this.stream.getTracks().forEach((track) => track.stop());
    this.stream = null;
  }

  // 3. 重置 worklet
  if (this.workletNode) {
    this.workletNode.port.postMessage({ type: 'reset' });
  }

  this.isStreaming = false;
}
```

---

## TwoWayAudioService

雙向音訊服務，整合 RTSP 與 WebRTC。

```javascript
// packages/app-vsaas-portal/src/models/Audio/TwoWayAudioService.js

class TwoWayAudioService extends VDSI {
  name = 'twoWayAudio';

  constructor(reference) {
    super(reference);
    this.connectionSSRC = null;

    this.updateState({
      status: TWO_WAY_AUDIO_STATUS.INITIAL,
      microphoneStreamer: null,
      audioBackChannel: null,
      rtspProtocol: null,
      audiobackConfig: {
        payloadType: 0,
        interleavedChannel: null,
      },
    });
  }
}
```

### 狀態機

```javascript
const TWO_WAY_AUDIO_STATUS = {
  INITIAL: 'INITIAL',                   // 初始狀態
  CONNECTING: 'CONNECTING',             // 連線中
  CONNECTED: 'CONNECTED',               // 已連線並串流
  DISCONNECTED: 'DISCONNECTED',         // 已斷線
  FAILED: 'FAILED',                     // 失敗
  MICROPHONE_STREAMING_ERROR: 'MICROPHONE_STREAMING_ERROR',
};
```

### 發起通話

```javascript
async call() {
  // 1. 檢查狀態
  if (!this.isDeviceOnline) {
    throw new Error('Device is not online');
  }

  if (!this.isWebRTCConnected) {
    throw new Error('Device webRTC connection is not connected');
  }

  if (!this.canCall) {
    return this.mediaStream;
  }

  // 2. 產生 SSRC
  this.connectionSSRC = Date.now() + Math.floor(Math.random() * 1000);

  // 3. 建立麥克風串流
  const microphoneStreamer = new MicrophoneStreamer({
    onPacketReceived: (audioData) => {
      // 產生 RTP 封包並發送
      const rtp = new AudioBackChannelRTP({
        payloadType: this.state.audiobackConfig.payloadType,
        sequence: audioData.sequenceNumber,
        timestamp: audioData.timestamp,
        ssrc: this.connectionSSRC,
        packet: audioData.data,
        interleavedChannel: this.state.audiobackConfig.interleavedChannel,
      });
      this.state.rtspProtocol.sendRtp(rtp.getPacket());
    },
    onError: (error) => {
      this.updateState({ status: TWO_WAY_AUDIO_STATUS.MICROPHONE_STREAMING_ERROR });
      throw error;
    },
  });

  // 4. 初始化並開始串流
  await microphoneStreamer.initialize();
  await microphoneStreamer.startStreaming();
  this.updateState({ microphoneStreamer, status: TWO_WAY_AUDIO_STATUS.CONNECTING });

  // 5. 建立 RTSP Audio Back Channel
  const audioBackChannel = await this.getAudioBackChannel();
  const rtspProtocol = new RtspProtocol(audioBackChannel);
  this.updateState({ audioBackChannel, rtspProtocol });

  // 6. RTSP 交握
  await rtspProtocol.sendOptions(this.rtspUrl);
  
  const desc = await rtspProtocol.sendDescribe(this.rtspUrl, { hasBackchannel: true });
  // 解析 audioback track 設定
  const audiobackTrack = desc.tracks.find(t => t.control === 'audioback');
  
  await rtspProtocol.sendSetup(`${this.rtspUrl}/audioback`, { hasBackchannel: true });
  await rtspProtocol.sendPlay(this.rtspUrl, { hasBackchannel: true });

  this.updateState({ status: TWO_WAY_AUDIO_STATUS.CONNECTED });
  return microphoneStreamer.stream;
}
```

### 掛斷

```javascript
async hangUp() {
  this.updateState({ status: TWO_WAY_AUDIO_STATUS.DISCONNECTED });
  await this.cleanup(true);  // 包含 RTSP TEARDOWN
}

async cleanup(includeRtspTeardown = false) {
  // 1. 停止麥克風
  if (this.state.microphoneStreamer) {
    this.stopMicrophoneStreamer();
  }

  // 2. 發送 RTSP TEARDOWN
  if (includeRtspTeardown && this.state.rtspProtocol) {
    await this.state.rtspProtocol.sendTeardown(this.rtspUrl);
  }

  // 3. 關閉 audio back channel
  if (this.state.audioBackChannel) {
    this.removeAudioBackChannel();
  }

  // 4. 重置狀態
  this.resetToInitialState();
}
```

---

## Audio Worklet

PCM 轉 G.711 μ-law 編碼。

```javascript
// PCMUToG711AudioWorkletProcessor.js

class PCMUToG711WorkletProcessor extends AudioWorkletProcessor {
  constructor() {
    super();
    this.sequenceNumber = 0;
    this.timestamp = 0;
    this.port.onmessage = this.handleMessage.bind(this);
  }

  process(inputs, outputs, parameters) {
    const input = inputs[0];
    if (input.length > 0) {
      const pcmData = input[0];  // Float32Array
      const g711Data = this.encodeG711(pcmData);
      
      this.port.postMessage({
        type: 'receivePacket',
        data: g711Data,
        sequenceNumber: this.sequenceNumber++,
        timestamp: this.timestamp,
      });
      
      this.timestamp += pcmData.length;
    }
    return true;
  }

  encodeG711(pcmData) {
    // PCM → G.711 μ-law 編碼
    const output = new Uint8Array(pcmData.length);
    for (let i = 0; i < pcmData.length; i++) {
      output[i] = this.linearToMulaw(pcmData[i] * 32768);
    }
    return output;
  }

  linearToMulaw(sample) {
    // μ-law 壓縮演算法
    // ...
  }
}

registerProcessor('pcmu-to-g711-worklet-processor', PCMUToG711WorkletProcessor);
```

---

## RTP 封包

```javascript
// AudioBackChannelRTP

class AudioBackChannelRTP {
  constructor({ payloadType, sequence, timestamp, ssrc, packet, interleavedChannel, marker }) {
    this.payloadType = payloadType;
    this.sequence = sequence;
    this.timestamp = timestamp;
    this.ssrc = ssrc;
    this.packet = packet;
    this.interleavedChannel = interleavedChannel;
    this.marker = marker;
  }

  getPacket() {
    // RTP Header (12 bytes) + Payload
    const header = new Uint8Array(12);
    
    // V=2, P=0, X=0, CC=0
    header[0] = 0x80;
    
    // M + PT
    header[1] = (this.marker << 7) | this.payloadType;
    
    // Sequence Number (16 bits)
    header[2] = (this.sequence >> 8) & 0xFF;
    header[3] = this.sequence & 0xFF;
    
    // Timestamp (32 bits)
    header[4] = (this.timestamp >> 24) & 0xFF;
    header[5] = (this.timestamp >> 16) & 0xFF;
    header[6] = (this.timestamp >> 8) & 0xFF;
    header[7] = this.timestamp & 0xFF;
    
    // SSRC (32 bits)
    header[8] = (this.ssrc >> 24) & 0xFF;
    header[9] = (this.ssrc >> 16) & 0xFF;
    header[10] = (this.ssrc >> 8) & 0xFF;
    header[11] = this.ssrc & 0xFF;

    // Combine header + payload
    return concat(header, this.packet);
  }
}
```

---

## RTSP Back Channel 交握

```
1. OPTIONS
   → RTSP OPTIONS request
   ← 200 OK

2. DESCRIBE
   → RTSP DESCRIBE (hasBackchannel: true)
   ← 200 OK with SDP
      m=audio 0 RTP/AVP 0
      a=control:audioback
      a=sendonly

3. SETUP
   → RTSP SETUP /audioback (Transport: RTP/AVP/TCP;interleaved=...)
   ← 200 OK with Transport

4. PLAY
   → RTSP PLAY (hasBackchannel: true)
   ← 200 OK

5. 開始發送 RTP 封包...

6. TEARDOWN
   → RTSP TEARDOWN
   ← 200 OK
```

---

## Vue Composable

```javascript
// packages/app-vsaas-portal/src/components/Speakers/composables/useTwoWayAudio.js

export function useTwoWayAudio(device) {
  const twoWayAudioService = computed(() => device.value?.services?.twoWayAudio);
  
  const isOnCall = computed(() => twoWayAudioService.value?.isOnCall);
  const canCall = computed(() => twoWayAudioService.value?.canCall);
  const status = computed(() => twoWayAudioService.value?.status);

  async function call() {
    if (!twoWayAudioService.value) return;
    await twoWayAudioService.value.call();
  }

  async function hangUp() {
    if (!twoWayAudioService.value) return;
    await twoWayAudioService.value.hangUp();
  }

  return {
    isOnCall,
    canCall,
    status,
    call,
    hangUp,
  };
}
```

---

## 常見問題

### 1. 麥克風權限被拒
- 檢查瀏覽器權限設定
- 確認 HTTPS 環境
- 處理 `NotAllowedError`

### 2. 音訊斷續
- 檢查網路連線品質
- 確認 AudioContext 狀態
- 調整 bufferSize

### 3. 無法連線
- 確認設備在線
- 檢查 WebRTC 連線狀態
- 確認 RTSP 交握完成

### 4. Echo 問題
- 確認 `echoCancellation: true`
- 調整喇叭音量
- 考慮硬體回音消除

---

## 相關模組

- `vsaas-webrtc-connection` - WebRTC 連線
- `vsaas-audio-player` - 音訊播放
- `vsaas-odyssey-connection` - 連線管理
