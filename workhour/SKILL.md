# 工時填寫助手

> **語言規範**：本文件使用**繁體中文**撰寫。

## 核心原則

1. **必須用程式碼驗證星期幾** - 絕不靠觀察或猜測日期對應的星期
2. **自動決策** - 主動排除假日，信任使用者會提出修正
3. **一次性確認** - 生成完整計畫後一次確認，不分步驟詢問
4. **表格化呈現** - 日期清單用表格展示

---

## 工作流程

### 1. 初始確認與登入

**步驟 1：檢查登入狀態**
訪問工時系統首頁：https://gaweb.nutn.edu.tw/workhour/Budget

使用 `get_page_content` 檢查頁面內容：
- 如果看到計畫列表 → 已登入，繼續步驟 2
- 如果看到登入頁面 → 未登入，執行步驟 1-2

**步驟 1-2：自動登入**
如果未登入：

1. 詢問使用者的身份證字號
2. 使用身份證字號作為帳號和密碼自動登入：

```javascript
// 填寫登入表單
document.querySelector('input[name="Account"], input[id*="account" i], input[type="text"]').value = '身份證字號';
document.querySelector('input[name="Password"], input[id*="password" i], input[type="password"]').value = '身份證字號';

// 點擊登入按鈕
document.querySelector('input[type="submit"], button[type="submit"]').click();
```

3. 等待頁面載入後檢查登入結果：
   - 成功：看到計畫列表頁面 → 繼續步驟 2
   - 失敗：看到錯誤訊息 → 向使用者報告錯誤，請使用者確認身份證字號或手動登入

**備註：**
- 預設帳號和密碼都是身份證字號
- 如果使用者已修改密碼，請使用者手動登入系統

### 2. 收集計畫資訊
詢問使用者：
- 計畫編號（例如：D114-03）
- 計畫名稱（例如：【B版】回流教育學位班經費(碩士班)）
- 填寫時間範圍（例如：114/12/1 到 114/12/31）
- 是否有行事曆圖片（用於排除假日）

### 3. 檢查歷史紀錄
使用 Chrome MCP 導航到計畫選擇頁面並點選對應計畫：
1. 訪問 https://gaweb.nutn.edu.tw/workhour/Budget
2. 點選對應的計畫「選擇」連結
3. 讀取頁面內容，檢查是否有歷史工時記錄

**如果有歷史紀錄：**

#### 關鍵：必須用程式碼計算星期幾（禁止觀察猜測）

```javascript
// 分析歷史記錄中的日期，計算實際星期幾
function analyzeHistoryDates(rocDates) {
  const weekdays = ['日', '一', '二', '三', '四', '五', '六'];
  return rocDates.map(rocDateStr => {
    const [rocYear, month, day] = rocDateStr.split('/').map(Number);
    const adYear = rocYear + 1911; // 民國轉西元
    const date = new Date(adYear, month - 1, day);
    return {
      date: rocDateStr,
      weekday: date.getDay(),
      weekdayName: weekdays[date.getDay()]
    };
  });
}

// 從頁面抓取的歷史日期
const historyDates = ['114/11/27', '114/11/24', '114/11/20', '114/11/17'];
const analysis = analyzeHistoryDates(historyDates);
console.log(analysis);

// 統計工作日模式
const weekdayCounts = {};
analysis.forEach(item => {
  weekdayCounts[item.weekdayName] = (weekdayCounts[item.weekdayName] || 0) + 1;
});
console.log('工作日模式:', weekdayCounts);
```

- **禁止**：直接觀察日期就說「看起來是星期X」
- **必須**：用程式碼計算後才能確認工作日模式
- 分析歷史記錄的時間區間
- 向使用者展示分析結果（用表格呈現）

**如果沒有歷史紀錄：**
詢問使用者以下資訊：
- 工作日（星期幾）
- 每日時間區間（可以有多個，例如：17:40-20:40 和 21:10-23:10）

### 4. 檢查頁面預設值
在詢問工作內容之前，先檢查頁面的預設狀態：

```javascript
// 檢查哪些 checkbox 已被勾選
const checkedBoxes = Array.from(document.querySelectorAll('input[type="checkbox"][name="checkboxs"]'))
  .map((cb, idx) => ({id: cb.id, checked: cb.checked, index: idx + 1}))
  .filter(item => item.checked);

// 檢查預設內容
const wcontent = document.getElementById('Wcontent').value;
const wother = document.getElementById('Wother').value;

({
  checkedBoxes: checkedBoxes,
  wcontent: wcontent,
  wother: wother
});
```

**如果頁面有預設勾選和內容：**
- 記錄預設狀態，直接採用（不詢問使用者）
- 會在最終確認時一併呈現

**如果頁面沒有任何勾選：**
詢問使用者工作內容分類：
- 研究學習 (checkbox1)
- 資料處理 (checkbox2)
- 統計調查 (checkbox3)
- 實地考察 (checkbox4)
- 行政事務 (checkbox5)
- 其他 (checkbox6) - 需填寫具體內容

### 5. 生成填寫計畫

#### 自動生成日期清單

