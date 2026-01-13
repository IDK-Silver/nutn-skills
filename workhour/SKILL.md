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

## 工作流程

### 1. 初始確認與登入

**步驟 1：開啟瀏覽器並檢查登入狀態**

```bash
agent-browser open https://gaweb.nutn.edu.tw/workhour/Budget --headed
agent-browser snapshot -i
```

檢查 snapshot 結果：
- 如果看到「計畫管理」連結 → 已登入，繼續步驟 2
- 如果看到「登入」按鈕和帳號密碼欄位 → 未登入，執行步驟 1-2

**步驟 1-2：自動登入**

```bash
# 點擊登入連結（如果在首頁）
agent-browser click @e2  # 登入連結的 ref

# 取得表單元素
agent-browser snapshot -i

# 填寫帳號密碼（預設都是身份證字號）
agent-browser fill @e5 "身份證字號"  # 帳號欄位
agent-browser fill @e6 "身份證字號"  # 密碼欄位

# 點擊登入按鈕（使用 eval 更穩定）
agent-browser eval "document.querySelector('input[type=submit], button[type=submit]').click()"

# 等待頁面載入
agent-browser wait 2000
agent-browser get url
```

登入成功後 URL 會變成 `/workhour/Budget`。

**處理登入後的 Modal 提示**

系統可能會顯示提示訊息的 Modal，用 eval 關閉：

```bash
agent-browser eval "document.querySelector('#myModal button[data-dismiss=modal]')?.click()"
```

### 2. 收集計畫資訊

詢問使用者：
- 計畫編號（例如：D115-03）
- 填寫時間範圍（例如：115/1/1 到 115/1/31）
- 是否有行事曆圖片（用於排除假日）

### 3. 選擇計畫並檢查歷史紀錄

```bash
# 取得計畫列表
agent-browser snapshot

# 點擊對應計畫的「選擇」連結
agent-browser click @e19  # 根據 snapshot 結果找到對應的 ref

# 等待頁面載入
agent-browser wait 1500
agent-browser snapshot
```

**分析歷史紀錄**

使用 eval 取得並分析歷史記錄的日期：

```bash
agent-browser eval "
const rows = Array.from(document.querySelectorAll('table tbody tr')).slice(1);
const dates = rows.map(r => r.cells[0]?.textContent?.trim()).filter(Boolean);
const weekdays = ['日', '一', '二', '三', '四', '五', '六'];
const analysis = dates.map(d => {
  const [y, m, day] = d.split('/').map(Number);
  const date = new Date(y + 1911, m - 1, day);
  return { date: d, weekday: weekdays[date.getDay()] };
});
JSON.stringify(analysis);
"
```

- **禁止**：直接觀察日期就說「看起來是星期X」
- **必須**：用程式碼計算後才能確認工作日模式

### 4. 檢查頁面預設值

```bash
agent-browser eval "
({
  checkbox6Checked: document.getElementById('checkbox6')?.checked,
  wother: document.getElementById('Wother')?.value
});
"
```

**如果頁面有預設勾選和內容**：直接採用，不詢問使用者。

**如果頁面沒有任何勾選**：詢問使用者工作內容分類。

### 5. 生成填寫計畫

使用 eval 生成日期清單：

```bash
agent-browser eval "
const results = [];
const weekdayNames = ['日', '一', '二', '三', '四', '五', '六'];
const workWeekdays = [1, 4];  // 星期一、四

for (let d = 1; d <= 31; d++) {
  const date = new Date(2026, 0, d);  // 115年1月
  if (workWeekdays.includes(date.getDay())) {
    results.push({
      date: '115/1/' + d,
      weekday: weekdayNames[date.getDay()]
    });
  }
}
JSON.stringify(results, null, 2);
"
```

### 6. 一次性最終確認

用表格呈現完整計畫，一次確認所有資訊：

