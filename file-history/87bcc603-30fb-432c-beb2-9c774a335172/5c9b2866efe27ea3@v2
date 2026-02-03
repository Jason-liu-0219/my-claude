---
name: vsaas-timelapse
description: Use when working with timelapse settings, snapshot frequency, capture schedule, timelapse export, or time-lapse video features. Triggers on keywords like "timelapse", "time-lapse", "snapshot", "capture schedule", "縮時攝影", "export timelapse".
version: 1.1.0
---

# VSaaS Portal - Timelapse 模組

縮時攝影模組提供設備的時間縮放設定功能和縮時影片導出功能。

## 模組概述

本模組包含兩個主要部分：
1. **設備設定** - 配置縮時攝影快照頻率和排程
2. **縮時導出** - 將快照匯出為縮時影片

### 商業邏輯
- 啟用/禁用設備縮時攝影功能
- 設定快照頻率（1小時至1天）
- 配置捕獲排程（全天或自訂時間範圍）
- 預估每日儲存使用量
- **匯出縮時影片**（選擇日期範圍、背景處理、下載）

### 功能特點
| 特點 | 說明 |
|------|------|
| 離線配置 | 可在設備離線時設定 |
| 多種頻率 | 1/3/6/12小時或1天 |
| 彈性排程 | 全天或自訂時間範圍 |
| 使用量預估 | 即時計算每日快照數和儲存量 |

---

## 路由結構

| 路徑 | 名稱 | 說明 |
|------|------|------|
| `/device/:deviceId/settings/media/timelapse` | DeviceSettingsTimelapse | 縮時攝影設定頁面 |

**路由特性**：
- `requiresOnline: false` - 可在設備離線時配置
- 權限要求：`DEVICE_SETTING`
- 功能標誌：`SETTING_TIMELAPSE`

---

## 目錄結構

```
src/pages/device/settings/media/DeviceSettingsTimelapse/
├── index.vue                        # 主組件
├── components/
│   ├── EnableTimelapseSetting.vue   # 啟用/禁用切換
│   ├── SnapshotFrequency.vue        # 快照頻率下拉選單
│   ├── CaptureSchedule.vue          # 捕獲排程設定
│   └── EstimatedDailyUsage.vue      # 預估每日使用量
└── composables/
    ├── useTimelapseFrequencyHelper.js      # 頻率邏輯
    ├── useTimelapseScheduleHelper.js       # 排程邏輯
    └── useTimelapseSettingAPIHelper.js     # API 交互
```

---

## 設定選項

### 快照頻率
```javascript
SNAPSHOT_FREQUENCY_MAP = {
  '1_HOUR':   { text: '1 hour',   value: 1 },
  '3_HOURS':  { text: '3 hours',  value: 3 },
  '6_HOURS':  { text: '6 hours',  value: 6 },
  '12_HOURS': { text: '12 hours', value: 12 },
  '1_DAY':    { text: '1 day',    value: 24 }
}
```

### 捕獲排程
| 類型 | 說明 |
|------|------|
| Always Active (24/7) | 全天捕獲（00:00 - 23:59） |
| Custom Schedule | 自訂時間範圍 |

**特殊情況**：1天頻率時只顯示單一時間選擇器（計畫時間）

---

## Composables

### useTimelapseFrequencyHelper
**職責**：管理快照頻率設定

```javascript
// 導出
currentSnapshotFrequency         // 當前頻率（小時）
snapshotFrequencyOptions         // 所有可用選項
snapshotFrequencyInSeconds       // 轉換為秒
isOneDayFrequency                // 是否為1天頻率
initializeCurrentSnapshotFrequency()  // 初始化
```

### useTimelapseScheduleHelper
**職責**：管理捕獲排程

```javascript
// 導出
customSchedule                   // { startTime, endTime } HH:MM 格式
captureScheduleIndex             // 選中的排程索引
captureScheduleOptions           // ['Always Active', 'Custom Schedule']
captureSchedule                  // 當前排程值
customScheduleInSeconds          // 轉換為秒
isAlwaysActiveSchedule           // 24/7 排程檢查
isCustomSchedule                 // 自訂排程檢查
verifyCustomSchedule()           // 驗證並糾正時間範圍
initializeCaptureSchedule()      // 初始化
```

### useTimelapseSettingAPIHelper
**職責**：API 交互

