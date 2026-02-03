---
name: vsaas-router
description: Use when working with routing, navigation guards, route middleware, authentication flow, permission checks, or page access control. Triggers on keywords like "router", "route", "middleware", "navigation guard", "beforeEach", "authStrategy", "路由", "導航守衛".
version: 1.0.0
---

# VSaaS Portal - Router 與 Middleware 模組

路由與中間件模組是 VSaaS Portal 的核心基礎設施，負責頁面導航、認證授權、權限檢查和功能開關控制。

## 模組概述

### 商業邏輯
- 統一的頁面導航和路由管理
- 多策略認證授權流程
- 三層權限檢查機制
- 功能開關與暗發佈控制
- 微前端動態路由加載

### 核心組件
| 組件 | 說明 |
|------|------|
| RouterMiddleware | 核心中間件類，處理所有路由決策 |
| Navigation Guards | Vue Router 生命週期鉤子 |
| Auth Strategy | 五種認證策略 |
| Feature Toggle | 功能開關系統 |
| Dark Release | 暗發佈功能控制 |

---

## 目錄結構

```
src/router/
├── index.js                           # 路由守衛和生命週期鉤子
├── router.js                          # Vue Router 實例和靜態路由定義
├── RouterMiddleware.js                # 核心中間件類
├── featureToggleRouteHelper.js        # 功能開關檢查
├── darkReleaseToggleRouteHelper.js    # 暗發佈功能檢查
└── dynamicVortexAIRouterLoader.js     # AI 模組動態路由加載

src/pages/{feature}/
├── route.js                           # 子路由定義
├── index.vue                          # 父級組件
└── {subpage}/index.vue                # 子頁面
```

---

## 認證策略類型

### AUTH_STRATEGY_TYPES

| 策略 | 說明 | 使用場景 |
|-----|------|--------|
| `SESSION_AUTH` | 需要有效會話 | 大多數受保護頁面 |
| `STEP_AUTH` | 分步驟認證 | 密碼重設、MFA 驗證 |
| `URL_TOKEN_AUTH` | URL 令牌認證 | 遠程訪問、共享鏈接 |
| `UNAUTH_ONLY` | 僅未認證用戶 | 登入、註冊頁面 |
| `ANY_ACCESS` | 無認證需求 | 設備共享、公開資源 |

### 策略使用示例

```javascript
// 需要登入的頁面（預設）
{
  path: 'management',
  name: 'DeviceManagement',
  meta: { permission: PERMISSION_KEYS.DEVICE }
}

// 僅未認證用戶
{
  path: '/login',
  name: 'Login',
  meta: { authStrategy: AUTH_STRATEGY_TYPES.UNAUTH_ONLY }
}

// 無認證需求
{
  path: '/shared-devices/:token',
  name: 'DeviceShare',
  meta: { authStrategy: AUTH_STRATEGY_TYPES.ANY_ACCESS }
}

// 分步驟認證
{
  path: '/setup-mfa',
  name: 'SetupMFA',
  meta: { authStrategy: AUTH_STRATEGY_TYPES.STEP_AUTH }
}
```

---

## 路由守衛系統

### 三個守衛鉤子

#### 1. beforeEach - 前置守衛
```javascript
// src/router/index.js
const beforeEach = (to, from, next) => {
  // 1. 檢查哈希重定向
  // 2. 設置路由加載狀態
  // 3. 加載登入頁面語言消息
  // 4. 加載微前端 AI 模組
  next();
};
```

#### 2. beforeResolve - 路由解析前
```javascript
const beforeResolve = async (to, from) => {
  // 1. 檢查路徑是否相同（避免重複導航）
  // 2. 調用 RouterMiddleware.decideRoute()
  // 3. 返回最終決定 (true/false/redirectPath)
  return await RouterMiddleware.decideRoute(to, from);
};
```

#### 3. afterEach - 後置守衛
```javascript
const afterEach = (to, from) => {
  // 1. 清除路由加載狀態
  // 2. 處理 shadcn-ui 頁面的 CSS 衝突
  // 3. 移除/恢復 v1-style 類
};
```

---

## RouterMiddleware 核心方法

### decideRoute - 路由決策入口

```javascript
async decideRoute(to, from) {
  // 1. 檢查 UI 骯髒狀態 (isDirty)
  if (this.isUiDirty) {
    return from; // 留在當前頁面
  }

  // 2. 檢查 forceRedirect 元標籤
  // 3. 根據 authStrategy 選擇認證策略
  switch (to.meta?.authStrategy) {
    case 'sessionAuth':
      return this.sessionLoginAuth(to, from);
    case 'unauthorizedOnly':
      return this.unAuthOnlyAuth(to, from);
    case 'anyAccess':
      return true;
    case 'urlTokenAuth':
      return this.urlTokenAuth(to, from);
    case 'stepAuth':
      return this.stepAuth(to, from);
    default:
      return this.sessionLoginAuth(to, from);
  }
}
```

