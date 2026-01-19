# 自訂技能索引

> **語言規範**：本專案所有文件均使用**繁體中文**撰寫。

每次對話開始時，請先閱讀此檔案以決定要載入哪些技能。

## 使用方式

1. 先閱讀此 INDEX.md
2. 比對使用者請求與下方的觸發關鍵字
3. 若符合，則在執行前先閱讀對應的 SKILL.md
4. 可視需要載入多個技能

---

## 可用技能

### agent-browser

| 欄位 | 值 |
|------|-----|
| 路徑 | `./agent-browser/SKILL.md` |
| 說明 | 瀏覽器自動化工具，支援網頁操作、表單填寫、截圖等 |
| 觸發關鍵字 | agent-browser, 瀏覽器自動化, browser automation, 網頁操作 |

**載入時機**：使用者需要進行瀏覽器自動化操作，或其他技能依賴此工具時。

---

### workhour

| 欄位 | 值 |
|------|-----|
| 路徑 | `./workhour/SKILL.md` |
| 說明 | 南大工時系統自動化填寫 |
| 觸發關鍵字 | 工時, workhour, 填工時, work hours, 南大工時, NUTN 工時系統 |

**載入時機**：使用者詢問填寫工時、南大工時系統，或自動化時數表登錄相關事宜。

**相依技能**：agent-browser

---

### academic-calendar

| 欄位 | 值 |
|------|-----|
| 路徑 | `./academic-calendar/SKILL.md` |
| 說明 | 取得並提供南大學校行事曆完整內容 |
| 觸發關鍵字 | 行事曆, 學校行事曆, 放假日, 假日, 校曆, academic calendar, holidays, 開學, 考試週 |

**載入時機**：使用者詢問學校行事曆、放假日、學期日期、開學日、考試週等相關事宜。

---

### csie-website

| 欄位 | 值 |
|------|-----|
| 路徑 | `./csie-website/INDEX.md` |
| 說明 | 資工系網站管理與更新（含多個子技能） |
| 觸發關鍵字 | 系網, 系網站, csie website, 專題演講, seminar, 演講更新, 學期更新 |

**載入時機**：使用者需要更新或管理資工系網站內容。

**子技能**：先閱讀 `./csie-website/INDEX.md` 以決定要載入哪個子技能。

---

### skill-creator

| 欄位 | 值 |
|------|-----|
| 路徑 | `./skill-creator/SKILL.md` |
| 說明 | 用於建立結構完善、易於維護的技能的元技能 |
| 觸發關鍵字 | 建立 skill, 新增 skill, create skill, new skill, skill 範本, skill template |

**載入時機**：使用者想要建立新技能或檢視技能設計規範。

---

### cloud-setup

| 欄位 | 值 |
|------|-----|
| 路徑 | `./cloud-setup/SKILL.md` |
| 說明 | 雲端環境檢測與設定，自動安裝 agent-browser |
| 觸發關鍵字 | 環境設定, cloud setup, 雲端環境, 安裝 agent-browser, 環境檢測 |

**載入時機**：
- 對話開始時自動檢測環境（見 CLAUDE.md）
- 使用者詢問環境設定相關問題
- 其他技能需要 agent-browser 但尚未安裝時

---

<!-- 使用以下範本新增技能：

### skill-name

| 欄位 | 值 |
|------|-----|
| 路徑 | `./skill-name/SKILL.md` |
| 說明 | 簡短說明 |
| 觸發關鍵字 | keyword1, keyword2, 關鍵字1, 關鍵字2 |

**載入時機**：描述觸發條件。

**相依技能**：列出此技能依賴的其他技能（如有）。

-->