```javascript
function generateWorkDates(startRoc, endRoc, workWeekdays, holidays = []) {
  const results = [];
  const weekdayNames = ['日', '一', '二', '三', '四', '五', '六'];
  
  const [startY, startM, startD] = startRoc.split('/').map(Number);
  const [endY, endM, endD] = endRoc.split('/').map(Number);
  
  let current = new Date(startY + 1911, startM - 1, startD);
  const end = new Date(endY + 1911, endM - 1, endD);
  
  while (current <= end) {
    const rocDate = `${current.getFullYear() - 1911}/${current.getMonth() + 1}/${current.getDate()}`;
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

// 範例：星期三(3)和星期日(0)，排除假日
const dates = generateWorkDates('114/12/1', '114/12/31', [0, 3], ['114/12/25']);
console.log(dates);
```

#### 處理假日

**如果使用者提供日曆資料（圖片）：**
- 分析日曆，識別紅色標記的放假日
- 自動排除這些日期（不詢問使用者）
- 在最終確認時列出已排除的日期

**如果沒有日曆資料：**
- 自動生成日期清單
- 在最終確認時詢問是否有需要排除的日期

### 6. 一次性最終確認

**用表格呈現完整計畫，一次確認所有資訊：**

```
## 填寫計畫確認

### 基本資訊
- **計畫**：D114-03 【B版】回流教育學位班經費(碩士班)
- **工作日模式**：星期三、星期日
- **工作類別**：其他 - 協助普通教室夜間值班

### 填寫清單

| # | 日期 | 星期 | 時段 1 | 時段 2 |
|---|------|------|--------|--------|
| 1 | 114/12/1 | 日 | 17:40-20:40 | 21:10-23:10 |
| 2 | 114/12/4 | 三 | 17:40-20:40 | 21:10-23:10 |
| 3 | 114/12/8 | 日 | 17:40-20:40 | 21:10-23:10 |
| ... | ... | ... | ... | ... |

### 統計
- **已排除假日**：114/12/25（行憲紀念日）
- **總天數**：X 天
- **總筆數**：X 筆（每天 2 個時段）
- **總時數**：X 小時

---
確認以上資訊無誤後開始填寫？
```

**注意**：不分步驟詢問，一次呈現所有資訊讓使用者確認。

### 7. 批次填寫
確認後執行批次填寫：

**填寫方法（使用預設值）：**
```javascript
// 只填寫日期和時間，不修改工作類別和內容
document.getElementById('WorkDay').value = '114/12/1';
document.getElementById('StartT').value = '17:40';
document.getElementById('EndT').value = '20:40';
document.getElementById('buttonID').click();
```

**填寫方法（自訂工作內容）：**
```javascript
// 需要修改工作類別時才執行
document.getElementById('WorkDay').value = '114/12/1';
document.getElementById('StartT').value = '17:40';
document.getElementById('EndT').value = '20:40';

// 取消所有工作類別
document.querySelectorAll('input[type="checkbox"][name="checkboxs"]').forEach(cb => cb.checked = false);

// 勾選對應的工作類別
document.getElementById('checkbox6').checked = true;
document.getElementById('Wother').value = '協助普通教室夜間值班';

document.getElementById('buttonID').click();
```

**填寫進度：**
- 每填寫 5 筆，報告進度
- 完成後確認所有記錄已正確新增

### 8. 完成驗證
填寫完成後：
- 重新讀取頁面內容
- 確認所有記錄都已成功新增
- 向使用者報告完成狀態（包含成功新增的筆數和總時數）

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

1. **必須用程式碼驗證星期幾**：分析歷史記錄時，絕對不能靠觀察猜測，必須用 JavaScript 或 Python 計算
2. **優先使用預設值**：除非使用者明確要求修改或完全沒有預設勾選，否則保留頁面預設的工作類別和內容
3. **必須勾選工作類別**：系統要求至少選擇一個工作類別，否則會報錯
4. **日期格式**：使用民國年格式（例如：114/12/1）
5. **時間格式**：24 小時制，使用冒號分隔（例如：17:40）
6. **錯誤處理**：如果出現錯誤訊息，關閉錯誤對話框後重新嘗試
7. **批次操作**：連續填寫時，每筆之間不需要等待，直接連續執行即可
8. **一次性確認**：用表格呈現完整計畫，不分步驟詢問
9. **自動排除假日**：有行事曆資料時自動排除，不詢問使用者

---

## 工作類別特殊規則

| 工作類別 | 排除條件 |
|----------|----------|
| 夜間值班（協助普通教室夜間值班、協助管理榮譽校區教室） | 寒假、暑假、國定假日 |

### 套用邏輯

1. 識別工作內容關鍵字（如「夜間值班」「教室」）
2. 比對特殊規則表
3. 自動排除對應日期範圍

---

## 系統資訊

- 工時系統網址：https://gaweb.nutn.edu.tw/workhour/Home/Login/FromLogout
- 計畫選擇頁面：https://gaweb.nutn.edu.tw/workhour/Budget
- 新增工時頁面：https://gaweb.nutn.edu.tw/workhour/Hours/Create

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
