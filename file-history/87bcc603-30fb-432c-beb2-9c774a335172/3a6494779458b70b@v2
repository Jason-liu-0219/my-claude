---
name: vsaas-group
description: Use when working with group management, device grouping, live view layouts, or site organization. Triggers on keywords like "group", "site", "live view", "layout", "device grouping", "rotation".
version: 1.0.0
---

# VSaaS Portal - Group 模組

群組管理模組提供設備分組功能，支援實時視頻檢視、多種版面配置、頁面輪播等功能。

## 模組概述

### 商業邏輯
- 建立和管理設備群組（Site）
- 以網格佈局顯示實時視頻串流
- 支援多種版面配置（1x1、2x2、3x3 等）
- 頁面自動輪播功能
- 設備搜尋和篩選

### 核心概念
| 概念 | 說明 |
|------|------|
| Group/Site | 設備群組容器 |
| Layout | 版面配置（決定每頁顯示數量） |
| Page | 根據版面分組的設備頁面 |
| ViewCell | 版面中的單個視圖單元 |

---

## 路由結構

| 路徑 | 名稱 | 說明 |
|------|------|------|
| `/group` | Group | 群組直播檢視頁面 |

**路由特性**：
- 權限要求：`PERMISSION_KEYS.GROUP`
- 許可證檢查：`licenseForbiddenPage: true`
- 應用預設路由指向此頁面

---

## 目錄結構

```
src/pages/group/
├── index.vue                    # 主頁面元件
├── index.test.js                # 測試檔案
└── components/
    ├── SingleGroup.vue          # 單個群組摺疊元件
    ├── SingleGroup.test.js
    ├── GroupDeviceList.vue      # 群組與裝置列表
    └── GroupDeviceList.test.js

src/store/group/
├── index.js                     # Store 定義
├── actions.js                   # 非同步動作
├── mutations.js                 # 狀態變更
├── getters.js                   # 計算屬性
└── *.test.js                    # 測試檔案
```

---

## Store 依賴

### 主要模組：group
**位置**：`src/store/group/`

```javascript
state: {
  isFetchingData: false,         // 資料抓取中
  rawGroupsMap: {},              // 原始群組映射 {groupId: group}
  activePageIndex: 0,            // 當前頁面索引
  activeViewIndex: -1,           // 當前視圖單元索引
  activeGroupId: '',             // 當前選中群組 ID
  activeDeviceId: '',            // 當前選中裝置 ID
  editingGroup: [],              // 編輯中的群組
  layout: LAYOUT_DEFAULT,        // 當前版面配置
  isSyncMode: false              // 同步模式
}
```

#### 關鍵 Actions

| Action | 參數 | 說明 |
|--------|------|------|
| `fetchGroupList` | `{ queryDevices? }` | 從 API 取得群組清單 |
| `setActiveGroupId` | `groupId` | 設置活躍群組 |
| `setActiveDeviceId` | `deviceId` | 設置活躍裝置 |
| `updateGroupInformation` | `{ id, name, location }` | 更新群組資訊 |
| `batchMoveDeviceToGroup` | `{ devices[], deviceGroupID }` | 批量移動裝置 |
| `changeLayout` | `{ layout }` | 變更版面配置 |
| `goPreviousPage` | - | 上一頁 |
| `goNextPage` | - | 下一頁 |
| `goNextNonEmptyPage` | - | 跳到下一個非空頁面 |
| `createGroup` | `{ name, location }` | 建立群組 |
| `removeGroup` | `group` | 刪除群組 |

#### 關鍵 Getters

| Getter | 說明 |
|--------|------|
| `groupsList` | 所有群組陣列 |
| `groupsMap` | 群組映射（含設備） |
| `validGroups` | 名稱不為空的群組 |
| `sortedGroups` | 排序函數 |
| `activeGroup` | 當前活躍群組 |
| `activeGroupPages` | 當前群組頁面陣列 |
| `activePageLayoutInfo` | 當前頁面視圖資訊 |
| `activeGroupAvailableDevices` | 可用設備列表 |
| `pageCount` | 頁面總數 |
| `currentPageIndex` | 當前頁碼（1-based） |
| `currentLayout` | 當前版面配置 |
| `defaultGroup` | 預設群組 |

---

## 版面配置系統

### 可用版面
```javascript
LAYOUT_COUNT = {
  '1x1': 1,
  '2x2': 4,
  '2x3': 6,
  '3x3': 9,
  '4x4': 16,
  // ...
}
```

### 頁面計算邏輯
```javascript
getDevicePages(devices, layout) {
  1. 迭代所有設備
  2. 填充視圖單元直到達到版面容量
  3. 創建新頁面當視圖單元滿時
  4. 用空設備填充最後一頁剩餘單元
}
```

---

## 關鍵程式碼路徑

### 進入點
- `src/pages/group/index.vue` - 主頁面
- `src/store/group/index.js` - Store 模組

### 頁面結構
```
SideMenusLayout
├── 左菜單: GroupDeviceList（群組和設備樹）
├── 中央區域: PlayerWrapper（視頻播放器網格）
├── 右菜單: StreamingControlPanel（控制面板）
└── 頁腳: 版面配置、分頁、全螢幕控制
```

