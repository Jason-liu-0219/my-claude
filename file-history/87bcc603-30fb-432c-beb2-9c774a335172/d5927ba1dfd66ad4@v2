---
name: vsaas-account
description: Use when working with account settings, user profile, API key management, MFA, language preferences, or notification center settings. Triggers on keywords like "account", "profile", "API key", "MFA", "language", "notification settings", "password".
version: 1.0.0
---

# VSaaS Portal - Account 模組

帳戶管理模組提供用戶個人設定功能，包括個人資料、語言偏好、多因素認證、API 金鑰管理等。

## 模組概述

### 商業邏輯
- 用戶個人資料管理（頭像、用戶名、密碼）
- 語言偏好設定
- 多因素認證（MFA）配置
- 通知中心勿擾設定
- API 金鑰管理

### 功能模組
| 模組 | 說明 |
|------|------|
| Profile | 個人資料管理 |
| Language | 語言設定 |
| MFA | 多因素認證 |
| Notification Center | 勿擾設定 |
| API Key Management | API 金鑰管理 |

---

## 路由結構

| 路徑 | 名稱 | 說明 |
|------|------|------|
| `/account` | AccountIndex | 重定向到 profile |
| `/account/profile` | AccountProfile | 個人資料頁面 |
| `/account/language` | AccountLanguage | 語言設定頁面 |
| `/account/mfa` | AccountMfa | MFA 設定頁面 |
| `/account/notification-center` | AccountNotificationCenter | 勿擾設定頁面 |
| `/account/api-key-management` | AccountAPIKeyManagement | API 金鑰管理 |

---

## 目錄結構

```
src/pages/account/
├── index.vue                        # 主帳戶頁面（左側導航）
├── route.js                         # 路由定義
├── profile/
│   └── index.vue                    # 個人資料頁面
├── language/
│   ├── index.vue                    # 語言設定頁面
│   └── components/
│       └── ChangeLanguageDialogue.vue
├── mfa/
│   ├── index.vue                    # MFA 設定頁面
│   └── components/
│       └── VerifyMFASetupDialogue.vue
├── notificationCenter/
│   └── index.vue                    # 勿擾設定頁面
└── apiKeyManagement/
    ├── index.vue                    # API 金鑰管理頁面
    └── components/
        ├── AddAPIKeyModal.vue
        ├── DeleteAPIKeyDialogue.vue
        └── APIKeyManagementTable.vue
```

---

## Store 依賴

### 主要模組：account
**位置**：`src/store/account/`

```javascript
state: {
  attributes: {},              // 用戶屬性 (email, sub)
  name: '',                    // 用戶名
  organizationRole: {},        // 組織角色
  apiKeyList: [],              // API 金鑰列表
  isSigningIn: false,          // 登錄狀態
  shouldDisplaySnoozeAllForMe: false,
  agreements: {}               // 用戶協議
}
```

#### 關鍵 Actions

| Action | 說明 |
|--------|------|
| `confirmPassword` | 密碼驗證 |
| `changeLocale` | 更改語言 |
| `updateMyOrganizationUsername` | 更新用戶名 |
| `setMySnoozeUntil` | 設置勿擾時間 |
| `removeMySnoozeUntil` | 移除勿擾 |
| `queryAPIKeyList` | 查詢 API 金鑰列表 |
| `generateAPIKey` | 生成 API 金鑰 |
| `deleteAPIKey` | 刪除 API 金鑰 |
| `deleteAccount` | 刪除帳戶 |

#### 關鍵 Getters

| Getter | 說明 |
|--------|------|
| `organizationRole` | 組織角色 |
| `isOwner` | 是否為所有者 |
| `email` | 用戶郵箱 |
| `sub` | 用戶 sub ID |
| `name` | 用戶名 |
| `shouldDisplaySnoozeAllForMe` | 是否顯示勿擾功能 |

### MFA Store
**位置**：`src/store/mfa/`

```javascript
state: {
  userMFA: undefined    // MFA 狀態
}
```

#### MFA Actions

| Action | 說明 |
|--------|------|
| `fetchUserMFA` | 獲取 MFA 狀態 |
| `getSetupUri` | 獲取 TOTP 設置 URI |
| `enableUserMFA` | 啟用 MFA |
| `disableUserMFA` | 禁用 MFA |

---

## 功能詳解

### 1. Profile（個人資料）

**功能**：
- 用戶頭像展示
- 用戶名編輯
- 郵箱地址顯示（只讀）
- 密碼更改
- 帳戶刪除

**對話框**：
- `ChangeUsernameDialogue` - 更改用戶名
- `ChangePasswordDialogue` - 更改密碼
- `DeleteAccountDialogue` - 刪除帳戶

### 2. Language（語言設定）

