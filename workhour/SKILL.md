# 工時填寫助手

> **語言規範**：本文件使用**繁體中文**撰寫。

## 前置需求

本技能使用 agent-browser 進行瀏覽器自動化，請先閱讀 `agent-browser/SKILL.md`。

## 核心原則

1. **必須用程式碼驗證星期幾** - 絕不靠觀察或猜測日期對應的星期
2. **自動決策** - 主動排除假日，信任使用者會提出修正
3. **一次性確認** - 生成完整計畫後一次確認，不分步驟詢問
4. **表格化呈現** - 日期清單用表格展示

---

## 執行環境

**重要**：agent-browser 需要透過 nvm 載入 Node.js 環境，必須使用 Desktop Commander 的 `start_process` 搭配 zsh shell：

```bash
# 正確寫法
Desktop Commander:start_process
  command: source ~/.zshrc 2>/dev/null; agent-browser open https://example.com --headed
  shell: /bin/zsh
  timeout_ms: 15000

# 錯誤寫法（會找不到 agent-browser）
Desktop Commander:start_process
  command: agent-browser open https://example.com --headed
  timeout_ms: 15000
```

**所有 agent-browser 命令都必須加上 `source ~/.zshrc 2>/dev/null;` 前綴並指定 `/bin/zsh` shell。**

---

## 工作流程

### 1. 初始確認與登入

**步驟 1：開啟瀏覽器並檢查登入狀態**

```bash
source ~/.zshrc 2>/dev/null; agent-browser close 2>/dev/null; sleep 1; agent-browser open https://gaweb.nutn.edu.tw/workhour/Budget --headed
```

```bash
source ~/.zshrc 2>/dev/null; agent-browser snapshot -i
```

檢查 snapshot 結果：
- 如果看到「計畫管理」連結 → 已登入，繼續步驟 2
- 如果看到「登入」按鈕和帳號密碼欄位 → 未登入，執行步驟 1-2

**步驟 1-2：自動登入**

```bash
# 點擊登入連結（如果在首頁）
source ~/.zshrc 2>/dev/null; agent-browser click @e2  # 登入連結的 ref

# 取得表單元素
source ~/.zshrc 2>/dev/null; agent-browser snapshot -i

# 填寫帳號密碼（預設都是身份證字號）
source ~/.zshrc 2>/dev/null; agent-browser fill @e5 "身份證字號"; agent-browser fill @e6 "身份證字號"

# 點擊登入按鈕（使用 eval 更穩定）
source ~/.zshrc 2>/dev/null; agent-browser eval "document.querySelector('input[type=submit], button[type=submit]').click()"

# 等待頁面載入
source ~/.zshrc 2>/dev/null; agent-browser wait 2000; agent-browser get url
```

登入成功後 URL 會變成 `/workhour/Budget`。

**處理登入後的 Modal 提示**

系統可能會顯示提示訊息的 Modal，用 eval 關閉：

```bash
source ~/.zshrc 2>/dev/null; agent-browser eval "document.querySelector('#myModal button[data-dismiss=modal]')?.click()"
```

### 2. 收集計畫資訊

詢問使用者：
- 計畫編號（例如：D115-03）
- 填寫時間範圍（例如：115/1/1 到 115/1/31）
- 是否有行事曆圖片（用於排除假日）

### 3. 選擇計畫並檢查歷史紀錄

```bash
# 取得計畫列表
source ~/.zshrc 2>/dev/null; agent-browser eval "
Array.from(document.querySelectorAll('table tbody tr')).slice(1).map(r => ({
  id: r.cells[0]?.textContent?.trim(),
  name: r.cells[1]?.textContent?.trim(),
  period: r.cells[2]?.textContent?.trim(),
  status: r.cells[3]?.textContent?.trim()
}));
"
```

```bash
# 點擊對應計畫的「選擇」連結
source ~/.zshrc 2>/dev/null; agent-browser click @eXX  # 根據 snapshot 結果找到對應的 ref

# 等待頁面載入
source ~/.zshrc 2>/dev/null; agent-browser wait 1500; agent-browser snapshot
```

