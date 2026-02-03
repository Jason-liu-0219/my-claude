---
name: vsaas-components
description: Use when working with VSaaS component libraries, importing components, understanding component APIs, or choosing the right component. Triggers on keywords like "component", "AppButton", "VsaasTable", "generic-vue", "advanced-vue", "shadcn".
version: 1.0.0
---

# VSaaS Portal - 組件庫指南

VSaaS Portal 使用四個內部組件庫，各有不同定位和用途。

## 組件庫概覽

| 套件 | 用途 | 數量 |
|------|------|------|
| `@vivotek/generic-vue-components` | 通用 UI 元件 | 48 |
| `@vivotek/advanced-vue-components` | 業務組件（設備、播放器） | 56 |
| `@vivotek/vsaas-vue-components` | VSaaS 專用輕量組件 | 17 |
| `@vivotek/shadcn-vue-components` | 現代設計系統 | 30+ |

---

## 1. generic-vue-components

**位置**：`packages/generic-vue-components/`
**定位**：通用 UI 元件，不含業務邏輯

### 常用組件

| 組件 | 用途 |
|------|------|
| `AppButton` | 按鈕 |
| `AppDialogue` | 對話框 |
| `AppDropdown` | 下拉選單 |
| `AppCheckBox` | 複選框 |
| `AppTable` | 基礎表格 |
| `AppDatePicker` | 日期選擇器 |
| `AppFormErrorMessage` | 表單錯誤訊息 |
| `AppPagination` | 分頁 |
| `AppTabs` | 標籤頁 |
| `AppTextField` | 文字輸入框 |

### 使用範例

```vue
<script setup>
import { AppButton, AppDialogue } from '@vivotek/generic-vue-components'
</script>

<template>
  <AppButton variant="primary" @click="handleClick">
    確認
  </AppButton>
  
  <AppDialogue v-model="showDialog" title="提示">
    <p>確定要執行此操作嗎？</p>
  </AppDialogue>
</template>
```

### 核心依賴
- `@tanstack/vue-table` - 表格功能
- `vue-final-modal` - 模態框
- `vue-tippy` - 提示框
- `vuedraggable` - 拖曳功能

---

## 2. advanced-vue-components

**位置**：`packages/advanced-vue-components/`
**定位**：業務組件，包含設備管理和視頻播放相關功能

### 常用組件

| 組件 | 用途 |
|------|------|
| `DeviceSelector` | 設備選擇器 |
| `DeviceThumbnail` | 設備縮圖 |
| `DeviceIndicator` | 設備狀態指示器 |
| `DisplayStreamingBlock` | 串流顯示區塊 |
| `PlayerControl` | 播放器控制 |
| `Timeline` | 時間軸 |
| `PlaybackControlPanel` | 回放控制面板 |
| `DraggableViewcellLayout` | 可拖曳視圖佈局 |
| `LayoutSwitch` | 佈局切換 |
| `SideMenusLayout` | 側邊選單佈局 |

### 使用範例

```vue
<script setup>
import { 
  DeviceSelector, 
  DisplayStreamingBlock,
  Timeline 
} from '@vivotek/advanced-vue-components'
</script>

<template>
  <DeviceSelector
    v-model="selectedDevices"
    :device-list="devices"
    multiple
  />
  
  <DisplayStreamingBlock
    :device-id="deviceId"
    :stream-type="'live'"
  />
  
  <Timeline
    :start-time="startTime"
    :end-time="endTime"
    @time-change="handleTimeChange"
  />
</template>
```

### 核心依賴
- `@vivotek/device` - 設備抽象層
- `@vivotek/lib-hls-player` - HLS 播放
- `@vivotek/lib-rtsp-player` - RTSP 播放
- `chart.js` + `vue-chartjs` - 圖表
- `vue-draggable-plus` - 拖曳功能

---

## 3. vsaas-vue-components

