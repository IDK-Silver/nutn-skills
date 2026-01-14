# 系網部署

> **語言規範**：本文件使用**繁體中文**撰寫。

## 概述

從 GitHub 拉取最新變更，自動偵測修改的檔案並同步至 FTP。支援新增、修改、刪除、重命名等操作。

---

## 資料來源

| 項目 | 位置 |
|------|------|
| 系網 GitHub | https://github.com/IDK-Silver/nutn-csie-web |
| 內網 repo 路徑 | `~/Documents/Github/nutn-csie-web` |
| FTP 根目錄 | `/ftp/html/` |
| 連線資訊 | 參考 Project 文件 `NUTN_內網連線資訊.md` |

---

## 排除規則

以下檔案與資料夾**不應同步至 FTP**，即使出現在 git diff 結果中也必須跳過。

### 排除檔案

| 檔案名稱 | 說明 |
|----------|------|
| `CLAUDE.md` | AI 助理指令檔 |
| `README.md` | 專案說明文件 |
| `LICENSE` | 授權檔案 |
| `.gitignore` | Git 忽略規則 |
| `.gitattributes` | Git 屬性設定 |
| `Dockerfile` | Docker 映像定義 |
| `docker-compose.yml` | Docker Compose 設定 |
| `docker-compose.yaml` | Docker Compose 設定 |
| `.dockerignore` | Docker 忽略規則 |
| `*.dockerfile` | 任何 dockerfile |

### 排除資料夾

| 資料夾模式 | 說明 |
|------------|------|
| `.*` | 所有隱藏資料夾（`.github/`、`.vscode/` 等） |
| `*dev*` | 任何包含 "dev" 字樣的資料夾（`dev/`、`devtools/`、`dev-assets/` 等） |
| `node_modules/` | npm 依賴 |
| `docker/` | Docker 相關資料夾 |

### 過濾邏輯

在步驟 1 取得變更清單後，必須過濾掉符合以下條件的路徑：

```
# Pseudocode
EXCLUDED_FILES = ["CLAUDE.md", "README.md", "LICENSE", ".gitignore", ".gitattributes", 
                  "Dockerfile", "docker-compose.yml", "docker-compose.yaml", ".dockerignore"]

def should_exclude(path):
    filename = basename(path)
    
    # Check excluded files
    if filename in EXCLUDED_FILES:
        return True
    if filename.endswith(".dockerfile"):
        return True
    
    # Check excluded folders
    parts = path.split("/")
    for part in parts[:-1]:  # exclude filename, check directories only
        if part.startswith("."):          # hidden folders
            return True
        if "dev" in part.lower():         # folders containing "dev"
            return True
        if part == "node_modules":
            return True
        if part == "docker":
            return True
    
    return False
```

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

### 步驟 1：取得變更清單（含狀態）

SSH 到內網，fetch 遠端並比較差異：

```bash
ssh -tt {host} "cd ~/Documents/Github/nutn-csie-web; git fetch origin; git diff --name-status HEAD origin/main"
```

**輸出格式**：
```
M       subsite/seminar/index.php
A       images/new_logo.png
D       old_file.html
R100    old_name.php    new_name.php
```

**狀態代碼**：
| 代碼 | 含義 | 操作 |
|------|------|------|
| `A` | Added | 上傳新檔案 |
| `M` | Modified | 上傳覆蓋 |
| `D` | Deleted | 從 FTP 刪除 |
| `R` | Renamed | 刪除舊路徑 + 上傳新路徑 |

若無變更（輸出為空），告知使用者已是最新狀態，結束流程。

### 步驟 2：過濾排除項目

根據上方「排除規則」過濾變更清單。若過濾後無剩餘項目，告知使用者「變更皆為非網頁檔案，無需同步」。

回報過濾結果：
```
變更清單（已過濾 N 個非網頁項目）：
- [M] subsite/seminar/index.php
- [A] images/new_logo.png

已排除：
- README.md（說明文件）
- .github/workflows/deploy.yml（隱藏資料夾）
```

### 步驟 3：執行 git pull

```bash
ssh -tt {host} "gh auth setup-git; cd ~/Documents/Github/nutn-csie-web; git pull"
```

### 步驟 4：同步至 FTP

根據變更類型執行對應操作：

**上傳檔案（A/M）**：
```bash
ssh -tt {host} "cd ~/Documents/Github/nutn-csie-web; curl.exe -T {file_path} ftp://{ftp_host}/ftp/html/{file_path} --user {ftp_user}:{ftp_password}"
```

**刪除檔案（D）**：
```bash
ssh -tt {host} "curl.exe -Q 'DELE /ftp/html/{file_path}' ftp://{ftp_host} --user {ftp_user}:{ftp_password}"
```

**重命名（R）**：
```bash
# 先刪除舊路徑
ssh -tt {host} "curl.exe -Q 'DELE /ftp/html/{old_path}' ftp://{ftp_host} --user {ftp_user}:{ftp_password}"
# 再上傳新路徑
ssh -tt {host} "cd ~/Documents/Github/nutn-csie-web; curl.exe -T {new_path} ftp://{ftp_host}/ftp/html/{new_path} --user {ftp_user}:{ftp_password}"
```

**重要**：
- 必須使用 `curl.exe` 而非 `curl`（PowerShell 環境）
- 從 Project 文件取得 FTP 連線資訊

### 步驟 5：回報結果

列出所有操作及結果，確認部署完成。

---

## 錯誤處理

| 情況 | 處理方式 |
|------|----------|
| SSH 連線失敗 | 確認網路狀態，提示手動執行指令 |
| gh token 失效 | 提示使用者在內網主機執行 `gh auth login -h github.com` |
| git credential 未設定 | 在 git pull 前執行 `gh auth setup-git` |
| FTP 上傳失敗 | 記錄失敗檔案，繼續處理其他檔案，最後彙總報告 |
| FTP 刪除失敗 | 檔案可能已不存在，記錄並繼續 |
| FTP 逾時 | 確認是否在內網環境，FTP 僅限內網存取 |

---

## 使用範例

**使用者**：「部署系網」

**預期行為**：

1. SSH 連線，執行 `git fetch` + `git diff --name-status` 取得變更清單
2. 過濾排除項目，回報過濾結果
3. 執行 `git pull` 拉取最新程式碼
4. 根據變更類型執行對應操作，回報進度：
   ```
   [1/2] 上傳 subsite/seminar/index.php ... 完成
   [2/2] 上傳 images/new_logo.png ... 完成
   
   已跳過（非網頁檔案）：
   - README.md
   - .github/workflows/deploy.yml
   ```
5. 回報部署完成
