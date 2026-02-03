---
name: code-review
description: Use when user says "/review", "review code", "review my code", "review PR", "code review", "幫我 review", "檢查程式碼". Supports reviewing local changes or PR.
version: 1.3.0
---

# Code Review Skill

程式碼審查工具，支援兩種模式：
1. **Local Review** - 審查自己寫的程式碼（commit 前）
2. **PR Review** - 審查 Pull Request

---

## 使用方式

```
/review              # 自動偵測模式（僅輸出報告）
/review local        # 強制本地審查模式
/review pr [number]  # 審查指定 PR（僅輸出報告）
/review pr           # 審查當前分支的 PR
/review pr 123 --submit  # 審查並直接提交到 GitHub
```

### 輸出模式

| 模式 | 說明 | 使用情境 |
|------|------|---------|
| **預設（無參數）** | 僅在 terminal 輸出報告 | 你是 reviewer，需要先看過再決定如何回應 |
| `--submit` | 直接提交 review 到 GitHub | 你是 PR 作者，自我審查後留下紀錄 |
| `--copy` | 輸出報告並複製到剪貼簿 | 你要手動貼到 GitHub 或其他地方 |

### 決策流程

```
你是 PR 的作者嗎？
├── 是 → 使用 --submit（自動提交 COMMENT）
└── 否 → 你想直接留言嗎？
          ├── 是 → 使用 --submit（可選 APPROVE/REQUEST_CHANGES）
          └── 否 → 不加參數（僅輸出報告，自己決定如何回應）
```

### 範例

```bash
# 情境 1：我是 reviewer，先看報告再決定
/review pr 123
# → 輸出報告到 terminal，不會在 GitHub 留言

# 情境 2：我是 PR 作者，自我審查
/review pr 123 --submit
# → 直接提交到 GitHub（使用 COMMENT）

# 情境 3：我是 reviewer，確認後要直接 approve
/review pr 123 --submit --approve
# → 直接提交 APPROVE 到 GitHub

# 情境 4：我想複製報告到其他地方
/review pr 123 --copy
# → 輸出報告並複製到剪貼簿
```

---

## 執行流程

### Step 1：偵測審查模式

```bash
# 檢查是否有指定 PR 號碼
if [[ "$1" == "pr" ]]; then
  MODE="pr"
  PR_NUMBER="$2"
elif [[ "$1" == "local" ]]; then
  MODE="local"
else
  # 自動偵測：檢查是否有未暫存的變更
  if [[ -n $(git status --porcelain) ]] || [[ -n $(git diff --cached --name-only) ]]; then
    MODE="local"
  else
    # 檢查當前分支是否有 PR
    CURRENT_BRANCH=$(git branch --show-current)
    PR_EXISTS=$(gh pr view "$CURRENT_BRANCH" --json number -q '.number' 2>/dev/null || echo "")
    if [[ -n "$PR_EXISTS" ]]; then
      MODE="pr"
      PR_NUMBER="$PR_EXISTS"
    else
      MODE="local"
    fi
  fi
fi

echo "審查模式: $MODE"
```

---

## Mode 1: Local Review（本地程式碼審查）

審查尚未 commit 或剛 commit 的程式碼。

### Step 1：取得變更檔案

```bash
# 優先順序：staged > unstaged > recent commits
if [[ -n $(git diff --cached --name-only) ]]; then
  # 已暫存的變更
  FILES=$(git diff --cached --name-only)
  DIFF_CMD="git diff --cached"
  echo "審查已暫存的變更"
elif [[ -n $(git diff --name-only) ]]; then
  # 未暫存的變更
  FILES=$(git diff --name-only)
  DIFF_CMD="git diff"
  echo "審查未暫存的變更"
else
  # 最近的 commits（相對於 main）
  FILES=$(git diff --name-only main)
  DIFF_CMD="git diff main"
  echo "審查相對於 main 的變更"
fi

echo "變更檔案:"
echo "$FILES"
```

### Step 2：執行程式碼審查

