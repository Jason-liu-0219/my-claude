---
name: vsaas-system
description: Use when working with system settings, organization configuration, license management, SSO, MFA, audit logs, or third-party integrations. Triggers on keywords like "system", "organization", "license", "SSO", "MFA", "audit", "alarm management".
version: 1.0.0
---

# VSaaS Portal - System 模組

系統設定模組負責組織層級的配置，包含組織資訊、授權、安全設定和第三方整合。

## 模組概述

### 功能範圍
- 組織資訊與設定
- 授權管理
- 告警管理
- 稽核日誌
- SSO 單一登入
- MFA 多因素驗證
- 全域設備設定
- 第三方整合

---

## 路由結構

```
/system
├── organization              # 組織詳情
├── license-information       # 授權資訊
├── alarm-management          # 告警管理
├── audit-log                 # 稽核日誌
├── reseller-management       # 經銷商管理
├── single-sign-on            # SSO 設定
├── organization-security     # 組織安全（MFA）
├── global-device-settings/   # 全域設備設定
│   ├── auto-firmware-update  # 自動韌體更新
│   ├── event-delay           # 事件延遲
│   └── ai-consent-and-control-settings # AI 同意設定
└── third-party-integration/  # 第三方整合
    ├── access-control        # 門禁系統
    └── smart-sensor          # 智慧感測器
```

---

## 目錄結構

```
src/pages/system/
├── index.vue                 # 路由容器
├── route.js                  # 路由定義
├── organization/             # 組織設定
├── LicenseInformation/       # 授權資訊
├── AlarmManagement/          # 告警管理
├── AuditLog/                 # 稽核日誌
├── ResellerManagement/       # 經銷商管理
├── SingleSignOn/             # SSO
├── OrganizationSecurity/     # MFA
├── GlobalDeviceSettings/     # 全域設備設定
│   ├── AutoFirmwareUpdate/
│   ├── EventDelay/
│   └── AIConsentAndControlSettings/
├── ThirdPartyIntegration/    # 第三方整合
│   ├── AccessControl/
│   └── SmartSensor/
└── reskin/                   # Reskin 版本
```

---

## Store 依賴

### organization 模組
**位置**：`src/store/organization/`

| Action | 說明 |
|--------|------|
| `fetchOrganization` | 取得組織資訊 |
| `updateOrganization` | 更新組織設定 |

### license 模組
**位置**：`src/store/license/`

| Action | 說明 |
|--------|------|
| `fetchLicense` | 取得授權資訊 |
| `checkLicenseFeature` | 檢查功能授權 |

### alarm 模組
**位置**：`src/store/alarm/`

| Action | 說明 |
|--------|------|
| `fetchAlarmRules` | 取得告警規則 |
| `updateAlarmRule` | 更新告警規則 |
| `createAlarmRule` | 建立告警規則 |

### auditLog 模組
**位置**：`src/store/auditLog/`

| Action | 說明 |
|--------|------|
| `fetchAuditLogs` | 取得稽核日誌 |
| `exportAuditLogs` | 匯出日誌 |

### singleSignOn 模組
**位置**：`src/store/singleSignOn/`

| Action | 說明 |
|--------|------|
| `fetchSSOConfig` | 取得 SSO 設定 |
| `updateSSOConfig` | 更新 SSO 設定 |

### mfa 模組
**位置**：`src/store/mfa/`

| Action | 說明 |
|--------|------|
| `fetchMFASettings` | 取得 MFA 設定 |
| `enableMFA` | 啟用 MFA |

### 第三方整合模組
- `accessControl` - 門禁整合
- `smartSensor` - 智慧感測器

---

## 功能詳解

### 1. 組織設定
```
位置：src/pages/system/organization/
Store：organization

功能：
- 組織名稱與資訊
- 時區設定
- 語言設定
```

### 2. 授權管理
```
位置：src/pages/system/LicenseInformation/
Store：license

功能：
- 查看授權狀態
- 設備授權數量
- 功能授權清單
```

### 3. 告警管理
```
位置：src/pages/system/AlarmManagement/
Store：alarm

功能：
- 告警規則設定
- 通知方式設定
- 暫停規則（Snooze）
```

### 4. SSO 設定
```
位置：src/pages/system/SingleSignOn/
Store：singleSignOn

支援的 IdP：
- SAML 2.0
- OAuth 2.0
- OpenID Connect
```

### 5. MFA 設定
```
位置：src/pages/system/OrganizationSecurity/
Store：mfa

支援方式：
- TOTP（Google Authenticator）
- Email OTP
```

### 6. 第三方整合
```
位置：src/pages/system/ThirdPartyIntegration/

Access Control（門禁）：
- 整合門禁系統事件
- 觸發攝影機錄影

Smart Sensor（智慧感測器）：
- IoT 感測器整合
- 環境監控
```

---

## 權限控制

```javascript
// constants/Permission.js
PERMISSION_KEYS = {
  SYSTEM: 'system',
  SYSTEM_ORGANIZATION: 'system/organization',
  SYSTEM_LICENSE: 'system/license-information',
  SYSTEM_ALARM: 'system/alarm-management',
  SYSTEM_AUDIT_LOG: 'system/audit-log',
  SYSTEM_SSO: 'system/single-sign-on',
  SYSTEM_MFA: 'system/organization-security',
  // ...
}
```

---

## 常見開發任務

### 1. 新增系統設定頁面
```bash
# 1. 建立頁面組件
src/pages/system/NewSetting/index.vue

# 2. 新增路由
src/pages/system/route.js

# 3. 新增權限定義
src/constants/Permission.js

# 4. 更新導航選單
```

### 2. 修改組織設定
```bash
# 主要檔案
src/pages/system/organization/index.vue
src/store/organization/actions.js
```

### 3. 新增第三方整合
```bash
# 1. 建立整合頁面
src/pages/system/ThirdPartyIntegration/NewIntegration/

# 2. 建立對應 Store 模組
src/store/newIntegration/

# 3. 新增 API 服務
src/models/API/NewIntegrationAPI.js
```

---

## 注意事項

1. **權限分層**：每個子功能有獨立權限
2. **組織層級**：設定影響整個組織
3. **敏感操作**：SSO/MFA 變更需額外確認
4. **稽核記錄**：重要操作會記錄到稽核日誌

---

## 相關資源

- `references/routes.md` - 完整路由結構
- `references/store.md` - Store 詳解