### sessionLoginAuth - 會話認證流程

```
sessionLoginAuth(to, from)
  ├─> 檢查會話是否有效
  │   └─> 如果無效 → handleRedirectToLogin()
  ├─> setAuthenticateConnection()
  │   ├─> setToken(jwtToken)
  │   ├─> OdysseyConnection.initialize()
  │   └─> SipConnection.createSipConnection()
  ├─> setUserAttributes()
  └─> getSessionLoginPath(to, from)
      ├─> verifyUserAccess()
      ├─> verifyOrganizationAccess()
      ├─> verifyPageAccess()
      │   ├─> checkLicensePhase()
      │   ├─> checkPermission()
      │   ├─> checkDarkRelease()
      │   └─> checkFeatureToggle()
      └─> requestAllData()
```

### 關鍵檢查方法

| 方法 | 說明 |
|------|------|
| `checkPermission()` | 三層權限檢查 |
| `checkLicensePhase()` | 授權狀態檢查 |
| `checkDarkRelease()` | 暗發佈功能檢查 |
| `checkFeatureToggle()` | 功能開關檢查 |
| `verifyUserAccess()` | 用戶訪問驗證 |
| `verifyOrganizationAccess()` | 組織訪問驗證 |
| `checkEnableMFA()` | MFA 強制執行 |

---

## 權限檢查機制

### 三層權限模型

```javascript
checkPermission(to, from) {
  // 第一層：計劃更新權限（NVR 通道限制）
  const canPlanUpdate = store.getters['permission/canPlanUpdate'](to.path);

  // 第二層：計劃權限（功能可用性）
  const hasPlanPermission = store.getters['permission/hasPlanPermission'](to.path);

  // 第三層：頁面權限（角色權限）
  const hasPagePermission = store.getters['permission/hasPagePermission'](to.path);

  if (!canPlanUpdate || !hasPlanPermission || !hasPagePermission) {
    return this.findAccessibleParentRoute(to);
  }
  return true;
}
```

### PERMISSION_KEYS 常量

```javascript
PERMISSION_KEYS = {
  GROUP: 'group',
  DEVICE: 'device',
  DEVICE_SETTING: 'deviceSetting',
  USER: 'user',
  ARCHIVE: 'archive',
  MESSAGE_CENTER: 'messageCenter',
  SYSTEM: 'system',
  ALARM_MANAGEMENT: 'alarmManagement',
  AUDIT_LOG: 'auditLog',
  ROLE: 'role',
  LICENSE_INFORMATION: 'licenseInformation',
  SSO: 'sso',
  MFA: 'mfa',
  GLOBAL_DEVICE_SETTINGS: 'globalDeviceSettings',
  THIRD_PARTY_INTEGRATION: 'thirdPartyIntegration',
}
```

---

## 路由元標籤 (Meta)

### 常見元標籤

| 元標籤 | 類型 | 用途 |
|--------|------|------|
| `permission` | string | 頁面權限需求 |
| `authStrategy` | enum | 認證策略類型 |
| `requiresWizard` | boolean | 設備需要完成初始化 |
| `requiresOnline` | boolean | 設備需要在線 |
| `requiresSupport` | boolean | 檢查設備功能支持 |
| `licenseForbiddenPage` | boolean | 授權不足時禁止 |
| `freePlanCanAccess` | boolean | 免費計劃可訪問 |
| `requiresAllData` | string | 等待數據加載 |
| `layout` | string | 頁面佈局類型 |
| `hideNavigation` | boolean | 隱藏導航欄 |
| `shadcnUI` | boolean | 使用 shadcn-ui |
| `activeNav` | string | 激活導航項 |

### 元標籤使用示例

```javascript
{
  path: 'device-information',
  name: 'DeviceSettingsSystemInformation',
  component: () => import('./system/DeviceInformation/index.vue'),
  meta: {
    permission: PERMISSION_KEYS.DEVICE_SETTING,
    requiresWizard: true,
    requiresOnline: false,
    licenseForbiddenPage: true,
    freePlanCanAccess: true,
  },
}
```

---

## 功能開關系統

### Feature Toggle 映射

**位置**：`src/router/featureToggleRouteHelper.js`

```javascript
FEATURE_TOGGLE_SERVICES_MAP = {
  AccountAPIKeyManagement: [RemoteConfigService.API_KEY],
  SmartSensor: [RemoteConfigService.SMART_SENSOR],
  DeviceVideoStreamSettings: [RemoteConfigService.DEVICE_SETTINGS_VIDEO_STREAM],
}

async checkFeatureToggle(routeName, featureMap) {
  const services = featureMap[routeName];
  if (!services) return true;

  const results = await Promise.allSettled(
    services.map(key => RemoteConfigService.getConfiguration(key))
  );
  return results.every(r => r.status === 'fulfilled' && r.value === true);
}
```

