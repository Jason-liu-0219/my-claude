---
name: vsaas-permission-guide
description: Use when need to find which permission controls a feature, understand permission system, or check permission usage. Triggers on keywords like "permission", "權限", "canDoDevice", "canDoOrg", "RBAC", "access control", "哪個權限".
version: 1.0.0
---

# VSaaS Portal - 權限查詢指南

此指南用於快速查找 VSaaS Portal 中的權限控制，包括哪個功能使用哪個權限、如何檢查權限等。

## 用途

當需要以下操作時使用此 skill：
- 查找某個功能/頁面使用哪個權限
- 理解權限系統架構
- 在代碼中添加權限檢查
- 除錯權限相關問題

---

## 🏗️ 權限系統架構

VSaaS Portal 使用 **三層權限系統**：

| 層級 | Manager | 用途 |
|------|---------|------|
| RBAC | `RbacPermissionManager` | 角色基礎存取控制（Casbin） |
| Plan | `PlanPermissionManager` | 方案功能限制（免費 vs 付費） |
| AI | `AIControlPermissionManager` | AI 功能控制和灰度發布 |

---

## 📋 權限範圍類型

### 1. 組織層級權限 (`PERMISSION_ORG_SCOPE`)

適用於整個組織的操作：

| 權限 Scope | 說明 | 使用場景 |
|------------|------|---------|
| `/org/all-users` | 所有人可存取 | 基本功能 |
| `/org/admin-restricted` | Admin 和 Owner 專用 | 系統設定 |
| `/org/user/read` | 讀取用戶列表 | 用戶管理 |
| `/org/user/update` | 更新用戶 | 用戶管理 |
| `/org/user/invite` | 邀請用戶 | 用戶管理 |
| `/org/user/delete` | 刪除用戶 | 用戶管理 |
| `/org/role/read` | 讀取角色 | 角色管理 |
| `/org/role/create` | 建立角色 | 角色管理 |
| `/org/role/update` | 更新角色 | 角色管理 |
| `/org/role/delete` | 刪除角色 | 角色管理 |
| `/org/role/assign` | 指派角色 | 角色管理 |
| `/org/system/alarm/*` | 警報管理 | 系統設定 |
| `/org/system/3rd-integration/*` | 第三方整合 | 系統設定 |
| `/org/ai/profile/*` | AI 設定檔管理 | AI 功能 |
| `/org/ai/case-vault/*` | AI Case Vault | AI 功能 |
| `/org/customized-view/send-copy` | 發送視圖副本 | 視圖功能 |

### 2. 設備層級權限 (`PERMISSION_DEVICE_SCOPE`)

適用於特定設備的操作：

| 權限 Scope | 說明 | 使用場景 |
|------------|------|---------|
| `/device/all-users` | 設備基本存取 | 基本功能 |
| `/device/live` | 即時串流 | 直播檢視 |
| `/device/playback` | 回放 | 歷史影像 |
| `/device/export-video` | 導出影片 | 影片導出 |
| `/device/archive/read` | 讀取存檔 | 存檔管理 |
| `/device/archive/create` | 建立存檔 | 存檔管理 |
| `/device/archive/play` | 播放存檔 | 存檔管理 |
| `/device/archive/download` | 下載存檔 | 存檔管理 |
| `/device/archive/delete` | 刪除存檔 | 存檔管理 |
| `/device/archive/share` | 分享存檔 | 存檔管理 |
| `/device/settings` | 設備設定 | 設備管理 |
| `/device/vca-settings` | VCA 設定 | AI 偵測 |
| `/device/talkdown` | 雙向語音 | 設備控制 |
| `/device/lock` | 鎖定設備 | 設備控制 |
| `/device/unlock` | 解鎖設備 | 設備控制 |
| `/device/share` | 分享設備 | 設備分享 |
| `/device/ai/search` | AI 搜尋 | AI 功能 |
| `/device/ai/event-insight` | 事件洞察 | AI 功能 |
| `/device/event/snooze-for-org` | 暫停規則 | 通知管理 |

---

## 🔍 快速查詢：頁面權限對照表

### 主要頁面權限

| 頁面 | Permission Key | 組織權限 | 設備權限 |
|------|---------------|---------|---------|
| **群組** | `group` | - | `/device/live` |
| **檢視** | `view` | - | `/device/live` |
| **設備管理** | `device` | - | - |
| **設備設定** | `device/setting` | - | `/device/settings` |
| **VCA 偵測** | `detection/vca-detection` | - | `/device/vca-settings` |
| **存檔** | `archive` | - | `/device/archive/read` |
| **訊息中心** | `notification` | - | `/device/all-users` |
| **樓層平面圖** | `floorplan` | - | - |