對每個檔案進行以下檢查：

#### 2.1 程式碼品質檢查

| 檢查項目 | 說明 |
|---------|------|
| 命名規範 | 變數、函數、組件命名是否清晰 |
| 程式碼結構 | 函數是否過長、邏輯是否清晰 |
| 重複程式碼 | 是否有可抽取的重複邏輯 |
| 註解品質 | 複雜邏輯是否有適當註解 |
| Magic Numbers | 是否有未定義的魔術數字 |

#### 2.2 Vue/TypeScript 特定檢查

| 檢查項目 | 說明 |
|---------|------|
| Props 定義 | 是否有完整的 type 和 default |
| Emits 定義 | 是否有定義 emits |
| Composition API | setup 函數是否過長 |
| Reactive 使用 | ref/reactive 使用是否正確 |
| 生命週期 | 是否有正確清理 (onUnmounted) |
| TypeScript | 是否有 any 類型需要修正 |

#### 2.3 安全性檢查

| 檢查項目 | 說明 |
|---------|------|
| XSS | 是否有 v-html 或未處理的用戶輸入 |
| 敏感資訊 | 是否有硬編碼的密鑰或密碼 |
| API 呼叫 | 是否有適當的錯誤處理 |
| 權限檢查 | 是否有遺漏的權限驗證 |

#### 2.4 效能檢查

| 檢查項目 | 說明 |
|---------|------|
| 不必要的渲染 | computed 是否可優化 |
| 記憶體洩漏 | 是否有未清理的監聽器 |
| 大型依賴 | 是否引入過大的套件 |
| 懶加載 | 是否有可延遲載入的組件 |

### Step 3：輸出審查報告

```markdown
## 📋 Code Review Report

### 📁 變更檔案
- file1.vue
- file2.ts

### ✅ 通過項目
- 命名規範良好
- 邏輯清晰

### ⚠️ 建議改進
1. **[file.vue:42]** 建議將此邏輯抽取為 composable
2. **[file.ts:15]** 變數命名可更清晰

### ❌ 需要修正
1. **[file.vue:78]** 潛在 XSS 風險：使用 v-html
2. **[file.ts:23]** 缺少錯誤處理

### 📊 總結
- 檔案數：X
- 問題數：X (嚴重: X, 建議: X)
- 整體評分：⭐⭐⭐⭐☆
```

---

## Mode 2: PR Review（Pull Request 審查）

### Step 1：取得 PR 資訊

```bash
# 取得 PR 詳情
if [[ -n "$PR_NUMBER" ]]; then
  PR_INFO=$(gh pr view "$PR_NUMBER" --json title,body,files,additions,deletions,author)
else
  PR_INFO=$(gh pr view --json title,body,files,additions,deletions,author)
fi

echo "PR 資訊:"
echo "$PR_INFO" | jq '.'

# 取得變更檔案列表
FILES=$(echo "$PR_INFO" | jq -r '.files[].path')
echo "變更檔案:"
echo "$FILES"
```

### Step 2：取得 PR Diff

```bash
# 取得完整 diff
if [[ -n "$PR_NUMBER" ]]; then
  gh pr diff "$PR_NUMBER"
else
  gh pr diff
fi
```

### Step 3：執行審查

除了 Local Review 的所有檢查項目外，額外檢查：

#### PR 特定檢查

| 檢查項目 | 說明 |
|---------|------|
| PR 標題 | 是否符合 Conventional Commits |
| PR 描述 | 是否清楚說明變更內容 |
| 變更範圍 | 是否過大需要拆分 |
| 測試覆蓋 | 是否有新增對應的測試 |
| Breaking Changes | 是否有破壞性變更需標註 |

### Step 4：輸出 PR 審查報告

