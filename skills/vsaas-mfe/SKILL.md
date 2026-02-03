---
name: vsaas-mfe
description: VSaaS Portal 微前端架構指南。觸發詞：「微前端」、「MFE」、「module federation」、「vortexai」、「AI 模組載入」、「遠端模組」
version: 1.0.0
---

# VSaaS Portal 微前端架構

VSaaS Portal 使用 Vite Module Federation 實現微前端架構，主應用 (host-app) 動態載入 AI 模組 (remote-vortexai)。

---

## 架構概覽

```
┌─────────────────────────────────────────────────────────────┐
│                    VSaaS Portal (host-app)                   │
│                      localhost:8080                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Vuex      │  │   Router    │  │ MicrofrontendManager│  │
│  │   Store     │  │             │  │                     │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
│         │                │                     │             │
│         └────────────────┼─────────────────────┘             │
│                          │                                   │
│                    import('vortexai/...')                    │
│                          │                                   │
└──────────────────────────┼───────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  VSaaS AI (remote-vortexai)                  │
│                      localhost:8681                          │
├─────────────────────────────────────────────────────────────┤
│  Exposes:                                                    │
│  ├── SearchModule          (Vuex Module)                     │
│  ├── ProfileSearchModule   (Vuex Module)                     │
│  ├── CaseVaultModule       (Vuex Module)                     │
│  ├── VCASettingsModule     (Vuex Module)                     │
│  ├── EventSearchModule     (Vuex Module)                     │
│  ├── VSaasAIRoutes         (Router Config)                   │
│  ├── VCARuleSettingsPanel  (Vue Component)                   │
│  ├── ExcludedAreaPanel     (Vue Component)                   │
│  ├── FaceAlignService      (Service)                         │
│  └── AppVersion            (Version Info)                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 關鍵檔案位置

| 檔案 | 說明 |
|------|------|
| `packages/app-vsaas-portal/vite.config.js` | 主應用 Module Federation 配置 |
| `packages/app-vsaas-ai/vite.config.js` | AI 模組 Module Federation 配置 |
| `packages/app-vsaas-portal/src/MicrofrontendManager.js` | 微前端載入管理器 |
| `packages/app-vsaas-portal/src/router/dynamicVortexAIRouterLoader.js` | 動態路由載入 |
| `packages/app-vsaas-portal/src/router/index.js` | 路由守衛（按需載入） |
| `packages/app-vsaas-portal/.env.dev` | 開發環境配置 |
| `packages/app-vsaas-ai/.env.dev` | AI 模組開發環境配置 |

---

## 開發模式

### 方式一：連接遠端 AI 模組（推薦日常開發）

```bash
# 只啟動 Portal，連接遠端 AI 模組
cd packages/app-vsaas-portal
pnpm dev
```

預設連接 `https://mfe-ai.dev.vortexcloud.com`

### 方式二：本地同時開發兩個模組

```bash
# 終端 1：啟動 AI 模組 (port 8681)
cd packages/app-vsaas-ai
pnpm dev

# 終端 2：啟動 Portal 微前端測試模式 (port 8080)
cd packages/app-vsaas-portal
pnpm dev:mfe
```

`pnpm dev:mfe` 會設定 `TEST=true`，載入本地 AI 模組。

### 方式三：使用建構產物測試

```bash
# 終端 1：建構並預覽 AI 模組
cd packages/app-vsaas-ai
pnpm build && pnpm serve  # port 8681

# 終端 2：建構並預覽 Portal
cd packages/app-vsaas-portal
VITE_MFE_REMOTE_AI_ORIGIN=http://localhost:8681 pnpm preview
```

---

## 常用指令

### Portal (主應用)

| 指令 | 說明 |
|------|------|
| `pnpm dev` | 開發模式（連接遠端 AI） |
| `pnpm dev:mfe` | 微前端測試模式（連接本地 AI） |
| `pnpm build` | 生產建構 |
| `pnpm build:mfe:test` | 測試建構（TEST=true） |
| `pnpm test:unit` | 單元測試 |
| `pnpm preview` | 預覽建構結果 |

### AI 模組 (遠端模組)

| 指令 | 說明 |
|------|------|
| `pnpm dev` | 開發模式 (port 8080) |
| `pnpm serve` | 預覽模式 (port 8681) |
| `pnpm build` | 生產建構 |
| `pnpm build:mfe:test` | 測試建構（TEST=true） |
| `pnpm test:unit` | 單元測試 |

