# Downtime Page 實作計畫

## 背景
權限 3.0 上線後會有 downtime，需要做歇業中的告知頁面。

**截止日期：** 週四前完成並上 stage

## 需求
- 兩個品牌版本：VORTEX 和 CloudSight
- Infra 部署時用環境變數控制，都是執行 `pnpm build`
- Build 維護模式時要快（不跑完整打包）

---

## 技術方案

### 使用 Vite Plugin + 靜態 HTML

在 build 時根據環境變數 `VITE_IS_MAINTENANCE` 決定輸出：
- **正常模式**：打包完整 Vue App
- **維護模式**：直接複製靜態 HTML（超快，跳過打包）

---

## 實作計畫

### 1. 建立靜態維護頁面 HTML

**VORTEX 版本：** `src/pages/maintenance/vortex.html`
**CloudSight 版本：** `src/pages/maintenance/cloudsight.html`

### 2. 修改 vite.config.js

新增 maintenance plugin，根據環境變數決定：
- `VITE_IS_MAINTENANCE=true` → 複製靜態 HTML 到 `dist/index.html`
- 否則 → 正常打包

```javascript
import fs from 'fs';
import path from 'path';

const maintenancePlugin = () => ({
  name: 'maintenance-only',
  closeBundle() {
    if (process.env.VITE_IS_MAINTENANCE === 'true') {
      const variant = process.env.VITE_VENDOR_NAME === 'MarchNetworks' 
        ? 'cloudsight' 
        : 'vortex';
      const src = path.resolve(__dirname, `src/pages/maintenance/${variant}.html`);
      const dest = path.resolve(__dirname, 'dist/index.html');
      fs.copyFileSync(src, dest);
    }
  },
});
```

### 3. 新增 build script（可選）

**package.json:**
```json
{
  "scripts": {
    "build:maintenance": "VITE_IS_MAINTENANCE=true pnpm build",
    "build:maintenance:cloudsight": "VITE_IS_MAINTENANCE=true VITE_VENDOR_NAME=MarchNetworks pnpm build"
  }
}
```

---

## 關鍵檔案

| 用途 | 路徑 |
|------|------|
| VORTEX 維護頁 HTML | `src/pages/maintenance/vortex.html` |
| CloudSight 維護頁 HTML | `src/pages/maintenance/cloudsight.html` |
| Vite 設定 | `vite.config.js` |

---

## Infra 部署方式

```bash
# 正常部署
pnpm build

# 維護模式部署 - VORTEX
VITE_IS_MAINTENANCE=true pnpm build

# 維護模式部署 - CloudSight
VITE_IS_MAINTENANCE=true VITE_VENDOR_NAME=MarchNetworks pnpm build
```

---

## 驗證方式

1. 執行 `VITE_IS_MAINTENANCE=true pnpm build`
2. 確認 `dist/index.html` 是靜態維護頁面
3. 用 `pnpm serve` 預覽確認顯示正確
4. 測試 CloudSight 版本

---

## 待確認

- [ ] 用戶提供 UI 設計圖確認樣式細節