### 用戶和角色管理

| 頁面 | Permission Key | 組織權限 |
|------|---------------|---------|
| **用戶管理** | `user` | `/org/user/read`, `/org/role/read` |
| **角色管理** | `role` | `/org/role/read` |

### 系統設定

| 頁面 | Permission Key | 組織權限 |
|------|---------------|---------|
| **系統設定** | `system` | `/org/admin-restricted` |
| **組織設定** | `system/organization` | `/org/admin-restricted` |
| **授權資訊** | `system/license-information` | `/org/admin-restricted` |
| **警報管理** | `system/alarm-management` | `/org/system/alarm/*` |
| **第三方整合** | `system/third-party-integration` | `/org/system/3rd-party/*` |

### AI 功能

| 頁面 | Permission Key | 組織權限 | 設備權限 |
|------|---------------|---------|---------|
| **AI 搜尋** | `search` | - | `/device/ai/search`, `/device/ai/event-insight` |
| **設定檔搜尋** | `profile-search` | `/org/ai/profile/*` | - |

---

## 🛠️ 權限檢查方法

### 方法 1：在 Store 中檢查

```javascript
import { useStore } from 'vuex';
import { PERMISSION_DEVICE_SCOPE, PERMISSION_ORG_SCOPE } from '@/constants/Permission';

const store = useStore();

// 檢查組織權限
const canManageUsers = store.getters['permission/canDoOrg']({
  scope: PERMISSION_ORG_SCOPE.ORG_USER_READ
});

// 檢查設備權限
const canEditSettings = store.getters['permission/canDoDevice']({
  deviceId: 'camera123',
  scope: PERMISSION_DEVICE_SCOPE.DEVICE_SETTINGS
});
```

### 方法 2：使用 Composable

```javascript
// 建立 composable
const usePermissionHelper = (device) => {
  const store = useStore();
  const canDoDevice = store.getters['permission/canDoDevice'];

  const hasSettingsPermission = computed(() =>
    canDoDevice({
      deviceId: device.value.deviceId,
      scope: PERMISSION_DEVICE_SCOPE.DEVICE_SETTINGS
    })
  );

  return { hasSettingsPermission };
};
```

### 方法 3：在 Template 中使用

```vue
<template>
  <button v-if="hasSettingsPermission" @click="editSettings">
    編輯設定
  </button>
</template>

<script setup>
const { hasSettingsPermission } = usePermissionHelper(device);
</script>
```

### 方法 4：路由層級權限

```javascript
// 在 route.js 中定義
{
  path: 'settings/:deviceId',
  name: 'DeviceSettings',
  meta: {
    permission: PERMISSION_KEYS.DEVICE_SETTING
  }
}
```

---

## 🔎 如何找出某功能使用哪個權限

### Step 1：找到頁面/功能的路由

```bash
# 搜尋路由定義
grep -r "path.*settings" packages/app-vsaas-portal/src/pages/*/route.js
```

### Step 2：查看路由 meta.permission

```javascript
// 找到類似這樣的定義
meta: { permission: PERMISSION_KEYS.DEVICE_SETTING }
```

### Step 3：在 Page.js 中查詢映射

```bash
cat packages/app-vsaas-portal/src/constants/Page.js | grep -A5 "DEVICE_SETTING"
```

### Step 4：找到實際的 scope

```javascript
// 結果類似：
PAGE_PERMISSION[PERMISSION_KEYS.DEVICE_SETTING] = {
  orgScope: [],
  deviceScope: [PERMISSION_DEVICE_SCOPE.DEVICE_SETTINGS]
}
// 即 '/device/settings'
```

### Step 5：搜尋組件中的使用

```bash
# 搜尋權限使用
grep -r "DEVICE_SETTINGS" packages/app-vsaas-portal/src/pages/device/
```

---

## 📁 關鍵檔案位置

| 檔案 | 用途 |
|------|------|
| `packages/const-vsaas/Permission.js` | 基礎權限常數定義 |
| `packages/app-vsaas-portal/src/constants/Permission.js` | 應用層權限 Key 和依賴關係 |
| `packages/app-vsaas-portal/src/constants/Page.js` | 頁面到權限的映射 |
| `packages/app-vsaas-portal/src/models/Permission/RbacPermissionManager.js` | RBAC 權限檢查引擎 |
| `packages/app-vsaas-portal/src/store/permission/` | Vuex 權限 Store |
| `packages/app-vsaas-portal/src/router/RouterMiddleware.js` | 路由權限守衛 |

