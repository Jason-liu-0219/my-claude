---
name: vsaas-notification
description: Use when working with notification center, message management, event alerts, snooze rules, or alarm notifications. Triggers on keywords like "notification", "message center", "event", "snooze", "alert", "tampering".
version: 1.0.0
---

# VSaaS Portal - Notification 模組

通知模組提供事件訊息管理功能，支援多種事件類型、暫停規則、事件詳情查看等功能。

## 模組概述

### 商業邏輯
- 查看和管理設備/系統事件通知
- 支援存取控制和智慧感測器事件
- 設定暫停規則（Snooze Rules）
- 事件詳情和視頻回放
- 假警報報告功能

### 事件類型
| 類型 | 常數 | 說明 |
|------|------|------|
| DEVICE | `MESSAGE_TYPE.DEVICE` | 設備事件（動作檢測、VCA 等） |
| SYSTEM | `MESSAGE_TYPE.SYSTEM` | 系統事件（NVR 錄影中斷等） |
| ACCESS | `MESSAGE_TYPE.ACCESS` | 存取控制事件（門禁） |
| SENSOR | `MESSAGE_TYPE.SENSOR` | 智慧感測器事件 |

---

## 路由結構

| 路徑 | 名稱 | 說明 |
|------|------|------|
| `/notification/` | MessageCenter | 訊息中心主頁 |
| `/notification/:source/:eventId/` | MessageCenterDetails | 事件詳情 |
| `/notification/:thirdPartyType/:provider/:id/` | MessageCenterThirdPartyEventDetails | 第三方事件詳情 |
| `/notification/settings` | NotificationSettings | 通知設定（重定向） |

**第三方類型**：`access-control`, `smart-sensor`

**權限要求**：`PERMISSION_KEYS.MESSAGE_CENTER`

---

## 目錄結構

```
src/pages/notification/
├── index.vue                        # 路由容器
├── route.js                         # 路由定義
├── message-center/
│   ├── index.vue                    # 訊息中心主元件
│   └── components/
│       ├── MessageCenterTable.vue           # 通知列表表格
│       ├── MessageCenterDetailPanel.vue     # 詳細面板
│       ├── MessageCenterInformationPanel.vue # 事件資訊
│       ├── EventThumbnail.vue               # 事件縮圖
│       ├── SystemEventThumbnail.vue         # 系統事件縮圖
│       ├── FacialInformation.vue            # 臉部辨識資訊
│       ├── LicensePlateInformation.vue      # 車牌辨識資訊
│       ├── FallDetectionInformation.vue     # 跌倒檢測資訊
│       ├── ResetTamperingDialogue.vue       # 重置防拆對話框
│       └── SnoozeRuleSidebar/
│           ├── SnoozeRuleSidebar.vue        # 暫停規則側欄
│           ├── DeviceSnoozeRuleCard.vue     # 設備暫停規則卡片
│           └── SnoozeRuleInfo.vue           # 規則詳情
└── settings/
    └── index.vue                    # 通知設定頁面

src/pages/account/notificationCenter/
└── index.vue                        # 全域暫停通知設定

src/store/notification/
├── index.js                         # 狀態定義
├── mutations.js                     # 狀態變更
├── actions.js                       # 非同步操作
└── getters.js                       # 計算屬性
```

---

## Store 依賴

### 主要模組：notification
**位置**：`src/store/notification/`

```javascript
state: {
  currentRequestId: null,            // 防止競態條件
  events: {
    [MESSAGE_TYPE.DEVICE]: [],       // 設備事件
    [MESSAGE_TYPE.SYSTEM]: [],       // 系統事件
    [MESSAGE_TYPE.ACCESS]: [],       // 存取控制事件
    [MESSAGE_TYPE.SENSOR]: [],       // 感測器事件
  },
  totalCount: {
    [MESSAGE_TYPE.DEVICE]: 0,
    [MESSAGE_TYPE.SYSTEM]: 0,
    [MESSAGE_TYPE.ACCESS]: 0,
    [MESSAGE_TYPE.SENSOR]: 0,
  },
  tamperingResetCache: [],           // 防拆重置快取
  snoozeRules: {},                   // 暫停規則映射
  shouldDisplaySnoozeRule: false,    // 是否顯示暫停規則功能
}
```

