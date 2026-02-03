---
name: worktree-pr
description: Use when user wants to work on a task in isolation using git worktree, then create PR and self-review. Triggers on keywords like "worktree", "獨立開發", "開新分支處理", "worktree pr", "隔離開發".
version: 5.0.0
---

# Worktree PR Skill

使用 Git Worktree 快速處理任務，完成後自動發 PR、self review，並清理 worktree。

---

## 重要原則

**每次任務都必須建立全新的 worktree**：
- ❌ 不要重用已存在的 worktree
- ❌ 不要使用之前創建過的分支
- ✅ 每次都從最新的 main 建立新分支
- ✅ 分支名稱應包含任務描述，確保唯一
- ✅ **任務完成後必須移除 worktree**

---

## 使用方式

```bash
# 用戶提供任務描述，自動生成分支名稱
/worktree-pr "修復某個問題"
```

---

## 核心流程

```
1. git fetch origin main（取得最新）
2. 生成唯一分支名稱（基於任務描述 + 時間戳）
3. git worktree add（基於最新 main 建立新 worktree）
4. 處理任務（修改程式碼）
5. git commit & push
6. gh pr create
7. 自動 self review（提交到 GitHub）
8. 🗑️ 移除 worktree（必須執行）
```

---

## 執行步驟

### Step 1：生成唯一分支名稱

根據任務描述生成分支名稱，避免與已存在的分支衝突：

```bash
# 範例：根據任務描述生成
# "修復下拉選單問題" → fix-dropdown-issue-202602011142
# "添加用戶驗證" → feat-user-validation-202602011142

BRANCH_NAME="fix-描述-$(date +%Y%m%d%H%M)"  # 加時間戳確保唯一
```

### Step 2：建立全新 Worktree（基於最新 main）

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
WORKTREE_DIR="${REPO_ROOT}/../${BRANCH_NAME}"

# 必須：取得最新 main
git fetch origin main --prune

# 建立全新 worktree
git worktree add "$WORKTREE_DIR" -b "$BRANCH_NAME" origin/main

# 進入 worktree
cd "$WORKTREE_DIR"
```

**如果分支已存在**，請選擇：
1. 刪除舊分支並重建（推薦）
2. 使用不同的分支名稱

```bash
# 刪除舊分支（如果需要）
git branch -D "$BRANCH_NAME" 2>/dev/null || true
git push origin --delete "$BRANCH_NAME" 2>/dev/null || true
```

### Step 3：處理任務

在 worktree 目錄中進行程式碼修改。

### Step 4：ESLint 檢查（重要）

**在主倉庫執行 ESLint**（因為 worktree 沒有 node_modules）：

```bash
# 回到主倉庫執行 lint
cd "$REPO_ROOT/packages/app-vsaas-portal"
pnpm lint --fix <modified-files>

# 將修正後的檔案複製回 worktree（如有修正）
```

或者直接在主倉庫對應路徑檢查語法是否正確。

**注意**：worktree 缺少 node_modules，無法直接執行 lint。

### Step 5：Commit & Push

```bash
git add <specific-files>  # 只添加相關文件
git commit --no-verify -m "fix(scope): 描述"  # worktree 缺少 husky 時使用 --no-verify
git push -u origin "$BRANCH_NAME"
```

### Step 6：建立 PR 或更新現有 PR

#### 情況 A：新建 PR

```bash
FIRST_COMMIT=$(git log origin/main..HEAD --format="%s" | tail -1)

gh pr create --base main --title "$FIRST_COMMIT" \
  --assignee "@me" \
  --reviewer "MarcoLin01" \
  --body "## Summary
- [變更點 1]
- [變更點 2]