### Dark Release 映射

**位置**：`src/router/darkReleaseToggleRouteHelper.js`

```javascript
DARK_RELEASE_FEATURE_MAP = {
  DeviceVideoStreamSettings: DARK_RELEASE_FEATURES.SETTING_H265,
  RecordingRetention: DARK_RELEASE_FEATURES.SETTING_RETENTION,
  DeviceSettingsTimelapse: DARK_RELEASE_FEATURES.SETTING_TIMELAPSE,
  FloorplansDefault: DARK_RELEASE_FEATURES.FLOOR_PLAN,
}

function checkDarkRelease(routeName, organizationSupportList, featureMap) {
  const services = featureMap[routeName];
  if (!services) return true;
  return organizationSupportList.includes(services);
}
```

---

## 路由定義模式

### 模式 1：簡單列表 + 詳情

```javascript
// src/pages/user/route.js
const UserRoute = [
  {
    path: 'management',
    name: 'UserManagement',
    component: () => import('./management/index.vue'),
    meta: { permission: PERMISSION_KEYS.USER },
  },
  {
    path: ':userEmail',
    name: 'UserDetail',
    component: () => import('./detail/index.vue'),
    meta: { permission: PERMISSION_KEYS.USER },
  },
  {
    path: ':pathMatch(.*)*',
    redirect: { name: 'UserManagement' },
  },
];
```

### 模式 2：多層嵌套

```javascript
// src/pages/device/settings/route.js
const SettingsRoute = [
  {
    path: 'system',
    name: 'DeviceSettingsSystem',
    redirect: { name: 'DeviceSettingsSystemInformation' },
    children: [
      {
        path: 'device-information',
        name: 'DeviceSettingsSystemInformation',
        meta: { requiresWizard: true, requiresOnline: false },
      },
      // ... 更多子路由
    ],
  },
];
```

### 模式 3：動態重定向

```javascript
// src/pages/system/route.js
{
  path: '',
  redirect: () => {
    const isFreePlan = window.store.getters['organization/isFreePlan'];
    if (!isFreePlan) {
      return { name: 'OrganizationDetails' };
    }
    return findFirstFreePlanRoute(SettingsRoute);
  },
}
```

### 模式 4：Props 函數

```javascript
{
  path: 'organization/:roleId',
  name: 'OrganizationRoleDetail',
  component: () => import('./Detail/index.vue'),
  props: (route) => ({
    roleType: PERMISSION_TYPES.ORGANIZATION,
    roleId: route.params.roleId,
  }),
}
```

### 模式 5：Reskin 處理

```javascript
{
  path: 'management',
  name: 'DeviceManagement',
  component: reskinRouterHandler(
    () => import('./management/index.vue'),
    () => import('./reskin/management/index.vue'),
  ),
}
```

---

## 微前端動態路由

**位置**：`src/router/dynamicVortexAIRouterLoader.js`

```javascript
export default function addVortexAIRoutesToDevice(router) {
  // 1. 在 DeviceSettingsDetection 下追加 AI 路由
  const detectionRoute = SettingsRoute.find(
    route => route.name === 'DeviceSettingsDetection'
  );
  if (detectionRoute && detectionRoute.children) {
    detectionRoute.children.push(...VSAAS_AI_ROUTE_SETTINGS);
    detectionRoute.children.forEach(child => {
      router.addRoute('DeviceSettingsDetection', child);
    });
  }

  // 2. 在 Device 路由下追加 AI 路由
  VSAAS_AI_ROUTE_DEVICE.forEach(route => {
    router.addRoute('Device', route);
  });
}
```

---

## 主要路由統計

### 頂級路由（18 個）

| 路徑 | 名稱 | 權限 |
|-----|------|------|
| `/` | root | - |
| `/login` | Login | UNAUTH_ONLY |
| `/device/` | Device | DEVICE |
| `/user/` | User | USER |
| `/system` | System | SYSTEM |
| `/group` | Group | GROUP |
| `/view` | View | - |
| `/account` | Account | - |
| `/archive/` | Archive | ARCHIVE |
| `/notification/` | Notification | MESSAGE_CENTER |
| `/floorplans` | Floorplans | - |
| `/role/` | Role | ROLE |
| `/alarm` | Alarm | - |
| `/timelapse` | Timelapse | - |
| `/upgrade-plan` | UpgradePlan | - |
| `/shared-devices/:token` | DeviceShare | ANY_ACCESS |
| `/nvr-forgot-password` | NVRForgotPassword | ANY_ACCESS |
| `/:pathMatch(.*)*` | - | - |

### 子路由文件（16 個）