#### 關鍵 Actions

**事件查詢**：
| Action | 說明 |
|--------|------|
| `queryMessage` | 查詢設備/系統事件（分頁串流） |
| `getMessage` | 取得單一事件詳情 |
| `queryThirdPartyMessages` | 查詢存取控制/感測器事件 |
| `queryThirdPartySingleMessage` | 取得單一第三方事件 |
| `queryAccessControlMessages` | 查詢存取控制事件 |
| `querySensorMessages` | 查詢感測器事件 |

**暫停規則管理**：
| Action | 說明 |
|--------|------|
| `querySnoozeRules` | 查詢組織暫停規則 |
| `createSnoozeRule` | 建立暫停規則 |
| `updateSnoozeRule` | 更新暫停規則 |
| `deleteSnoozeRule` | 刪除暫停規則 |
| `fetchShouldDisplaySnoozeRule` | 檢查功能開關 |

#### 關鍵 Getters

| Getter | 說明 |
|--------|------|
| `isFetchingData` | 是否正在擷取資料 |
| `snoozeRules` | 所有暫停規則 |
| `totalSnoozeRules` | 活躍暫停規則數 |
| `snoozeRulesDeviceMap` | 按設備/事件類型組織的規則 |
| `shouldDisplaySnoozeRule` | 是否顯示暫停規則功能 |

---

## 暫停規則系統

### SnoozeRule 類別
**位置**：`src/models/SnoozeRule/SnoozeRule.js`

```javascript
class SnoozeRule {
  constructor({
    id,                  // 規則 ID
    thingName,           // 設備主名稱
    derivant,            // 衍生設備標識
    messageTypeID,       // 事件類型 ID
    eventType,           // 事件類型
    ruleName,            // 規則名稱
    snoozeFor,           // 暫停範圍
    snoozeUntil,         // ISO 時間字串
    editorEmail,         // 建立者郵箱
  })

  // 方法
  async create(organizationID, snoozeUntil, editorEmail)
  async update({organizationID, snoozeUntil, editorEmail})
  async delete({id, organizationID, snoozeFor})
  reset()               // 清除規則狀態
}
```

### 暫停範圍
| 類型 | 說明 |
|------|------|
| `ONLY_ME` | 僅暫停此用戶的通知 |
| `EVERYONE` | 暫停整個組織的通知 |

---

## 關鍵程式碼路徑

### 進入點
- `src/pages/notification/index.vue` - 路由容器
- `src/pages/notification/route.js` - 路由定義
- `src/store/notification/index.js` - Store 模組

### 資料流

#### 事件查詢流程
```
用戶搜尋/篩選
    ↓
queryMessage({filter, type}) [Action]
    ↓
API 呼叫（callback pattern）
    ↓
items, totalCount, nextToken
    ↓
commit('setEvents') + commit('setTotalCount')
```

#### 暫停規則流程
```
用戶點擊暫停
    ↓
開啟 SnoozeRuleDialogue
    ↓
選擇時長和範圍
    ↓
createSnoozeRule() [Action]
    ↓
規則建立 + 設置自動重置計時器
```

---

## 主要元件

### 1. MessageCenter (index.vue)
**功能**：訊息中心主頁面

**主要特性**：
- 四種事件類型標籤頁
- 動態搜尋欄
- 分頁顯示（每頁 50 筆）
- 事件詳情面板
- 暫停規則側欄

**主要計算屬性**：
```javascript
notificationList     // 當前類型事件列表
eventTypeOptions     // 可用事件類型
tabsData             // 標籤頁配置
```

### 2. MessageCenterTable.vue
**功能**：事件列表表格

