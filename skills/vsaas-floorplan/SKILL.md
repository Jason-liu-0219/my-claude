---
name: vsaas-floorplan
description: Use when working with floor plan management, device placement, FOV editing, or spatial visualization. Triggers on keywords like "floorplan", "floor plan", "device placement", "FOV", "spatial", "map view".
version: 1.0.0
---

# VSaaS Portal - Floorplan 模組

樓層平面圖模組提供設備空間視覺化功能，支援在平面圖上放置設備、編輯 FOV（視野範圍）、以及實時視頻查看。

## 模組概述

### 商業邏輯
- 上傳樓層平面圖背景圖像
- 在平面圖上拖放放置設備
- 編輯設備 FOV（縮放、方向、深度）
- 實時視頻查看（點擊設備標記）
- 跨站點管理多個樓層平面圖

### 限制
- 每個站點最多 10 個樓層平面圖
- 支援的圖像格式：PNG、JPG、JPEG
- 設備只能放置在一個樓層平面圖上

---

## 路由結構

| 路徑 | 名稱 | 說明 |
|------|------|------|
| `/floorplans` | FloorplansDefault | 預設空狀態頁面 |
| `/sites/:siteId/floorplans/:floorplanId` | FloorplanView | 檢視模式（唯讀） |
| `/sites/:siteId/floorplans/:floorplanId/edit` | FloorplanEdit | 編輯模式 |

**路由特性**：
- 所有路由需要 `requiresAllData: 'device'` 確保設備數據已載入
- 使用動態導入支援代碼分割

---

## 目錄結構

```
src/pages/floorplan/
├── index.vue                    # 路由容器
├── route.js                     # 路由定義
├── default/
│   └── index.vue                # 預設頁面（空狀態）
├── view/
│   └── index.vue                # 檢視模式頁面
├── edit/
│   └── index.vue                # 編輯模式頁面
├── composables/
│   ├── useFloorplanNavigation.js    # 路由導航邏輯
│   └── useDevicePositions.js        # 設備位置處理
├── components/
│   ├── FloorPlanCanvas.vue          # 主畫布渲染器
│   ├── DevicePanel.vue              # 設備列表面板
│   ├── SiteFloorPlanList.vue        # 階層式網站/樓層列表
│   ├── FloorPlanFormDialog.vue      # 建立/編輯對話框
│   ├── FloorPlanLiveViewModal.vue   # 實時視頻模式
│   ├── CameraMarker.vue             # 相機標記
│   ├── FOVValueDisplay.vue          # FOV 值顯示
│   └── StatusLegend.vue             # 狀態圖例
├── classes/
│   ├── managers/
│   │   ├── CanvasManager.js         # RAF 批次渲染
│   │   └── InteractionManager.js    # 事件委派
│   ├── handlers/
│   │   ├── PanZoomHandler.js        # 平移縮放
│   │   ├── DragDropHandler.js       # 拖放處理
│   │   └── FOVInteractionHandler.js # FOV 編輯
│   ├── renderers/
│   │   ├── ImageRenderer.js         # 背景圖像
│   │   ├── DeviceRenderer.js        # 設備標記
│   │   └── FOVRenderer.js           # FOV 扇形
│   └── utilities/
│       ├── CoordinateTransform.js   # 坐標轉換
│       └── Viewport.js              # 視埠管理
└── utils/
    ├── coordinateUtils.js           # 坐標工具
    └── deviceIconResolver.js        # 設備圖標
```

---

## Store 依賴

### 主要模組：floorPlan
**位置**：`src/store/floorPlan/`

```javascript
state: {
  floorPlans: [],                    // 所有樓層平面圖
  currentFloorPlan: null,            // 當前選中樓層平面圖
  devicePositions: [],               // 設備位置列表
  devicePositionsBySerial: Map,      // 設備序號 → 位置查詢
  expandedSiteId: null,              // 展開的網站 ID
  isLoading: false,
  uploadProgress: 0,
  loadedSiteIds: Set,                // 懶加載緩存
}
```

#### 關鍵 Actions

| Action | 說明 |
|--------|------|
| `fetchAllFloorPlans` | 取得所有樓層平面圖 |
| `fetchFloorPlansInBatches` | 批次懶加載（初始 + 背景） |
| `createFloorPlan` | 建立新樓層平面圖 |
| `updateFloorPlan` | 更新樓層平面圖名稱 |
| `deleteFloorPlan` | 刪除樓層平面圖 |
| `uploadFloorPlanImage` | 上傳背景圖像 |
| `fetchDevicePositionsForFloorPlan` | 取得設備位置 |
| `createDevicePosition` | 新增設備到樓層平面圖 |
| `updateDevicePosition` | 更新設備位置/FOV |
| `deleteDevicePosition` | 移除設備 |

#### 關鍵 Getters