#### 分析歷史紀錄

**關鍵：必須用程式碼計算星期幾（禁止觀察猜測）**

```bash
source ~/.zshrc 2>/dev/null; agent-browser eval "
// 取得歷史紀錄
const rows = Array.from(document.querySelectorAll('table tbody tr')).slice(1);
const history = rows.map(r => ({
  date: r.cells[0]?.textContent?.trim(),
  start: r.cells[1]?.textContent?.trim(),
  end: r.cells[2]?.textContent?.trim(),
  hours: r.cells[3]?.textContent?.trim()
})).filter(h => h.date && h.date.includes('/'));

// 用程式碼計算星期幾
const weekdays = ['日', '一', '二', '三', '四', '五', '六'];
const analyzed = history.map(h => {
  const parts = h.date.split('/').map(Number);
  if (parts.length === 3) {
    const [rocYear, month, day] = parts;
    const adYear = rocYear + 1911;
    const date = new Date(adYear, month - 1, day);
    return {
      ...h,
      weekday: date.getDay(),
      weekdayName: weekdays[date.getDay()]
    };
  }
  return h;
});

// 統計工作日模式
const weekdayCounts = {};
analyzed.forEach(item => {
  if (item.weekdayName) {
    weekdayCounts[item.weekdayName] = (weekdayCounts[item.weekdayName] || 0) + 1;
  }
});

// 統計時段模式
const timeSlots = {};
analyzed.forEach(item => {
  const slot = item.start + '-' + item.end;
  timeSlots[slot] = (timeSlots[slot] || 0) + 1;
});

JSON.stringify({ 
  recentRecords: analyzed.slice(0, 20),
  weekdayCounts, 
  timeSlots,
  total: analyzed.length 
}, null, 2);
"
```

- **禁止**：直接觀察日期就說「看起來是星期X」
- **必須**：用程式碼計算後才能確認工作日模式
- 分析歷史記錄的時間區間
- 向使用者展示分析結果（用表格呈現）

#### 如果目標計畫沒有歷史紀錄（新計畫或新年度）

**重要**：查找同名的舊計畫歷史紀錄作為參考。

例如：D115-03 沒有歷史，但 D114-03 有相同名稱「【B版】回流教育學位班經費(碩士班)」

1. 回到計畫列表頁面：
   ```bash
   source ~/.zshrc 2>/dev/null; agent-browser open https://gaweb.nutn.edu.tw/workhour/Budget
   ```

2. 找出同名舊計畫並選擇：
   ```bash
   source ~/.zshrc 2>/dev/null; agent-browser click @eXX  # 舊計畫的選擇連結
   ```

3. 分析舊計畫的歷史紀錄（使用上方相同的 eval 程式碼）

4. 取得工作模式後，再切回新計畫進行填寫

**如果找不到任何相關歷史紀錄**：詢問使用者以下資訊：
- 工作日（星期幾）
- 每日時間區間（可以有多個，例如：17:40-20:40 和 21:10-23:10）

### 4. 檢查頁面預設值

```bash
source ~/.zshrc 2>/dev/null; agent-browser eval "
const checkedBoxes = Array.from(document.querySelectorAll('input[type=\"checkbox\"]'))
  .map(cb => ({id: cb.id, name: cb.name, checked: cb.checked, label: cb.parentElement?.textContent?.trim()}))
  .filter(item => item.checked);

const wcontent = document.getElementById('Wcontent')?.value || '';
const wother = document.getElementById('Wother')?.value || '';

({checkedBoxes, wcontent, wother});
"
```

**如果頁面有預設勾選和內容**：記錄預設狀態，直接採用（不詢問使用者）。

**如果頁面沒有任何勾選**：詢問使用者工作內容分類：
- 研究學習 (checkbox1)
- 資料處理 (checkbox2)
- 統計調查 (checkbox3)
- 實地考察 (checkbox4)
- 行政事務 (checkbox5)
- 其他 (checkbox6) - 需填寫具體內容

