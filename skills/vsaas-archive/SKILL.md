---
name: vsaas-archive
description: Use when working with video archives, archive playback, archive download, or archive sharing. Triggers on keywords like "archive", "recorded video", "clip", "export video", "archive management".
version: 1.0.0
---

# VSaaS Portal - Archive 模組

存檔管理模組負責視頻錄像的管理、播放、下載和分享功能。

## 模組概述

### 功能範圍
- 存檔列表查詢和過濾
- HLS 流媒體播放
- MP4 下載
- 分享連結管理
- 批量操作

---

## 目錄結構

```
src/pages/archive/
├── index.vue                           # 路由根組件
├── route.js                            # 路由配置
└── management/
    ├── index.vue                       # 存檔管理主頁面
    └── components/
        ├── ArchiveManagementTable.vue  # 表格主組件
        ├── ArchivePlayModal.vue        # 播放模態框
        ├── ArchivePlayerPanel.vue      # 播放面板
        ├── ArchiveDetailPanel.vue      # 詳情面板
        ├── ArchivePlayerControl.vue    # 播放控制條
        ├── ArchivePlayerTimeline.vue   # 時間軸
        └── ArchiveManagementTable/
            ├── ArchiveGridModeView.vue # 網格視圖
            ├── ArchiveThumbnail.vue    # 縮圖組件
            ├── ChangeArchiveNameDialogue.vue
            ├── DeleteArchiveDialogue.vue
            └── ShareArchiveDialogue.vue

src/store/archive/
├── index.js
├── actions.js
├── mutations.js
└── getters.js

src/models/Archive/
├── Archive.js          # 存檔模型
└── SharedArchive.js    # 分享存檔模型
```

---

## 路由結構

```
/archive
└── '' → ArchiveManagement (存檔管理頁面)
    meta: { permission: ARCHIVE, requiresAllData: 'device' }
```

---

## Store 模組

### State

```javascript
{
  filter: {},               // 查詢過濾器
  archives: [],             // 存檔列表
  nvrArchiveMap: {},        // NVR 設備存檔映射
  downloadingList: Set(),   // 下載中的檔案
  genMp4Requests: Set(),    // MP4 生成請求隊列
  status: APP_TABLE_STATUS, // 表格狀態
  totalCount: 0,
  isFetchingData: false
}
```

### 主要 Actions

| Action | 功能 |
|--------|------|
| `queryArchive` | 查詢存檔列表 |
| `createArchive` | 創建新存檔 |
| `deleteArchives` | 批量刪除存檔 |
| `updateArchive` | 更新存檔名稱 |
| `modifyArchiveSharedLink` | 修改分享連結 |
| `tryDownloadArchive` | 嘗試下載存檔 |

---

## 資料模型

### Archive 模型

```javascript
class Archive {
  // 屬性
  device              // 關聯設備
  fileName            // 文件名
  cameraName          // 相機名稱
  status              // HLS_READY | MP4_READY | GEN_MP4 | GEN_MP4_FAIL
  videoLength         // 視頻長度（秒）
  startAt, endAt      // 視頻時間範圍
  createdAt           // 歸檔時間
  sharedLink          // 分享設置

  // 計算屬性
  playbackUrl         // HLS 播放 URL
  mp4Url              // MP4 下載 URL
  isReadyToDownload   // 是否可下載

  // 方法
  download()          // 下載檔案
  queryGetCookie()    // 獲取授權 Cookie
  genArchivesMp4()    // 生成 MP4
  update()            // 更新信息
}
```

---

## 存檔狀態流程

```
HLS_READY    → 可播放（HLS 流媒體）
MP4_READY    → 可下載（MP4 格式）
GEN_MP4      → 正在生成 MP4
GEN_MP4_FAIL → MP4 生成失敗（可重試）
```

---

## 下載流程

```
downloadArchive(archives[])
    ↓
archive.download()
    ├─ 檢查狀態：MP4_READY？
    ├─ No → genMp4() + registGenMp4Request()
    └─ Yes → 
       ├─ 獲取授權令牌
       ├─ 設置雲存儲憑證
       ├─ 生成預簽署 URL
       └─ DownloadHelper.downloadFile()
    ↓
sendDownloadArchiveLogs()  // 審計日誌
```

---

## 分享功能

```javascript
// 分享設置
sharedLink: {
  enabled: boolean,
  password: string | null,
  expireTime: number | null,
  url: string | null,
  enableDownload: boolean
}

// 修改分享
modifyArchiveSharedLink({
  enabled, password, expireTime, enableDownload
})
```

---

## 權限檢查

| 權限 | 說明 |
|------|------|
| `DEVICE_ARCHIVE_PLAY` | 播放存檔 |
| `DEVICE_ARCHIVE_UPDATE` | 編輯存檔 |
| `DEVICE_ARCHIVE_DOWNLOAD` | 下載存檔 |
| `DEVICE_ARCHIVE_SHARE` | 分享存檔 |
| `DEVICE_ARCHIVE_DELETE` | 刪除存檔 |

---

## 視圖模式

- **網格模式**：卡片式縮圖展示
- **列表模式**：表格展示，可選列

---

## 播放控制

- 播放/暫停
- 快進/快退（10 秒）
- 播放速度（0.5x - 2x）
- 音量控制
- 快照功能
- 全屏模式
- 時間軸拖拽

---

## 外部依賴

```javascript
import { SearchBar, PlaybackControlPanel } from '@vivotek/advanced-vue-components'
import { AppTable } from '@vivotek/generic-vue-components'
import '@vivotek/lib-hls-player'
```