| Getter | 說明 |
|--------|------|
| `currentFloorPlan` | 當前樓層平面圖 |
| `floorPlansBySite` | 依站點取得樓層平面圖 |
| `devicePositions` | 當前樓層平面圖設備位置 |
| `placedDevices` | 已放置設備序號列表 |
| `unplacedDevices` | 未放置的可用設備 |
| `isFloorPlanLimitReached` | 檢查限制（10 個/站點） |

---

## 架構特點

### 混合式渲染（Canvas + DOM）
- **Canvas 層**：背景圖像、FOV 扇形、選擇突出顯示
- **DOM 層**：設備標記（CameraMarker Vue 組件）
- **Portal 層**：提示和模式（Teleport）

### OOP 類層次結構
```
CanvasManager（RAF 批次化）
    └── InteractionManager（事件委派）
        ├── PanZoomHandler（平移縮放）
        ├── DragDropHandler（拖放）
        ├── FOVInteractionHandler（FOV 編輯）
        └── DeviceInteractionHandler（設備選擇）
```

### 坐標轉換管道
```
標準化坐標 (0-1)
    ↕ × 圖像尺寸
影像空間坐標 (像素)
    ↕ × 縮放 + 平移
螢幕空間坐標 (視埠)
```

---

## 關鍵程式碼路徑

### 進入點
- `src/pages/floorplan/index.vue` - 路由容器
- `src/pages/floorplan/route.js` - 路由定義
- `src/store/floorPlan/index.js` - Store 模組

### 資料流

#### 檢視模式流程
```
用戶進入 FloorplanView
    ↓
加載樓層平面圖 (getFloorPlan)
    ↓
加載設備位置 (fetchDevicePositions)
    ↓
渲染 FloorPlanCanvas + 設備標記
    ↓
用戶點擊設備 → 開啟 FloorPlanLiveViewModal
```

#### 編輯模式流程
```
用戶進入 FloorplanEdit
    ↓
加載樓層平面圖和設備位置
    ↓
計算可用設備 (未放置的)
    ↓
渲染 FloorPlanCanvas + DevicePanel
    ↓
用戶拖放設備 → createDevicePosition
用戶拖動設備 → updateDevicePosition（延遲 300ms）
用戶編輯 FOV → updateDevicePosition（延遲 300ms）
```

---

## 使用的組件

### 來自 advanced-vue-components
- `DisplayStreamingBlock` - 串流顯示
- `PlayerControl` - 播放器控制

### 來自 generic-vue-components
- `AppButton` - 按鈕
- `AppDialogue` - 對話框
- `AppDropdown` - 下拉選單
- `AppFileUpload` - 檔案上傳

---

## 常見開發任務

### 1. 新增樓層平面圖操作
```bash
# 主要檔案
src/pages/floorplan/edit/index.vue
src/store/floorPlan/actions.js
```

### 2. 修改 FOV 編輯邏輯
```bash
# 主要檔案
src/pages/floorplan/classes/handlers/FOVInteractionHandler.js
src/pages/floorplan/classes/renderers/FOVRenderer.js
```

### 3. 除錯設備位置
```javascript
// 在 Vue DevTools 中檢查
this.$store.state.floorPlan.devicePositions

// 或使用 composable
import { useDevicePositions } from './composables/useDevicePositions'
const { filteredDevicePositions } = useDevicePositions()
```

---

## 效能最佳化

1. **RAF 批次化**：合併多個渲染調用到單一 requestAnimationFrame
2. **記憶化查詢**：O(1) 設備位置查詢
3. **懶加載**：初始加載 20 個站點，背景加載剩餘
4. **樂觀更新**：即時 UI 更新 + 延遲 API 呼叫

---

## 注意事項

1. **坐標系統**：標準化坐標 (0-1) 用於儲存，轉換為像素進行渲染
2. **WebRTC 整合**：實時視頻使用 `@vivotek/lib-webrtc-odyssey`
3. **設備刪除級聯**：系統刪除設備時自動移除位置
4. **圖像驗證**：客戶端驗證格式、大小、尺寸

---

## API 服務

### FloorPlanService
```javascript
listFloorPlans(siteId)
createFloorPlan(siteId, name)
getFloorPlan(floorPlanId)
updateFloorPlan(floorPlanId, name)
deleteFloorPlan(floorPlanId)
uploadFloorPlanImage(floorPlanId, file, onProgress)
```

### DevicePositionService
```javascript
listDevicePositions(floorPlanId)
createDevicePosition(floorPlanId, devicePosition)
updateDevicePosition(floorPlanId, devicePositionId, updates)
deleteDevicePosition(floorPlanId, devicePositionId)
```

---

## 相關資源

- 樓層平面圖常數：`@vivotek/const-vsaas` - `FLOORPLAN.*`
- 設備狀態常數：`FLOORPLAN.DEVICE_ONLINE_STATUS.*`
