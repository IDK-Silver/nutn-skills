# agent-browser 瀏覽器自動化

> **語言規範**：本文件使用**繁體中文**撰寫。

## 安裝

```bash
npm install -g agent-browser
agent-browser install  # 下載 Chromium
```

若未安裝，請先閱讀 `./skills/cloud-setup/SKILL.md` 進行環境設定。

---

## 執行環境

### 環境類型判斷

執行以下指令判斷環境類型：

```bash
[ -f ~/.nvm/nvm.sh ] && echo "LOCAL" || echo "CLOUD"
```

### 雲端環境（Claude Code Cloud）

在雲端環境中，**直接執行**即可，不需要 zsh 或 source：

```bash
agent-browser open https://example.com --headed
agent-browser snapshot -i
agent-browser close
```

### 本地環境（使用 nvm/zsh）

**重要**：agent-browser 透過 nvm 安裝，必須使用 Desktop Commander 的 `start_process` 搭配 zsh shell 來載入環境：

```
Desktop Commander:start_process
  command: source ~/.zshrc 2>/dev/null; agent-browser open https://example.com --headed
  shell: /bin/zsh
  timeout_ms: 15000
```

**本地環境所有 agent-browser 命令都必須：**
1. 加上 `source ~/.zshrc 2>/dev/null;` 前綴
2. 指定 `shell: /bin/zsh`

---

## 核心工作流程

根據環境類型選擇正確的執行方式：

### 雲端環境

```bash
agent-browser open <url>        # 1. 導航到頁面
agent-browser snapshot -i       # 2. 取得互動元素與 ref
agent-browser click @e1         # 3. 使用 ref 操作元素
agent-browser fill @e2 "text"   # 4. 填寫表單
agent-browser close             # 5. 關閉瀏覽器
```

### 本地環境

```bash
agent-browser open <url>        # 1. 導航到頁面
agent-browser snapshot -i       # 2. 取得互動元素與 ref
agent-browser click @e1         # 3. 使用 ref 操作元素
agent-browser fill @e2 "text"   # 4. 填寫表單
agent-browser close             # 5. 關閉瀏覽器
```

**重要**：DOM 變化後必須重新執行 `snapshot` 取得新的 ref。

---

## 常用命令

### 導航

```bash
agent-browser open <url>      # 導航到 URL
agent-browser back            # 上一頁
agent-browser forward         # 下一頁
agent-browser reload          # 重新載入
agent-browser close           # 關閉瀏覽器
```

### 頁面快照

```bash
agent-browser snapshot        # 完整 accessibility tree
agent-browser snapshot -i     # 僅互動元素（推薦）
agent-browser snapshot -c     # 精簡輸出
agent-browser snapshot -d 3   # 限制深度為 3 層
```

### 元素互動（使用 snapshot 的 @ref）

```bash
agent-browser click @e1           # 點擊
agent-browser dblclick @e1        # 雙擊
agent-browser fill @e2 "text"     # 清除並輸入
agent-browser type @e2 "text"     # 直接輸入（不清除）
agent-browser press Enter         # 按鍵
agent-browser press Control+a     # 組合鍵
agent-browser hover @e1           # 滑鼠懸停
agent-browser check @e1           # 勾選 checkbox
agent-browser uncheck @e1         # 取消勾選
agent-browser select @e1 "value"  # 選擇下拉選單
agent-browser scroll down 500     # 捲動頁面
agent-browser scrollintoview @e1  # 捲動至元素可見
```

### 取得資訊

```bash
agent-browser get text @e1        # 取得元素文字
agent-browser get value @e1       # 取得輸入值
agent-browser get title           # 取得頁面標題
agent-browser get url             # 取得目前 URL
```

### 截圖與 PDF

```bash
agent-browser screenshot          # 截圖到 stdout
agent-browser screenshot path.png # 儲存到檔案
agent-browser screenshot --full   # 完整頁面截圖
agent-browser pdf path.pdf        # 儲存為 PDF（注意：不支援自訂紙張大小）
```

**PDF 限制**：`agent-browser pdf` 命令不支援指定紙張大小（如 A4）。如需 A4 格式，請參考下方「進階：自訂格式 PDF」章節。

### 等待

```bash
agent-browser wait @e1                     # 等待元素出現
agent-browser wait 2000                    # 等待毫秒數
agent-browser wait --text "Success"        # 等待文字出現
agent-browser wait --load networkidle      # 等待網路閒置
```

### 語意定位器（ref 的替代方案）

```bash
agent-browser find role button click --name "Submit"
agent-browser find text "Sign In" click
agent-browser find label "Email" fill "user@test.com"
```

---

## 進階技巧

### eval 執行 JavaScript

`eval` 是處理複雜場景的關鍵命令，官方文件未詳述但非常實用：

```bash
agent-browser eval "<JavaScript 程式碼>"
```

**適用場景**：

1. **關閉 Modal Dialog**
   ```bash
   agent-browser eval "document.querySelector('#myModal button[data-dismiss=modal]').click()"
   ```

