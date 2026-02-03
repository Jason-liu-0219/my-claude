# VSaaS Portal - Audio Player

音訊播放模組，支援 G.711 PCM 和 AAC 格式的音訊串流播放。

## 模組概述

### 支援格式
| 格式 | 編碼 | 播放器 | 說明 |
|------|------|--------|------|
| G.711 | PCM (μ-law/A-law) | AudioPlayer | WebAudio API |
| G.726 | ADPCM | AudioPlayer | WebAudio API |
| AAC | MPEG-4 AAC | AACPlayer | MSE + SourceBuffer |

---

## 檔案結構

```
packages/lib-rtsp-player/src/
├── remuxer/
│   ├── AbstractAudioRemuxer.js  # 音訊 remuxer 基類
│   ├── AudioRemuxer.js          # G.711/G.726 remuxer
│   └── AACRemuxer.js            # AAC remuxer
├── player/
│   ├── AudioPlayer.js           # PCM 播放器 (WebAudio)
│   ├── AACPlayer.js             # AAC 播放器 (MSE)
│   └── WebCodecsAudioPlayer.js  # WebCodecs 音訊播放器
├── media/
│   └── WebCodecsAudioController.js
└── utils/
    └── AudioTools.js            # 音訊工具函式
```

---

## 音訊 Remuxing

### AbstractAudioRemuxer

```javascript
// packages/lib-rtsp-player/src/remuxer/AbstractAudioRemuxer.js

class AbstractAudioRemuxer {
  constructor({ streamProcessor, track }) {
    this.streamProcessor = streamProcessor;
    const { rtpmap, payloadType } = track;
    const entry = rtpmap[payloadType];
    
    this.track = {
      type: 'audio',
      codec: ENCODING_NAME_TO_CANONICAL_CODEC_MAP[entry.encodingName],
      clockRate: entry.clock || entry.rate,
      numberOfChannels: entry.encoding || entry.encodingParameters,
    };
  }

  remux(trunk, applicationData) {
    const bitstream = trunk.data || new Uint8Array();
    
    // 組裝封包格式
    // MSE: [4 bytes appData length] + [appData] + [bitstream]
    // WebCodecs: [bitstream only]
    
    const packet = {
      data: combined,
      applicationData,
      meta: {
        codec: this.track.codec,
        clockRate: this.track.clockRate,
        dts: trunk.dts,
      },
      type: 'audio',
    };

    this.streamProcessor.inputPacket(packet);
  }
}
```

### G.711 Remuxer

```javascript
// packages/lib-rtsp-player/src/remuxer/AudioRemuxer.js

class AudioRemuxer extends AbstractAudioRemuxer {
  remux(trunk, applicationData = []) {
    // 直接轉發 bitstream
    const bitstream = trunk.data || new Uint8Array();
    super.remux({ ...trunk, data: bitstream }, applicationData);
  }
}
```

### AAC Remuxer

```javascript
// packages/lib-rtsp-player/src/remuxer/AACRemuxer.js

class AACRemuxer extends AbstractAudioRemuxer {
  remux(trunk, applicationData = []) {
    // 跳過前 4 bytes (AU header)
    const bitstream = trunk.data?.subarray(4) || new Uint8Array();
    super.remux({ ...trunk, data: bitstream }, applicationData);
  }
}
```

---

## AudioPlayer (G.711/G.726)

使用 Web Audio API 播放 PCM 資料。

```javascript
// packages/lib-rtsp-player/src/player/AudioPlayer.js

class AudioPlayer extends EventEmitter {
  constructor({ snapshotMode, muted, volume }) {
    this.snapshotMode = snapshotMode;
    this.currentMuted = muted;
    this.currentVolume = volume / 100;
    
    this.audioContext = null;
    this.gainNode = null;
    this.audioQueue = [];
    this.activeSources = new Set();
    this.lastAudioEnd = 0;
    
    this.createAudio();
  }
}
```

### 初始化 AudioContext

```javascript
createAudio() {
  const AudioContext = window.AudioContext || window.webkitAudioContext;
  
  this.audioContext = new AudioContext();
  this.gainNode = audioContext.createGain();
  this.gainNode.connect(audioContext.destination);
  this.gainNode.gain.value = this.muted ? 0 : this.volume;
}
```

### 同步音訊

```javascript
syncAudio({ videoCurrentTime, videoTimeStamp }) {
  const { timeShift, audioQueue, MERGE_PACKET_TIME_RANGE } = this;
  
  // 合併多個封包
  while (audioQueue[0].info.timestamp.stream <= videoTimeStamp + MERGE_PACKET_TIME_RANGE) {
    if (audioQueue.length < COMBINE_PACKET_NUMBER) break;
    const packetPair = audioQueue.shift();
    packetCombine = combineBuffer(packetCombine, packetPair.packet.buffer);
  }

  const packetStartTime = (packetInfo.timestamp.audio / 1000) - timeShift;
  this.playAudioPacket([new Float32Array(packetCombine), packetStartTime, sampleRate]);
}
```

