# 學校行事曆

> **語言規範**：本文件使用**繁體中文**撰寫。

## 概述

取得南大學校行事曆 PDF，提供完整的學年度資訊，包括：
- 學期開始/結束日期
- 放假日與補假日
- 考試週、註冊期間
- 重要學術活動與行政事項

---

## 資料來源

| 項目 | 值 |
|------|-----|
| 行事曆列表頁面 | `https://academic.nutn.edu.tw/index.php?option=module&lang=cht&task=showlist&id=239` |
| PDF 網址格式 | `https://academic.nutn.edu.tw/upload/cht/152/{file_id}_file_1.pdf` |
| 備註 | `file_id` 無法預測，必須從列表頁面擷取 |

**重要**：務必取得最新的 PDF，完整載入內容。絕對不要使用快取或寫死的資料。

---

## 工作流程

### 步驟 1：判斷目標學年度

將目前日期轉換為民國學年度：

```javascript
function getCurrentAcademicYear() {
  const now = new Date();
  const year = now.getFullYear();
  const month = now.getMonth() + 1; // 1-12
  
  // 學年度從 8 月開始
  // 8 月前 = 前一學年度
  const adAcademicYear = month >= 8 ? year : year - 1;
  const rocAcademicYear = adAcademicYear - 1911;
  
  return rocAcademicYear;
}

// 範例：2026/01/12 → 114 學年度（114年8月 ~ 115年7月）
```

### 步驟 2：取得行事曆列表頁面

使用 `web_search` 找到行事曆頁面，再用 `web_fetch` 取得內容：

```
1. web_search: "site:academic.nutn.edu.tw 行事曆 {rocYear}學年度 pdf"
2. web_fetch: 行事曆列表頁面網址
3. 解析 HTML 找出目標學年度的 PDF 網址
```

**HTML 中預期的 PDF 網址格式：**
```
國立臺南大學114學年度全校行事曆
  檔案下載: upload/cht/152/3619_file_1.pdf
```

### 步驟 3：取得完整 PDF 內容

使用 `web_fetch` 並設定 `web_fetch_pdf_extract_text=true`：

```
web_fetch(pdf_url, web_fetch_pdf_extract_text=true)
```

**重要**：完整載入 PDF 文字內容，不要只擷取特定資訊。PDF 內容包含兩學期的完整行事曆表格與「重要記事」區段。

---

## 錯誤處理

| 情況 | 處理方式 |
|------|----------|
| 行事曆頁面無法存取 | 回報錯誤，建議手動查詢 |
| 找不到目標學年度的 PDF 網址 | 嘗試前一學年度，或回報尚未公佈 |
| PDF 取得失敗 | 回報錯誤，提供列表頁面連結供使用者自行查閱 |
| 網路逾時 | 重試一次，然後回報錯誤 |

---

## 使用範例

**使用者請求**：「下學期什麼時候開學？」

**助理工作流程**：
1. 判斷目前學年度
2. 取得行事曆 PDF
3. 載入完整 PDF 內容
4. 從內容中找出下學期開學日期
5. 回答使用者

**使用者請求**：「期中考是哪幾天？」

**助理工作流程**：
1. 取得目前學年度行事曆 PDF
2. 載入完整 PDF 內容
3. 從內容中找出考試週資訊
4. 回答使用者

**使用者請求**：「114 學年度有哪些放假日？」

**助理工作流程**：
1. 取得 114 學年度行事曆 PDF
2. 載入完整 PDF 內容
3. 從內容中整理放假日資訊
4. 回答使用者