```javascript
// State
state = {
  timelapseEnable: boolean,
  snapshotFrequency: number,     // 小時
  startTime: string,             // HH:MM
  endTime: string,               // HH:MM
  snapshotSize: number           // KB
}

// 方法
getTimelapseSettings(deviceId)   // 取得設定
updateTimelapseSettings(payload) // 保存設定
snapshotSizeInMB                 // 轉換為 MB
```

---

## 關鍵程式碼路徑

### 進入點
- `src/pages/device/settings/media/DeviceSettingsTimelapse/index.vue`
- `src/pages/device/settings/route.js` - 路由定義
- `src/utils/DeviceSettingsMenu.js` - 側邊選單

### 資料流

#### 載入流程
```
頁面掛載
    ↓
loadSettings()
    ↓
getTimelapseSettings(deviceId) [API]
    ↓
initializeCurrentSnapshotFrequency()
initializeCaptureSchedule()
    ↓
emit('loaded')
```

#### 保存流程
```
用戶修改設定
    ↓
watch(isChanged) → setDirty()
    ↓
點擊保存
    ↓
handleUpdateTimelapseSettings(payload)
    ↓
updateTimelapseSettings() [API]
    ↓
updateOptions({ isTimelapseEnabled })
```

---

## 主要組件

### 1. index.vue（主組件）
**功能**：縮時攝影設定頁面

**主要邏輯**：
```javascript
// 變更檢測
isChanged = computed(() => {
  return (
    frequency !== state.snapshotFrequency ||
    startTime !== state.startTime ||
    endTime !== state.endTime
  )
})

// Dirty 狀態管理
watch(isChanged, (newVal) => {
  $_setDirty(newVal)
})
```

### 2. EnableTimelapseSetting.vue
**功能**：啟用/禁用切換

**Props**：`modelValue`, `disabled`
**Emits**：`update`

### 3. SnapshotFrequency.vue
**功能**：快照頻率下拉選單

使用 `AppDropdown` 元件

### 4. CaptureSchedule.vue
**功能**：捕獲排程設定

**條件渲染**：
- 1天頻率：單個時間選擇器
- 其他頻率：
  - 無線電按鈕（Always Active / Custom）
  - Custom：兩個時間選擇器

### 5. EstimatedDailyUsage.vue
**功能**：預估每日使用量

**計算邏輯**：
```javascript
// 捕獲時長
duration = endTime - startTime  // 秒

// 每日快照數
snapshotCount = 1 + Math.floor(duration / frequency)

// 每日使用量
usage = snapshotCount × snapshotSize  // MB
```

---

## API 端點

```javascript
// 取得設定
GET /v1/devices/{deviceId}/timelapse

// 更新設定
PATCH /v1/devices/{deviceId}/timelapse
// body: { timelapseEnable, snapshotFrequency, startTime, endTime }

// 清理資源
POST /v1/timelapse/resources-cleanup
```

---

## 常數定義

**位置**：`packages/const-vsaas/Timelapse.js`

```javascript
SNAPSHOT_FREQUENCY_MAP = {...}
TIMELAPSE_TASK_STATUS = {
  RUNNING, DONE, CANCELLED, FAILED
}
MAX_EXPORT_DURATION = 366 * 24 * 60 * 60 * 1000  // 366 天
```

---

## 常見開發任務

### 1. 新增快照頻率選項
```bash
# 主要檔案
packages/const-vsaas/Timelapse.js  # 新增到 SNAPSHOT_FREQUENCY_MAP
```

### 2. 修改排程邏輯
```bash
# 主要檔案
composables/useTimelapseScheduleHelper.js
components/CaptureSchedule.vue
```

### 3. 除錯設定狀態
```javascript
// 在組件中
const { state, snapshotSizeInMB } = useTimelapseSettingAPIHelper()
console.log(state.value)
```

---

## 使用的組件

### 來自 generic-vue-components
- `AppDropdown` - 下拉選單
- `AppRadioButton` - 無線電按鈕
- `AppScrollTimePicker` - 時間選擇器
- `AppStatefulUpdateButton` - 保存按鈕
- `AppToggle` - 切換開關

---

## 側邊選單配置

**位置**：`src/utils/DeviceSettingsMenu.js`

```javascript
{
  path: 'media/timelapse',
  label: 'Timelapse',
  isSupport: hasSettingsPermission &&
             supportList.includes(DARK_RELEASE_FEATURES.SETTING_TIMELAPSE)
}
```

---

## 功能標誌

```javascript
// 需要功能標誌才能顯示
DARK_RELEASE_FEATURES.SETTING_TIMELAPSE
```

---

## Vuex 整合