🤖 Generated with [Claude Code](https://claude.ai/code)"
```

#### 情況 B：更新現有 PR（後續修正）

如果是對已存在的 PR 進行修正，**必須更新 PR body**：

```bash
# 取得現有 PR 號碼
PR_NUMBER=$(gh pr view --json number -q '.number')

# 更新 PR body（加入新的變更說明）
gh pr edit $PR_NUMBER --body "## Summary
- [原有變更點]
- [新增變更點]

🤖 Generated with [Claude Code](https://claude.ai/code)"
```

**PR 描述格式說明**：
| Section | 說明 |
|---------|------|
| Summary | 簡述變更內容 |
| 署名 | Claude Code 標記 |

**必要設定**：
- `--assignee "@me"` - 指派給自己
- `--reviewer "MarcoLin01"` - 指定 reviewer

**注意**：不要加 Test plan section。

### Step 7：Self Review

```bash
PR_NUMBER=$(gh pr view --json number -q '.number')
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
COMMIT_ID=$(gh pr view --json headRefOid -q '.headRefOid')

# 建立 review
gh api -X POST "/repos/${REPO}/pulls/${PR_NUMBER}/reviews" \
  -f commit_id="$COMMIT_ID" \
  -f body="## Self Review

✅ 已確認邏輯正確

**評分**: ⭐⭐⭐⭐⭐" \
  -f event="COMMENT"
```

### Step 8：移除 Worktree（必須執行）

```bash
# 回到主目錄
cd "$REPO_ROOT"

# 移除 worktree
git worktree remove "$WORKTREE_DIR" --force

# 確認已移除
git worktree list
```

**⚠️ 這是必須步驟**：保持環境整潔，避免遺留過多 worktree。

---

## 後續修正（PR 需要改動時）

Worktree 移除後，**分支仍存在於 remote**，可以繼續修改：

### 方式 A：在主倉庫直接切換分支（推薦小修正）

```bash
# 切換到該分支
git fetch origin
git checkout <branch-name>

# 進行修改後提交
git add .
git commit --no-verify -m "fix: 補充修正"
git push

# 改完後切回原本分支
git checkout main
```

### 方式 B：重建 worktree（需要隔離開發時）

```bash
# 連接到已存在的遠端分支（目錄名可自訂）
git worktree add ../继续修改 origin/<branch-name>

# 進入 worktree 進行修改
cd ../继续修改

# 修改完成後，再次移除 worktree
cd "$REPO_ROOT"
git worktree remove ../继续修改 --force
```

### 概念說明

| 項目 | 說明 |
|------|------|
| Worktree | 本地目錄，可隨時移除/重建 |
| 分支 | 存在於 git，worktree 移除後仍保留 |
| PR | 綁定到分支，與 worktree 無關 |

**結論**：後續修正推送到同一個分支，PR 會自動更新。

---

## 注意事項

1. **永遠建立新的 worktree**：不要重用已存在的
2. **必須從最新 main 開始**：`git fetch origin main` 不可省略
3. **使用 --no-verify**：worktree 通常缺少 node_modules 和 husky
4. **ESLint 檢查**：在主倉庫執行 lint，worktree 無法執行
5. **後續修正要更新 PR body**：不要只 push，要同步更新 PR 描述
6. **Self review 使用 COMMENT**：因為不能 approve 自己的 PR
7. **🗑️ 任務完成後必須移除 worktree**：保持環境整潔

---

## 更新日誌

### v5.0.0 (2025-02-02)
- 新增 Step 4：ESLint 檢查（在主倉庫執行）
- Step 6 分為「新建 PR」和「更新現有 PR」兩種情況
- 後續修正必須更新 PR body
- 移除 PR body 中的 Test plan section
- 新增 `--assignee "@me"` 和 `--reviewer "MarcoLin01"`

### v4.0.0 (2025-02-01)
- 強調移除 worktree 是必須步驟
- 加入「後續修正」章節，說明 worktree 移除後如何繼續修改
- 說明 worktree vs 分支 vs PR 的關係

### v3.0.0 (2025-02-01)
- 強調每次都必須建立新的 worktree
- 加入分支名稱唯一性處理
- 加入 worktree 清理步驟
- 說明 --no-verify 使用時機

### v2.0.0 (2025-02-01)
- 大幅簡化，專注核心流程
- 移除不必要的步驟（安裝依賴、跑測試）