**欄位**：
- `snapshot` - 事件縮圖
- `deviceName` - 設備名稱 + 狀態
- `eventState` - 事件狀態
- `translatedEventType` - 事件類型
- `clientTime` / `localTime` - 時間
- `information` - 額外資訊
- `snooze-rule` - 暫停按鈕

### 3. MessageCenterDetailPanel.vue
**功能**：事件詳細信息面板

**操作按鈕**：
- `Resolved` - 重置防拆狀態
- `Archive` - 建立事件存檔
- `Snooze Rule` - 設定暫停規則

### 4. SnoozeRuleSidebar.vue
**功能**：暫停規則管理側欄

**結構**：
- 按設備分組
- 每個設備下顯示事件規則卡片
- 支援編輯和刪除

---

## 搜尋和過濾

### 過濾器（按事件類型）

| 事件類型 | 可用過濾器 |
|---------|----------|
| DEVICE | 設備、事件類型、時間範圍 |
| SYSTEM | 設備、事件類型、時間範圍 |
| ACCESS | 存取點、群組、事件類型、時間範圍 |
| SENSOR | 感測器、事件類型、群組、時間範圍 |

### 時間範圍
- 預設：7 天
- 最大：90 天

---

## 使用的組件

### 來自 advanced-vue-components
- `DisplayStreamingBlock` - 串流顯示
- `PlayerControl` - 播放器控制
- `Timeline` - 時間軸

### 來自 generic-vue-components
- `AppTable` - 表格
- `AppTabs` - 標籤頁
- `AppPagination` - 分頁
- `AppSearchInput` - 搜尋輸入

---

## 常見開發任務

### 1. 新增事件類型支援
```bash
# 主要檔案
src/pages/notification/message-center/index.vue
src/store/notification/actions.js
packages/const-vsaas/EventType.js
```

### 2. 修改暫停規則邏輯
```bash
# 主要檔案
src/models/SnoozeRule/SnoozeRule.js
src/store/notification/actions.js
```

### 3. 新增事件資訊元件
```bash
# 建立元件
src/pages/notification/message-center/components/NewEventInformation.vue

# 在 MessageCenterInformationPanel.vue 中引用
```

---

## 權限控制

| 權限 | 用途 |
|------|------|
| `PERMISSION_KEYS.MESSAGE_CENTER` | 存取訊息中心 |
| `PERMISSION_DEVICE_SCOPE.DEVICE_ALL_USERS` | 事件檢視 |
| `hasPlanPermission('device/reportfalsealarm')` | 假警報報告 |
| `aiControlPermissionManager` | AI 功能權限 |

---

## 帳戶級通知設定

**位置**：`src/pages/account/notificationCenter/index.vue`

**功能**：全域暫停組織所有通知

**特性**：
- 選擇暫停時間（持續時間或自訂）
- 只暫停行動和電子郵件通知
- 訊息中心仍記錄事件

---

## API 整合

```javascript
// 事件查詢
queryMessage({ filter, type })
getMessage({ eventID, source })

// 第三方事件
queryAccessControlMessages({ filter, offset, limit })
querySensorMessages({ filter, offset, limit })

// 暫停規則
querySnoozeRules()
createSnoozeRule()
updateSnoozeRule()
deleteSnoozeRule()
```

---

## 效能考量

1. **分頁**：每頁 50 筆，支援前後翻頁
2. **請求去重**：`currentRequestId` 防止競態條件
3. **快取**：`tamperingResetCache` 防止重複操作
4. **虛擬滾動**：AppTable 支持大型資料集

---

## 功能開關

```javascript
// 遠端配置
RemoteConfigService.SNOOZE_MESSAGE     // 暫停規則功能
RemoteConfigService.SMART_SENSOR       // 智慧感測器功能
```

---

## 注意事項

1. **事件來源**：區分 `device` 和 `system` 來源
2. **第三方整合**：存取控制和感測器使用不同 API
3. **暫停計時器**：規則到期自動重置
4. **假警報**：需要特定權限才能報告
