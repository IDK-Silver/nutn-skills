# NUTN Skills

國立臺南大學（NUTN）助理技能庫，提供工時系統、學校行事曆等自動化功能。

## 可用技能

| 技能 | 說明 |
|------|------|
| workhour | 南大工時系統自動化填寫 |
| academic-calendar | 學校行事曆查詢 |
| skill-creator | 技能建立範本 |

---

## 使用方式

### 步驟 1：安裝 MCP

#### 從 Extensions 安裝

前往 **Settings → Extensions → Browse extensions**，搜尋並安裝：

- **Control Chrome** — 瀏覽器自動化
- **Filesystem** — 檔案讀寫

#### 手動設定

編輯 `claude_desktop_config.json`，加入以下內容：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<your_token>"
      }
    },
    "desktop-commander": {
      "command": "npx",
      "args": ["-y", "@anthropic/desktop-commander"]
    }
  }
}
```

**GitHub Token 權限：** `repo` 或 `public_repo`

Desktop Commander 詳細說明請參考 [DesktopCommanderMCP](https://github.com/wonderwhy-er/DesktopCommanderMCP)。

### 步驟 2：重啟 Claude Desktop

設定完成後重啟 Claude Desktop，確認所有 MCP 載入正常。

### 步驟 3：設定 Project Instructions

複製以下內容到 Claude Desktop 的 **Project Instructions**：

```
你是 NUTN 助理。對話開始時，使用 GitHub MCP 讀取 IDK-Silver/nutn-skills 的 PROJECT_INSTRUCTIONS.md，並依照其內容執行。
```
