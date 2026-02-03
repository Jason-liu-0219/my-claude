# VSaaS Portal - Player Components

Vue 播放器元件層，提供串流播放的 UI 封裝與整合。

## 模組概述

### 元件層級

```
PlayerWrapper (頁面級)
    │
    ├── ViewcellPageLayout (佈局)
    │
    └── DisplayStreamingBlock (串流區塊)
            │
            ├── DisplayStatusBar (狀態列)
            ├── ViewcellHud (HUD 顯示)
            ├── PluginlessWrapper (播放器封裝)
            └── FisheyeDewarpDropdown (魚眼校正)
```

---

## 檔案結構

```
packages/app-vsaas-portal/src/components/Player/
└── PlayerWrapper.vue              # 主播放器容器

packages/advanced-vue-components/components/
├── DisplayStreamingBlock.vue      # 串流顯示區塊
├── DisplayStatusBar.vue           # 狀態列
├── ViewcellHud.vue               # HUD 顯示
├── PluginlessWrapper.vue         # 無插件播放器封裝
├── FisheyeDewarpDropdown.vue     # 魚眼校正選單
├── ViewcellPageLayout.vue        # 頁面佈局
├── PlayerControl.vue             # 播放控制列
└── Timeline.vue                  # 時間軸
```

---

## PlayerWrapper

主播放器容器，管理多視窗佈局與時間軸。

```vue
<!-- packages/app-vsaas-portal/src/components/Player/PlayerWrapper.vue -->

<template>
  <div class="viewcell-layout">
    <!-- 視窗區域 -->
    <div class="viewcell-wrapper" :class="{ 'sync-mode': isSyncMode }">
      <ViewcellPageLayout
        :layout="layout"
        :isCellExpanded="isViewcellExpanded"
        :expandedIndex="activeViewIndex"
      >
        <template #cell="{ index }">
          <div
            :class="['viewcell', { active: activeViewIndex === index }]"
            @click.prevent="selectActiveView(index)"
            @dblclick="toggleViewcellExpand"
          >
            <DisplayStreamingBlock
              :device="activeGroupCurrentPageDevices[index]"
              :isActiveView="activeViewIndex === index"
              :isSyncMode="isSyncMode"
            />
          </div>
        </template>
      </ViewcellPageLayout>
    </div>

    <!-- 控制面板 -->
    <div class="control-panel-wrapper">
      <PlayerControl
        :activeDeviceView="activeDeviceView"
        :isSyncMode="isSyncMode"
        @updatePlayTime="updatePlayTime"
      />
      <Timeline
        :device="activeDeviceView"
        :isSyncMode="isSyncMode"
        @change="updatePlayTime"
      />
    </div>
  </div>
</template>
```

### Props

| Prop | Type | 說明 |
|------|------|------|
| `layout` | String | 佈局格式（如 '3x3', '1M+7'） |
| `activeViewIndex` | Number | 當前選中的視窗索引 |
| `activeDeviceView` | Object | 當前選中的設備 |
| `isSyncMode` | Boolean | 是否為同步模式 |
| `isFullscreen` | Boolean | 是否全螢幕 |

### 關鍵方法

```javascript
// 更新播放時間
updatePlayTime(localTime) {
  const { timestamp, tzoffs } = localTime.useClientTimezone();
  const { stream } = this.activeDeviceView.services;
  stream.resetPlayTime({ timestamp, tzoffs });
}

// 同步模式：重設所有設備播放時間
resetGroupDevicesPlayTime({ timestamp, tzoffs }) {
  this.activeGroupCurrentPageDevices.forEach((device) => {
    device.services?.stream?.resetPlayTime({ timestamp, tzoffs });
  });
}

// 切換視窗展開
toggleViewcellExpand() {
  this.activeDeviceView.services.ui.toggleViewcellExpand();
}
```

---

## DisplayStreamingBlock

串流顯示區塊，封裝播放器與狀態顯示。

```vue
<!-- packages/advanced-vue-components/components/DisplayStreamingBlock.vue -->

<template>
  <div class="streaming-block">
    <!-- 狀態列 -->
    <DisplayStatusBar
      v-if="shouldDisplayStatusBar"
      :device="device"
      :globalDisplayConfig="globalDisplayConfig"
    />

    <!-- 魚眼校正 -->
    <div v-if="shouldDisplayPlayerToolbar" class="top-toolbar">
      <FisheyeDewarpDropdown
        v-if="isFisheye"
        :modelValue="dewarpType"
        @update:modelValue="onChangeDewarpType"
      />
    </div>

    <!-- 播放器區域 -->
    <div class="camera-wrapper">
      <PluginlessWrapper
        v-if="!isEmpty && hasLivePermission"
        :mode="mode"
        :device="device"
        :streamingInterface="streamingInterface"
      />
      <ViewcellHud
        :device="device"
        :playbackIndex="playbackIndex"
      />
      <slot name="overlay" />
    </div>
  </div>
</template>
```

### Feature Options (Provide/Inject)

```javascript
provide() {
  return {
    featureOptions: {
      allowMute: false,
      useWorker: true,
      useWebCodecs: this.isWebCodecsSupported,
    },
  };
}
```

### Props

| Prop | Type | 說明 |
|------|------|------|
| `device` | Object | 設備物件 |
| `autoPlay` | Boolean | 自動播放 |
| `isActiveView` | Boolean | 是否為活動視窗 |
| `mode` | String | 模式（liveview/playback） |
| `needLiteMode` | Boolean | 精簡模式 |
| `enableUserIdle` | Boolean | 閒置檢測 |

---

## PluginlessWrapper

無插件播放器封裝層。

