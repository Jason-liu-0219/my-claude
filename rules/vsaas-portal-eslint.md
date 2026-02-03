# VSaaS Portal ESLint 規範

**適用專案：** `webtech-monorepo/packages/app-vsaas-portal`

當編寫或修改此專案內的 JavaScript/Vue 程式碼時，遵守以下規則：

## Import 規則

- **不要加 `.js` 副檔名**：使用 `import x from '@/utils/file'` 而非 `'@/utils/file.js'`
- **單一 export 使用 default export**：當檔案只有一個 export 時，使用 `export default` 而非 `export const`

## 程式碼風格

- **if 語句必須使用大括號**，即使只有一行：
  ```javascript
  // 正確
  if (condition) {
    return value;
  }

  // 錯誤
  if (condition) return value;
  ```

- **if 後的語句必須換行**：
  ```javascript
  // 正確
  if (condition) {
    doSomething();
  }

  // 錯誤
  if (condition) { doSomething(); }
  ```

## 其他重要規則

- 使用 single quotes（單引號）
- 結尾加分號
- 最大行寬 120 字元
- camelCase 命名（但不強制）
- 禁止 `console.log`，只能用 `console.warn` 或 `console.error`
