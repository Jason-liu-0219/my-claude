---
name: vsaas-auth
description: Use when working with authentication, login, logout, SSO, MFA, password reset, or user sessions. Triggers on keywords like "login", "logout", "authentication", "SSO", "MFA", "password", "session", "token".
version: 1.0.0
---

# VSaaS Portal - Auth 模組

認證模組負責用戶身份驗證、會話管理、SSO 單點登入和 MFA 多因素認證。

## 模組概述

### 功能範圍
- 標準郵箱/密碼登入
- SSO 單點登入
- MFA 多因素認證
- 密碼重置流程
- 用戶註冊
- 會話管理

---

## 目錄結構

```
src/pages/auth/
└── index.vue                          # IDP 認證處理頁面

src/pages/login/
├── index.vue                          # 登入頁面容器
├── route.js                           # 路由配置
└── components/
    ├── SignIn.vue                     # 標準登入
    ├── SignInSSO.vue                  # SSO 登入
    ├── VerifyMFA.vue                  # MFA 驗證
    ├── CreateAccount.vue              # 帳戶註冊
    ├── ForgotPassword.vue             # 忘記密碼
    ├── SetNewPassword.vue             # 設置新密碼
    ├── AccountCertificationRequired.vue # 帳戶激活
    ├── NoAssociatedOrganization.vue   # 無組織關聯
    ├── UserAgreement.vue              # 用戶協議
    └── CreateOrganizationDialogue.vue # 創建組織

src/store/
├── account/       # 帳戶狀態
├── mfa/           # MFA 狀態
├── singleSignOn/  # SSO 狀態
└── user/          # 用戶管理
```

---

## 路由結構

```
/login
├── ''                  → SignIn (標準登入)
├── /create-account     → CreateAccount (註冊)
├── /login-sso          → SignInSSO (SSO)
├── /verify-mfa         → VerifyMFA (MFA 驗證)
├── /forgot-password    → ForgotPassword
├── /set-new-password   → SetNewPassword
├── /activation-required → AccountCertificationRequired
├── /invitation-required → AccountCertificationRequired
├── /login/:token       → RemoteAccess (遠程訪問)
├── /new-to-portal      → NoAssociatedOrganization
└── /user-agreement     → UserAgreement
```

---

## 認證策略 (authStrategy)

| 策略 | 說明 |
|------|------|
| `sessionAuth` | 需要已認證用戶（默認） |
| `unauthOnly` | 僅允許未認證用戶 |
| `stepAuth` | 多步驟認證過程中 |
| `urlTokenAuth` | URL Token 認證 |
| `anyAccess` | 任何人可訪問 |

---

## Account Store

### State

```javascript
{
  attributes: {},           // 用戶屬性
  name: '',                 // 用戶名
  organizationRole: {},     // 組織角色
  isSigningIn: false,       // 登入中
  agreements: {}            // 用戶協議狀態
}
```

### 主要 Actions

| Action | 功能 |
|--------|------|
| `signIn` | 標準登入 |
| `signInSSO` | SSO 登入 |
| `signInMFA` | MFA 驗證 |
| `signUp` | 用戶註冊 |
| `forgotPassword` | 發送密碼重置郵件 |
| `confirmResetPassword` | 確認新密碼 |
| `completeNewPassword` | 完成新密碼設置 |
| `signOutToLogin` | 登出 |
| `onAuthSuccess` | 認證成功後初始化 |

---

## 登入流程

```
用戶輸入 → 驗證字段 → signIn() → 後端返回 nextStep
    ↓
根據 nextStep.signInStep：
├─ NEW_PASSWORD_REQUIRED → SetNewPassword
├─ CONFIRM_SIGN_UP → ActivationRequired
├─ SIGN_IN_WITH_TOTP_CODE → VerifyMFA
└─ 其他 → 重定向到 /group
```

---

## MFA 流程

### 啟用 MFA

```
登入成功 (新用戶) → /setup-mfa
    ↓
掃描二維碼 (getSetupUri)
    ↓
輸入 TOTP 碼 → enableUserMFA(code)
    ↓
驗證成功 → 返回首頁
```

### 驗證 MFA

```
登入返回 SIGN_IN_WITH_TOTP_CODE → /verify-mfa
    ↓
輸入 6 位 TOTP 碼 → signInMFA(totpCode)
    ↓
驗證成功 → 根據組織狀態重定向
```

---

## SSO 流程

```
輸入郵箱 → querySSOProvider(email)
    ↓
獲取 customProvider → signInSSO()
    ↓
重定向到 OAuth 提供商
    ↓
返回 sso.html (Cognito 處理)
    ↓
自動重定向到應用首頁
```

---

## 密碼驗證規則

```javascript
{
  charactersWithNoSpace: true,  // 無空格
  oneUppercase: true,           // 大寫字母
  oneLowercase: true,           // 小寫字母
  oneNumberic: true,            // 數字
  oneSymbol: true,              // 特殊符號
}
```

---

## 認證成功後初始化 (onAuthSuccess)

```javascript
1. fetchUserOrganizations()
2. setSubscribeOnLogin()
3. initIntegration()
4. getAll() (RemoteConfig)
5. getPlanPermission()
6. setStorageLocation()
7. setUserName()
8. fetchGroupList()
9. fetchListDeviceInfoOfOrganization()
10. getPermissionPolicies()
```

---

## 錯誤類型

| 錯誤 | 說明 |
|------|------|
| `PASSWORD_ATTEMPTS_EXCEEDED` | 嘗試次數超限 |
| `INCORRECT_USERNAME_OR_PASSWORD` | 郵箱或密碼錯誤 |
| `TEMPORARY_PASSWORD_EXPIRED` | 臨時密碼過期 |
| `USER_DOES_NOT_EXIST` | 用戶不存在 |
| `NETWORK_ERROR` | 網絡連接失敗 |
| `NO_ORGANIZATION_FOUND` | 未找到組織 |

---

## 外部依賴

- **AWS Cognito**：用戶認證和管理
- **@vivotek/lib-aws-amplify**：Amplify SDK 封裝
