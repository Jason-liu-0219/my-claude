---
name: vsaas-device
description: Use when working with device management, device settings, adding devices, or camera/NVR configuration. Triggers on keywords like "device", "camera", "NVR", "DeviceManagement", "DeviceSettings", "AddDevice", "IoT", "Bridge".
version: 1.0.0
---

# VSaaS Portal - Device 模組

設備管理模組是 VSaaS Portal 的核心功能，負責攝影機、NVR、IoT 設備的完整生命週期管理。

## 模組概述

### 商業邏輯
- 新增/移除雲端連接的設備
- 查看設備列表和即時狀態
- 配置設備參數（影像、偵測、錄影）
- 管理設備韌體和維護

### 設備類型
| 類型 | 說明 |
|------|------|
| Camera | 攝影機 |
| NVR | 網路錄影機 |
| NVRChannel | NVR 頻道 |
| Bridge | 橋接設備 |
| IoT | 物聯網設備 |
| VSS | 視頻串流伺服器 |

---

## 路由結構

### 主路由
| 路徑 | 名稱 | 說明 |
|------|------|------|
| `/device` | Device | 根路由容器 |
| `/device/management` | DeviceManagement | 設備列表 |
| `/device/add-device/*` | AddDevice | 新增設備精靈 |
| `/device/:deviceId/*` | DeviceSettings | 設備設定 |

### 設備設定子路由（5 大類）

#### System 系統設定
```
/device/:deviceId/settings/
├── device-information    # 設備資訊
├── maintenance           # 維護
├── remote-support        # 遠端支援
├── nvr-settings          # NVR 設定
├── installation          # 安裝設定
└── poe-switch-settings   # PoE 交換器
```

#### Media 媒體設定
```
/device/:deviceId/media/
├── video-stream-settings # 影像串流
├── image-settings        # 影像設定
├── image-focus-adjustment # 對焦調整
├── privacy-mask          # 隱私遮罩
├── audio-settings        # 音訊設定
├── rtsp-access           # RTSP 存取
├── ptz-settings/         # PTZ 設定
│   ├── (preset-patrol)   # 預設點與巡邏
│   └── preference        # 偏好設定
└── timelapse             # 縮時攝影
```

#### Detection 偵測設定
```
/device/:deviceId/detection/
├── audio-detection       # 音訊偵測
├── tampering-detection   # 防篡改偵測
└── [VCA 動態路由]        # AI 偵測（微前端）
```

#### Device I/O 數位輸入輸出
```
/device/:deviceId/device-io/
├── digital-input         # 數位輸入
└── digital-output        # 數位輸出
```

#### Recording & Storage 錄影與儲存
```
/device/:deviceId/recording-and-storage/
├── recording-retention   # 錄影保留
└── cloud-backup          # 雲端備份
```

---

## 目錄結構

```
src/pages/device/
├── index.vue              # 路由容器
├── route.js               # 主路由定義
├── management/            # 設備列表
│   ├── index.vue
│   └── components/
│       ├── DeviceManagementTable.vue
│       ├── NVRDeviceManagementTable.vue
│       ├── VSSDeviceManagementTable.vue
│       └── BridgeManagementTable.vue
├── settings/              # 設備設定
│   ├── DeviceSettings.vue # 設定容器
│   ├── route.js           # 設定路由
│   ├── system/            # 系統設定頁面
│   ├── media/             # 媒體設定頁面
│   ├── detection/         # 偵測設定頁面
│   ├── DIDO/              # I/O 設定頁面
│   ├── RecordingAndStorage/ # 錄影儲存頁面
│   └── components/        # 共用組件
├── AddDevice/             # 新增設備
│   ├── index.vue
│   ├── route.js
│   ├── EnterDeviceId/
│   ├── AddDeviceSettings/
│   └── AddDeviceSuccess/
└── reskin/                # Reskin 版本
    └── management/
```

---

## Store 依賴

### 主要模組：device
**位置**：`src/store/device/`

```javascript
// State 重要屬性
state: {
  deviceMap: {},      // 設備 Map（deviceId -> device）
  nvrMap: {},         // NVR Map
  bridgeMap: {},      // Bridge Map
  isLoading: false,
  addDeviceFlow: {},  // 新增設備流程狀態
}
```