### 資料流

#### 初始化流程
```
App 啟動
    ↓
fetchGroupList() [Action]
    ↓
API: listSite()
    ↓
commit('setGroupsMap', sites)
    ↓
setActiveGroupId(defaultGroup.id)
```

#### 選擇群組流程
```
SingleGroup @clickToggleBlock
    ↓
setActiveGroupId(groupId)
    ↓
setLayout() + changeLayout() + resetAvailableDeviceUiSettings()
```

#### 選擇設備流程
```
selectDevice(deviceId)
    ↓
setActiveDeviceId(deviceId)
    ↓
1. 檢查群組所有權
2. 如需切換群組
3. 定位設備在版面配置中位置
4. 更新 activePageIndex 和 activeViewIndex
```

---

## 主要元件

### 1. Group/index.vue
**功能**：群組直播檢視主頁面

**主要計算屬性**：
```javascript
currentGroup        // 當前活躍群組
currentLayout       // 版面配置
pageCount           // 頁面總數
hasActiveDevice     // 是否有選中設備
```

**主要方法**：
```javascript
onLayoutChange()    // 版面配置變更
startRotate()       // 啟動輪播
stopRotate()        // 停止輪播
goPreviousPage()    // 上一頁
goNextPage()        // 下一頁
preConnectDevices() // 預連接下一頁設備
```

### 2. GroupDeviceList.vue
**功能**：階層式群組和設備列表

**特性**：
- 搜尋框（即時搜尋群組和設備）
- 虛擬滾動（`LazyloadList`）
- 群組摺疊（`SingleGroup`）

### 3. SingleGroup.vue
**功能**：可摺疊的群組容器

**事件**：
```javascript
@clickToggleBlock   // 摺疊狀態改變
@selectDevice       // 設備選擇
```

---

## 輪播功能

### 啟動流程
```javascript
startRotate()
    ↓
標記當前頁面設備為 isActiveInRotation = true
    ↓
計算下一頁索引
    ↓
preConnectDevices()（預先連接下一頁設備）
    ↓
計時器觸發 → goNextNonEmptyPage()
```

### 預連接優化
提前建立下一頁設備的 WebSocket 連接，避免切換頁面時延遲。

---

## 使用的組件

### 來自 advanced-vue-components
- `SideMenusLayout` - 三欄佈局
- `PlayerWrapper` - 播放器容器
- `StreamingControlPanel` - 控制面板
- `DisplayStreamingBlock` - 串流顯示

### 來自 generic-vue-components
- `AppUniqueAccordionToggleBlock` - 手風琴摺疊
- `AppDeviceList` - 設備列表
- `LazyloadList` - 虛擬滾動

---

## 常見開發任務

### 1. 修改群組列表功能
```bash
# 主要檔案
src/pages/group/components/GroupDeviceList.vue
src/store/group/actions.js
```

### 2. 新增版面配置
```bash
# 主要檔案
packages/const-vsaas/Layout.js  # 新增版面定義
src/store/group/getters.js      # 更新頁面計算
```

### 3. 除錯群組狀態
```javascript
// 在 Vue DevTools 中檢查
this.$store.state.group.rawGroupsMap
this.$store.getters['group/activeGroup']
this.$store.getters['group/activeGroupPages']
```

---

## 跨模組依賴

| 模組 | 用途 |
|------|------|
| `device` | 設備資料（groupDevicesMap） |
| `permission` | 權限檢查 |
| `organization` | 組織設定 |
| `ui` | 載入狀態 |

---

## 本地儲存整合

```javascript
localStorage:
- leftMenuOpened    // 左菜單狀態
- showTimeline      // 時間線可見性
- showGroupRight    // 右面板可見性

sessionStorage:
- layout            // 當前版面配置
```

---

## 權限控制

| 權限 | 用途 |
|------|------|
| `PERMISSION_KEYS.GROUP` | 存取群組頁面 |
| `PERMISSION_DEVICE_SCOPE.DEVICE_PLAYBACK` | 回放權限 |
| `PERMISSION_ORG_SCOPE.ORG_ADMIN_RESTRICTED` | 編輯群組權限 |

---

## API 整合

```javascript
// 通過 rootState.apiService
listSite()                              // 取得群組清單
createSite({ name, location })          // 建立群組
updateSite({ siteId, name, location })  // 更新群組
deleteSite({ siteId })                  // 刪除群組
moveDevicesToSite({ siteId, devices })  // 移動設備
```

---

## 效能最佳化

1. **虛擬滾動**：只渲染可見群組列表項
2. **預連接**：輪播時提前建立下一頁連接
3. **計算屬性緩存**：Vuex getters 自動緩存
4. **版面配置緩存**：sessionStorage 保存用戶選擇

---

## 注意事項

1. **默認路由**：應用預設導向群組頁面
2. **組織群組**：特殊群組，不可刪除
3. **設備所有權**：設備只能屬於一個群組
4. **即時更新**：設備狀態透過 WebSocket 更新
