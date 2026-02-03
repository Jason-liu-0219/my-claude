---
name: vsaas-architecture
description: Use when working with VSaaS Portal architecture, understanding project structure, routing patterns, state management, or microfrontend setup. Triggers on keywords like "vsaas", "portal architecture", "monorepo structure", "module federation".
version: 1.0.0
---

# VSaaS Portal - 整體架構

VSaaS Portal 是一個企業級 Vue 3 視頻監控管理平台，採用微前端架構和模組化設計。

## 技術棧

| 類別 | 技術 |
|------|------|
| 框架 | Vue 3.5 + Composition API |
| 路由 | Vue Router 4 |
| 狀態管理 | Vuex 4（26 個模組） |
| 構建工具 | Vite + Module Federation |
| 樣式 | LESS + Tailwind CSS |
| API | GraphQL (Apollo) + REST |
| 即時通訊 | WebSocket + SIP.js |

## 目錄結構

```
packages/app-vsaas-portal/
├── src/
│   ├── pages/              # 頁面模組（按路由分）
│   ├── components/         # 共用組件（29 類）
│   ├── store/              # Vuex 模組（26 個）
│   ├── composables/        # 組合式函數（13 個）
│   ├── models/             # 資料模型與 API 服務
│   ├── router/             # 路由配置
│   ├── layouts/            # 佈局組件
│   ├── constants/          # 常數定義
│   ├── utils/              # 工具函數
│   └── graphql/            # GraphQL 查詢
├── public/
└── vite.config.js
```

## 路由模組

| 路由 | 模組 | 說明 |
|------|------|------|
| `/device` | Device | 設備管理（最複雜） |
| `/system` | System | 系統設定 |
| `/account` | Account | 帳戶設定 |
| `/group` | Group | 群組管理 |
| `/view` | View | 自訂檢視 |
| `/user` | User | 使用者管理 |
| `/role` | Role | 角色管理 |
| `/archive` | Archive | 存檔管理 |
| `/notification` | Notification | 訊息中心 |
| `/floorplans` | Floorplan | 樓層平面圖 |
| `/alarm` | Alarm | 告警 |
| `/login` | Auth | 認證流程 |

## 微前端架構

使用 Vite Module Federation 動態載入 AI 模組：

```javascript
// MicrofrontendManager.js
const modules = {
  SearchModule,        // /search
  ProfileSearchModule, // /profile-search
  VCASettingsModule,   // VCA 設定
  EventSearchModule,   // 事件搜尋
}
```

**動態路由載入位置**：`src/router/dynamicVortexAIRouterLoader.js`

## Store 模組架構

### 核心模組
- `device` - 設備資料（deviceMap, nvrMap, bridgeMap）
- `organization` - 組織設定
- `permission` - 權限管理
- `ui` - UI 狀態（loading, error）

### 功能模組
- `account`, `user`, `role` - 使用者相關
- `archive`, `notification`, `alarm` - 功能模組
- `group`, `view`, `floorPlan` - 檢視相關

### Store 標準結構
```
store/{module}/
├── index.js       # 模組配置
├── actions.js     # 非同步操作
├── mutations.js   # 同步更新
├── getters.js     # 狀態選擇器
└── *.test.js      # 測試
```

## 組件庫

| 套件 | 數量 | 用途 |
|------|------|------|
| generic-vue-components | 48 | 通用 UI 元件 |
| advanced-vue-components | 56 | 業務組件（設備、播放器） |
| vsaas-vue-components | 17 | VSaaS 專用樣式 |
| shadcn-vue-components | 30+ | 現代設計系統 |

## 權限系統

採用 RBAC + Casbin：

```javascript
// constants/Permission.js
PERMISSION_KEYS = {
  DEVICE: 'device',
  DEVICE_SETTING: 'device/setting',
  USER: 'user',
  ROLE: 'role',
  // ...
}
```

**路由守衛**：`src/router/RouterMiddleware.js`

## 關鍵設計模式

### 1. Reskin 雙版本
部分頁面有新舊兩版 UI，使用 `reskinRouterHandler` 切換：
```
pages/device/management/       # 舊版
pages/device/reskin/management/ # 新版
```

### 2. 頁面模組結構
```
pages/{module}/
├── index.vue      # 頁面容器
├── route.js       # 路由定義
├── components/    # 模組組件
└── composables/   # 模組邏輯
```

### 3. API 服務層
```
models/API/
├── DeviceAPI.js
├── UserAPI.js
└── ...
```

## 開發常用指令

```bash
cd packages/app-vsaas-portal
pnpm dev          # 開發伺服器 http://localhost:8080
pnpm dev:mfe      # 微前端模式
pnpm build        # 生產構建
pnpm test:unit    # 單元測試
pnpm lint         # ESLint
```

## 相關 Skill

- `vsaas-components` - 組件庫詳細使用指南
- `vsaas-device` - 設備模組詳解
- `vsaas-system` - 系統設定詳解
- `vsaas-user-role` - 使用者與角色詳解
