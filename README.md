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

### 1. 安裝 MCP

| MCP | 用途 | 安裝方式 |
|-----|------|----------|
| Control Chrome | 瀏覽器自動化 | Extensions 搜尋安裝 |
| GitHub | 技能庫讀取、PR 建立 | [設定說明](#github-mcp-設定) |
| Desktop Commander | 檔案操作、指令執行 | [安裝說明](https://github.com/wonderwhy-er/DesktopCommanderMCP) |

### 2. 設定 Project Instructions

複製以下內容到 Claude Desktop 的 **Project Instructions**：

```
你是 NUTN 助理。對話開始時，使用 GitHub MCP 讀取 IDK-Silver/nutn-skills 的 PROJECT_INSTRUCTIONS.md，並依照其內容執行。
```

---

## GitHub MCP 設定

編輯 `claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<your_token>"
      }
    }
  }
}
```

**Token 權限：** `repo` 或 `public_repo`

設定完成後重啟 Claude Desktop。