### 播放封包

```javascript
playAudioPacket([packet, startTime, sampleRate]) {
  const { audioContext, gainNode } = this;

  // 時間校正
  let correctedStartTime = startTime;
  const gap = startTime - this.lastAudioEnd;
  
  if (gap < -TOLERABLE_OVERLAP) {
    correctedStartTime = Math.max(startTime, now - MAX_ADVANCE_SECONDS);
  } else if (gap > TOLERABLE_OVERLAP) {
    correctedStartTime = Math.min(startTime, now + MAX_ADVANCE_SECONDS);
  }

  // 建立 AudioBufferSourceNode
  const source = audioContext.createBufferSource();
  const audioBuffer = audioContext.createBuffer(1, packet.length, sampleRate);
  
  // 填入 PCM 資料並套用 fade window
  for (let sample = 0; sample < packet.length; sample++) {
    audioBuffer.getChannelData(0)[sample] = packet[sample] * win[sample];
  }

  source.buffer = audioBuffer;
  source.connect(gainNode);
  source.start(correctedStartTime);
  
  this.activeSources.add(source);
  source.onended = () => this.activeSources.delete(source);
  
  this.lastAudioEnd = correctedStartTime + duration;
}
```

### 音量控制

```javascript
set volume(value) {
  this.currentVolume = value / 100;
  if (this.gainNode) {
    this.gainNode.gain.value = this.currentVolume;
  }
}

set muted(value) {
  this.currentMuted = value;
  if (this.gainNode) {
    this.gainNode.gain.value = this.muted ? 0 : this.volume;
  }
}
```

---

## AACPlayer (AAC over MSE)

使用 MediaSource Extensions 播放 AAC 音訊。

```javascript
// packages/lib-rtsp-player/src/player/AACPlayer.js

class AACPlayer extends EventEmitter {
  constructor({ snapshotMode, muted, volume }) {
    this.snapshotMode = snapshotMode;
    this.currentMuted = muted;
    this.currentVolume = volume / 100;
    
    this.queue = [];
    this.mediaSource = null;
    this.sourceBuffer = null;
    this.player = null;  // <audio> element
    
    this.createAudio();
    this.createMediaSource();
  }
}
```

### 建立 Audio 元素

```javascript
createAudio() {
  const player = document.createElement('audio');
  player.muted = this.muted;
  player.autoplay = false;
  
  if (isManagedMediaSourceSupported) {
    player.disableRemotePlayback = true;
  }

  ['play', 'timeupdate', 'pause', 'loadeddata', 'error'].forEach((evt) => {
    player.addEventListener(evt, this);
  });
  
  this.player = player;
}
```

### 建立 MediaSource

```javascript
createMediaSource() {
  let mediaSource = null;
  
  if (isManagedMediaSourceSupported) {
    mediaSource = new window.ManagedMediaSource();
  } else if (window.MediaSource) {
    mediaSource = new MediaSource();
  }
  
  mediaSource.addEventListener('sourceopen', this);
  this.player.src = URL.createObjectURL(mediaSource);
  this.mediaSource = mediaSource;
}

onMediaSourceSourceopen() {
  const { mediaSource } = this;
  mediaSource.duration = Infinity;

  // AAC MIME Type
  const sourceBuffer = mediaSource.addSourceBuffer('audio/mp4; codecs="mp4a.40.2"');
  sourceBuffer.addEventListener('updateend', this);
  this.initMedia(mediaSource, sourceBuffer);
}
```

### 添加封包

```javascript
appendPacket({ packet, info }) {
  const { canAppendBuffer, sourceBuffer, queue } = this;

  this.notifyHandler.appendNotify(info);

  if (queue.length === 0 && canAppendBuffer) {
    sourceBuffer.appendBuffer(packet);
  }
  queue.push({ packet, info });
}

onSourceBufferUpdateend() {
  const { sourceBuffer, canAppendBuffer, queue } = this;
  queue.shift();

  if (queue.length && canAppendBuffer) {
    sourceBuffer.appendBuffer(queue[0].packet);
  }
}
```

### 同步音訊

```javascript
syncAudio({ videoTimeStamp }) {
  if (this.snapshotMode || this.paused) return;

  const gap = videoTimeStamp - this.referenceNotify.timestamp.stream;
  const SYNC_GAP_THRESHOLD = 100; // ms

  // 根據差距調整播放速率
  if (Math.abs(gap) > SYNC_GAP_THRESHOLD) {
    this.playbackRate = (gap > 0) ? 1.5 : 0.5;
  } else {
    this.playbackRate = 1;
  }
}
```