2. **調用頁面內部函數**
   ```bash
   agent-browser eval "DeleteEmployee()"
   ```

3. **繞過確認對話框**
   ```bash
   agent-browser eval "document.querySelector('#hiddenId').value = '123'; DeleteRecord();"
   ```

4. **檢查 Modal 是否存在**
   ```bash
   agent-browser eval "document.querySelector('.modal.in') ? 'modal visible' : 'no modal'"
   ```

5. **批次取得資料**
   ```bash
   agent-browser eval "Array.from(document.querySelectorAll('table tr')).map(r => r.textContent)"
   ```

### ref 不穩定的處理

有時 `click @ref` 會失敗，解決方案：

1. 重新執行 `snapshot` 取得新 ref
2. 改用 `eval` 直接操作 DOM：
   ```bash
   agent-browser eval "document.querySelector('button[type=submit]').click()"
   ```

### 批次操作的穩定寫法

**注意**：`eval` 中的 async/await 不可靠，改用逐筆執行 + `wait`：

```bash
# 錯誤：async/await 在 eval 中可能不執行
agent-browser eval "async function run() { ... } run();"

# 正確：逐筆執行
agent-browser eval "document.getElementById('field').value = 'value1'; submit();"
agent-browser wait 1500
agent-browser eval "document.getElementById('field').value = 'value2'; submit();"
agent-browser wait 1500
```

### 保存與載入登入狀態

避免每次都要重新登入：

```bash
# 登入後保存狀態
agent-browser state save auth.json

# 下次直接載入
agent-browser state load auth.json
agent-browser open https://example.com/dashboard
```

---

## 進階：自訂格式 PDF

`agent-browser pdf` 不支援自訂紙張大小。如需 A4 格式，使用 Node.js + Playwright：

```javascript
// save as /tmp/print_a4.mjs
import { chromium } from '/Users/yufeng/.nvm/versions/node/v22.14.0/lib/node_modules/agent-browser/node_modules/playwright-core/index.mjs';
import os from 'os';
import path from 'path';

const browser = await chromium.launch({ headless: true });
const context = await browser.newContext();
const page = await context.newPage();

// 導航到目標頁面（如需登入，先處理登入流程）
await page.goto('https://example.com/report');
await page.waitForLoadState('networkidle');

// 處理可能的 Modal
await page.evaluate(() => {
  const modal = document.querySelector('#myModal');
  if (modal) modal.style.display = 'none';
  const backdrop = document.querySelector('.modal-backdrop');
  if (backdrop) backdrop.remove();
});

// 生成 A4 PDF
await page.pdf({
  path: path.join(os.homedir(), 'Downloads', 'report_A4.pdf'),
  format: 'A4',
  printBackground: true,
  margin: { top: '10mm', bottom: '10mm', left: '10mm', right: '10mm' }
});

console.log('PDF saved');
await browser.close();
```

執行：
```bash
node /tmp/print_a4.mjs
```

---

## Session 管理（多瀏覽器並行）

```bash
agent-browser --session test1 open site-a.com
agent-browser --session test2 open site-b.com
agent-browser session list
```

---

## JSON 輸出（程式化處理）

```bash
agent-browser snapshot -i --json
agent-browser get text @e1 --json
```

---

## 除錯

```bash
agent-browser open example.com --headed  # 顯示瀏覽器視窗
agent-browser console                    # 檢視 console 訊息
agent-browser errors                     # 檢視頁面錯誤
```

---

## 範例：表單提交

```bash
agent-browser open https://example.com/form
agent-browser snapshot -i
# 輸出: textbox "Email" [ref=e1], textbox "Password" [ref=e2], button "Submit" [ref=e3]

agent-browser fill @e1 "user@example.com"
agent-browser fill @e2 "password123"
agent-browser click @e3
agent-browser wait --load networkidle
agent-browser snapshot -i  # 檢查結果
```

---

## 範例：處理確認對話框

許多網站的刪除操作需要確認，無法用單純的 click 處理：

```bash
# 1. 點擊刪除按鈕會彈出確認對話框
agent-browser click @e5

# 2. 檢查對話框內容
agent-browser eval "document.querySelector('.modal.in')?.innerHTML"

# 3. 直接調用刪除函數（繞過對話框）
agent-browser eval "document.querySelector('#hiddenId').value = '123'; DeleteRecord();"
```

---

## 常見問題

### agent-browser: command not found

確保已正確安裝：
```bash
npm install -g agent-browser
agent-browser --version
```

### PDF 不是 A4 大小

`agent-browser pdf` 不支援自訂紙張大小，請使用「進階：自訂格式 PDF」章節的 Node.js 腳本。

### 瀏覽器已關閉錯誤

如果出現 "Target page, context or browser has been closed" 錯誤，先關閉再重新開啟：
```bash
agent-browser close 2>/dev/null; sleep 1; agent-browser open https://example.com --headed
```