**功能**：
- 顯示當前語言
- 語言切換對話框

**資料流**：
```
ChangeLanguageDialogue
  → dispatch('account/changeLocale', targetLocale)
  → API: putUserLocale()
  → loadLocaleMessages()
```

### 3. MFA（多因素認證）

**MFA 狀態**：
| 狀態 | 說明 |
|------|------|
| `ORG_REQUIRED` | 組織強制要求 |
| `ENABLED` | 已啟用 |
| `DISABLED` | 已禁用 |

**功能**：
- MFA 狀態切換
- TOTP 驗證器設置
- 組織級別強制標記

### 4. Notification Center（勿擾設定）

**功能**：
- 設置通知勿擾時間
- 預設時間選項（1/3/6小時等）
- 自訂結束時間
- 時間驗證

**狀態機制**：
```javascript
SNOOZE_STATE = {
  DEFAULT: 'default',
  TIME_ERROR: 'time-error',
  DEFAULT_ERROR: 'default-error',
  DISABLED: 'disabled'
}
```

### 5. API Key Management（API 金鑰管理）

**功能**：
- API 金鑰列表查看
- 新增 API 金鑰
- 刪除 API 金鑰
- 配額限制顯示

**新增金鑰流程**：
```
1. 密碼驗證 → confirmPassword()
2. 配置過期設置（NEVER / 30-365天 / 自訂）
3. 生成金鑰 → generateAPIKey()
4. 顯示金鑰（一次性）
```

**金鑰狀態**：
- `Active` - 有效
- `Expired` - 已過期

---

## 主頁面佈局

```
┌─────────────────────────────────────┐
│  My account (Header)                │
├──────────────┬──────────────────────┤
│              │                      │
│  Left Menu   │  router-view         │
│              │                      │
│  - Profile   │  (子頁面內容)        │
│  - Language  │                      │
│  - MFA       │                      │
│  - Notif.    │                      │
│  - API Keys  │                      │
│              │                      │
└──────────────┴──────────────────────┘
```

**菜單條件渲染**：
```javascript
accountMenu = [
  { name: 'Profile', path: 'profile' },
  { name: 'Preferred language', path: 'language' },
  { name: 'Multi-factor authentication', path: 'mfa' },
  // 條件顯示
  ...(shouldDisplaySnoozeAllForMe ? notificationCenterItem : []),
  ...(shouldDisplayAPIKeyManagement ? apiKeyItem : [])
]
```

---

## 常見開發任務

### 1. 新增帳戶設定項
```bash
# 1. 建立新頁面
src/pages/account/newSetting/index.vue

# 2. 更新路由
src/pages/account/route.js

# 3. 更新選單
src/pages/account/index.vue
```

### 2. 修改 API 金鑰邏輯
```bash
# 主要檔案
src/pages/account/apiKeyManagement/index.vue
src/pages/account/apiKeyManagement/components/
src/store/account/actions.js
```

### 3. 除錯帳戶狀態
```javascript
// 在 Vue DevTools 中
this.$store.state.account
this.$store.getters['account/email']
this.$store.getters['mfa/MFAStatus']
```

---

## 使用的組件

### 來自 generic-vue-components
- `AppAvatar` - 用戶頭像
- `AppButton` - 按鈕
- `AppDialogue` - 對話框
- `AppTable` - 表格
- `AppToggle` - 切換開關

---

## 功能開關

```javascript
// 遠端配置
RemoteConfigService.API_KEY              // API 金鑰功能
RemoteConfigService.SNOOZE_ALL_FOR_ME    // 勿擾功能
```

---

## API 整合

```javascript
// 帳戶操作
apiService.putUserLocale({ userSub, locale })
apiService.updateMyOrganizationUsername({ organizationID, username })
apiService.deleteAccount()

// MFA 操作
apiService.queryUserMFA({ email })
apiService.setUserMFA({ enableMFA })

// 勿擾操作
apiService.setMySnoozeUntil({ organizationID, snoozeUntil })

// API 金鑰操作
apiService.queryAPIKeys()
apiService.createAPIKey({ organizationID, keyName, expiresAt })
apiService.deleteAPIKey({ keyID })
apiService.deleteAllAPIKey()
```

---

## 許可控制

```javascript
// API 金鑰管理特殊配置
meta: {
  licenseForbiddenPage: true,    // 受限功能
  freePlanCanAccess: true        // 免費計畫可訪問
}
```

---

## 注意事項

1. **一次性金鑰**：API 金鑰生成後只顯示一次
2. **組織強制 MFA**：組織可強制所有用戶啟用 MFA
3. **密碼驗證**：敏感操作需要密碼確認
4. **時區處理**：勿擾時間支援時區感知
5. **功能開關**：部分功能受遠端配置控制
