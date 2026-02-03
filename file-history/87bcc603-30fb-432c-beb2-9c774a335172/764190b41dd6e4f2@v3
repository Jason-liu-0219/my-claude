---
name: vsaas-skill-updater
description: Use when need to create, update, or audit VSaaS Portal module skills. Triggers on keywords like "update skill", "create skill", "audit skills", "sync skills", "skill maintenance", "更新 skill", "skill 維護".
version: 1.1.0
---

# VSaaS Portal - Skill 更新工具

此工具用於創建、更新和審核 VSaaS Portal 模組的 Claude Code skills。

## 用途

當需要以下操作時使用此 skill：
- 為新的 VSaaS Portal 模組創建 skill
- 更新現有模組的 skill 內容
- 審核所有模組是否都有對應的 skill
- 檢查 skill 與代碼的一致性
- **定期維護和更新 skills**

---

## 🔄 定期更新指南

### 建議更新頻率

| 情況 | 建議頻率 |
|------|---------|
| 主要版本發布後 | 立即檢查 |
| Sprint 結束後 | 每 2 週檢查 |
| 定期維護 | 每月一次 |
| 發現 skill 與代碼不符 | 立即更新 |

### 快速更新指令

當用戶說「更新 vsaas skills」或「檢查 skill 更新」時，執行以下流程：

```
/vsaas-skill-updater
```

---

## 📋 定期更新流程（完整版）

### Phase 1：變更偵測

```bash
# 1. 檢查自上次更新以來的所有變更
git log --oneline --since="30 days ago" -- packages/app-vsaas-portal/src/pages/

# 2. 列出有變更的模組
git diff --name-only HEAD~100 -- packages/app-vsaas-portal/src/pages/ | \
  cut -d'/' -f5 | sort -u

# 3. 檢查是否有新增的頁面模組
ls packages/app-vsaas-portal/src/pages/

# 4. 檢查是否有新增的 Store 模組
ls packages/app-vsaas-portal/src/store/
```

### Phase 2：優先級判斷

根據變更內容決定更新優先級：

| 變更類型 | 優先級 | 需要更新的章節 |
|---------|--------|---------------|
| 新增路由 | 高 | 路由結構 |
| 新增組件 | 高 | 目錄結構、主要組件 |
| 新增 Store Action/Getter | 高 | Store 依賴 |
| 新增 API 端點 | 高 | API 整合 |
| 修改商業邏輯 | 中 | 模組概述、資料流 |
| 重構代碼 | 中 | 目錄結構 |
| Bug 修復 | 低 | 通常不需更新 |
| UI 調整 | 低 | 通常不需更新 |

### Phase 3：逐一更新 Skills

對每個有變更的模組執行：

```bash
# 1. 查看模組最近變更
git log --oneline -10 -- packages/app-vsaas-portal/src/pages/{module}/

# 2. 讀取現有 skill
cat ~/.claude/skills/vsaas-{module}/SKILL.md

# 3. 使用 Explore agent 分析變更
# 聚焦於：新增路由、新增組件、新增 Store 功能、新增 API

# 4. 更新 SKILL.md
# - 更新版本號（minor: 新功能, patch: 修正）
# - 更新相關章節
# - 保留未變更的內容
```

### Phase 4：驗證和記錄

```bash
# 1. 驗證所有 skills 存在
ls ~/.claude/skills/vsaas-*/SKILL.md | wc -l

# 2. 檢查版本號更新
grep -h "^version:" ~/.claude/skills/vsaas-*/SKILL.md

# 3. 記錄更新日誌（見下方）
```

---

## 📝 更新日誌管理

在每個 skill 的 SKILL.md 末尾維護更新日誌：

```markdown
---

## 更新日誌

### v1.2.0 (2025-02-01)
- 新增 XXX 路由
- 新增 XXX 組件
- 更新 Store Actions

### v1.1.0 (2025-01-15)
- 新增 XXX 功能
- 修正路由結構

### v1.0.0 (2025-01-01)
- 初始版本
```

---

## 🎯 單一模組快速更新流程

當只需要更新特定模組的 skill 時：

### Step 1：分析變更

```bash
# 查看最近 30 天的變更
git log --oneline --since="30 days ago" -- packages/app-vsaas-portal/src/pages/{module}/

# 查看詳細變更內容
git diff HEAD~30 -- packages/app-vsaas-portal/src/pages/{module}/
```

### Step 2：使用 Explore Agent