---

## 🎯 常見權限查詢場景

### 場景 1：「設備設定頁面需要什麼權限？」

```
答案：PERMISSION_DEVICE_SCOPE.DEVICE_SETTINGS
      即 '/device/settings'

使用：canDoDevice({ deviceId, scope: '/device/settings' })
```

### 場景 2：「用戶管理需要什麼權限？」

```
答案：PERMISSION_ORG_SCOPE.ORG_USER_READ 和 ORG_ROLE_READ
      即 '/org/user/read' 和 '/org/role/read'

使用：canDoOrg({ scope: '/org/user/read' })
```

### 場景 3：「存檔下載需要什麼權限？」

```
答案：PERMISSION_DEVICE_SCOPE.DEVICE_ARCHIVE_DOWNLOAD
      即 '/device/archive/download'

使用：canDoDevice({ deviceId, scope: '/device/archive/download' })
```

### 場景 4：「如何檢查用戶是否為 Admin？」

```javascript
const rbacPermissionManager = store.getters['permission/rbacPermissionManager'];
const role = rbacPermissionManager.getOrganizationRole();
// 返回: 'OWNER', 'ADMIN', 'USER' 等
```

---

## 🔗 權限依賴關係

某些權限有依賴關係：

### 設備權限依賴

```
播放存檔 (play-archive-video)
  └── 依賴：存檔影片 (archive-video)

匯出影片 (export-video)
  └── 依賴：回放 (playback)
```

### 組織權限依賴

```
編輯設定檔 (create-edit-profile)
  └── 不可同時有：觀看設定檔 (watch-profile)
```

---

## 🚀 快速指令

```bash
# 搜尋特定權限的使用位置
grep -r "DEVICE_SETTINGS" packages/app-vsaas-portal/src/

# 列出所有權限常數
grep "PERMISSION_" packages/const-vsaas/Permission.js

# 查看頁面權限映射
cat packages/app-vsaas-portal/src/constants/Page.js

# 搜尋 canDoDevice 使用
grep -r "canDoDevice" packages/app-vsaas-portal/src/pages/

# 搜尋 canDoOrg 使用
grep -r "canDoOrg" packages/app-vsaas-portal/src/pages/
```

---

## 📊 系統角色 ID

| 角色 | ID | 說明 |
|------|-----|------|
| Owner | `role-org-owner` | 組織擁有者 |
| Admin | `role-org-admin` | 管理員 |
| Member | `role-org-member` | 一般成員 |
| Device Supervisor | `role-device-supervisor` | 設備管理者 |
| Device Viewer | `role-device-viewer` | 設備檢視者 |

---

## ⚡ 權限檢查模式總結

| 模式 | 適用場景 | 代碼範例 |
|------|---------|---------|
| 路由 Meta | 整頁存取控制 | `meta: { permission: KEY }` |
| Store Getter | 組件邏輯中 | `canDoDevice({ deviceId, scope })` |
| Composable | 可複用邏輯 | `usePermissionHelper()` |
| Template | UI 條件渲染 | `v-if="hasPermission"` |

---

## 🔧 新增權限檢查的步驟

### 1. 確定需要的權限 Scope

```javascript
// 查看 packages/const-vsaas/Permission.js
// 或 packages/app-vsaas-portal/src/constants/Permission.js
```

### 2. 在組件中導入常數

```javascript
import { PERMISSION_DEVICE_SCOPE } from '@/constants/Permission';
```

### 3. 建立權限檢查

```javascript
const hasPermission = computed(() =>
  store.getters['permission/canDoDevice']({
    deviceId: props.deviceId,
    scope: PERMISSION_DEVICE_SCOPE.DEVICE_SETTINGS
  })
);
```

### 4. 在 Template 中使用

```vue
<button v-if="hasPermission">操作按鈕</button>
```

---

## 📝 注意事項

1. **設備 ID 格式**：支援 `deviceId` 或 `thingName:derivant` 格式
2. **權限緩存**：權限在登入時載入，登出時清除
3. **方案限制**：某些功能受方案限制（免費 vs 付費）
4. **灰度發布**：部分 AI 功能有 Dark Release 控制
5. **萬用字元**：Admin/Owner 使用 `/device/*/*` 存取所有設備