```markdown
## 📋 PR Review Report

### 📌 PR 資訊
- **標題**: fix(vsaas-portal): fix user table sorting
- **作者**: @username
- **變更**: +120 -45

### 📁 變更檔案 (5)
- src/pages/user/index.vue
- src/pages/user/components/UserTable.vue
- src/store/user/actions.js
- src/store/user/getters.js
- src/pages/user/index.test.js

### ✅ 通過項目
- PR 標題符合規範
- 有對應的測試更新
- 變更範圍適當

### ⚠️ 建議改進
1. **[UserTable.vue:42]** 建議使用 computed 取代 method
2. 建議在 PR 描述中補充變更原因

### ❌ 需要修正
1. **[actions.js:78]** 缺少 API 錯誤處理
2. **[index.vue:156]** 未使用的 import

### 💬 Review 評語

整體變更品質良好，邏輯清晰。請處理上述需要修正的項目後即可合併。

### 📊 總結
- 檔案數：5
- 問題數：4 (嚴重: 2, 建議: 2)
- 建議：**Request Changes** / Approve / Comment
```

---

## 🚀 提交 Review 到 GitHub（--submit 模式）

當使用 `--submit` 參數時，會直接將 review comments 提交到 GitHub PR 上的對應行數。

### ⚠️ 重要限制

1. **不能 Approve 自己的 PR**：GitHub 不允許 approve 自己的 pull request，會報錯 `Can not approve your own pull request`
2. **每個用戶只能有一個 Pending Review**：如果有未提交的 pending review，需要先刪除才能建立新的
3. **行號必須在 diff 範圍內**：`line` 參數必須是 diff 中實際變更的行號

### 🎯 推薦方式：一次性提交（JSON Payload）

這是最可靠的方式，一次性提交 review body + 所有行內評論：

```bash
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
COMMIT_ID=$(gh pr view $PR_NUMBER --json headRefOid -q '.headRefOid')

# 建立 JSON payload 檔案
cat > /tmp/review_payload.json << 'JSONEOF'
{
  "commit_id": "YOUR_COMMIT_ID",
  "body": "## Code Review Summary\n\n### ✅ 通過項目\n- 邏輯正確\n- 程式碼品質良好\n\n**評分**: ⭐⭐⭐⭐⭐",
  "event": "COMMENT",
  "comments": [
    {
      "path": "src/pages/user/index.vue",
      "line": 42,
      "side": "RIGHT",
      "body": "👍 這個實作很優雅"
    },
    {
      "path": "src/store/user/actions.js",
      "line": 78,
      "side": "RIGHT",
      "body": "💡 建議加上錯誤處理"
    }
  ]
}
JSONEOF

# 替換 commit_id
sed -i '' "s/YOUR_COMMIT_ID/$COMMIT_ID/" /tmp/review_payload.json

# 提交 review
gh api \
  -X POST \
  "/repos/${REPO}/pulls/${PR_NUMBER}/reviews" \
  --input /tmp/review_payload.json
```

### event 類型說明

| Event | 說明 | 使用時機 |
|-------|------|---------|
| `COMMENT` | 僅評論 | 一般 review、自己的 PR |
| `APPROVE` | 通過 | 確認可合併（不能用於自己的 PR）|
| `REQUEST_CHANGES` | 要求修改 | 有問題需要修正（不能用於自己的 PR）|

### 簡化版：使用 gh pr review（無行內評論）

對於簡單的整體評論（不含行內評論）：

```bash
# 通過 PR（不能用於自己的 PR）
gh pr review $PR_NUMBER --approve -b "LGTM! 程式碼品質良好。"

# 要求修改（不能用於自己的 PR）
gh pr review $PR_NUMBER --request-changes -b "請修正上述問題後再合併。"

# 僅評論（可用於自己的 PR）
gh pr review $PR_NUMBER --comment -b "有幾個建議，請參考。"
```

### 完整實戰範例