```
請分析 VSaaS Portal {module} 模組的最近變更：
1. 檢查 packages/app-vsaas-portal/src/pages/{module}/ 的結構
2. 識別新增的路由、組件、Store 功能
3. 與現有 skill 比較差異
```

### Step 3：更新 SKILL.md

1. 更新 frontmatter 中的 version
2. 更新變更的章節
3. 在末尾新增更新日誌

---

## 🔍 變更偵測腳本

建議的變更偵測流程：

```bash
#!/bin/bash
# 檢查每個模組的變更狀態

MODULES=(
  "account:vsaas-account"
  "alarm:vsaas-alarm"
  "archive:vsaas-archive"
  "auth:vsaas-auth"
  "device:vsaas-device"
  "floorplan:vsaas-floorplan"
  "group:vsaas-group"
  "notification:vsaas-notification"
  "system:vsaas-system"
  "Timelapse:vsaas-timelapse"
  "user:vsaas-user-role"
  "view:vsaas-view"
)

echo "=== VSaaS Portal Skills 變更檢查 ==="
echo ""

for item in "${MODULES[@]}"; do
  PAGE="${item%%:*}"
  SKILL="${item##*:}"

  CHANGES=$(git log --oneline --since="30 days ago" -- \
    "packages/app-vsaas-portal/src/pages/${PAGE}/" 2>/dev/null | wc -l)

  if [ "$CHANGES" -gt 0 ]; then
    echo "⚠️  ${SKILL}: ${CHANGES} commits in last 30 days"
  else
    echo "✓  ${SKILL}: No changes"
  fi
done
```

---

## 📊 Skills 狀態總覽指令

```bash
# 查看所有 skills 的版本和最後更新時間
for f in ~/.claude/skills/vsaas-*/SKILL.md; do
  SKILL=$(dirname $f | xargs basename)
  VERSION=$(grep "^version:" $f | cut -d' ' -f2)
  MTIME=$(stat -f "%Sm" -t "%Y-%m-%d" $f 2>/dev/null || stat -c "%y" $f 2>/dev/null | cut -d' ' -f1)
  echo "$SKILL: v$VERSION (updated: $MTIME)"
done
```

---

## 現有 VSaaS Portal Skills

### 已實現的模組 Skills

| Skill 名稱 | 說明 | 主要關鍵字 |
|-----------|------|-----------|
| `vsaas-account` | 帳戶設定、個人資料、API 金鑰 | account, profile, API key, MFA |
| `vsaas-alarm` | 警報管理、Webhook | alarm, alert, webhook |
| `vsaas-architecture` | 專案架構、路由、狀態管理 | architecture, monorepo, module federation |
| `vsaas-archive` | 視頻存檔、下載、分享 | archive, recorded video, clip |
| `vsaas-auth` | 認證、登入、SSO、MFA | login, logout, authentication, SSO |
| `vsaas-components` | 組件庫使用指南 | component, AppButton, generic-vue |
| `vsaas-device` | 設備管理、設定 | device, camera, NVR, DeviceSettings |
| `vsaas-floorplan` | 樓層平面圖、設備放置 | floorplan, device placement, FOV |
| `vsaas-group` | 群組管理、直播檢視 | group, site, live view, layout |
| `vsaas-notification` | 通知中心、事件警報 | notification, message center, event |
| `vsaas-system` | 系統設定、組織配置 | system, organization, license |
| `vsaas-timelapse` | 縮時攝影、導出 | timelapse, snapshot, capture schedule |
| `vsaas-user-role` | 用戶角色、權限管理 | user, role, permission, RBAC |
| `vsaas-view` | 自訂檢視、串流佈局 | view, customized view, streaming layout |

---

## 創建新 Skill 流程

### Step 1：探索模組結構

```bash
# 1. 檢查頁面目錄
ls packages/app-vsaas-portal/src/pages/{module}/

# 2. 檢查路由定義
cat packages/app-vsaas-portal/src/pages/{module}/route.js

# 3. 檢查 Store 模組
ls packages/app-vsaas-portal/src/store/{module}/

# 4. 檢查相關組件
ls packages/app-vsaas-portal/src/pages/{module}/components/
```

### Step 2：分析模組內容

使用 Explore agent 深入分析：
- 路由結構
- 目錄結構
- Store 狀態、Actions、Getters
- 主要組件功能
- 資料流模式
- API 整合

### Step 3：創建 SKILL.md

```bash
# 創建目錄
mkdir -p ~/.claude/skills/vsaas-{module}

# 創建 SKILL.md
# 內容應包含以下章節
```

