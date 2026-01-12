# NUTN Skills

國立臺南大學（NUTN）助理技能庫，提供工時系統、學校行事曆等自動化功能。

## 可用技能

| 技能 | 說明 |
|------|------|
| workhour | 南大工時系統自動化填寫 |
| academic-calendar | 學校行事曆查詢 |
| skill-creator | 技能建立範本 |

---

## Claude Code 使用方式

### 需求

- 電腦需安裝 git

### MCP 安裝

安裝 **Control Chrome** MCP：

```bash
claude mcp add anthropic-claude-in-chrome -- npx -y anthropic-claude-in-chrome
```

### 使用方式

直接在專案目錄使用，會自動讀取 `CLAUDE.md`。

---

## Claude Desktop 使用方式

### 需求

- Control Chrome MCP
- Filesystem MCP
- GitHub MCP

### MCP 安裝

前往 **Settings → Extensions → Browse extensions** 搜尋並安裝：

- **Control Chrome**
- **Filesystem**

* **GitHub MCP Server** 需手動設定 `claude_desktop_config.json`：
	**Token 權限需求：**
  - `repo`（完整控制）- 建立 PR 所需
  - 或 `public_repo` - 僅對公開 repo
  ```json
  {
    "mcpServers": {
      "github": {
        "command": "docker",
        "args": [
          "run", "-i", "--rm",
          "-e", "GITHUB_PERSONAL_ACCESS_TOKEN",
          "ghcr.io/github/github-mcp-server"
        ],
        "env": {
          "GITHUB_PERSONAL_ACCESS_TOKEN": "<your_token>"
        }
      }
    }
  }
  ```


設定完成後重啟 Claude Desktop。

### 使用方式

1. 將 `PROJECT_INSTRUCTIONS.md` 的內容複製到 Claude Desktop 的 Project Instructions

2. Claude 會在對話開始時自動執行：
   ```bash
   git clone https://github.com/IDK-Silver/nutn-skills.git ~/nutn-skills
   ```
