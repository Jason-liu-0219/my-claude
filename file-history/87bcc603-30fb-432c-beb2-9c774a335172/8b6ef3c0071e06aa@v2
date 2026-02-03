---
name: vsaas-view
description: Use when working with customized views, view layouts, live streaming, view rotation, or multi-device display. Triggers on keywords like "view", "customized view", "live view", "streaming layout", "view rotation", "sync mode".
version: 1.0.0
---

# VSaaS Portal - View 模組

自訂檢視模組提供用戶自定義的設備視圖管理，支援多種佈局、頁面輪播、同步播放等功能。

## 模組概述

### 商業邏輯
- 建立和管理自訂視圖
- 拖放配置設備位置
- 支援多種網格佈局（1x1、2x2、3x3 等）
- 頁面自動輪播功能
- 同步播放（多設備同時播放）
- 視圖分享和複製

### 核心概念
| 概念 | 說明 |
|------|------|
| CustomizedView | 用戶自訂視圖物件 |
| Cell | 視圖中的設備槽位 |
| Layout | 網格佈局配置 |
| Page | 根據佈局分組的頁面 |
| Sync Mode | 同步播放模式 |

---

## 路由結構

| 路徑 | 名稱 | 說明 |
|------|------|------|
| `/view/` | ListView | 視圖列表和直播檢視 |
| `/view/create` | CreateView | 建立新視圖 |
| `/view/edit/:id` | EditView | 編輯現有視圖 |

---

## 目錄結構

```
src/pages/view/
├── index.vue                        # 根元件（loading wrapper）
├── route.js                         # 路由定義
├── list/
│   ├── index.vue                    # 直播檢視主頁面
│   └── components/
│       ├── CustomizedViewList.vue   # 側邊欄視圖列表
│       ├── ViewThumbnail.vue        # 視圖縮圖預覽
│       ├── EmptyView.vue            # 空狀態元件
│       ├── ListViewPlayersWrapper/
│       │   ├── ListViewPlayersWrapper.vue  # 播放器容器
│       │   └── composables/
│       │       ├── useViewControl.js       # 視圖控制
│       │       ├── usePlaybackControl.js   # 播放回放
│       │       └── useDeviceDisplay.js     # 設備顯示
│       └── Footer/
│           ├── ViewCentralPanel.vue        # 頁腳中央面板
│           ├── ViewControlsModel.js        # 控制模型
│           └── CountdownTimer.js           # 倒計時計時器
├── edit/
│   ├── index.vue                    # 編輯頁面
│   └── components/
│       ├── EditViewPlayersWrapper.vue      # 編輯播放器
│       ├── DeviceSiteList.vue              # 設備/站點列表
│       ├── DeviceThumbnailList.vue         # 設備縮圖列表
│       ├── ViewLayoutThumbnail.vue         # 佈局預覽
│       ├── ViewPagesPagination.vue         # 頁面管理
│       └── LeaveEditViewDialogue.vue       # 離開確認
└── components/
    └── DeleteViewDialogue.vue       # 刪除視圖對話框

src/store/view/
├── index.js                         # Store 定義
├── actions.js                       # 非同步操作
├── getters.js                       # 派生狀態
└── mutations.js                     # 狀態變更
```

---

## Store 依賴

### 主要模組：view
**位置**：`src/store/view/`

```javascript
state: {
  isFetchingData: false,         // 正在獲取數據
  viewsList: [],                 // 所有用戶視圖列表
  currentViewId: '',             // 當前選中視圖 ID
  createView: null,              // 新建視圖臨時物件
  isSyncMode: false,             // 同步播放模式
  copyViewId: null               // 待複製視圖 ID
}
```

#### 關鍵 Actions

