# 系網部署

> **語言規範**：本文件使用**繁體中文**撰寫。

## 概述

從 GitHub 拉取最新變更，自動偵測修改的檔案並上傳至 FTP。適用於本地開發完成、push 到 GitHub 後，需要同步到正式環境的場景。

---

## 資料來源

| 項目 | 位置 |
|------|------|
| 系網 GitHub | https://github.com/IDK-Silver/nutn-csie-web |
| 內網 repo 路徑 | `~/Documents/Github/nutn-csie-web` |
| FTP 根目錄 | `/ftp/html/` |
| 連線資訊 | 參考 Project 文件 `NUTN_內網連線資訊.md` |

---

## FTP 路徑映射

repo 根目錄對應 FTP 的 `/ftp/html/`。

| 範例 repo 路徑 | FTP 路徑 |
|----------------|----------|
| `subsite/seminar/index.php` | `/ftp/html/subsite/seminar/index.php` |
| `images/logo.png` | `/ftp/html/images/logo.png` |
| `index.html` | `/ftp/html/index.html` |

---

## 工作流程

### 步驟 1：取得變更清單

SSH 到內網，fetch 遠端並比較差異：

```bash
ssh -tt {host} "cd ~/Documents/Github/nutn-csie-web; git fetch origin; git diff --name-only HEAD origin/main"
```

**輸出範例**：
```
subsite/seminar/index.php
subsite/seminar/old_seminar.php
```

### 步驟 2：確認變更

顯示變更檔案清單，詢問使用者是否繼續部署。

若無變更（輸出為空），告知使用者已是最新狀態，結束流程。

### 步驟 3：執行 git pull

```bash
ssh -tt {host} "gh auth setup-git; cd ~/Documents/Github/nutn-csie-web; git pull"
```

### 步驟 4：上傳至 FTP

對每個變更的檔案，使用 `curl.exe` 上傳：

```bash
ssh -tt {host} "cd ~/Documents/Github/nutn-csie-web; curl.exe -T {file_path} ftp://{ftp_host}/ftp/html/{file_path} --user {ftp_user}:{ftp_password}"
```

**重要**：
- 必須使用 `curl.exe` 而非 `curl`（PowerShell 環境）
- 逐一上傳並回報進度
- 從 Project 文件取得 FTP 連線資訊

### 步驟 5：回報結果

列出所有已上傳的檔案，確認部署完成。

---

## 錯誤處理

| 情況 | 處理方式 |
|------|----------|
| SSH 連線失敗 | 確認網路狀態，提示手動執行指令 |
| gh token 失效 | 提示使用者在內網主機執行 `gh auth login -h github.com` |
| git credential 未設定 | 在 git pull 前執行 `gh auth setup-git` |
| FTP 上傳失敗 | 記錄失敗檔案，繼續上傳其他檔案，最後彙總報告 |
| FTP 逾時 | 確認是否在內網環境，FTP 僅限內網存取 |
| 檔案被刪除 | git diff 顯示的刪除檔案不上傳，提示使用者手動從 FTP 刪除 |

---

## 使用範例

**使用者**：「部署系網」

**預期行為**：

1. SSH 連線，執行 `git fetch` + `git diff`
2. 顯示變更清單：
   ```
   偵測到 2 個檔案變更：
   - subsite/seminar/index.php
   - subsite/seminar/old_seminar.php
   
   是否繼續部署？
   ```
3. 使用者確認後，執行 `git pull`
4. 逐一上傳檔案，回報進度：
   ```
   [1/2] 上傳 subsite/seminar/index.php ... 完成
   [2/2] 上傳 subsite/seminar/old_seminar.php ... 完成
   ```
5. 回報部署完成

---

**使用者**：「同步系網到 FTP」

**預期行為**：同上流程。
