# cloud-setup 雲端環境設定

> **語言規範**：本文件使用**繁體中文**撰寫。

## 概述

此技能用於檢測和設定 Claude Code Cloud 環境，確保 agent-browser 等工具可正常運作。

---

## 環境檢測

執行以下指令檢測環境狀態：

```bash
echo "=== 環境檢測 ===" && \
echo "Node: $(which node 2>/dev/null || echo '未安裝')" && \
echo "Node 版本: $(node --version 2>/dev/null || echo 'N/A')" && \
echo "npm: $(which npm 2>/dev/null || echo '未安裝')" && \
echo "agent-browser: $(which agent-browser 2>/dev/null || echo '未安裝')" && \
echo "Shell: $SHELL" && \
echo "環境類型: $([ -f ~/.nvm/nvm.sh ] && echo '本地 (nvm)' || echo '雲端 (系統 Node)')"
```

---

## 環境類型判斷

| 特徵 | 雲端環境 | 本地環境 |
|------|----------|----------|
| Node.js 位置 | `/opt/node22/bin/node` | `~/.nvm/versions/node/...` |
| Shell | bash | zsh |
| nvm | 無 | 有 |

---

## 安裝 agent-browser

### 雲端環境（無需 nvm）

```bash
# 1. 安裝 agent-browser
npm install -g agent-browser

# 2. 安裝相容版本的 Playwright 和 Chromium
npm install -g playwright@1.57.0
npx playwright install chromium

# 3. 驗證安裝
agent-browser --version
```

**重要**：`agent-browser install` 可能安裝版本不相容的瀏覽器。建議直接使用上述 `npx playwright install chromium` 來確保版本匹配。

### 本地環境（使用 nvm/zsh）

參考 `agent-browser/SKILL.md` 的安裝指引。

---

## 執行 agent-browser

### 雲端環境

在雲端環境中，**不需要** `source ~/.zshrc` 前綴，直接執行即可：

```bash
agent-browser open https://example.com --headed
agent-browser snapshot -i
agent-browser close
```

### 本地環境

需要 zsh 環境載入：

```bash
source ~/.zshrc 2>/dev/null; agent-browser open https://example.com --headed
```

---

## 自動設定流程

當檢測到環境未設定時，執行以下步驟：

### 步驟 1：確認 Node.js 可用

```bash
node --version && npm --version
```

若失敗，表示環境異常，需人工處理。

### 步驟 2：檢查 agent-browser

```bash
which agent-browser
```

### 步驟 3：安裝（若未安裝）

```bash
npm install -g agent-browser
npm install -g playwright@1.57.0
npx playwright install chromium
```

### 步驟 4：驗證

```bash
agent-browser --version
```

### 步驟 5：測試（可選）

```bash
agent-browser open "data:text/html,<h1>Test</h1>" && agent-browser snapshot -i && agent-browser close
```

---

## 完整檢測與設定腳本

一鍵檢測並設定環境：

```bash
#!/bin/bash
echo "🔍 檢測環境..."

# 檢測環境類型
if [ -f ~/.nvm/nvm.sh ]; then
    ENV_TYPE="local"
    echo "📍 環境類型：本地 (nvm)"
else
    ENV_TYPE="cloud"
    echo "📍 環境類型：雲端"
fi

# 檢查 Node.js
if ! command -v node &> /dev/null; then
    echo "❌ Node.js 未安裝"
    exit 1
fi
echo "✅ Node.js: $(node --version)"

# 檢查 agent-browser
if command -v agent-browser &> /dev/null; then
    echo "✅ agent-browser 已安裝: $(agent-browser --version 2>/dev/null | head -1)"
else
    echo "⚠️  agent-browser 未安裝，開始安裝..."
    npm install -g agent-browser
    if [ $? -eq 0 ]; then
        echo "📦 安裝 Playwright 和 Chromium..."
        npm install -g playwright@1.57.0
        npx playwright install chromium
        echo "✅ agent-browser 安裝完成"
    else
        echo "❌ 安裝失敗"
        exit 1
    fi
fi

echo ""
echo "🎉 環境設定完成！"
echo "環境類型: $ENV_TYPE"
if [ "$ENV_TYPE" = "cloud" ]; then
    echo "提示：雲端環境直接執行 agent-browser 命令即可，不需要 source ~/.zshrc"
fi
```

---

## 常見問題

### Q: 如何判斷我在哪種環境？

執行環境檢測指令，看 Node.js 位置：
- `/opt/node22/...` → 雲端環境
- `~/.nvm/...` → 本地環境

### Q: 雲端環境的 agent-browser 指令失敗？

確認是否已安裝：
```bash
which agent-browser
```

若未安裝，執行：
```bash
npm install -g agent-browser
npm install -g playwright@1.57.0
npx playwright install chromium
```

### Q: 出現「Executable doesn't exist」錯誤？

這通常是 Playwright 瀏覽器版本不匹配。執行以下指令重新安裝正確版本：
```bash
npm install -g playwright@1.57.0
npx playwright install chromium
```

### Q: 需要每次新對話都重新安裝嗎？

不需要。雲端環境的全域安裝會保留。使用檢測指令確認狀態即可。

### Q: 網路連線失敗（ERR_TUNNEL_CONNECTION_FAILED）？

這是雲端環境的網路限制，某些外部網站可能無法存取。可用 `data:text/html,...` 格式測試本地 HTML 確認瀏覽器正常運作。

---

## 與其他技能的整合

當其他技能（如 workhour）需要使用 agent-browser 時：

1. 先執行環境檢測
2. 若未安裝，執行安裝流程
3. 根據環境類型選擇正確的執行方式：
   - 雲端：直接執行 `agent-browser ...`
   - 本地：使用 `source ~/.zshrc 2>/dev/null; agent-browser ...`