| Action | 說明 |
|--------|------|
| `fetchViews` | 從 API 獲取用戶視圖 |
| `updateViewsSort` | 更新視圖排序（防抖 500ms） |
| `setCurrentViewId` | 設置當前視圖 ID |
| `changeLayout` | 更改視圖佈局 |
| `goPreviousPage` | 上一頁 |
| `goNextPage` | 下一頁 |
| `goNextNonEmptyPage` | 跳到非空頁面 |
| `setEditView` | 設置編輯視圖 |
| `setCreateView` | 初始化新視圖 |
| `addView` | 建立並添加視圖 |
| `removeView` | 刪除視圖 |
| `updateView` | 保存視圖更改 |
| `copyView` | 複製視圖給其他用戶 |

#### 關鍵 Getters

| Getter | 說明 |
|--------|------|
| `hasView` | 是否有任何視圖 |
| `viewsList` | 所有視圖列表 |
| `currentView` | 當前活動視圖 |
| `currentViewPageDevices` | 當前頁面設備列表 |
| `activeDevice` | 當前活動設備 |
| `defaultView` | 預設視圖 |
| `isSyncMode` | 同步播放模式狀態 |

---

## CustomizedView 模型

**位置**：`src/models/CustomizedView/CustomizedView.js`

### 核心屬性
```javascript
{
  id: string,                    // 視圖唯一標識符
  name: string,                  // 視圖名稱
  layout: string,                // 佈局類型（'3x3'）
  cells: Array,                  // 分配的設備列表
  shareFrom: string,             // 分享者郵件
  activeDeviceId: string,        // 當前活動設備 ID
  currentPageIndex: number,      // 當前頁面索引
  pages: Array,                  // 計算的頁面結構
  isNewSharedView: boolean,      // 是否為新共享視圖
  status: string,                // 'ACTIVE' 或 'INACTIVE'
  isCellEmpty: boolean,          // 視圖是否為空
  isEdited: boolean              // 是否有未保存編輯
}
```

### 主要方法
```javascript
initialize()                     // 初始化內部狀態
changeLayout(layout)             // 更改佈局
goPreviousPage() / goNextPage()  // 頁面導航
getCurrentPageDevices()          // 取得當前頁面設備
create() / update() / delete()   // CRUD 操作
```

---

## 關鍵程式碼路徑

### 進入點
- `src/pages/view/index.vue` - 根元件
- `src/pages/view/route.js` - 路由定義
- `src/store/view/index.js` - Store 模組
- `src/models/CustomizedView/CustomizedView.js` - 視圖模型

### 資料流

#### 視圖初始化
```
應用啟動
    ↓
fetchViews() [Action]
    ↓
API 獲取視圖 → CustomizedView 包裝
    ↓
setViewsList [Mutation]
    ↓
initializeCurrentView()
```

#### 設備串流流程
```
選擇視圖 / 頁面變更
    ↓
currentView [Getter]
    ↓
currentViewPageDevices [Getter]
    ↓
DisplayStreamingBlock 接收設備
    ↓
建立 WebSocket/WebRTC 連線
```

#### 視圖保存流程
```
編輯視圖
    ↓
點擊保存
    ↓
新視圖: addView() → create() API
現有視圖: updateView() → update() API
```

---

## 主要元件

### 1. ListView (list/index.vue)
**功能**：直播檢視主頁面

**特性**：
- 三面板佈局（側邊欄、中央播放器、控制面板）
- 頁面輪播
- 同步播放
- 全屏模式
- 時間線回放

**主要方法**：
```javascript
startRotate()       // 啟動輪播
cancelRotate()      // 停止輪播
preConnectDevices() // 預連接設備
onLayoutChange()    // 佈局變更
toggleCollapse()    // 折疊時間線
```

### 2. EditView (edit/index.vue)
**功能**：視圖編輯頁面

**特性**：
- 視圖名稱編輯（最大 128 字符）
- 佈局切換
- 拖放設備配置
- 交換模式（點擊兩個單元格交換）
- 變更追踪

**路由守衛**：
- `beforeRouteEnter`：驗證視圖 ID
- `beforeRouteLeave`：提示未保存更改