```bash
#!/bin/bash
# submit-review.sh - 提交帶有行內評論的 review 到 GitHub

PR_NUMBER=$1
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
COMMIT_ID=$(gh pr view $PR_NUMBER --json headRefOid -q '.headRefOid')

echo "📝 準備 Review..."
echo "PR: #$PR_NUMBER"
echo "Repo: $REPO"
echo "Commit: $COMMIT_ID"

# 建立 JSON payload
cat > /tmp/review_payload.json << JSONEOF
{
  "commit_id": "${COMMIT_ID}",
  "body": "## Code Review Summary\n\n### ✅ 通過項目\n\n1. 邏輯正確\n2. 程式碼品質良好\n\n### 📊 總結\n\nLGTM!\n\n**評分**: ⭐⭐⭐⭐⭐",
  "event": "COMMENT",
  "comments": [
    {
      "path": "src/pages/user/index.vue",
      "line": 42,
      "side": "RIGHT",
      "body": "👍 這個實作很優雅"
    },
    {
      "path": "src/store/user/actions.js",
      "line": 78,
      "side": "RIGHT",
      "body": "💡 建議加上錯誤處理：\n\n\`\`\`javascript\ntry {\n  await apiCall();\n} catch (error) {\n  handleError(error);\n}\n\`\`\`"
    }
  ]
}
JSONEOF

# 提交 review
echo "🚀 提交 Review..."
gh api \
  -X POST \
  "/repos/${REPO}/pulls/${PR_NUMBER}/reviews" \
  --input /tmp/review_payload.json > /dev/null

echo "✅ Review 已提交到 PR #${PR_NUMBER}"
```

### 疑難排解

#### 問題 1：`user_id can only have one pending review per pull request`

**原因**：你有一個未提交的 pending review

**解決方式**：
```bash
# 查看 pending reviews
gh api "/repos/${REPO}/pulls/${PR_NUMBER}/reviews" --jq '.[] | select(.state == "PENDING") | {id, state}'

# 刪除 pending review
gh api -X DELETE "/repos/${REPO}/pulls/${PR_NUMBER}/reviews/REVIEW_ID"
```

#### 問題 2：`Can not approve your own pull request`

**原因**：不能 approve 自己的 PR

**解決方式**：改用 `"event": "COMMENT"`

#### 問題 3：`line must be part of the diff`

**原因**：指定的行號不在 diff 變更範圍內

**解決方式**：
```bash
# 查看 PR diff 確認行號
gh pr diff $PR_NUMBER
```

### 評論格式建議

使用 emoji 標記問題嚴重程度：

| Emoji | 意義 | 範例 |
|-------|------|------|
| ❌ | 必須修正 | `❌ 安全風險：未驗證用戶輸入` |
| ⚠️ | 建議改進 | `⚠️ 建議抽取為 composable` |
| 💡 | 小建議 | `💡 可以考慮使用 optional chaining` |
| ❓ | 疑問 | `❓ 這個邏輯的目的是什麼？` |
| 👍 | 稱讚 | `👍 這個實作很優雅！` |

### 多行評論（Multi-line Comments）

對於跨多行的評論，在 JSON payload 的 comments 中加入 `start_line`：

```json
{
  "path": "src/components/Table.vue",
  "start_line": 42,
  "line": 50,
  "start_side": "RIGHT",
  "side": "RIGHT",
  "body": "這整段邏輯可以簡化"
}
```

---

## 審查清單（Checklist）

### 通用清單

```markdown
## Code Review Checklist

### 功能性
- [ ] 程式碼符合需求
- [ ] 邊界條件已處理
- [ ] 錯誤處理完善

### 程式碼品質
- [ ] 命名清晰有意義
- [ ] 函數職責單一
- [ ] 沒有重複程式碼
- [ ] 適當的註解

### 安全性
- [ ] 無 XSS 風險
- [ ] 無敏感資訊外洩
- [ ] 適當的權限檢查

### 效能
- [ ] 無不必要的渲染
- [ ] 無記憶體洩漏風險
- [ ] 合理的資源使用

### 測試
- [ ] 有對應的單元測試
- [ ] 測試覆蓋主要場景

### 文件
- [ ] PR 描述清楚
- [ ] 必要的 README 更新
```

### VSaaS Portal 特定清單

