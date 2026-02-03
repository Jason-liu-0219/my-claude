---
name: vsaas-alarm
description: Use when working with alarm management, alarm rules, webhook configuration, snooze settings, or event-triggered notifications. Triggers on keywords like "alarm", "alert", "webhook", "notification rule", "snooze", "alarm management".
version: 1.0.0
---

# VSaaS Portal - Alarm 模組

告警管理模組負責事件觸發的通知規則配置，包含告警規則、Webhook、靜音設置和多種通知方式。

## 模組概述

### 功能範圍
- 告警規則 CRUD
- 事件來源配置（設備、群組、訪問控制、傳感器）
- 通知動作配置（Webhook、郵件、SMS、揚聲器）
- 時間表設定
- 靜音規則管理

---

## 目錄結構

```
src/pages/alarm/
├── index.vue                 # 主頁面（路由包裹器）
├── route.js                  # 路由配置
└── mock/
    └── index.vue             # UI 原型頁面

src/store/alarm/
├── index.js                  # Store 模組
├── actions.js                # 異步操作
├── mutations.js              # 狀態變化
└── getters.js                # 計算屬性

src/pages/system/components/
├── AlarmManagement.vue       # 告警管理主組件
├── AlarmManagementTable.vue  # 告警表格
├── AlarmManagementDetails.vue # 告警詳情面板
└── EditAlarmModal/           # 編輯告警模態框（5 步驟）
    ├── index.vue
    ├── EditAlarmModalStepEvent.vue
    ├── EditAlarmModalStepSource.vue
    ├── EditAlarmModalStepAction.vue
    ├── EditAlarmModalStepSchedule.vue
    └── EditAlarmModalStepSummary.vue

src/models/
├── Alarm/Alarm.js            # Alarm 資料模型
└── Webhook/WebhookProfile.js # Webhook 設定檔模型
```

---

## 路由結構

```
/alarm
├── ''          → 重定向到 AlarmMock
└── /mock       → AlarmMock (UI 原型)

# 生產環境路由在 /system/alarm-management
```

---

## Store 模組

### State

```javascript
{
  alarms: [],                    // 告警列表
  status: APP_TABLE_STATUS,      // 載入狀態
  activeAlarmID: null,           // 選中的告警 ID
  webhookProfileMap: {},         // Webhook 設定檔映射
  loadedAlarmSourceIds: {},      // 已載入 S3 源的告警
  loadingAlarmSourceIds: {}      // 載入中的告警
}
```

### 主要 Actions

| Action | 功能 |
|--------|------|
| `fetchAlarmSettings` | 獲取組織所有告警 |
| `createAlarm` | 創建新告警 |
| `deleteAlarm` | 刪除告警（支援批量） |
| `switchAlarmEnable` | 啟用/禁用告警 |
| `queryWebhookProfile` | 查詢 Webhook 設定檔 |
| `batchFetchAlarmSources` | 批量載入 S3 源數據（分頁優化） |

### 主要 Getters

| Getter | 說明 |
|--------|------|
| `alarmCount` | 告警總數 |
| `activeAlarm` | 當前活躍告警 |
| `webhookProfiles` | Webhook 設定檔列表 |
| `isAlarmSourceLoaded` | 告警源是否已載入 |

---

## 資料模型

### Alarm 模型

```javascript
class Alarm {
  // 基本屬性
  alarmID, alarmName, enable, scheduleBitwise
  events, actions, createdBy
  getSettingsURL, putSettingsURL  // S3 URLs
  
  // S3 物件屬性（複雜配置）
  recipients          // 收件人 (EMAIL/SMS/VOICE)
  source              // 設備來源
  doSettings          // 數字輸出設置
  webhook             // Webhook 配置
  networkSpeakers     // 網絡揚聲器
  audioDeterrentCameras // 音頻威懾攝像機
  accessControlSource // 訪問控制源
  smartSensorSource   // 智能傳感器源
  pauseUntil          // 暫停截止時間
  city                // 時區城市

  // 方法
  async update(payload)
  async queryS3Object()
  async updateS3Object(payload)
  async setPauseAlarm(pauseFor)
}
```

### WebhookProfile 模型

```javascript
class WebhookProfile {
  webhookID, organizationID
  form: { name, url, contentType, sender, headers, payload }
  alarmIDBindings     // 綁定的告警
  
  async create()
  async update()
  async test()
  async delete()
}
```

---

## 告警創建流程（5 步驟）

```
Step 1: EVENT      → 選擇告警事件類型
Step 2: SOURCE     → 選擇設備、群組、訪問點、傳感器
Step 3: ACTION     → 配置通知動作和收件人
Step 4: SCHEDULE   → 設置時間表和時區
Step 5: SUMMARY    → 審核並保存
```

---

## S3 數據分離策略

**目的**：告警配置包含複雜嵌套數據，使用 S3 分離存儲

```
主 API (GraphQL)        S3 服務器
├─ alarmID              ├─ recipients
├─ alarmName            ├─ source
├─ events               ├─ doSettings
├─ actions              ├─ webhook
├─ scheduleBitwise      ├─ networkSpeakers
├─ enable               ├─ audioDeterrentCameras
├─ getSettingsURL       ├─ accessControlSource
└─ putSettingsURL       └─ smartSensorSource
```

---

## 延遲載入優化

```javascript
// 分頁場景優化
batchFetchAlarmSources(alarmIds, chunkSize = 5)

// 狀態追蹤
loadedAlarmSourceIds    // 已載入
loadingAlarmSourceIds   // 載入中

// Getters
isAlarmSourceLoaded(alarmId)
areAllSourcesLoaded(alarmIds)
```

---

## 通知方式

| 類型 | 說明 |
|------|------|
| Webhook | 自定義 HTTP 回調 |
| Email | 郵件通知 |
| SMS | 短信通知 |
| File Archive/DO | 文件存檔/數字輸出 |
| Network Speaker | 網絡揚聲器 |
| Audio Deterrent | 音頻威懾 |

---

## API 端點

```javascript
// 告警操作
apiService.queryOrganizationAlarmSetting()
apiService.createOrganizationAlarmSetting()
apiService.updateOrganizationAlarmSetting()
apiService.deleteOrganizationAlarmSettings()
apiService.switchOrganizationAlarmSetting()
apiService.pauseOrganizationAlarmSetting()

// Webhook 操作
apiService.createWebhook()
apiService.updateWebhook()
apiService.deleteWebhook()
apiService.cloudTestWebhook()

// S3 操作
apiService.requestToS3Server()
```

---

## 開發指南

### 新增告警類型

1. 在常量文件定義事件類型
2. 在 EditAlarmModalStepEvent.vue 添加選項
3. AccessControlHelper 解析新事件
4. 更新類型文檔

### 新增通知方式

1. 定義動作類型和常量
2. 在 EditAlarmModalStepAction.vue 添加 UI
3. 更新 Alarm 模型處理邏輯
4. 實現 API 端點
5. 添加單元測試