---

## Module Federation 配置

### 主應用配置 (vsaas-portal)

```javascript
// vite.config.js
federation({
  name: 'host-app',
  filename: 'remoteEntry.js',
  remotes: {
    vortexai: mfeConfig.REMOTE_AI_PATH,  // 動態決定路徑
  },
  shared: generateSharedList(),
})
```

### 遠端模組配置 (vsaas-ai)

```javascript
// vite.config.js
federation({
  name: 'remote-vortexai',
  filename: 'remoteEntry.js',
  exposes: {
    './SearchModule': './src/modules/search/index.js',
    './ProfileSearchModule': './src/modules/profileSearch/index.js',
    './CaseVaultModule': './src/modules/caseVault/index.js',
    './VCASettingsModule': './src/modules/vcaSettings/index.js',
    './EventSearchModule': './src/modules/eventSearch/index.js',
    './FaceAlignService': './src/models/FaceAlignService.js',
    './VCARuleSettingsPanel': './src/components/VCARuleSettingsPanel.vue',
    './VSaasAIRoutes': './src/routes/index.js',
    './AppVersion': './src/AppVersion.js',
    // ... 更多組件
  },
  shared: mfeConfig.SHARED_LIST,
})
```

---

## 共享依賴

兩個應用共享以下依賴，避免重複載入：

```javascript
const sharedModules = [
  // Vue 生態系
  'vue',
  'vuex', 
  'vue-router',
  
  // VIVOTEK 內部套件
  '@vivotek/advanced-vue-components',
  '@vivotek/generic-vue-components',
  '@vivotek/vsaas-vue-components',
  '@vivotek/const-vsaas',
  '@vivotek/device',
  '@vivotek/context',
  
  // 串流相關
  '@vivotek/lib-rtsp-player',
  '@vivotek/lib-webrtc-odyssey',
  '@vivotek/lib-hls-player',
  
  // 其他
  '@vivotek/lib-utility',
  '@vivotek/lib-aws-amplify',
  '@vivotek/theme-color',
  // ...
];
```

---

## 載入機制

### MicrofrontendManager

```javascript
// src/MicrofrontendManager.js
class MicrofrontendManager {
  loaded = false;
  loading = false;
  
  async load(store, router) {
    if (this.loaded) return;
    
    this.loading = true;
    
    // 並行載入所有模組
    const [search, profileSearch, caseVault, vcaSettings, eventSearch, info] = 
      await Promise.all([
        import('vortexai/SearchModule'),
        import('vortexai/ProfileSearchModule'),
        import('vortexai/CaseVaultModule'),
        import('vortexai/VCASettingsModule'),
        import('vortexai/EventSearchModule'),
        import('vortexai/AppVersion'),
      ]);

    // 註冊 Vuex 模組
    loadModule(search.default, store, router);
    loadModule(profileSearch.default, store, router);
    // ...

    // 註冊動態路由
    addVortexAIRoutesToDevice(router);
    
    this.loaded = true;
    this.loading = false;
  }
}
```

### 路由守衛按需載入

```javascript
// src/router/index.js
const beforeEach = (to, from, next) => {
  // 當路由包含 AI 功能且尚未載入時
  if (
    to?.redirectedFrom &&
    (to?.redirectedFrom.path.includes('/search') ||
      to?.redirectedFrom.path.includes('/profile-search') ||
      to?.redirectedFrom.path.includes('/detection/vca-detection')) &&
    !MicrofrontendManager.loaded
  ) {
    const toPath = to?.redirectedFrom.path;
    MicrofrontendManager.load(store, router).then(() => {
      router.replace({ path: toPath });
    });
    return;
  }
  next();
};
```

---

## 環境變數

### Portal (.env.dev)

```bash
VITE_MFE_REMOTE_AI_ORIGIN=https://mfe-ai.dev.vortexcloud.com
```

### Portal (.env.prod)

```bash
VITE_MFE_REMOTE_AI_ORIGIN=https://mfe-ai.vortexcloud.com
VITE_MFE_HOST_ORIGIN=https://portal.vortexcloud.com
```

### AI 模組 (.env.dev)

```bash
VITE_MFE_HOST_ORIGIN=https://portal.dev.vortexcloud.com
```