---

## 音訊同步機制

### Video-Audio 時間同步

```javascript
// StreamProcessor.onVideoTimeUpdate

onVideoTimeUpdate() {
  const { audioPlayer, streamPlayer, currentTime, audioTimeShift } = this;

  // 計算時間偏移
  if (audioPlayer.audioContext) {
    const mediaOffset = currentTime - audioPlayer.audioContext.currentTime;
    audioPlayer.timeShift = (audioTimeShift || 0) / 1000 + mediaOffset;
  }

  // 同步音訊
  const referenceTime = {
    videoCurrentTime: currentTime,
    videoTimeStamp: streamPlayer.getReferenceNotify()?.timestamp.stream || 0
  };
  audioPlayer.syncAudio(referenceTime);
}
```

### TimeShift 計算

```javascript
// audioTimeShift = 影片開始時間 - 音訊開始時間
get audioTimeShift() {
  const videoReady = this.streamPlayer?.startPlayTime;
  const audioReady = this.audioPlayer?.startPlayTime;
  return (videoReady && audioReady)
    ? this.streamPlayer.startPlayTime - this.audioPlayer.startPlayTime
    : undefined;
}
```

---

## 音訊工具

### Fade In/Out Window

```javascript
// packages/lib-rtsp-player/src/utils/AudioTools.js

export function createFadeInOutWindow(length, fadeInSamples, fadeOutSamples, amplitude) {
  const window = new Float32Array(length);
  
  for (let i = 0; i < length; i++) {
    if (i < fadeInSamples) {
      // Fade in
      window[i] = amplitude * (i / fadeInSamples);
    } else if (i > length - fadeOutSamples) {
      // Fade out
      window[i] = amplitude * ((length - i) / fadeOutSamples);
    } else {
      window[i] = amplitude;
    }
  }
  
  return window;
}

export function getAudioWindow(length, amplitude) {
  return new Float32Array(length).fill(amplitude);
}
```

---

## 播放器生命週期

### AudioPlayer

```javascript
play(time = 0) {
  return this.resume()
    .then(() => {
      this.emit(PLAYER_EVENT_NAMES.PLAY, this);
    });
}

pause() {
  this.stopAllActiveSources();
  return this.audioContext.suspend();
}

resume() {
  if (this.audioContext?.state === 'suspended') {
    return this.audioContext.resume();
  }
  return Promise.resolve();
}

cutover() {
  this.audioQueue = [];
  this.stopAllActiveSources();
  this.removeAudio();
  this.createAudio();
  this.lastAudioEnd = 0;
}

destroy() {
  this.removeAudio();
  this.startPlayTime = 0;
}
```

### AACPlayer

```javascript
play(time = 0) {
  this.playPromise = this.player.play()
    .then(() => {
      this.currentTime = time;
    });
  return this.playPromise;
}

pause() {
  const playPromise = this.playPromise || Promise.resolve();
  playPromise.then(() => this.player?.pause());
}

cutover() {
  this.notifyHandler.clear();
  this.player.currentTime += this.bufferedLength;
}

destroy() {
  this.removeAudio();
  this.notifyHandler.destroy();
  this.queue = [];
}
```

---

## Safari 相容性

```javascript
// Safari 需要固定的 sample rate
if (isSafari) {
  audioBuffer = audioContext.createBuffer(1, packet.length, 44100);
  source.playbackRate.value = 8000 / 44100;  // 調整播放速率
} else {
  audioBuffer = audioContext.createBuffer(1, packet.length, sampleRate);
}
```

---

## 常見問題

### 1. 音訊不播放
- 確認 `AudioContext.state` 是否為 `running`
- 檢查 `muted` 和 `volume` 設定
- 確認用戶已與頁面互動（瀏覽器自動播放限制）

### 2. 音訊延遲/不同步
- 檢查 `timeShift` 計算是否正確
- 確認 `syncAudio` 有被呼叫
- 調整 `SYNC_GAP_THRESHOLD`

### 3. 音訊斷續
- 檢查封包佇列長度
- 確認 `combineBuffer` 邏輯
- 調整 `MERGE_PACKET_TIME_RANGE`

### 4. 記憶體洩漏
- 確保 `activeSources` 有清理
- 確認 `AudioContext` 有 `close()`
- 確認 `URL.revokeObjectURL()` 有呼叫

---

## 相關模組

- `vsaas-two-way-audio` - 雙向音訊
- `vsaas-mse-player` - MSE 播放器
- `vsaas-streaming-architecture` - 串流架構