### 3. CustomizedViewList.vue
**功能**：視圖列表側邊欄

**特性**：
- 視圖搜尋
- 拖放排序（防抖更新）
- 視圖操作（編輯、複製、刪除）
- 虛擬滾動
- 共享視圖標籤

### 4. ListViewPlayersWrapper.vue
**功能**：播放器網格容器

**組合函式**：
- `useDeviceDisplay`：格式化設備顯示
- `usePlaybackControl`：同步播放邏輯
- `useViewControl`：視圖交互

### 5. ViewControlsModel.js
**功能**：頁腳控制模型

**特性**：
- 輪播管理（啟動/暫停/取消）
- 全屏控制
- 時間線管理
- 倒計時計時器

---

## 同步播放模式

### 啟用流程
```javascript
changeSyncMode({ timestamp, tzoffs, isSyncMode: true })
    ↓
resetViewDevicesPlayTime({ timestamp, tzoffs })
    ↓
遍歷所有設備 → device.services.stream.resetPlayTime()
    ↓
所有設備同時跳轉到相同時間點
```

---

## 輪播功能

### 啟動流程
```javascript
startRotate()
    ↓
設置 shouldPreserveStreaming = true
    ↓
標記當前頁面設備為活躍
    ↓
rotationTimer.start() (預設 15 秒)
    ↓
計時器完成 → goNextNonEmptyPage()
    ↓
preConnectDevices()（預連接下一頁）
```

---

## 使用的組件

### 來自 advanced-vue-components
- `SideMenusLayout` - 三欄佈局
- `DisplayStreamingBlock` - 串流顯示
- `ViewcellPageLayout` - 網格佈局
- `LayoutSwitchButton` - 佈局切換
- `PlayerControl` - 播放器控制
- `Timeline` - 時間軸

### 來自 generic-vue-components
- `AppButton` - 按鈕
- `AppDialogue` - 對話框
- `VirtualScrollerList` - 虛擬滾動

---

## 常見開發任務

### 1. 新增佈局類型
```bash
# 主要檔案
packages/const-vsaas/Layout.js
src/models/CustomizedView/CustomizedView.js
```

### 2. 修改視圖編輯功能
```bash
# 主要檔案
src/pages/view/edit/index.vue
src/pages/view/edit/components/
```

### 3. 除錯視圖狀態
```javascript
// 在 Vue DevTools 中檢查
this.$store.state.view.viewsList
this.$store.getters['view/currentView']

// 或在元件中
const view = this.$store.getters['view/currentView']
console.log(view.pages, view.cells)
```

---

## 權限控制

| 權限 | 用途 |
|------|------|
| `PERMISSION_DEVICE_SCOPE.DEVICE_LIVE` | 直播權限 |
| `PERMISSION_DEVICE_SCOPE.DEVICE_PLAYBACK` | 回放權限 |
| `PERMISSION_DEVICE_SCOPE.DEVICE_UNLOCK` | 遠程解鎖 |

---

## API 整合

```javascript
// 視圖 CRUD
getUserViews(params)
customizedView.create()
customizedView.update()
customizedView.delete()

// 排序和複製
updateUserViewsSort(params)
copyUserView({ userID, viewID, emails })
```

---

## 效能最佳化

1. **虛擬滾動**：優化大型設備和視圖列表
2. **防抖更新**：排序更改延遲 500ms
3. **預先連接**：輪播時預連接下一頁設備
4. **代碼分割**：路由使用動態導入
5. **EventEmitter**：ViewControlsModel 鬆散耦合

---

## 注意事項

1. **視圖狀態**：`ACTIVE`（已保存）vs `INACTIVE`（新建）
2. **變更追踪**：離開編輯前檢查未保存更改
3. **共享視圖**：標記分享者信息
4. **同步播放**：所有設備同時播放相同時間點
5. **佈局持久化**：視圖佈局保存在後端
