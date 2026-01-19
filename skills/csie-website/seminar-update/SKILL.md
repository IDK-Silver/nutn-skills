# 專題演講資料庫更新

> **語言規範**：本文件使用**繁體中文**撰寫。

## 概述

建立新學期的專題演講 SQL table，並更新系網頁面內容至新學期。

---

## ⚠️ 重要執行規則

> **階段 1 完成後必須停下來等待使用者確認，不可自動執行階段 2。**
>
> 階段 1 產生的 SQL 需要使用者手動匯入 phpMyAdmin，必須等使用者確認匯入完成後才能繼續階段 2。若在資料庫尚未建立前就更新網頁，會導致網站錯誤。

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

## Status Code 含義

| 代碼 | 含義 | 說明 |
|------|------|------|
| 0 | 特殊日期 | 不可填寫演講，例如：開學、國慶日放假、期中考、期末考 |
| 1 | 待填 | 該週可供新增演講（顯示在「新增演講」的日期選單中） |
| 2 | 已填 | 該週已有演講資訊 |

**注意**：建立新學期時，所有可安排日期的 status 初始值為 `1`，當演講資訊填入後會更新為 `2`。

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

其他週 `status` 設為 `1`（待填）。

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

### 步驟 1.5：提供 SQL 檔案下載並等待確認

將 SQL 檔案以 artifact 形式提供下載，並告知使用者：

1. 下載 SQL 檔案
2. 前往 phpMyAdmin（網址見資料來源）
3. 選擇資料庫 `www_csie`
4. 點選「匯入」分頁
5. 選擇檔案並執行

**重要**：
- 不要在回應中顯示資料庫帳號密碼，請使用者參考 Project 文件
- **階段 1 完成後必須停止，詢問使用者是否已完成 SQL 匯入，確認後才能執行階段 2**

---

## 階段 2：更新網頁檔案（自動部署）

> **前置條件**：使用者已確認 SQL 匯入完成

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

從 Project 文件取得 SSH 連線資訊（`{host}`）。

**正確指令**（需先設定 git credential helper）：

```bash
ssh -tt {host} "gh auth setup-git; cd ~/Documents/Github/nutn-csie-web; git pull"
```

**說明**：
- `gh auth setup-git` 會將 GitHub CLI 的認證連結到 git credential helper
- 若不執行此步驟，git pull 會因 `wincredman` 無法取得憑證而要求輸入帳密

### 步驟 2.5：FTP 上傳

從 Project 文件取得 FTP 連線資訊（`{ftp_host}`、`{ftp_user}`、`{ftp_password}`），上傳變更的檔案。

**正確指令**（Windows PowerShell 環境）：

```bash
ssh -tt {host} "cd ~/Documents/Github/nutn-csie-web; curl.exe -T subsite/seminar/index.php ftp://{ftp_host}/ftp/html/subsite/seminar/index.php --user {ftp_user}:{ftp_password}"
ssh -tt {host} "cd ~/Documents/Github/nutn-csie-web; curl.exe -T subsite/seminar/old_seminar.php ftp://{ftp_host}/ftp/html/subsite/seminar/old_seminar.php --user {ftp_user}:{ftp_password}"
```

**重要**：
- 必須使用 `curl.exe` 而非 `curl`
- PowerShell 中 `curl` 是 `Invoke-WebRequest` 的 alias，參數語法不同
- `curl.exe` 才是真正的 curl 執行檔

| 本地檔案 | FTP 目標路徑 |
|----------|-------------|
| `subsite/seminar/index.php` | `/ftp/html/subsite/seminar/index.php` |
| `subsite/seminar/old_seminar.php` | `/ftp/html/subsite/seminar/old_seminar.php` |

---

## 錯誤處理

| 情況 | 錯誤訊息 | 處理方式 |
|------|----------|----------|
| 無法取得學校行事曆 | - | 請使用者手動提供開學日、考試週、放假日資訊 |
| GitHub API 失敗 | - | 提供修改內容，請使用者手動建立 PR |
| gh token 失效 | `The token in default is invalid` | 提示使用者在內網主機執行 `gh auth login -h github.com` 重新認證 |
| git credential 未設定 | `Unable to persist credentials with the 'wincredman' credential store` | 在 git pull 前先執行 `gh auth setup-git` |
| PowerShell curl 參數錯誤 | `參數名稱不明確` 或 `AmbiguousParameter` | 改用 `curl.exe` 而非 `curl` |
| SSH 連線失敗 | - | 確認 VPN 或網路狀態，提示手動執行指令 |
| FTP 上傳逾時 | `Timeout was reached` | 確認是否在內網環境，FTP 僅限內網存取 |

### 常見問題排查流程

**SSH git pull 失敗時**：

1. 先檢查 gh 認證狀態：
   ```bash
   ssh -tt {host} "gh auth status"
   ```

2. 若顯示 token invalid，請使用者重新登入：
   ```bash
   ssh -tt {host} "gh auth login -h github.com"
   ```

3. 設定 git credential helper 後再 pull：
   ```bash
   ssh -tt {host} "gh auth setup-git; cd ~/Documents/Github/nutn-csie-web; git pull"
   ```

---

## 使用範例

**使用者**：「更新專題演講到 114 學年度第 2 學期」

**預期行為**：

### 階段 1（自動執行）：
1. 從學校行事曆取得 114-2 學期資訊
2. 計算 18 週週四，標記不可用日期（status=0 或 1）
3. 生成 `seminar_info_1142.sql` 並提供下載
4. 告知 phpMyAdmin 匯入步驟
5. **⏸️ 停止並詢問：「請確認 SQL 已匯入完成，我再繼續執行階段 2」**

### 階段 2（使用者確認後）：
1. 建立分支 `update-seminar-1142`
2. 修改 `index.php`（1141 → 1142）
3. 修改 `old_seminar.php`（新增 1141 選項）
4. 建立 PR 並合併
5. SSH 執行 `gh auth setup-git` + `git pull`
6. FTP 使用 `curl.exe` 上傳變更檔案
7. 回報完成