### 5. 處理假日

**重要：必須載入 `academic-calendar` 技能取得假日資訊**

#### 如果使用者提供日曆資料（圖片）：
- 分析日曆，識別紅色標記的放假日
- 自動排除這些日期（不詢問使用者）
- 在最終確認時列出已排除的日期

#### 如果沒有日曆資料：
- **載入 `academic-calendar/SKILL.md` 取得學校行事曆**
- 從行事曆識別寒假、暑假、國定假日等日期
- 自動排除這些日期
- 在最終確認時列出已排除的日期並詢問是否還有需要排除的

### 6. 生成填寫計畫

#### 自動生成日期清單

```bash
source ~/.zshrc 2>/dev/null; agent-browser eval "
function generateWorkDates(startRoc, endRoc, workWeekdays, holidays = []) {
  const results = [];
  const weekdayNames = ['日', '一', '二', '三', '四', '五', '六'];
  
  const [startY, startM, startD] = startRoc.split('/').map(Number);
  const [endY, endM, endD] = endRoc.split('/').map(Number);
  
  let current = new Date(startY + 1911, startM - 1, startD);
  const end = new Date(endY + 1911, endM - 1, endD);
  
  while (current <= end) {
    const rocDate = (current.getFullYear() - 1911) + '/' + (current.getMonth() + 1) + '/' + current.getDate();
    const weekday = current.getDay();
    
    if (workWeekdays.includes(weekday) && !holidays.includes(rocDate)) {
      results.push({
        date: rocDate,
        weekday: weekday,
        weekdayName: weekdayNames[weekday]
      });
    }
    current.setDate(current.getDate() + 1);
  }
  return results;
}

// 範例：星期一(1)和星期四(4)，排除假日
const dates = generateWorkDates('115/1/1', '115/1/31', [1, 4], ['115/1/1', '115/1/11']);
JSON.stringify(dates, null, 2);
"
```

### 7. 一次性最終確認

**用表格呈現完整計畫，一次確認所有資訊：**

```
## 填寫計畫確認

### 基本資訊
- **計畫**：D115-03 【B版】回流教育學位班經費(碩士班)
- **工作日模式**：星期一、星期四（從 D114-03 歷史分析）
- **工作類別**：其他 - 協助管理榮譽校區教室

### 填寫清單

| # | 日期 | 星期 | 時段 1 | 時段 2 |
|---|------|------|--------|--------|
| 1 | 115/1/5 | 一 | 17:40-20:40 | 21:10-23:10 |
| 2 | 115/1/8 | 四 | 17:40-20:40 | 21:10-23:10 |
| ... | ... | ... | ... | ... |

### 已排除日期
- **國定假日**：115/1/1 開國紀念日
- **寒假**：115/1/11 起

### 統計
- **總天數**：X 天
- **總筆數**：X 筆（每天 2 個時段）
- **總時數**：X 小時

---
確認以上資訊無誤後開始填寫？
```

**注意**：不分步驟詢問，一次呈現所有資訊讓使用者確認。

### 8. 批次填寫

**重要**：eval 中的 async/await 不可靠，必須逐筆執行並加入 wait：

```bash
# 第 1 筆
source ~/.zshrc 2>/dev/null; agent-browser eval "
document.getElementById('WorkDay').value = '115/1/5';
document.getElementById('StartT').value = '17:40';
document.getElementById('EndT').value = '20:40';
document.getElementById('buttonID').click();
"

source ~/.zshrc 2>/dev/null; agent-browser wait 1500

# 第 2 筆
source ~/.zshrc 2>/dev/null; agent-browser eval "
document.getElementById('WorkDay').value = '115/1/5';
document.getElementById('StartT').value = '21:10';
document.getElementById('EndT').value = '23:10';
document.getElementById('buttonID').click();
"

source ~/.zshrc 2>/dev/null; agent-browser wait 1500

# 繼續...
```

**填寫進度**：每填寫 5 筆，報告進度。

### 9. 完成驗證

