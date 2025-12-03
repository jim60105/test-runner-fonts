# GitHub Coding Agent 多語言字型渲染測試

此專案用於測試 GitHub Coding Agent 搭配 Playwright 在不同配置下的多語言字型渲染能力，並提供解決方案指南。

## 目錄

- [背景知識](#背景知識)
- [測試結果總覽](#測試結果總覽)
- [詳細測試結果](#詳細測試結果)
- [解決方案指南](#github-coding-agent-搭配-playwright-正確顯示字型指南)
- [參考資料](#參考資料)

---

## 背景知識

### GitHub Coding Agent 與 Playwright MCP

[GitHub Copilot Coding Agent](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent) 是 GitHub Copilot 的自主編程代理，能夠獨立完成開發任務。它內建了 [Playwright MCP Server](https://github.com/microsoft/playwright-mcp)，讓 Coding Agent 能夠進行瀏覽器自動化操作，包括網頁截圖、元素互動等功能。

### Model Context Protocol (MCP)

[Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) 是一個開放標準，定義了應用程式如何與大型語言模型 (LLM) 共享上下文。GitHub Coding Agent 透過 MCP 連接到各種工具和服務，擴展其能力。

根據 [GitHub 官方文件](https://docs.github.com/en/copilot/concepts/agents/coding-agent/mcp-and-coding-agent#default-mcp-servers)，Coding Agent 預設配置了以下 MCP 伺服器：

- **GitHub MCP Server**：提供對 GitHub 資料（如 Issues 和 Pull Requests）的存取
- **Playwright MCP Server**：提供瀏覽器自動化能力，可讀取、互動和截取網頁

### 防火牆與 MCP 的關係

根據 [GitHub 官方文件](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/customize-the-agent-firewall#limitations)：

> **Only applies to processes started by the agent**: The firewall only applies to processes started by the agent via its Bash tool. It does not apply to Model Context Protocol (MCP) servers or processes started in configured Copilot setup steps.

**重要**：Coding Agent 的防火牆設定**不會套用在 MCP 伺服器上**。防火牆只適用於透過 Bash 工具啟動的程式。

> **注意**：「關閉防火牆」意味著網路**暢通**（可以存取外部網路），而非阻斷狀態。

### Playwright MCP Server 的網路限制

根據 [GitHub 官方文件](https://docs.github.com/en/copilot/concepts/agents/coding-agent/mcp-and-coding-agent#default-mcp-servers)：

> By default, the Playwright MCP server is only able to access web resources hosted within Copilot's own environment, accessible on `localhost` or `127.0.0.1`.

Playwright MCP Server **預設只能存取本機資源**（localhost 或 127.0.0.1），無法存取外部網路資源。這是一個**獨立於防火牆設定的限制**。

---

## 測試結果總覽

我們進行了四組測試，驗證不同配置下字型載入的情況：

| 測試案例 | 防火牆設定 | Playwright 設定 | 字型安裝 | 結果 |
|---------|-----------|----------------|---------|------|
| [PR #1](https://github.com/jim60105/test-runner-fonts/pull/1) | 開啟（網路受限） | 預設值 | 無 | ❌ 字體未載入 |
| [PR #2](https://github.com/jim60105/test-runner-fonts/pull/2) | 關閉（網路暢通） | `--allowed-hosts *` | 無 | ✅ 字體成功載入 |
| [PR #3](https://github.com/jim60105/test-runner-fonts/pull/3) | 關閉（網路暢通） | 預設值 | 無 | ❌ 字體未載入 |
| [PR #4](https://github.com/jim60105/test-runner-fonts/pull/4) | 開啟（網路受限） | 預設值 | 使用 [action-install-google-fonts](https://github.com/jim60105/action-install-google-fonts) | ✅ 字體成功載入 |

### 關鍵發現

1. **防火牆設定對 MCP 無效**：即使關閉防火牆（PR #3），Playwright 仍無法載入外部字型，因為防火牆設定不適用於 MCP 伺服器。
2. **Playwright 需要明確允許外部存取**：必須透過 `--allowed-hosts *` 參數明確允許外部資源存取（PR #2）。
3. **本地安裝字型可繞過網路限制**：透過 setup steps 預先安裝字型（PR #4），可在保持預設安全設定的情況下正確渲染。

---

## 詳細測試結果

### PR #1：防火牆開啟 & Playwright 預設值

- **配置**：
  - 防火牆設定：開啟（網路受限）
  - Playwright 設定：預設值（僅允許 localhost）
  - 字型安裝：無
- **結果**：字體未載入，中文字顯示為方塊

![PR #1 截圖](https://github.com/user-attachments/assets/b2a16c51-cd42-47d5-93d4-e080fda418fb)

### PR #2：防火牆關閉 & Playwright `--allowed-hosts *`

- **配置**：
  - 防火牆設定：關閉（網路暢通）
  - Playwright 設定：`--allowed-hosts *`（允許所有外部存取）
  - 字型安裝：無
- **結果**：字體成功載入，所有字型正確顯示

此測試使用以下 MCP 配置：

```json
{
  "mcpServers": {
    "playwright": {
      "type": "local",
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--allowed-hosts", "*"],
      "tools": ["*"]
    }
  }
}
```

![PR #2 截圖](https://github.com/user-attachments/assets/ee734d45-bf41-4bb3-9087-7de262c3c491)

### PR #3：防火牆關閉 & Playwright 預設值

- **配置**：
  - 防火牆設定：關閉（網路暢通）
  - Playwright 設定：預設值（僅允許 localhost）
  - 字型安裝：無
- **結果**：字體未載入

> **重要發現**：即使防火牆設定關閉（網路暢通），Playwright MCP Server 的預設設定仍會阻擋外部資源存取。這證明了 Playwright 的網路限制是獨立於防火牆設定的。

![PR #3 截圖](https://github.com/user-attachments/assets/c7729b0b-a68f-4814-b82d-387e74dc2cbf)

### PR #4：防火牆開啟 & 本地安裝字型

- **配置**：
  - 防火牆設定：開啟（網路受限）
  - Playwright 設定：預設值（僅允許 localhost）
  - 字型安裝：使用 [action-install-google-fonts](https://github.com/jim60105/action-install-google-fonts) 安裝字型在 runner 本地
- **結果**：字體成功載入

> **重要發現**：透過在 Copilot setup steps 中預先安裝字型，Playwright 可以直接使用系統字型渲染，不需要網路存取外部字型資源。

![PR #4 截圖](https://github.com/user-attachments/assets/6ef7b186-ea03-45e8-859f-7f9d5d7672b3)

---

## GitHub Coding Agent 搭配 Playwright 正確顯示字型指南

基於上述測試結果，我們提供兩種解決方案來讓 Playwright MCP Server 正確渲染多語言字型。

### 方案一：修改 Playwright MCP Server 設定

透過配置 `--allowed-hosts *` 參數，允許 Playwright 存取外部網路資源。

**設定方式**：

1. 前往倉庫設定：**Settings** → **Copilot** → **Coding agent**
2. 在 **MCP configuration** 區塊加入以下配置：

```json
{
  "mcpServers": {
    "playwright": {
      "type": "local",
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--allowed-hosts", "*"],
      "tools": ["*"]
    }
  }
}
```

**Playwright MCP Server 支援的相關參數**：

根據 [Playwright MCP 官方文件](https://github.com/microsoft/playwright-mcp)：

| 參數 | 說明 |
|-----|------|
| `--allowed-hosts <hosts...>` | 允許存取的主機列表（逗號分隔）。預設僅允許伺服器綁定的主機。傳入 `*` 可停用主機檢查。 |
| `--allowed-origins <origins>` | 允許瀏覽器請求的**受信任**來源（分號分隔）。預設允許所有。**注意**：不作為安全邊界，不影響重定向。 |
| `--blocked-origins <origins>` | 阻擋瀏覽器請求的來源（分號分隔）。阻擋列表在允許列表之前評估。 |

**優點**：

- ✅ 設定簡單，只需修改 MCP 配置即可
- ✅ 能夠載入所有外部資源（Google Fonts、CDN 等）
- ✅ 適合需要完整外部網路存取的場景
- ✅ 不需要事先知道所需字型

**缺點**：

- ⚠️ 開放所有外部存取可能帶來安全風險
- ⚠️ 可能載入不必要的外部資源，增加載入時間
- ⚠️ 每次渲染都需要網路存取

### 方案二：使用 action-install-google-fonts 安裝本地字型

透過 Copilot setup steps 預先安裝字型到 runner 本地，讓 Playwright 可以直接使用系統字型。

**設定方式**：

建立 `.github/workflows/copilot-setup-steps.yml` 檔案：

```yaml
name: Copilot Setup Steps

on: workflow_dispatch

jobs:
  copilot-setup-steps:
    runs-on: ubuntu-latest
    steps:
      - name: Install fonts
        uses: jim60105/action-install-google-fonts@v2
        with:
          fonts: |
            Noto Sans TC
            Iansui
            Noto Color Emoji
```

根據 [action-install-google-fonts 文件](https://github.com/jim60105/action-install-google-fonts)，此 Action 支援以下參數：

| 參數 | 必填 | 預設值 | 說明 |
|-----|------|-------|------|
| `fonts` | ✅ 是 | - | 要安裝的字型列表（逗號或換行分隔） |
| `weights` | ❌ 否 | `'400,700'` | 要安裝的字型粗細（100-900） |

**常用 Google Fonts**：

- 中文字型：`Noto Sans TC`、`Noto Serif TC`、`Noto Sans SC`、`Noto Sans HK`
- 英文字型：`Roboto`、`Open Sans`、`Lato`、`Montserrat`
- 特殊字型：`Noto Color Emoji`、`Noto Emoji`

完整字型列表請參考 [Google Fonts](https://fonts.google.com/)。

**優點**：

- ✅ 可保持預設安全設定，安全性較高
- ✅ 字型安裝在 runner 本地，不需要網路存取即可渲染
- ✅ Playwright 使用預設設定即可正常運作
- ✅ 字型載入更快速穩定，不受網路影響
- ✅ 適合需要特定字型且注重安全性的場景

**缺點**：

- ⚠️ 需要事先知道並指定所需的字型名稱
- ⚠️ 僅支援 Ubuntu runner（不支援 Windows 或 macOS）
- ⚠️ 若需要新增字型，需修改配置檔

---

## 方案比較

| 面向 | 方案一：修改 Playwright 設定 | 方案二：安裝本地字型 |
|-----|---------------------------|-------------------|
| 安全性 | 較低（開放外部存取） | 較高（保持預設限制） |
| 設定複雜度 | 簡單 | 中等 |
| 網路依賴 | 每次渲染都需要 | 僅安裝時需要 |
| 適用場景 | 開發測試、需要完整外部存取 | 生產環境、安全優先 |
| Runner 相容性 | 所有 | 僅 Ubuntu |
| 彈性 | 高（支援任意外部資源） | 中（需預先指定字型） |

---

## 建議

對於大多數使用情況，**方案二（安裝本地字型）是推薦的做法**，原因如下：

1. **安全性優先**：保持 Playwright 的預設安全設定，減少潛在的安全風險
2. **穩定性**：本地字型不受網路波動影響，渲染結果更穩定
3. **效能**：減少外部資源載入時間，截圖更快完成
4. **一致性**：相同的字型在不同執行環境中產生一致的渲染結果

若你的使用場景需要存取多種外部資源，或無法事先確定所需字型，則可考慮**方案一**。

---

## 參考資料

### 官方文件

- [GitHub Copilot Coding Agent 文件](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent)
- [Customizing or disabling the firewall for GitHub Copilot coding agent](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/customize-the-agent-firewall)
- [Model Context Protocol (MCP) and GitHub Copilot coding agent](https://docs.github.com/en/copilot/concepts/agents/coding-agent/mcp-and-coding-agent)
- [Extending GitHub Copilot coding agent with the Model Context Protocol (MCP)](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/extend-coding-agent-with-mcp)
- [Customizing the development environment for GitHub Copilot coding agent](https://docs.github.com/en/copilot/customizing-copilot/customizing-the-development-environment-for-copilot-coding-agent)

### 相關專案

- [Playwright MCP Server](https://github.com/microsoft/playwright-mcp) - Microsoft 官方的 Playwright MCP 伺服器
- [action-install-google-fonts](https://github.com/jim60105/action-install-google-fonts) - 在 GitHub Workflows 中安裝 Google Fonts 的 Action
- [Model Context Protocol](https://modelcontextprotocol.io/introduction) - MCP 官方文件

---

## 授權條款

本專案採用 [AGPL-3.0](LICENSE) 授權條款。