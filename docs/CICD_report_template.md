# 學號*姓名\_CICD*作業

## 1. CI Pipeline 說明

本次作業在專案中新增 `.github/workflows/ci_reson320.yaml`，讓 GitHub Actions 在每次 `push` 到任一分支時自動執行 CI Pipeline。Pipeline 使用 Node.js 22，先透過 `npm ci` 安裝 `package-lock.json` 鎖定的依賴，再依序執行 TypeScript typecheck、Prettier 格式檢查與 Vitest 測試。

若 TypeScript 型別錯誤、Prettier 格式錯誤或測試失敗，對應步驟會回傳非零 exit code，因此 GitHub Actions job 會顯示 failed，符合「任一檢查失敗時 pipeline 應顯示失敗」的要求。

測試步驟使用 Vitest 的 JUnit reporter 輸出 `reports/vitest-junit.xml`，再透過 `EnricoMi/publish-unit-test-result-action@v2` 將測試結果發布到 GitHub Actions 結果頁面，同時使用 `actions/upload-artifact@v7` 上傳測試報告 artifact，方便在執行紀錄中查看。

## 2. `.github/workflows/ci_reson320.yaml` 主要內容

```yaml
name: CI Homework

on:
  push:
    branches:
      - '**'

permissions:
  contents: read
  checks: write
  pull-requests: write

jobs:
  ci:
    name: Typecheck, format, and test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Setup Node.js
        uses: actions/setup-node@v5
        with:
          node-version: '22'
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: TypeScript typecheck
        run: npm run typecheck

      - name: Prettier check
        run: npm run format:check

      - name: Run tests with JUnit report
        run: |
          mkdir -p reports
          npm test -- --reporter=default --reporter=junit --outputFile.junit=reports/vitest-junit.xml

      - name: Publish test results
        if: always() && hashFiles('reports/vitest-junit.xml') != ''
        uses: EnricoMi/publish-unit-test-result-action@v2
        with:
          files: reports/vitest-junit.xml
          check_name: Vitest Test Results

      - name: Upload test report artifact
        if: always() && hashFiles('reports/vitest-junit.xml') != ''
        uses: actions/upload-artifact@v7
        with:
          name: vitest-junit-report
          path: reports/vitest-junit.xml
          if-no-files-found: warn
```

## 3. CI 執行結果截圖

請在 GitHub repository 的 `Actions` 頁面選擇成功的 workflow run，截圖後貼在此處。

成功案例截圖：

`[請貼上 GitHub Actions 成功執行截圖]`

成功案例中應可看到：

- Workflow 名稱為 `CI Homework`
- Job `Typecheck, format, and test` 成功
- `TypeScript typecheck`、`Prettier check`、`Run tests with JUnit report` 均成功
- Actions 結果頁面顯示 `Vitest Test Results`
- Artifact 中有 `vitest-junit-report`

## 4. 失敗案例說明

本次故意製造的錯誤建議使用「測試失敗」。可在 `test/app.test.ts` 中暫時將以下測試期待值：

```ts
expect(response.statusCode).toBe(200);
```

改成：

```ts
expect(response.statusCode).toBe(500);
```

接著 commit 並 push 到 GitHub，GitHub Actions 會重新執行。因為 `/health` API 實際回傳 HTTP 200，但測試期待值被改成 500，Vitest 會判定測試失敗，`Run tests with JUnit report` 步驟回傳失敗，因此整個 CI Pipeline 會顯示 failed。

失敗案例截圖：

`[請貼上 GitHub Actions failed 執行截圖]`

錯誤原因：

測試期待 `/health` 回傳狀態碼 500，但實際 API 正常回傳 200，造成 assertion failed。

修正方式：

將 `expect(response.statusCode).toBe(500);` 改回 `expect(response.statusCode).toBe(200);`，重新 commit 並 push 後，CI Pipeline 會恢復成功。

## 5. 使用工具與策略

本次 CI 採用 GitHub Actions 實作，並搭配專案既有 npm scripts：

- `npm run typecheck`：使用 TypeScript compiler 檢查型別。
- `npm run format:check`：使用 Prettier 檢查程式碼格式。
- `npm test`：使用 Vitest 執行單元測試。
- `EnricoMi/publish-unit-test-result-action@v2`：解析 JUnit XML 並將測試結果顯示於 Actions 頁面。
- `actions/upload-artifact@v7`：保留 JUnit 測試結果檔案供下載查看。

設計策略是讓每個檢查拆成獨立 step，使失敗原因能在 Actions 頁面清楚定位；同時測試結果使用標準 JUnit XML 格式輸出，方便 GitHub Actions 與其他 CI 工具整合。