```bash
source ~/.zshrc 2>/dev/null; agent-browser eval "
const rows = Array.from(document.querySelectorAll('table tbody tr')).slice(1);
const records = rows.map(r => ({
  date: r.cells[0]?.textContent?.trim(),
  start: r.cells[1]?.textContent?.trim(),
  end: r.cells[2]?.textContent?.trim(),
  hours: r.cells[3]?.textContent?.trim()
})).filter(h => h.date && h.date.includes('/'));

const totalHours = records.reduce((sum, r) => sum + parseFloat(r.hours || 0), 0);
JSON.stringify({ records, count: records.length, totalHours }, null, 2);
"
```

確認所有記錄都已成功新增，向使用者報告完成狀態（包含成功新增的筆數和總時數）。

### 10. 下載 A4 PDF 工時紀錄表

填寫完成後，可下載 A4 格式的 PDF 工時紀錄表。

**注意**：agent-browser 的 `pdf` 命令不支援自訂紙張大小，必須使用 Node.js + Playwright 直接生成。

#### 步驟 1：建立下載腳本

```bash
source ~/.zshrc 2>/dev/null; cat << 'EOF' > /tmp/print_workhour_a4.mjs
import { chromium } from '/Users/yufeng/.nvm/versions/node/v22.14.0/lib/node_modules/agent-browser/node_modules/playwright-core/index.mjs';
import os from 'os';
import path from 'path';

const args = process.argv.slice(2);
const year = args[0] || '2026';  // 西元年
const month = args[1] || '1';    // 月份
const userId = args[2] || '<使用者帳號>';  // 身份證字號

const browser = await chromium.launch({ headless: true });
const context = await browser.newContext();
const page = await context.newPage();

// 登入
await page.goto('https://gaweb.nutn.edu.tw/workhour/Home/Login');

// 關閉 Modal（如果有）
await page.evaluate(() => {
  const modal = document.querySelector('#myModal');
  if (modal) modal.style.display = 'none';
  const backdrop = document.querySelector('.modal-backdrop');
  if (backdrop) backdrop.remove();
});

await page.fill('input[type="text"]', userId);
await page.fill('input[type="password"]', userId);
await page.evaluate(() => {
  document.querySelector('input[type="submit"]').click();
});
await page.waitForURL('**/Budget', { timeout: 10000 });

// 前往報表頁面
await page.goto(`https://gaweb.nutn.edu.tw/workhour/Hours/SetYM?WorkYear=${year}&WorkMonth=${month}`);
await page.waitForLoadState('networkidle');

// 生成 A4 PDF
const rocYear = parseInt(year) - 1911;
const pdfPath = path.join(os.homedir(), 'Downloads', `workhour_${rocYear}_${month.padStart(2, '0')}_A4.pdf`);
await page.pdf({
  path: pdfPath,
  format: 'A4',
  printBackground: true,
  margin: { top: '10mm', bottom: '10mm', left: '10mm', right: '10mm' }
});

console.log('PDF saved to', pdfPath);
await browser.close();
EOF
```

#### 步驟 2：執行下載

```bash
# 下載 115 年 1 月的工時紀錄表
source ~/.zshrc 2>/dev/null; node /tmp/print_workhour_a4.mjs 2026 1 <使用者帳號>
```

參數說明：
- 第 1 個參數：西元年（例如 2026）
- 第 2 個參數：月份（例如 1）
- 第 3 個參數：身份證字號（用於登入）

PDF 會儲存到 `~/Downloads/workhour_115_01_A4.pdf`。

---

## 刪除工時記錄

系統的刪除功能需要確認對話框，使用 eval 繞過：

### 步驟 1：取得記錄 ID

```bash
source ~/.zshrc 2>/dev/null; agent-browser eval "
Array.from(document.querySelectorAll('a')).filter(a => a.textContent.includes('刪除')).map(a => ({
  id: a.name,
  onclick: a.getAttribute('onclick')
}));
"
```

### 步驟 2：執行刪除

```bash
# 設定要刪除的 ID 並調用刪除函數
source ~/.zshrc 2>/dev/null; agent-browser eval "document.querySelector('#hiddenEmployeeId').value = '455930'; DeleteEmployee();"
source ~/.zshrc 2>/dev/null; agent-browser wait 1500

