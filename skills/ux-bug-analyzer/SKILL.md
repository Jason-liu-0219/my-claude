---
name: ux-bug-analyzer
description: 分析指定頁面的 UX 相關 bug。觸發詞：「確認...操作是否有問題」、「檢查...UX bug」、「/ux-check」
version: 1.1.0
---

# UX Bug Analyzer

分析 VSaaS Portal 頁面程式碼，找出使用者操作相關的潛在 bug。

## 觸發條件

- 「請幫我確認 XXX 頁面相關操作是否會有問題」
- 「請檢查 XXX 的 UX bug」
- `/ux-check {module}`

---

## 核心原則

### ⚠️ 自動深入分析，不中斷詢問

**重要**：當發現可疑問題時，必須自動追蹤到底，不要中途詢問用戶「是否要確認」或「要不要繼續分析」。

執行流程：
1. 發現可疑模式 → 自動讀取相關程式碼
2. 需要追蹤上下游 → 自動查找引用和被引用
3. 確認是否為真正的 bug → 自動驗證
4. 全部分析完成後 → 一次性輸出報告

**❌ 錯誤做法**：
```
發現 CopyRoleDialogue 可能有問題...
→ 「請確認是否要深入分析？」  // 不要這樣！
```

**✅ 正確做法**：
```
發現 CopyRoleDialogue 可能有問題...
→ 自動讀取 CopyRoleDialogue.vue
→ 自動追蹤 useCopyRole.js
→ 確認問題存在
→ 輸出完整報告
```

---

## 執行流程

### Step 1: 定位相關檔案

根據使用者指定的頁面/模組，找出相關檔案：

```bash
# 找出頁面檔案
find packages/app-vsaas-portal/src/pages -name "*.vue" | xargs grep -l "{keyword}"

# 找出相關 components
find packages/app-vsaas-portal/src/components -name "*.vue" | xargs grep -l "{keyword}"

# 找出相關 store
find packages/app-vsaas-portal/src/store -name "*.js" | xargs grep -l "{keyword}"
```

### Step 2: 分析常見 UX Bug 模式

#### 2.1 Dropdown/Select 重複選擇問題

```javascript
// ❌ 問題：選同一個選項不會觸發
@change="handleChange"

// ✅ 正確：每次點擊都觸發
@click="handleClick"
// 或
@optionClick="handleOptionClick"
```

#### 2.2 v-model 雙向綁定陷阱

```javascript
// ❌ 問題：直接修改 prop
props: ['value']
this.value = newValue

// ✅ 正確：emit 事件
this.$emit('update:value', newValue)
```

#### 2.3 非同步狀態不同步

```javascript
// ❌ 問題：沒等待完成就更新 UI
async save() {
  api.save(data)
  this.showSuccess = true
}

// ✅ 正確：等待完成
async save() {
  await api.save(data)
  this.showSuccess = true
}
```

#### 2.4 陣列/物件響應性問題（Vue 2）

```javascript
// ❌ 問題：Vue 2 無法偵測
this.items[index] = newItem
this.obj.newKey = value

// ✅ 正確：使用 Vue.set 或展開
this.$set(this.items, index, newItem)
this.items = [...this.items.slice(0, index), newItem, ...this.items.slice(index + 1)]
```

#### 2.5 條件判斷錯誤

```javascript
// ❌ 問題：falsy 值判斷錯誤
if (value) // 0, '', false 都會是 false

// ✅ 正確：明確判斷
if (value !== undefined && value !== null)
```

#### 2.6 事件重複綁定/未清理

```javascript
// ❌ 問題：每次 mount 都綁定，沒有清理
mounted() {
  window.addEventListener('resize', this.handleResize)
}

// ✅ 正確：清理
beforeUnmount() {
  window.removeEventListener('resize', this.handleResize)
}
```

#### 2.7 批次操作問題

