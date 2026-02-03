# 建立 PR Skill 計劃

## 目標
建立一個自動發 PR 到 VIVOTEK-IT/webtech-monorepo 的 skill，功能包括：
1. 依照專案規則格式化 PR
2. 精簡描述自動生成
3. 自動帶上相關 reviewers

---

## PR Skill 設計

### Skill 名稱
`pr` 或 `create-pr`

### 觸發條件
- 使用者說 "發 PR"、"create PR"、"開 PR"
- 使用者說 "/pr"

### 核心功能

#### 1. 自動生成 PR 標題
依照 Conventional Commits 格式：
```
<type>(<scope>): <description>
```

從 git commits 自動提取：
- type: feat, fix, docs, refactor 等
- scope: 從修改的 package 推斷（如 vsaas-portal）
- description: 從 commit 訊息提取

#### 2. 自動生成 PR 描述
精簡格式：
```markdown
## Changes
- 變更項目 1
- 變更項目 2

## Related Issues
- Fixes #123 (如有)
```

#### 3. 自動指派 Reviewers
依照修改的 package 指派對應 reviewer：

| Package | Reviewers |
|---------|-----------|
| app-vsaas-portal | @vsaas-team |
| app-one-portal | @one-portal-team |
| app-reseller-portal | @si-team |
| generic-vue-components | @component-team |
| advanced-vue-components | @component-team |

### 實作步驟

#### Step 1: 建立 Skill 目錄
```bash
mkdir -p ~/.claude/skills/pr
```

#### Step 2: 建立 SKILL.md
包含：
- 觸發條件描述
- PR 格式規則
- Reviewer 對照表
- gh CLI 指令範本

#### Step 3: Skill 邏輯

```bash
# 1. 取得變更的 packages
git diff --name-only main | grep "^packages/" | cut -d'/' -f2 | sort -u

# 2. 從 commits 生成標題
git log main..HEAD --oneline --format="%s" | head -1

# 3. 生成描述
git log main..HEAD --oneline --format="- %s"

# 4. 對應 reviewers
# 根據 package 查表

# 5. 執行 gh pr create
gh pr create \
  --base main \
  --title "<generated-title>" \
  --body "<generated-body>" \
  --reviewer "<reviewers>"
```

---

## 關鍵檔案位置

| 檔案 | 用途 |
|------|------|
| `commitlint.config.js` | Commit 類型定義 |
| `.github/workflows/*.yml` | CI 工作流程 |
| `.husky/commit-msg` | Commit 驗證 |
| `CLAUDE.md` | 專案規則 |

---

## Commit 類型對照

| Type | 說明 | 版本影響 |
|------|------|---------|
| feat | 新功能 | MINOR |
| fix | 修復 | PATCH |
| docs | 文件 | - |
| style | 格式 | - |
| refactor | 重構 | PATCH |
| perf | 效能 | PATCH |
| test | 測試 | - |
| chore | 維護 | - |
| ci | CI/CD | - |

---

## Reviewer 規則

### 範圍
僅針對 `app-vsaas-portal` 專案

### Reviewer 邏輯
1. **優先**：查詢修改檔案的上次提交者（`git log -1 --format='%an'`）
2. **預設**：如無法取得或為自己，使用 `@MarcoLin01`

### 實作邏輯
```bash
# 取得修改檔案的最後提交者
LAST_AUTHOR=$(git log -1 --format='%aN' -- <changed-files>)
CURRENT_USER=$(git config user.name)

if [ "$LAST_AUTHOR" != "$CURRENT_USER" ] && [ -n "$LAST_AUTHOR" ]; then
  REVIEWER="$LAST_AUTHOR"
else
  REVIEWER="MarcoLin01"
fi
```

---

## 驗證方式

1. 執行 `/pr` 指令
2. 確認 PR 標題符合 conventional commits
3. 確認 reviewers 正確指派
4. 確認 CI 檢查通過

---

# VSaaS Portal 模組 Skill 文件建立計劃（已完成部分）

## 已建立的 Skill

| Skill | 狀態 |
|-------|------|
| vsaas-architecture | ✅ 完成 |
| vsaas-components | ✅ 完成 |
| vsaas-device | ✅ 完成 |
| vsaas-system | ✅ 完成 |
| vsaas-user-role | ✅ 完成 |

---

## 模組清單（依路由分類）

