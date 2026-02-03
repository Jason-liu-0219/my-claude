# PR Review Analyzer

分析 PR 上的 review comments，整理出需要修改的項目。

## 觸發條件

當使用者說：
- `/pr-comments`
- `/review-comments`
- `分析 PR comments`
- `看一下 PR 上的回饋`
- `PR 要改什麼`

## 使用方式

```
/pr-comments [PR number]
```

如果沒有提供 PR number，會自動偵測當前分支對應的 PR。

## 執行流程

### 1. 取得 PR 資訊

```bash
# 取得當前分支的 PR
gh pr view --json number,title,url

# 或指定 PR number
gh pr view <PR_NUMBER> --json number,title,url
```

### 2. 拉取所有 review comments

```bash
# 取得 PR reviews
gh pr view <PR_NUMBER> --json reviews --jq '.reviews[] | select(.state != "") | {author: .author.login, state: .state, body: .body}'

# 取得 review comments (行內評論)
gh api repos/{owner}/{repo}/pulls/<PR_NUMBER>/comments --jq '.[] | {path: .path, line: .line, body: .body, author: .user.login}'

# 取得一般 comments
gh pr view <PR_NUMBER> --json comments --jq '.comments[] | {author: .author.login, body: .body}'
```

### 3. 分析並分類 comments

將 comments 分類為：
- **必須修改** - reviewer 明確要求改動
- **建議修改** - 可選的改進建議
- **討論/問題** - 需要回覆或澄清的問題
- **已解決** - 標記為 resolved 的 comments

### 4. 輸出格式

```markdown
## PR #<number>: <title>

### 必須修改 (X 項)

1. **[檔案路徑:行號]** - @reviewer
   > comment 內容
   
   **建議做法：** 具體修改建議

### 建議修改 (X 項)

1. **[檔案路徑:行號]** - @reviewer
   > comment 內容

### 待討論 (X 項)

1. @reviewer 問：
   > 問題內容
   
   **可能的回覆：** 建議回覆方向

### 總結

- 必須修改：X 項
- 建議修改：X 項
- 待討論：X 項
```

## 進階用法

### 只看特定 reviewer 的 comments

```
/pr-comments --reviewer=<username>
```

### 只看未解決的 comments

```
/pr-comments --unresolved
```

### 自動生成修改計畫

```
/pr-comments --plan
```

這會額外生成一個修改計畫，包含：
- 修改順序建議
- 預估每項修改的複雜度
- 相關檔案的依賴關係

## 範例

```bash
# 分析當前分支的 PR
/pr-comments

# 分析指定 PR
/pr-comments 6700

# 分析並生成修改計畫
/pr-comments --plan
```