# 繼續刪除下一筆...
```

---

## 工作類別特殊規則

| 工作類別 | 排除條件 |
|----------|----------|
| 夜間值班（協助普通教室夜間值班、協助管理榮譽校區教室） | 寒假、暑假、國定假日 |

### 套用邏輯

1. 識別工作內容關鍵字（如「夜間值班」「教室」）
2. 比對特殊規則表
3. **載入 `academic-calendar` 技能取得寒假、暑假日期**
4. 自動排除對應日期範圍

---

## 日期計算備忘

```
民國年 → 西元年：+1911
西元年 → 民國年：-1911

JavaScript Date.getDay(): 0=日, 1=一, 2=二, 3=三, 4=四, 5=五, 6=六
Python datetime.weekday(): 0=一, 1=二, 2=三, 3=四, 4=五, 5=六, 6=日
```

---

## 重要注意事項

1. **必須使用 zsh shell**：所有 agent-browser 命令都要加 `source ~/.zshrc 2>/dev/null;` 前綴並指定 `/bin/zsh` shell
2. **必須用程式碼驗證星期幾**：分析歷史記錄時，絕對不能靠觀察猜測，必須用 JavaScript 計算
3. **空計畫查舊計畫**：如果目標計畫沒有歷史紀錄，要找同名的舊計畫（如 D114-03 → D115-03）分析工作模式
4. **必須處理假日**：載入 academic-calendar 技能取得放假日資訊
5. **套用特殊規則**：夜間值班需排除寒假、暑假、國定假日
6. **優先使用預設值**：除非使用者明確要求修改或完全沒有預設勾選，否則保留頁面預設的工作類別和內容
7. **必須勾選工作類別**：系統要求至少選擇一個工作類別，否則會報錯
8. **日期格式**：使用民國年格式（例如：115/1/5）
9. **時間格式**：24 小時制，使用冒號分隔（例如：17:40）
10. **批次操作**：逐筆執行 + wait 1500ms，不要用 async/await
11. **一次性確認**：用表格呈現完整計畫，不分步驟詢問
12. **自動排除假日**：有行事曆資料時自動排除，不詢問使用者
13. **ref 不穩定時**：改用 `eval` 直接操作 DOM
14. **下載 PDF 用 Node.js**：agent-browser pdf 命令不支援 A4 格式，需用 Playwright 腳本
15. **完成後關閉瀏覽器**：任務完成後執行 `agent-browser close` 釋放資源

---

## 系統資訊

| 頁面 | URL |
|------|-----|
| 登入頁面 | https://gaweb.nutn.edu.tw/workhour/Home/Login |
| 計畫選擇 | https://gaweb.nutn.edu.tw/workhour/Budget |
| 新增工時 | https://gaweb.nutn.edu.tw/workhour/Hours/Create |
| 報表年月選擇 | https://gaweb.nutn.edu.tw/workhour/Hours/Report2 |
| 報表頁面 | https://gaweb.nutn.edu.tw/workhour/Hours/SetYM?WorkYear={西元年}&WorkMonth={月} |

---

## 表單欄位對應

| 欄位 ID | 用途 |
|---------|------|
| WorkDay | 工作日期 |
| StartT | 開始時間 |
| EndT | 結束時間 |
| checkbox1 | 研究學習 |
| checkbox2 | 資料處理 |
| checkbox3 | 統計調查 |
| checkbox4 | 實地考察 |
| checkbox5 | 行政事務 |
| checkbox6 | 其他 |
| Wcontent | 工作內容（勾選非「其他」類別時使用） |
| Wother | 其他工作內容（勾選「其他」時使用） |
| buttonID | 提交按鈕 |
| hiddenEmployeeId | 刪除用的隱藏 ID 欄位 |