```javascript
// 初始化串流
async initStreaming() {
  const { device, mode, featureOptions } = this;
  
  // 使用 VsaasStreamingHandler 建立串流
  const streaming = await VsaasStreamingHandler.create({
    device,
    mode,
    ...featureOptions,
  });

  // 設定播放器
  this.streaming = streaming;
  this.videoElement = streaming.player;
}
```

---

## ViewcellPageLayout

多視窗佈局元件。

### 支援佈局

| 佈局 | 說明 |
|------|------|
| `1x1` | 單視窗 |
| `2x2` | 2x2 格 |
| `3x3` | 3x3 格 |
| `4x4` | 4x4 格 |
| `1M+7` | 1 主 + 7 副 |
| `1M+12` | 1 主 + 12 副 |

### 使用方式

```vue
<ViewcellPageLayout
  :layout="layout"
  :isCellExpanded="isViewcellExpanded"
  :expandedIndex="activeViewIndex"
>
  <template #cell="{ index }">
    <YourViewcellComponent :index="index" />
  </template>
</ViewcellPageLayout>
```

---

## PlayerControl

播放控制列。

### 功能

- 播放/暫停
- 快進/快退
- 下一幀
- 播放速度
- 同步模式切換
- 錄影/截圖

### Events

| Event | 說明 |
|-------|------|
| `updatePlayTime` | 更新播放時間 |
| `changeSyncMode` | 切換同步模式 |
| `movePointerTimeSeconds` | 移動時間指標 |

---

## Timeline

時間軸元件。

### 功能

- 錄影區間顯示
- 拖曳跳轉
- 縮放控制
- 事件標記

### Events

| Event | 說明 |
|-------|------|
| `change` | 時間變更 |
| `dragend` | 拖曳結束 |
| `seeking` | 尋找中 |
| `seekend` | 尋找結束 |

---

## 快捷鍵

```javascript
// PlayerWrapper.vue

keyDownHandler(event) {
  const parsedEvent = new KeyboardEventParser(event);

  // Ctrl/Cmd + → : 下一幀
  parsedEvent.matchesCodeHook({
    code: CODE_MAP.ARROW_RIGHT,
    modifiers: [parsedEvent.primaryModifier],
    callback: () => {
      this.activeDeviceView.services.stream.nextFrame();
    },
  });

  // ← : 後退 10 秒
  parsedEvent.matchesCodeHook({
    code: CODE_MAP.ARROW_LEFT,
    callback: () => {
      this.movePointerTimeSeconds(TIMELINE_CONTROL.BACKWARD_TEN_SECONDS);
    },
  });

  // → : 前進 10 秒
  parsedEvent.matchesCodeHook({
    code: CODE_MAP.ARROW_RIGHT,
    callback: () => {
      this.movePointerTimeSeconds(TIMELINE_CONTROL.FORWARD_TEN_SECONDS);
    },
  });
}
```

---

## 同步模式

多設備同步播放。

```javascript
// 切換同步模式
changeSyncMode({ timestamp, tzoffs, isSyncMode }) {
  this.setSyncMode(isSyncMode);

  if (isSyncMode) {
    this.resetGroupDevicesPlayTime({ timestamp, tzoffs });
  }
}

// 同步所有設備
resetGroupDevicesPlayTime({ timestamp, tzoffs }) {
  this.activeGroupCurrentPageDevices.forEach((device) => {
    device.services?.stream?.resetPlayTime({ timestamp, tzoffs });
  });
}
```

視覺提示：
```css
.viewcell-wrapper.sync-mode {
  border: solid 3px var(--color-primary);
}
```

---

## 權限控制

```javascript
computed: {
  hasLivePermission() {
    return this.canDoDevice({
      deviceId: this.device.deviceId,
      scope: PERMISSION_DEVICE_SCOPE.DEVICE_LIVE,
    });
  },

  hasPlaybackPermission() {
    return this.canDoDevice({
      deviceId: this.activeDeviceView.deviceId,
      scope: PERMISSION_DEVICE_SCOPE.DEVICE_PLAYBACK,
    });
  },
}
```

---

## 魚眼校正

```vue
<FisheyeDewarpDropdown
  v-if="isFisheye"
  :modelValue="dewarpType"
  @update:modelValue="onChangeDewarpType"
/>
```

Dewarp 類型：
- 原始（Original）
- 單視角（Single View）
- 雙視角（Dual View）
- 四視角（Quad View）
- 全景（Panorama）

---

## 遠端配置

```javascript
import RemoteConfigService, { PORTAL_FEATURE_CONFIG_KEY } from '@vivotek/lib-remote-config';

// 功能開關
const isNextFrameEnabled = RemoteConfigService.getConfiguration(
  PORTAL_FEATURE_CONFIG_KEY.FRAME_BY_FRAME
);

const isTalkDownEnabled = RemoteConfigService.getConfiguration(
  PORTAL_FEATURE_CONFIG_KEY.TALK_DOWN
);

const twoWayAudioEnabled = RemoteConfigService.getConfiguration(
  PORTAL_FEATURE_CONFIG_KEY.TWO_WAY_AUDIO
);
```

---

## 常見問題

### 1. 視窗不顯示
- 檢查 `device` prop 是否有效
- 確認 `hasLivePermission` 狀態
- 檢查 `streaming` service 是否初始化

### 2. 同步模式不同步
- 確認所有設備都支援 playback
- 檢查時間戳格式
- 確認 `resetPlayTime` 有呼叫

### 3. 快捷鍵無效
- 確認 `canHandleHotkey` 條件
- 檢查焦點是否在播放器區域
- 確認沒有其他元素攔截事件

---

## 相關模組

- `vsaas-streaming-architecture` - 串流架構
- `vsaas-playback` - 回放控制
- `vsaas-view` - 視圖管理