---

## 測試配置

### 單元測試 Mock

Portal 測試時需要 mock AI 模組：

```javascript
// vitest.config.js 或測試設定
alias: {
  'vortexai/VCARuleSettingsPanel': './mock/MockRemoteComponent.vue',
  'vortexai/FaceProfileDialogue': './mock/MockRemoteComponent.vue',
  'vortexai/FaceAlignService': './mock/MockService.js',
  'vortexai/VSaasAIRoutes': './mock/MockVSaasAIRoute.js',
}
```

### 測試建構模式

```bash
# Portal 測試建構（載入本地 AI 模組）
cd packages/app-vsaas-portal
pnpm build:mfe:test

# AI 測試建構
cd packages/app-vsaas-ai
pnpm build:mfe:test
```

---

## 常見問題排查

### 問題 1：AI 模組載入失敗

**症狀**：Console 出現 `Failed to fetch dynamically imported module: vortexai/...`

**排查步驟**：

1. 確認 AI 模組伺服器是否啟動
   ```bash
   curl http://localhost:8681/assets/remoteEntry.js
   ```

2. 檢查 CORS 設定
   ```bash
   # 應該要有正確的 CORS header
   curl -I http://localhost:8681/assets/remoteEntry.js
   ```

3. 確認環境變數
   ```bash
   # 在 Portal 目錄
   cat .env.dev | grep MFE
   ```

### 問題 2：共享依賴版本不一致

**症狀**：執行時出現多個 Vue 實例或 Vuex store 問題

**解決方式**：

1. 確保兩個專案的共享依賴版本一致
   ```bash
   # 檢查版本
   cd packages/app-vsaas-portal && pnpm list vue vuex vue-router
   cd packages/app-vsaas-ai && pnpm list vue vuex vue-router
   ```

2. 檢查 `vite.config.js` 中的 shared 配置

### 問題 3：熱更新不生效

**症狀**：修改 AI 模組後，Portal 沒有更新

**解決方式**：

1. 微前端開發模式下，需要手動重新整理頁面
2. 或重啟 Portal 開發伺服器

### 問題 4：路由找不到

**症狀**：跳轉到 AI 功能頁面時出現 404

**排查步驟**：

1. 確認 MicrofrontendManager 是否已載入
   ```javascript
   console.log(MicrofrontendManager.loaded);
   ```

2. 檢查動態路由是否註冊
   ```javascript
   console.log(router.getRoutes());
   ```

### 問題 5：本地開發連不上

**症狀**：`pnpm dev:mfe` 無法連接本地 AI 模組

**解決方式**：

1. 確保先啟動 AI 模組
   ```bash
   cd packages/app-vsaas-ai
   pnpm dev  # 先啟動這個
   ```

2. 確保 AI 模組已建構（如果使用 TEST=true）
   ```bash
   cd packages/app-vsaas-ai
   pnpm build
   ```

---

## 新增遠端模組步驟

如果需要在 AI 模組中暴露新的模組或組件：

### 1. 在 AI 模組中暴露

```javascript
// packages/app-vsaas-ai/vite.config.js
exposes: {
  // 新增這行
  './NewModule': './src/modules/newModule/index.js',
}
```

### 2. 在 Portal 中使用

```javascript
// 動態載入
const newModule = await import('vortexai/NewModule');

// 或在 MicrofrontendManager 中載入
// src/MicrofrontendManager.js
const [search, ..., newModule] = await Promise.all([
  import('vortexai/SearchModule'),
  // ...
  import('vortexai/NewModule'),
]);
```

### 3. 更新測試 Mock

```javascript
// vitest 配置
alias: {
  'vortexai/NewModule': './mock/MockNewModule.js',
}
```

---

## 版本追蹤

AI 模組透過 `AppVersion` 暴露版本資訊：

```javascript
// 在 Portal 中取得 AI 模組版本
const info = await import('vortexai/AppVersion');
console.log(info.default);  // AI 模組版本
```

MicrofrontendManager 會在載入後記錄版本：

```javascript
MicrofrontendManager.aiVersion  // 取得 AI 模組版本
```

---

## 相關文件

- [Vite Module Federation Plugin](https://github.com/nicholaslee119/vite-plugin-federation)
- [Module Federation 官方文件](https://webpack.js.org/concepts/module-federation/)
