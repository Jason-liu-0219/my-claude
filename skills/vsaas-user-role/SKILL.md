---
name: vsaas-user-role
description: Use when working with user management, role management, permissions, RBAC, or access control policies. Triggers on keywords like "user", "role", "permission", "RBAC", "Casbin", "access control", "organization role", "resource role".
version: 1.0.0
---

# VSaaS Portal - User & Role 模組

使用者與角色管理模組實作 RBAC（Role-Based Access Control）權限架構，支援組織級和資源級兩種角色類型。

## 模組概述

### 權限架構
```
使用者 (User)
  └── 指派角色 (Role)
        ├── 組織角色 (Organization Role)
        │     └── 系統功能權限
        └── 資源角色 (Resource Role)
              └── 設備/群組操作權限
```

### 角色類型

| 類型 | 說明 | 範例 |
|------|------|------|
| Organization Role | 組織層級權限 | Admin, Operator, Viewer |
| Resource Role | 資源層級權限 | Device Manager, Group Viewer |

---

## 路由結構

### User 使用者管理
```
/user/
├── management            # 使用者列表
└── :userEmail            # 使用者詳情
```

### Role 角色管理
```
/role/
├── organization          # 組織角色列表
├── resource              # 資源角色列表
├── organization/:roleId  # 組織角色詳情
└── resource/:roleId      # 資源角色詳情
```

---

## 目錄結構

### User 模組
```
src/pages/user/
├── index.vue             # 路由容器
├── route.js              # 路由定義
├── management/           # 使用者列表
│   ├── index.vue
│   └── components/
│       └── UserManagementTable.vue
└── UserDetail/           # 使用者詳情
    ├── index.vue
    └── components/
```

### Role 模組
```
src/pages/role/
├── index.vue             # 路由容器
├── route.js              # 路由定義
├── OrganizationRole/     # 組織角色
│   ├── index.vue         # 列表
│   └── RoleDetail.vue    # 詳情
├── ResourceRole/         # 資源角色
│   ├── index.vue         # 列表
│   └── RoleDetail.vue    # 詳情
└── components/           # 共用組件
    ├── RolePermissionTree.vue
    └── ResourceSelector.vue
```

---

## Store 依賴

### user 模組
**位置**：`src/store/user/`

```javascript
state: {
  userList: [],
  currentUser: null,
  isLoading: false,
}
```

| Action | 說明 |
|--------|------|
| `fetchUsers` | 取得使用者列表 |
| `fetchUserDetail` | 取得使用者詳情 |
| `inviteUser` | 邀請新使用者 |
| `updateUser` | 更新使用者資訊 |
| `deleteUser` | 刪除使用者 |
| `assignRole` | 指派角色 |

### role 模組
**位置**：`src/store/role/`

```javascript
state: {
  organizationRoles: [],
  resourceRoles: [],
  currentRole: null,
}
```

| Action | 說明 |
|--------|------|
| `fetchOrganizationRoles` | 取得組織角色列表 |
| `fetchResourceRoles` | 取得資源角色列表 |
| `createRole` | 建立新角色 |
| `updateRole` | 更新角色 |
| `deleteRole` | 刪除角色 |
| `cloneRole` | 複製角色 |

### permission 模組
**位置**：`src/store/permission/`

```javascript
state: {
  permissions: [],        // 權限定義
  userPermissions: [],    // 當前使用者權限
}
```

| Action | 說明 |
|--------|------|
| `fetchPermissions` | 取得所有權限定義 |
| `checkPermission` | 檢查權限 |

### accessControl 模組
**位置**：`src/store/accessControl/`

用於 Casbin 政策管理。

---

## 權限定義

### 權限結構
```javascript
// src/constants/Permission.js
export const PERMISSION_KEYS = {
  // 設備相關
  DEVICE: 'device',
  DEVICE_SETTING: 'device/setting',
  
  // 系統相關
  SYSTEM: 'system',
  SYSTEM_ORGANIZATION: 'system/organization',
  
  // 使用者相關
  USER: 'user',
  ROLE: 'role',
  
  // 功能相關
  ARCHIVE: 'archive',
  NOTIFICATION: 'notification',
  // ...
}
```

### 權限檢查 Composable
```javascript
// src/composables/usePermission/usePermission.js
import { usePermission } from '@/composables/usePermission'

const { hasPermission, checkPermission } = usePermission()

// 檢查單一權限
if (hasPermission('device/setting')) {
  // 有權限
}

// 檢查多個權限（任一）
if (checkPermission(['device', 'system'], 'any')) {
  // 有任一權限
}
```

---

## Casbin 整合

VSaaS Portal 使用 Casbin 作為權限引擎：

### 政策模型
```
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
```

### 相關檔案
- `src/models/Permission/CasbinAdapter.js` - Casbin 適配器
- `src/store/accessControl/` - 政策管理

---

## Composables

### usePermission
```javascript
// src/composables/usePermission/usePermission.js
export function usePermission() {
  return {
    hasPermission,      // 檢查單一權限
    checkPermission,    // 檢查多個權限
    getUserPermissions, // 取得使用者權限
  }
}
```

### usePermissionHierarchy
```javascript
// src/composables/usePermission/usePermissionHierarchy.js
export function usePermissionHierarchy() {
  return {
    getPermissionTree,   // 取得權限樹狀結構
    flattenPermissions,  // 扁平化權限
  }
}
```

### useCopyRole
```javascript
// src/composables/usePermission/useCopyRole.js
export function useCopyRole() {
  return {
    copyRole,            // 複製角色
    validateRoleName,    // 驗證角色名稱
  }
}
```

---

## 常見開發任務

### 1. 新增權限定義
```javascript
// 1. 在 constants/Permission.js 新增
export const PERMISSION_KEYS = {
  // ...
  NEW_FEATURE: 'new-feature',
}

// 2. 在路由 meta 中使用
{
  path: 'new-feature',
  meta: {
    permission: PERMISSION_KEYS.NEW_FEATURE
  }
}
```

### 2. 新增角色類型
```bash
# 1. 建立角色頁面
src/pages/role/NewRoleType/

# 2. 更新 Store
src/store/role/actions.js

# 3. 更新路由
src/pages/role/route.js
```

### 3. 權限檢查
```vue
<template>
  <!-- 使用 v-if 檢查權限 -->
  <button v-if="hasPermission('device/setting')">
    編輯設備
  </button>
</template>

<script setup>
import { usePermission } from '@/composables/usePermission'
const { hasPermission } = usePermission()
</script>
```

### 4. 邀請新使用者流程
```javascript
// 1. 輸入 Email
// 2. 選擇組織角色
// 3. 選擇資源角色（可選）
// 4. 呼叫 user/inviteUser action
// 5. 系統發送邀請郵件
```

---

## 注意事項

1. **角色繼承**：組織角色權限 > 資源角色權限
2. **預設角色**：系統預設角色不可刪除
3. **權限快取**：使用者權限會快取，變更後需重新登入
4. **資源範圍**：資源角色需指定適用的設備/群組

---

## 相關工具

### RoleHelper
```javascript
// src/utils/RoleHelper.js
import { 
  getPermissionLabel,
  validateRolePermissions,
  formatRoleForAPI,
} from '@/utils/RoleHelper'
```

---

## 相關資源

- `references/routes.md` - 完整路由結構
- `references/store.md` - Store 詳解
- `references/permission.md` - 權限系統詳解