| 文件 | 路由數 |
|------|--------|
| device/settings/route.js | 20+ |
| system/route.js | 13 |
| login/route.js | 11 |
| account/route.js | 6 |
| role/route.js | 4 |
| notification/route.js | 4 |
| device/route.js | 4 |
| floorplan/route.js | 3 |
| view/route.js | 3 |
| device/AddDevice/route.js | 3 |
| user/route.js | 2 |
| alarm/route.js | 2 |
| NVRForgotPassword/route.js | 2 |
| archive/route.js | 1 |
| Timelapse/route.js | 1 |

---

## 常見開發任務

### 1. 新增路由

```bash
# 1. 建立頁面目錄和組件
mkdir -p src/pages/myFeature
touch src/pages/myFeature/index.vue
touch src/pages/myFeature/route.js

# 2. 定義子路由
# src/pages/myFeature/route.js
export default [
  {
    path: 'list',
    name: 'MyFeatureList',
    component: () => import('./list/index.vue'),
    meta: { permission: PERMISSION_KEYS.MY_FEATURE },
  },
];

# 3. 在 router.js 中註冊
# src/router/router.js
import MyFeatureRoute from '@/pages/myFeature/route.js';

{
  path: '/my-feature',
  name: 'MyFeature',
  component: () => import('@/pages/myFeature/index.vue'),
  children: MyFeatureRoute,
}
```

### 2. 新增功能開關

```javascript
// 1. 在 featureToggleRouteHelper.js 添加映射
FEATURE_TOGGLE_SERVICES_MAP = {
  ...existing,
  MyNewFeature: [RemoteConfigService.MY_NEW_FEATURE],
};

// 2. 在路由中使用該名稱
{
  path: 'new-feature',
  name: 'MyNewFeature',  // 必須匹配映射表中的 key
  component: () => import('./NewFeature/index.vue'),
}
```

### 3. 新增暗發佈控制

```javascript
// 1. 在 darkReleaseToggleRouteHelper.js 添加映射
DARK_RELEASE_FEATURE_MAP = {
  ...existing,
  MyDarkFeature: DARK_RELEASE_FEATURES.MY_DARK_FEATURE,
};

// 2. 確保組織的 supportList 包含該功能
```

### 4. 除錯路由問題

```javascript
// 在 RouterMiddleware.js 中添加日誌
console.log('decideRoute:', { to, from });
console.log('authStrategy:', to.meta?.authStrategy);
console.log('permission check:', {
  canPlanUpdate: store.getters['permission/canPlanUpdate'](to.path),
  hasPlanPermission: store.getters['permission/hasPlanPermission'](to.path),
  hasPagePermission: store.getters['permission/hasPagePermission'](to.path),
});
```

---

## 特殊處理

### UI 骯髒狀態

```javascript
// 防止用戶在未保存編輯時離開頁面
if (this.isUiDirty) {
  await this.store.dispatch('ui/setRouterLeaveGuideTarget', to.path);
  return from; // 留在當前頁面
}
```

### 防止循環重定向

```javascript
wouldCauseLoop(parentRoute, childRoute) {
  const { redirect } = parentRoute;
  if (redirect.name) {
    return childRoute.matched.some(r => r.name === redirect.name);
  }
  if (redirect.path) {
    return childRoute.path.startsWith(redirect.path);
  }
  return false;
}
```

### 網絡恢復機制

```javascript
// 當用戶從離線恢復在線時
window.addEventListener('online', async () => {
  try {
    await store.dispatch('account/getAuthTokenStatus');
  } catch (error) {
    router.push(await handleRedirectToLogin(currentRoute));
  }
});
```

---

## 關鍵文件位置

| 文件 | 用途 |
|------|------|
| `src/router/router.js` | Vue Router 實例和靜態路由 |
| `src/router/index.js` | 路由守衛定義 |
| `src/router/RouterMiddleware.js` | 核心中間件邏輯 |
| `src/router/featureToggleRouteHelper.js` | 功能開關檢查 |
| `src/router/darkReleaseToggleRouteHelper.js` | 暗發佈檢查 |
| `src/router/dynamicVortexAIRouterLoader.js` | AI 模組動態路由 |
| `src/constants/Permission.js` | 權限常量定義 |
| `src/constants/Page.js` | 頁面權限映射 |

---

## 注意事項

1. **認證策略**：未指定時預設使用 `SESSION_AUTH`
2. **權限檢查順序**：Plan Update → Plan Permission → Page Permission
3. **路由回退**：權限不足時自動查找可訪問的父路由
4. **MFA 強制**：組織可強制要求所有用戶啟用 MFA
5. **動態路由**：AI 模組路由在應用啟動時動態加載
6. **CSS 衝突**：shadcn-ui 頁面需要特殊處理 v1-style 類
7. **骯髒狀態**：離開編輯頁面前會檢查未保存的更改