```
## 填寫計畫確認

### 基本資訊
- **計畫**：D115-03 【B版】回流教育學位班經費(碩士班)
- **工作日模式**：星期一、星期四
- **工作類別**：其他 - 協助管理榮譽校區教室

### 填寫清單

| # | 日期 | 星期 | 時段 1 | 時段 2 |
|---|------|------|--------|--------|
| 1 | 115/1/5 | 一 | 17:40-20:40 | 21:10-23:10 |
| 2 | 115/1/8 | 四 | 17:40-20:40 | 21:10-23:10 |
| ... | ... | ... | ... | ... |

### 統計
- **總天數**：X 天
- **總筆數**：X 筆（每天 2 個時段）
- **總時數**：X 小時

---
確認以上資訊無誤後開始填寫？
```

### 7. 批次填寫

**重要**：eval 中的 async/await 不可靠，必須逐筆執行並加入 wait：

```bash
# 第 1 筆
agent-browser eval "
document.getElementById('WorkDay').value = '115/1/5';
document.getElementById('StartT').value = '17:40';
document.getElementById('EndT').value = '20:40';
document.getElementById('buttonID').click();
"
agent-browser wait 1500

# 第 2 筆
agent-browser eval "
document.getElementById('WorkDay').value = '115/1/5';
document.getElementById('StartT').value = '21:10';
document.getElementById('EndT').value = '23:10';
document.getElementById('buttonID').click();
"
agent-browser wait 1500

# 繼續...
```

**填寫進度**：每填寫 5 筆，報告進度。

### 8. 完成驗證

```bash
agent-browser eval "
Array.from(document.querySelectorAll('table tbody tr')).slice(1).map(r => ({
  date: r.cells[0]?.textContent?.trim(),
  start: r.cells[1]?.textContent?.trim(),
  end: r.cells[2]?.textContent?.trim(),
  hours: r.cells[3]?.textContent?.trim()
}));
"
```

確認所有記錄都已成功新增，向使用者報告完成狀態。

---

## 刪除工時記錄

系統的刪除功能需要確認對話框，使用 eval 繞過：

### 步驟 1：取得記錄 ID

```bash
agent-browser eval "
Array.from(document.querySelectorAll('a')).filter(a => a.textContent.includes('刪除')).map(a => ({
  id: a.name,
  onclick: a.getAttribute('onclick')
}));
"
```

### 步驟 2：執行刪除

```bash
# 設定要刪除的 ID 並調用刪除函數
agent-browser eval "document.querySelector('#hiddenEmployeeId').value = '455930'; DeleteEmployee();"
agent-browser wait 1500

# 繼續刪除下一筆...
```

---

## 日期計算備忘

```
民國年 → 西元年：+1911
西元年 → 民國年：-1911

JavaScript Date.getDay(): 0=日, 1=一, 2=二, 3=三, 4=四, 5=五, 6=六
```

---

## 重要注意事項

1. **必須用程式碼驗證星期幾**：絕對不能靠觀察猜測
2. **優先使用預設值**：保留頁面預設的工作類別和內容
3. **必須勾選工作類別**：系統要求至少選擇一個
4. **日期格式**：民國年格式（例如：115/1/5）
5. **時間格式**：24 小時制（例如：17:40）
6. **批次操作**：逐筆執行 + wait 1500ms，不要用 async/await
7. **一次性確認**：用表格呈現完整計畫，不分步驟詢問
8. **ref 不穩定時**：改用 `eval` 直接操作 DOM

---

## 工作類別特殊規則

| 工作類別 | 排除條件 |
|----------|----------|
| 夜間值班（協助普通教室夜間值班、協助管理榮譽校區教室） | 寒假、暑假、國定假日 |

---

## 系統資訊

| 頁面 | URL |
|------|-----|
| 登入頁面 | https://gaweb.nutn.edu.tw/workhour/Home/Login |
| 計畫選擇 | https://gaweb.nutn.edu.tw/workhour/Budget |
| 新增工時 | https://gaweb.nutn.edu.tw/workhour/Hours/Create |

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
| Wcontent | 工作內容 |
| Wother | 其他工作內容 |
| buttonID | 提交按鈕 |
| hiddenEmployeeId | 刪除用的隱藏 ID 欄位 |