```javascript
// Dirty 狀態管理
store.dispatch('ui/setDirty', {
  name: 'Timelapse',
  value: true
})

// 設備選項更新
updateOptions({ isTimelapseEnabled: true/false })
```

---

## 注意事項

1. **離線配置**：`requiresOnline: false` 允許離線設定
2. **時間格式**：使用 HH:MM 格式（24小時制）
3. **自動驗證**：End Time < Start Time 時自動交換
4. **設備同步**：保存後更新設備的 `isTimelapseEnabled` 選項
5. **i18n 支援**：所有文本使用 `$t()` 進行多語言支援
6. **1天頻率特殊處理**：只顯示單一時間選擇器

---

# Part 2: Timelapse Export 頁面

獨立的縮時導出頁面，用於將快照匯出為縮時影片。

## 路由結構

| 路徑 | 名稱 | 說明 |
|------|------|------|
| `/timelapse/export/:deviceId` | TimeLapseExport | 縮時影片導出頁面 |

**權限要求**：`PERMISSION_KEYS.SYSTEM`

---

## 目錄結構

```
src/pages/Timelapse/
├── index.vue                        # 包裝組件
├── route.js                         # 路由定義
└── export/
    ├── index.vue                    # 主導出頁面
    ├── composables/
    │   ├── dateHelpers.js           # 日期工具函數
    │   ├── useTimelapseExport.js    # 導出任務管理
    │   └── useTimelapseModel.js     # 數據獲取和過濾
    └── components/
        ├── TimelapseStep1.vue       # 日期範圍選擇
        ├── TimelapseSort.vue        # 排序切換
        ├── TimelapseList.vue        # 虛擬滾動列表
        ├── TimelapseItem.vue        # 快照卡片
        ├── TimelapseExportTaskButton.vue    # 導出列表按鈕
        ├── TimelapseExportTaskList.vue      # 導出任務列表
        └── TimelapseExportTaskCard.vue      # 導出任務卡片
```

---

## 三步驟導出流程

### Step 1：日期選擇
- 日期範圍選擇器
- 根據可用快照禁用日期
- 預設：最近 30 天

### Step 2：開始導出
- 點擊「Start Export」按鈕
- 背景處理視頻生成
- 同一時間只能有一個導出任務

### Step 3：導出管理
- 查看導出任務狀態
- 下載已完成的影片
- 取消進行中的任務

---

## 核心 Composables

### useTimelapseModel
**職責**：快照數據管理

```javascript
// 數據獲取
fetchDailySnapshotSummary()      // 獲取快照摘要
fetchNextTimelapseListData()      // 懶加載下一批

// 數據結構
snapshotSummaryList              // 按月分組的快照
timelapseListData                // 排序後的列表
disabledDays                     // 無快照的日期
```

**特點**：
- 4 個月窗口緩存優化
- 虛擬滾動支援大量數據

### useTimelapseExport
**職責**：導出任務管理

```javascript
// 任務操作
exportTimelapse({ beginDate, endDate })  // 建立導出任務
cancelExportTask(task)                    // 取消任務
downloadExportedTimelapse(task)           // 下載影片

// 任務狀態
RUNNING, DONE, FAILED, CANCELLED
```

**輪詢機制**：每 60 秒自動更新任務狀態

---

## 導出 API

```javascript
// 獲取快照摘要
GET /v1/timelapse/daily-snapshot-summary
// params: { beginDate, endDate, thingName, derivant }

// 建立導出任務
POST /v1/timelapse/export-tasks
// body: { beginDate, endDate, thingName, derivant }

// 獲取導出任務列表
GET /v1/timelapse/export-tasks

// 取消導出任務
POST /v1/timelapse/export-task-cancel
// body: { taskID }
```

---

## 下載機制

```
1. 請求設備 Token (grantDeviceToken)
2. 建立 AWS CloudStorage 服務
3. 生成預簽名 URL
4. 觸發瀏覽器下載
5. 檔名格式：Timelapse 2025/01/01~2025/01/31.mp4
```

---

## 效能最佳化

1. **虛擬滾動**：支援大量快照數據
2. **懶加載**：滾動時獲取缺失月份
3. **4 個月緩存**：減少 API 請求
4. **防抖更新**：日曆月份變更 500ms 防抖

---

## 相關資源

- 縮時攝影常數：`@vivotek/const-vsaas` - `TIMELAPSE.*`
- 設備設定路由：`src/pages/device/settings/route.js`
- 導出頁面路由：`src/pages/Timelapse/route.js`
- 側邊選單：`src/utils/DeviceSettingsMenu.js`