#### 關鍵 Actions
| Action | 說明 |
|--------|------|
| `fetchDevices` | 取得設備列表 |
| `fetchDeviceDetail` | 取得單一設備詳情 |
| `updateDevice` | 更新設備設定 |
| `deleteDevice` | 刪除設備 |
| `addDevice` | 新增設備 |
| `refreshDeviceStatus` | 刷新設備狀態 |

#### 關鍵 Getters
| Getter | 說明 |
|--------|------|
| `deviceList` | 設備陣列 |
| `getDeviceById` | 依 ID 取得設備 |
| `onlineDevices` | 在線設備 |
| `offlineDevices` | 離線設備 |

### 跨模組依賴
| 模組 | 用途 |
|------|------|
| `group` | 設備群組關聯 |
| `organization` | 組織設定 |
| `license` | 授權檢查 |
| `permission` | 權限控制 |
| `ui` | 載入狀態 |

---

## 使用的組件

### 來自 advanced-vue-components
- `DeviceSelector` - 設備選擇器
- `DeviceThumbnail` - 設備縮圖
- `DeviceIndicator` - 設備狀態指示
- `DisplayStreamingBlock` - 串流顯示
- `PlayerControl` - 播放器控制
- `Timeline` - 時間軸

### 來自 generic-vue-components
- `AppTable` - 表格
- `AppButton` - 按鈕
- `AppDialogue` - 對話框
- `AppDropdown` - 下拉選單

### 專案內組件
```
src/components/
├── Dialogues/
│   ├── DeviceSettingDialogue/
│   ├── AddDeviceDialogue/
│   └── DeleteDeviceDialogue/
└── Cards/
    └── DeviceCard/
```

---

## 關鍵程式碼路徑

### 進入點
- `src/pages/device/index.vue` - 路由容器
- `src/pages/device/route.js` - 路由定義
- `src/store/device/index.js` - Store 模組

### 資料流

#### 設備列表載入
```
1. 進入 /device/management
2. 觸發 device/fetchDevices action
3. API 呼叫 → 回傳設備資料
4. mutation 更新 deviceMap
5. 組件透過 getter 取得 deviceList 渲染
```

#### 設備設定更新
```
1. 使用者修改設定
2. 呼叫 device/updateDevice action
3. API 更新設備
4. mutation 更新 deviceMap 中的設備
5. UI 顯示成功訊息
```

### API 服務
- `src/models/API/DeviceAPI.js` - REST API
- `src/graphql/queries.gql` - GraphQL 查詢

---

## Reskin 雙版本

部分頁面有新舊兩版 UI：

```javascript
// route.js 中使用 reskinRouterHandler
{
  path: 'management',
  component: reskinRouterHandler(
    () => import('./management/index.vue'),      // 舊版
    () => import('./reskin/management/index.vue') // 新版
  )
}
```

判斷邏輯在 `src/utils/reskinRouterHandler.js`

---

## 微前端 AI 模組

VCA 偵測設定透過微前端動態載入：

```javascript
// src/router/dynamicVortexAIRouterLoader.js
// 動態新增 AI 相關路由到 detection 子路由
```

**載入時機**：訪問 `/device/:deviceId/detection/vca-detection`

---

## 常見開發任務

### 1. 新增設備設定子頁面
```bash
# 1. 在對應類別建立 Vue 組件
src/pages/device/settings/{category}/NewSetting.vue

# 2. 在 route.js 新增路由
src/pages/device/settings/route.js

# 3. 更新導航選單
src/utils/DeviceSettingsMenu.js
```

### 2. 修改設備列表功能
```bash
# 主要檔案
src/pages/device/management/index.vue
src/pages/device/management/components/DeviceManagementTable.vue

# Store 操作
src/store/device/actions.js
```

### 3. 除錯設備狀態
```javascript
// 在 Vue DevTools 中檢查
this.$store.state.device.deviceMap

// 或在組件中
import { useStore } from 'vuex'
const store = useStore()
console.log(store.state.device.deviceMap)
```

---

## 注意事項

1. **設備類型差異**：不同設備類型功能不同，需條件渲染
2. **即時更新**：設備狀態透過 WebSocket 即時更新
3. **權限控制**：所有路由有 `permission` meta
4. **Reskin 支援**：部分頁面有新舊版本
5. **微前端模組**：AI 功能動態載入

---

## 相關資源

- `references/routes.md` - 完整路由結構
- `references/store.md` - Store 詳解
- `references/components.md` - 組件使用