---

## SKILL.md 標準模板

```markdown
---
name: vsaas-{module}
description: Use when working with {描述}. Triggers on keywords like "{關鍵字1}", "{關鍵字2}".
version: 1.0.0
---

# VSaaS Portal - {Module} 模組

{模組簡介}

## 模組概述

### 商業邏輯
- {功能1}
- {功能2}

---

## 路由結構

| 路徑 | 名稱 | 說明 |
|------|------|------|
| `/path` | RouteName | 說明 |

---

## 目錄結構

\```
src/pages/{module}/
├── index.vue
├── route.js
└── components/
\```

---

## Store 依賴

### 主要模組：{module}
**位置**：`src/store/{module}/`

\```javascript
state: {
  // 狀態結構
}
\```

#### 關鍵 Actions
| Action | 說明 |
|--------|------|

#### 關鍵 Getters
| Getter | 說明 |
|--------|------|

---

## 主要組件

### 1. ComponentName.vue
**功能**：描述

---

## 關鍵程式碼路徑

### 進入點
- 路由定義位置
- Store 位置

### 資料流
\```
流程圖
\```

---

## 常見開發任務

### 1. 任務名稱
\```bash
# 主要檔案
path/to/file
\```

---

## 使用的組件

### 來自 advanced-vue-components
- ComponentName

### 來自 generic-vue-components
- ComponentName

---

## API 整合

\```javascript
// API 端點
\```

---

## 注意事項

1. 注意事項1
2. 注意事項2
```

---

## 更新現有 Skill 流程

### Step 1：檢查變更

```bash
# 檢查模組最近的 Git 變更
git log --oneline -20 -- packages/app-vsaas-portal/src/pages/{module}/

# 檢查新增的檔案
git diff --name-only HEAD~50 -- packages/app-vsaas-portal/src/pages/{module}/
```

### Step 2：更新 SKILL.md

1. 讀取現有 SKILL.md
2. 分析代碼變更
3. 更新相關章節
4. 更新版本號

### Step 3：驗證更新

- 確認路由結構正確
- 確認 Store 結構正確
- 確認組件列表完整

---

## 審核所有 Skills 流程

### Step 1：列出所有頁面模組

```bash
ls packages/app-vsaas-portal/src/pages/
```

### Step 2：列出現有 Skills

```bash
ls ~/.claude/skills/vsaas-*/SKILL.md
```

### Step 3：比對缺失的模組

對比兩個列表，找出：
- 需要新增的 skill
- 需要更新的 skill
- 可以合併或刪除的 skill

---

## 模組到 Skill 的映射規則

### 需要獨立 Skill 的模組

| 頁面模組 | Skill 名稱 | 原因 |
|---------|-----------|------|
| device | vsaas-device | 核心功能，內容豐富 |
| view | vsaas-view | 獨立功能模組 |
| group | vsaas-group | 獨立功能模組 |
| notification | vsaas-notification | 獨立功能模組 |
| alarm | vsaas-alarm | 獨立功能模組 |
| archive | vsaas-archive | 獨立功能模組 |
| floorplan | vsaas-floorplan | 獨立功能模組 |
| user + role | vsaas-user-role | 相關功能合併 |
| system | vsaas-system | 獨立功能模組 |
| account | vsaas-account | 獨立功能模組 |
| login + auth | vsaas-auth | 相關功能合併 |
| Timelapse + device/timelapse | vsaas-timelapse | 相關功能合併 |

### 不需要獨立 Skill 的模組

| 頁面模組 | 原因 |
|---------|------|
| noPrivileges | 錯誤頁面，無開發需求 |
| upgradePlan | 簡單頁面，無複雜邏輯 |
| NVRForgotPassword | 簡單頁面，無複雜邏輯 |
| share | 分享功能，已在其他模組涵蓋 |
| deviceShare | 設備分享，已在 device 模組涵蓋 |
| searchlight | 可在未來需要時創建 |

---

## 驗證清單

創建或更新 Skill 後，確認以下項目：

- [ ] Frontmatter 格式正確（name, description, version）
- [ ] Description 包含觸發關鍵字
- [ ] 路由結構與代碼一致
- [ ] 目錄結構與代碼一致
- [ ] Store 結構與代碼一致
- [ ] 主要組件列表完整
- [ ] API 端點正確
- [ ] 常見開發任務實用
- [ ] 注意事項有價值

---

## 常用指令