| 優先級 | 路由 | Skill 名稱 | 說明 |
|--------|------|-----------|------|
| 1 | `/device` | `vsaas-device` | 設備管理（最複雜） |
| 1 | `/system` | `vsaas-system` | 系統設定 |
| 1 | `/user` + `/role` | `vsaas-user-role` | 使用者與角色管理 |
| 2 | `/archive` | `vsaas-archive` | 存檔管理 |
| 2 | `/notification` | `vsaas-notification` | 訊息中心 |
| 2 | `/view` | `vsaas-view` | 自訂檢視 |
| 3 | `/group` | `vsaas-group` | 群組管理 |
| 3 | `/floorplans` | `vsaas-floorplan` | 樓層平面圖 |
| 3 | `/alarm` | `vsaas-alarm` | 告警 |
| 3 | `/timelapse` | `vsaas-timelapse` | 縮時攝影 |
| 基礎 | - | `vsaas-architecture` | 整體架構 |
| 基礎 | - | `vsaas-components` | 組件庫指南 |
| 基礎 | `/login` | `vsaas-auth` | 認證流程 |

---

## Skill 文件結構

每個模組的 skill 目錄結構：
```
~/.claude/skills/vsaas-{module}/
├── SKILL.md                    # 主文件（精簡版）
└── references/                 # 詳細參考
    ├── routes.md              # 路由結構
    ├── store.md               # Store 模組
    └── components.md          # 組件使用
```

---

## SKILL.md 模板

每個 skill 文件包含以下區塊：

### 1. Frontmatter
```yaml
---
name: vsaas-{module}
description: 觸發條件描述
version: 1.0.0
---
```

### 2. 模組概述
- 商業邏輯（解決什麼問題）
- 功能範圍

### 3. 路由結構
- 主路由表格
- 子路由摘要

### 4. Store 依賴
- 主要 Store 模組
- 跨模組依賴

### 5. 組件架構
- 頁面目錄結構
- 使用的共用組件庫

### 6. 關鍵程式碼路徑
- 進入點文件
- 資料流描述

### 7. 常見開發任務
- 新增功能步驟
- 修改指引
- 除錯方向

---

## 實作步驟

### Phase 1: 基礎架構 Skill
1. 建立 `vsaas-architecture` - 整體架構概覽
2. 建立 `vsaas-components` - 四個組件庫使用指南

### Phase 2: 核心模組 Skill
3. 建立 `vsaas-device` - 設備管理（最大最複雜）
4. 建立 `vsaas-system` - 系統設定
5. 建立 `vsaas-user-role` - 使用者與角色

### Phase 3: 功能模組 Skill
6. 建立 `vsaas-archive` - 存檔管理
7. 建立 `vsaas-notification` - 訊息中心
8. 建立 `vsaas-view` - 自訂檢視

### Phase 4: 輔助模組 Skill
9. 建立其餘模組 skill

---

## 關鍵檔案位置

### 路由配置
- `packages/app-vsaas-portal/src/router/router.js` - 主路由
- `packages/app-vsaas-portal/src/pages/*/route.js` - 各模組路由

### Store 模組
- `packages/app-vsaas-portal/src/store/index.js` - Store 入口
- `packages/app-vsaas-portal/src/store/*/` - 26 個模組

### 頁面組件
- `packages/app-vsaas-portal/src/pages/` - 所有頁面
- `packages/app-vsaas-portal/src/components/` - 共用組件

### 組件庫
- `packages/generic-vue-components/` - 通用 UI（48 個）
- `packages/advanced-vue-components/` - 業務組件（56 個）
- `packages/vsaas-vue-components/` - VSaaS 專用（17 個）
- `packages/shadcn-vue-components/` - 現代設計

---

## 每個模組記錄重點

### Device 模組（最複雜）
- 三大子模組：management、settings、AddDevice
- 設備類型差異：Camera、NVR、IoT、Bridge
- Reskin 雙版本支援
- 微前端 AI 模組動態載入

### System 模組
- 組織設定、授權、SSO、MFA
- 第三方整合（Access Control、Smart Sensor）
- 全域設備設定

### User/Role 模組
- RBAC 權限架構
- 組織級 vs 資源級角色
- Casbin 政策引擎

### Archive 模組
- 存檔匯出流程
- 共享存檔功能

### Notification 模組
- 訊息中心
- 第三方事件整合

---

## 驗證方式

1. 檢查 skill 文件語法正確
2. 確認關鍵路徑檔案存在
3. 測試 skill 觸發條件是否正確匹配

---

## 預估工作量

| Phase | 內容 | Skill 數量 |
|-------|------|-----------|
| 1 | 基礎架構 | 2 |
| 2 | 核心模組 | 3 |
| 3 | 功能模組 | 3 |
| 4 | 輔助模組 | 5 |
| **總計** | | **13** |
