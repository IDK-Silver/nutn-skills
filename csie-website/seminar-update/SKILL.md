# 專題演講資料庫更新

> **語言規範**：本文件使用**繁體中文**撰寫。

## 概述

建立新學期的專題演講 SQL table，並更新系網頁面內容至新學期。

---

## 資料來源

| 項目 | 位置 |
|------|------|
| 學校行事曆 | 使用 `academic-calendar` 技能取得 |
| phpMyAdmin | https://phpweb3.nutn.edu.tw/phpMyAdmin/index.php |
| 系網 GitHub | https://github.com/IDK-Silver/nutn-csie-web |
| FTP 路徑 | `/ftp/html/subsite/seminar` |
| 連線資訊 | 參考 Project 文件 `NUTN_內網連線資訊.md` |

---

## 學期代碼格式

**4 位數字** = 民國年(3位) + 學期(1位)

| 代碼 | 含義 |
|------|------|
| `1141` | 114 學年度第 1 學期 |
| `1142` | 114 學年度第 2 學期 |

---

## 階段 1：建立 SQL 資料表（手動匯入）

### 步驟 1.1：取得學期資訊

使用 `academic-calendar` 技能取得目標學期的：
- 開學日
- 期中考週
- 期末考週
- 放假日（影響週四的）

### 步驟 1.2：計算 18 週的週四日期

從開學日所在週開始，計算連續 18 週的週四日期。

### 步驟 1.3：標記不可安排演講的日期

以下情況 `status` 設為 `0`，並填入對應 `topic`：

| 情況 | topic 值 |
|------|----------|
| 開學第一週 | `開學` |
| 期中考週的週四 | `期中考` |
| 期末考週的週四 | `期末考` |
| 國定假日 | 假日名稱（如 `調整放假`、`端午節`） |

其他週 `status` 設為 `1`（待安排）。

### 步驟 1.4：生成 SQL 檔案

Table 結構：

```sql
CREATE TABLE `seminar_info_{學期代碼}` (
  `week` int(11) NOT NULL,
  `date` date NOT NULL,
  `speaker` text NOT NULL,
  `unit` text NOT NULL,
  `title` text NOT NULL,
  `topic` text NOT NULL,
  `content` text DEFAULT NULL,
  `invite` varchar(255) NOT NULL,
  `status` int(11) NOT NULL,
  `summary` text NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

生成 INSERT 語句，每週一筆資料。

### 步驟 1.5：提供 SQL 檔案下載

將 SQL 檔案以 artifact 形式提供下載，並告知使用者：

1. 下載 SQL 檔案
2. 前往 phpMyAdmin（網址見資料來源）
3. 選擇資料庫 `www_csie`
4. 點選「匯入」分頁
5. 選擇檔案並執行

**重要**：不要在回應中顯示資料庫帳號密碼，請使用者參考 Project 文件。

---

## 階段 2：更新網頁檔案（自動部署）

### 步驟 2.1：建立 GitHub 分支

分支名稱：`update-seminar-{新學期代碼}`

### 步驟 2.2：修改檔案

#### 檔案 1: index.php

**路徑**: `subsite/seminar/index.php`

找到 `$_SESSION['current_semester']` 並更新為新學期代碼：

```php
$_SESSION['current_semester'] = "{新學期代碼}";
```

#### 檔案 2: old_seminar.php

**路徑**: `subsite/seminar/old_seminar.php`

在 `<select>` 的 `<option>` 列表最上方新增舊學期：

```html
<option value="{舊學期代碼}">{舊學年度}學年度第{舊學期}學期</option>
```

### 步驟 2.3：建立並合併 Pull Request

1. 建立 PR，標題：`更新專題演講至 {學年度} 學年度第 {學期} 學期`
2. 直接合併（私人 repo）

### 步驟 2.4：SSH 同步到內網

從 Project 文件取得 SSH 連線資訊，執行：

```bash
ssh -tt {host} "cd ~/Documents/Github/nutn-csie-web; git pull"
```

### 步驟 2.5：FTP 上傳

從 Project 文件取得 FTP 連線資訊，上傳變更的檔案：

| 本地檔案 | FTP 目標路徑 |
|----------|-------------|
| `subsite/seminar/index.php` | `/ftp/html/subsite/seminar/index.php` |
| `subsite/seminar/old_seminar.php` | `/ftp/html/subsite/seminar/old_seminar.php` |

使用 PowerShell 的 WebClient 或 FTP 命令上傳。

---

## 錯誤處理

| 情況 | 處理方式 |
|------|----------|
| 無法取得學校行事曆 | 請使用者手動提供開學日、考試週、放假日資訊 |
| GitHub API 失敗 | 提供修改內容，請使用者手動建立 PR |
| SSH 連線失敗 | 確認 VPN 或網路狀態，提示手動執行 git pull |
| FTP 上傳失敗 | 提供檔案內容，請使用者手動上傳 |

---

## 使用範例

**使用者**：「更新專題演講到 114 學年度第 2 學期」

**預期行為**：

**階段 1**：
1. 從學校行事曆取得 114-2 學期資訊
2. 計算 18 週週四，標記不可用日期
3. 生成 `seminar_info_1142.sql` 並提供下載
4. 告知 phpMyAdmin 匯入步驟

**階段 2**（使用者確認後）：
1. 建立分支 `update-seminar-1142`
2. 修改 `index.php`（1141 → 1142）
3. 修改 `old_seminar.php`（新增 1141 選項）
4. 建立 PR 並合併
5. SSH 執行 git pull
6. FTP 上傳變更檔案
7. 回報完成