```markdown
## VSaaS Portal Review Checklist

### Vue 組件
- [ ] Props 有完整定義 (type, default, validator)
- [ ] Emits 有定義
- [ ] 使用 Composition API 最佳實踐
- [ ] 組件命名符合規範 (PascalCase)

### Store
- [ ] Actions 有錯誤處理
- [ ] State 結構合理
- [ ] 使用正確的 mutation 模式

### 路由
- [ ] 路由守衛正確
- [ ] 權限檢查完整
- [ ] 懶加載配置正確

### 樣式
- [ ] 使用 Less 變數 (顏色、間距)
- [ ] Scoped 樣式
- [ ] 響應式設計

### 國際化
- [ ] 所有文字使用 $t()
- [ ] 翻譯 key 有意義
```

---

## 常用審查指令

```bash
# 查看變更的檔案
git diff --name-only main

# 查看特定檔案的變更
git diff main -- path/to/file.vue

# 查看 staged 變更
git diff --cached

# 取得 PR 資訊
gh pr view [number]

# 取得 PR diff
gh pr diff [number]

# 列出 PR 的變更檔案
gh pr view [number] --json files -q '.files[].path'

# 查看 PR 的 review 狀態
gh pr view [number] --json reviews
```

---

## 自動化輔助

### ESLint 檢查

```bash
# 對變更檔案執行 ESLint
git diff --name-only main -- '*.vue' '*.ts' '*.js' | xargs pnpm eslint
```

### TypeScript 檢查

```bash
# 執行 TypeScript 編譯檢查
cd packages/app-vsaas-portal
pnpm vue-tsc --noEmit
```

### 單元測試

```bash
# 執行相關測試
cd packages/app-vsaas-portal
pnpm test:unit --related
```

---

## 輸出格式選項

### 格式 1：詳細報告（預設）
完整的 Markdown 報告，包含所有檢查項目。

### 格式 2：簡潔清單
```
✅ 3 passed | ⚠️ 2 warnings | ❌ 1 error

Errors:
- [file.vue:78] XSS risk: v-html usage

Warnings:
- [file.ts:15] Consider extracting to composable
- [file.vue:42] Unclear variable name
```

### 格式 3：GitHub Comment（PR Review）
直接產生可貼到 GitHub PR 的 comment 格式。

---

## 整合其他 Skills

此 skill 可搭配以下 skills 使用：

| Skill | 用途 |
|-------|------|
| `/pr` | Review 完成後建立 PR |
| `vsaas-components` | 確認組件使用正確性 |
| `vsaas-architecture` | 確認架構模式正確 |

---

## 注意事項

1. **審查範圍**：一次審查不要超過 400 行變更，過大的 PR 建議拆分
2. **建設性回饋**：提供具體的改進建議，而非只指出問題
3. **優先級**：先處理安全性和功能性問題，再處理程式碼風格
4. **上下文**：考慮變更的業務背景，避免過度設計

---

## 更新日誌

### v1.3.0 (2025-02-01)
- **新增輸出模式選擇**：預設僅輸出報告，不自動提交到 GitHub
- 新增 `--copy` 參數：複製報告到剪貼簿
- 新增 `--approve` / `--request-changes` 參數：指定 review 類型
- 新增決策流程圖：幫助判斷何時使用哪種模式
- 適用於 reviewer 不是自己的情況

### v1.2.0 (2025-02-01)
- **重大更新**：改用 JSON payload 一次性提交 review + comments（更可靠）
- 新增重要限制說明（不能 approve 自己的 PR、pending review 限制）
- 新增疑難排解章節
- 新增 event 類型說明表格
- 修正錯誤的 API 用法

### v1.1.0 (2025-02-01)
- 新增 `--submit` 模式：直接提交 review 到 GitHub PR
- 支援行內評論（line comments）
- 支援多行評論（multi-line comments）
- 新增評論格式建議（emoji 標記）

### v1.0.0 (2025-02-01)
- 初始版本
- 支援 Local Review 和 PR Review 兩種模式
- 包含通用和 VSaaS Portal 特定檢查清單