**位置**：`packages/vsaas-vue-components/`
**定位**：VSaaS 專用樣式的輕量組件

### 常用組件

| 組件 | 用途 |
|------|------|
| `VsaasButton` | 按鈕 |
| `VsaasTable` | 表格 |
| `VsaasDialogue` | 對話框 |
| `VsaasDropdownMenu` | 下拉選單 |
| `VsaasFilter` | 篩選器 |
| `VsaasPagination` | 分頁 |
| `VsaasSearch` | 搜尋框 |
| `VsaasSelect` | 選擇器 |
| `VsaasTabs` | 標籤頁 |
| `VsaasTooltip` | 提示框 |

### 使用範例

```vue
<script setup>
import { VsaasTable, VsaasSearch } from '@vivotek/vsaas-vue-components'
</script>

<template>
  <VsaasSearch v-model="keyword" placeholder="搜尋..." />
  
  <VsaasTable
    :columns="columns"
    :data="tableData"
    :loading="isLoading"
  />
</template>
```

---

## 4. shadcn-vue-components

**位置**：`packages/shadcn-vue-components/`
**定位**：現代設計系統，基於 Tailwind CSS

### 常用組件

| 組件 | 用途 |
|------|------|
| `Button` | 按鈕 |
| `Input` | 輸入框 |
| `SearchInput` | 搜尋輸入框 |
| `Card` | 卡片 |
| `Dialog` | 對話框 |
| `AlertDialog` | 警告對話框 |
| `DropdownMenu` | 下拉選單 |
| `Select` | 選擇器 |
| `Tabs` | 標籤頁 |
| `Table` | 表格 |
| `Badge` | 徽章 |
| `StatusIndicator` | 狀態指示器 |

### 使用範例

```vue
<script setup>
import { Button, Dialog, DialogContent, DialogTrigger } from '@vivotek/shadcn-vue-components'
</script>

<template>
  <Dialog>
    <DialogTrigger as-child>
      <Button variant="outline">開啟對話框</Button>
    </DialogTrigger>
    <DialogContent>
      <h2>對話框標題</h2>
      <p>對話框內容</p>
    </DialogContent>
  </Dialog>
</template>
```

### 核心依賴
- `reka-ui` - Headless UI
- `tailwind-merge` - Tailwind 類別合併
- `lucide-vue-next` - 圖示
- `clsx` - 類別名稱工具

---

## 組件選擇指南

### 何時使用哪個組件庫？

| 場景 | 推薦組件庫 |
|------|-----------|
| 通用表單元素 | generic-vue-components |
| 設備相關功能 | advanced-vue-components |
| 視頻播放 | advanced-vue-components |
| 輕量列表頁 | vsaas-vue-components |
| 新功能開發 | shadcn-vue-components |
| 現代化重構 | shadcn-vue-components |

### 命名規則

| 組件庫 | 前綴 |
|--------|------|
| generic-vue | `App*` |
| advanced-vue | 無統一前綴 |
| vsaas-vue | `Vsaas*` |
| shadcn-vue | PascalCase（無前綴） |

---

## 組件匯入

### 從 node_modules

```javascript
// generic-vue-components
import { AppButton } from '@vivotek/generic-vue-components'

// advanced-vue-components  
import { DeviceSelector } from '@vivotek/advanced-vue-components'

// vsaas-vue-components
import { VsaasTable } from '@vivotek/vsaas-vue-components'

// shadcn-vue-components
import { Button, Dialog } from '@vivotek/shadcn-vue-components'
```

### 相對路徑匯入（專案內）

```javascript
// 從專案內的 components 目錄
import CustomDialog from '@/components/Dialogues/CustomDialog.vue'
```

---

## 相關資源

- 各組件庫的 `README.md` 有完整 API 文件
- shadcn-vue-components 有 Storybook 文件
- 組件使用範例可參考 `packages/app-vsaas-portal/src/components/`
