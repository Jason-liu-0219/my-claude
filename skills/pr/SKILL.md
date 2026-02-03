---
name: pr
description: Use when user says "發 PR", "create PR", "開 PR", "/pr", or wants to create a pull request for vsaas-portal changes. Automatically generates PR title, description, and assigns reviewers.
version: 2.3.0
---

# PR - 自動建立或更新 Pull Request

針對 `app-vsaas-portal` 專案自動建立或更新 PR，使用統一格式。

---

## PR 描述固定格式

**所有 PR 必須使用此格式**：

```markdown
## Summary
- [變更點 1]
- [變更點 2]
- [變更點 3]
```

### 範例

```markdown
## Summary
- Use `@optionClick` instead of `@update:modelValue` to trigger selection event
- This ensures the `selected` event fires even when clicking the same option
- Fixes batch assign dropdown not responding when re-selecting the same role
```

---

## 使用方式

```bash
/pr
```

---

## 執行步驟

### Step 1: 確認環境

```bash
CURRENT_BRANCH=$(git branch --show-current)

# 確認不是 main 分支
if [ "$CURRENT_BRANCH" = "main" ]; then
  echo "❌ 不能從 main 分支建立 PR"
  exit 1
fi

# 確認有變更
if [ -z "$(git diff --name-only origin/main)" ]; then
  echo "❌ 沒有相對於 main 的變更"
  exit 1
fi
```

### Step 2: 檢查是否已有 PR

```bash
# 檢查當前分支是否已有 PR
EXISTING_PR=$(gh pr view --json number,title,body 2>/dev/null)

if [ -n "$EXISTING_PR" ]; then
  echo "📝 發現已存在的 PR，將進行整合更新"
  # 進入 Step 3a: 整合模式
else
  echo "🆕 建立新的 PR"
  # 進入 Step 3b: 新建模式
fi
```

### Step 3a: 整合模式（PR 已存在）

**重要：不要覆蓋原有內容，而是整合新變更**

```bash
# 取得現有 PR 資訊
EXISTING_BODY=$(gh pr view --json body -q '.body')
EXISTING_TITLE=$(gh pr view --json title -q '.title')

# 分析新增的 commits（從上次 PR 建立後的變更）
NEW_COMMITS=$(git log origin/main..HEAD --format="%s" --not --remotes=origin/$(gh pr view --json headRefName -q '.headRefName') 2>/dev/null || git log origin/main..HEAD --format="%s")
```

**整合策略：**

1. **保留原有 Summary 內容**
2. **在 Summary 區塊新增新的變更點**

**整合範例：**

原有 PR body：
```markdown
## Summary
- Fix button spacing issue
```

新增變更後，整合為：
```markdown
## Summary
- Fix button spacing issue
- Add hover effect to button
- Update button color scheme
```

**執行更新：**

```bash
# 使用 gh pr edit 更新 PR
gh pr edit --body "$MERGED_BODY"

# 如果標題需要更新（可選）
# gh pr edit --title "$NEW_TITLE"
```

### Step 3b: 新建模式（無現有 PR）

生成 PR 標題（使用第一個 commit 訊息）：

```bash
FIRST_COMMIT=$(git log origin/main..HEAD --format="%s" | tail -1)
PR_TITLE="$FIRST_COMMIT"
```

生成 PR 描述：

```bash
PR_BODY="$(cat <<'EOF'
## Summary
- [根據變更內容填寫]
EOF
)"
```

### Step 4: 推送並建立/更新 PR

```bash
# 推送分支
git push -u origin "$CURRENT_BRANCH"

# 根據模式執行
if [ -n "$EXISTING_PR" ]; then
  # 更新現有 PR
  gh pr edit --body "$MERGED_BODY"
  echo "✅ PR 已更新"
else
  # 建立新 PR
  gh pr create --base main --title "$PR_TITLE" --body "$PR_BODY"
  echo "✅ PR 已建立"
fi
```

---

## PR 標題格式

遵循 Conventional Commits：

```
type(scope): description
```

| Type | 說明 | 範例 |
|------|------|------|
| feat | 新功能 | `feat(vsaas-portal): add user filter` |
| fix | 修復 | `fix(vsaas-portal): fix dropdown reselect issue` |
| docs | 文件 | `docs(vsaas-portal): update README` |
| refactor | 重構 | `refactor(vsaas-portal): simplify auth logic` |
| style | 格式 | `style(vsaas-portal): format code` |
| test | 測試 | `test(vsaas-portal): add unit tests` |
| chore | 維護 | `chore(vsaas-portal): update deps` |

---

## 注意事項

1. **格式必須統一**：所有 PR 使用相同的 Summary / Test plan 格式
2. **使用 HEREDOC**：確保 markdown 格式正確
3. **BREAKING CHANGE**：避免使用，會觸發 MAJOR 版本
4. **署名**：必須包含 `🤖 Generated with [Claude Code]`

---

## 更新日誌

### v2.3.0 (2026-02-01)
- 移除 Claude Code 署名

### v2.2.0 (2026-02-01)
- 移除 Test plan 區塊，簡化 PR 格式

### v2.1.0 (2026-02-01)
- **新增整合模式**：當 PR 已存在時，整合新變更而非覆蓋
- 新增 Step 2：檢查是否已有 PR
- 新增 Step 3a：整合模式處理邏輯
- 使用 `gh pr edit` 更新現有 PR
- 保留原有 Summary 內容

### v2.0.0 (2025-02-01)
- 統一 PR 描述格式：Summary + Test plan
- 移除 Checklist，改用 Test plan
- 加入 Claude Code 署名
- 簡化流程，移除複雜的 reviewer 邏輯

### v1.0.0
- 初始版本