```bash
# 列出所有 vsaas skills
ls ~/.claude/skills/vsaas-*/SKILL.md

# 查看 skill 內容
cat ~/.claude/skills/vsaas-{module}/SKILL.md

# 統計 skills 數量
ls ~/.claude/skills/vsaas-*/SKILL.md | wc -l

# 搜尋特定關鍵字在 skills 中的出現
grep -r "keyword" ~/.claude/skills/vsaas-*/SKILL.md
```

---

## 🚀 常見更新場景

### 場景 1：Sprint 結束後的例行檢查

```
用戶：「Sprint 結束了，幫我檢查 vsaas skills 是否需要更新」

執行步驟：
1. 執行變更偵測腳本
2. 列出有變更的模組
3. 對每個有變更的模組進行簡要分析
4. 詢問用戶是否要更新特定 skill
```

### 場景 2：新功能開發完成後

```
用戶：「我剛完成 {module} 的新功能，幫我更新對應的 skill」

執行步驟：
1. 使用 Explore agent 分析該模組的新功能
2. 讀取現有 SKILL.md
3. 更新相關章節
4. 更新版本號和更新日誌
```

### 場景 3：發現 Skill 與代碼不符

```
用戶：「vsaas-{module} skill 的內容好像過時了」

執行步驟：
1. 讀取現有 SKILL.md
2. 對比實際代碼結構
3. 識別差異
4. 更新不一致的章節
```

### 場景 4：定期全面審核

```
用戶：「幫我全面審核所有 vsaas skills」

執行步驟：
1. 列出所有頁面模組
2. 列出所有現有 skills
3. 檢查是否有遺漏的模組
4. 對每個 skill 進行快速驗證
5. 生成審核報告
```

---

## 📌 版本號規則

遵循語義化版本 (Semantic Versioning)：

| 變更類型 | 版本變更 | 範例 |
|---------|---------|------|
| 新增主要功能模組 | Major (X.0.0) | 1.0.0 → 2.0.0 |
| 新增路由/組件/功能 | Minor (x.X.0) | 1.0.0 → 1.1.0 |
| 修正錯誤/小調整 | Patch (x.x.X) | 1.0.0 → 1.0.1 |

---

## 🔧 疑難排解

### Q: Skill 內容太長，如何精簡？

保留核心章節：
1. 路由結構（必須）
2. 目錄結構（必須）
3. Store 依賴（核心 Actions/Getters）
4. 常見開發任務（最實用的 3-5 個）

可精簡的章節：
- 詳細組件說明 → 只列出組件名稱
- 完整 API 列表 → 只列出主要端點
- 資料流細節 → 簡化為高層概述

### Q: 如何判斷是否需要更新？

檢查以下關鍵變更：
- [ ] 新增或修改路由
- [ ] 新增頁面或組件
- [ ] Store 結構變更
- [ ] API 端點變更
- [ ] 商業邏輯重大調整

如果以上都沒有變更，則不需要更新 skill。

### Q: 多個模組有關聯變更怎麼辦？

1. 先更新被依賴的模組（如 device 常被其他模組依賴）
2. 再更新依賴它的模組
3. 確保跨模組引用保持一致

---

## 📅 維護排程建議

```
┌─────────────────────────────────────────────────────┐
│  VSaaS Portal Skills 維護日曆                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│  每週：                                              │
│  └─ 無需主動維護                                     │
│                                                     │
│  每兩週（Sprint 結束）：                              │
│  └─ 執行變更偵測腳本                                 │
│  └─ 更新有變更的 skills                              │
│                                                     │
│  每月：                                              │
│  └─ 全面審核所有 skills                              │
│  └─ 檢查是否有新模組需要創建 skill                    │
│  └─ 更新 vsaas-skill-updater 的模組清單              │
│                                                     │
│  每季度：                                            │
│  └─ 深度審核 skills 品質                             │
│  └─ 精簡過時或冗餘內容                               │
│  └─ 更新 SKILL.md 模板                              │
│                                                     │
│  主要版本發布後：                                     │
│  └─ 立即執行全面審核                                 │
│  └─ 更新所有受影響的 skills                          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 🎯 總結：更新 Skills 的最佳實踐

1. **保持同步**：代碼變更後及時更新對應 skill
2. **版本管理**：使用語義化版本號追蹤變更
3. **更新日誌**：記錄每次更新的內容
4. **定期審核**：每月檢查 skills 與代碼的一致性
5. **精簡內容**：保持 skill 簡潔實用，避免過度冗長
6. **驗證測試**：更新後確認 skill 內容正確