```javascript
// ❌ 問題：沒有處理空選擇
async batchDelete() {
  await api.delete(this.selectedIds)
}

// ✅ 正確：檢查空陣列
async batchDelete() {
  if (!this.selectedIds.length) return
  await api.delete(this.selectedIds)
}
```

#### 2.8 Watch 無限迴圈

```javascript
// ❌ 問題：watch 中修改被 watch 的值
watch: {
  items(newVal) {
    this.items = newVal.filter(...)  // 無限迴圈
  }
}

// ✅ 正確：使用 computed 或另一個變數
computed: {
  filteredItems() {
    return this.items.filter(...)
  }
}
```

#### 2.9 Dialog 狀態殘留

```javascript
// ❌ 問題：關閉 dialog 後資料沒清空
closeDialog() {
  this.showDialog = false
}

// ✅ 正確：清空表單資料
closeDialog() {
  this.showDialog = false
  this.formData = { ...initialFormData }
}
```

#### 2.10 Loading 狀態不正確

```javascript
// ❌ 問題：錯誤時 loading 沒關閉
async fetchData() {
  this.loading = true
  const data = await api.fetch()
  this.loading = false
}

// ✅ 正確：finally 確保關閉
async fetchData() {
  this.loading = true
  try {
    const data = await api.fetch()
  } finally {
    this.loading = false
  }
}
```

### Step 3: 評估嚴重程度

| 嚴重程度 | 條件 | 動作 |
|----------|------|------|
| **High** | 影響核心功能、資料遺失風險、操作無效 | 發 Issue |
| **Medium** | 影響使用體驗、需要 workaround | 發 Issue |
| **Low** | 效能、樣式、可優化 | 只報告 |

### Step 4: 發 GitHub Issue（僅 High/Medium）

**發 Issue 到 upstream (VIVOTEK-IT/webtech-monorepo)**

```bash
gh issue create \
  --repo VIVOTEK-IT/webtech-monorepo \
  --title "🐛 [UX Bug] {頁面}: {摘要}" \
  --body "{issue_body}" \
  --label "bug"
```

**Issue 格式**：

```markdown
## 🐛 [UX Bug] {頁面名稱}: {問題摘要}

### 問題描述
{詳細說明}

### 重現步驟
1. 前往 {頁面}
2. {操作步驟}
3. 預期：{預期行為}
4. 實際：{實際行為}

### 程式碼位置
- 檔案：`{file_path}`
- 行數：{line_number}

### 建議修復
```{language}
{suggested_fix}
```

### 嚴重程度
- [x] High - 影響核心功能
- [ ] Medium - 影響使用體驗
- [ ] Low - 輕微問題

---
🤖 Generated by UX Bug Analyzer
```

### Step 5: 輸出報告

```markdown
## UX Bug 分析報告：{頁面名稱}

### 🔴 嚴重問題（已發 Issue）

1. **{問題標題}** - Issue #{number}
   - 位置：`{file}:{line}`
   - 說明：{description}

### 🟡 中等問題（已發 Issue）

1. ...

### 🟢 輕微問題（僅供參考）

1. **{問題標題}**
   - 位置：`{file}:{line}`
   - 說明：{description}
   - 類型：效能 / 可優化

### 統計
- 嚴重：{high_count} 個
- 中等：{medium_count} 個
- 輕微：{low_count} 個
- 已發 Issue：{issue_count} 個
```

---

## 注意事項

1. **發 Issue 前先搜尋**是否已存在類似 issue
2. 不確定的問題標記為「待確認」
3. **效能問題不發 issue**，除非嚴重影響使用
4. 遵循專案的 issue 命名規範
5. Issue 發到 **VIVOTEK-IT/webtech-monorepo** (upstream)

---

## 常見模組對應

| 關鍵字 | 路徑 |
|--------|------|
| role | `pages/system/role/` |
| user | `pages/system/user/` |
| device | `pages/device/` |
| group | `pages/group/` |
| view | `pages/view/` |
| archive | `pages/archive/` |
| account | `pages/account/` |
| alarm | `pages/alarm/` |
| floorplan | `pages/floorplan/` |
