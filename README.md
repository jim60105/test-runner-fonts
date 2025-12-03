# GitHub Coding Agent 多語言字型渲染測試

此專案用於測試 GitHub Coding Agent 搭配 Playwright 在不同配置下的多語言字型渲染能力。

## 測試結果總覽

我們進行了四組測試，驗證不同配置下字型載入的情況：

| 測試案例 | 防火牆設定 | Playwright 設定 | 字型安裝 | 結果 |
|---------|-----------|----------------|---------|------|
| [PR #1](https://github.com/jim60105/test-runner-fonts/pull/1) | 開啟 | 預設值 | 無 | ❌ 字體未載入 |
| [PR #2](https://github.com/jim60105/test-runner-fonts/pull/2) | 關閉 | `--allowed-hosts *` | 無 | ✅ 字體成功載入 |
| [PR #3](https://github.com/jim60105/test-runner-fonts/pull/3) | 關閉 | 預設值 | 無 | ❌ 字體未載入 |
| [PR #4](https://github.com/jim60105/test-runner-fonts/pull/4) | 開啟 | 預設值 | 使用 [action-install-google-fonts](https://github.com/jim60105/action-install-google-fonts) | ✅ 字體成功載入 |

## 詳細測試結果

### PR #1：防火牆開啟 & Playwright 預設值

- **配置**：防火牆設定開啟、Playwright 設定如預設值
- **結果**：字體未載入，中文字顯示為方塊

![PR #1 截圖](https://github.com/user-attachments/assets/b2a16c51-cd42-47d5-93d4-e080fda418fb)

### PR #2：防火牆關閉 & Playwright `--allowed-hosts *`

- **配置**：防火牆設定關閉、Playwright 設定 `--allowed-hosts *`
- **結果**：字體成功載入，所有字型正確顯示

![PR #2 截圖](https://github.com/user-attachments/assets/ee734d45-bf41-4bb3-9087-7de262c3c491)

### PR #3：防火牆關閉 & Playwright 預設值

- **配置**：防火牆關閉、Playwright 設定如預設值
- **結果**：字體未載入

> 注意：即使防火牆設定關閉，Playwright 的預設設定仍會阻擋外部資源存取。

![PR #3 截圖](https://github.com/user-attachments/assets/c7729b0b-a68f-4814-b82d-387e74dc2cbf)

### PR #4：防火牆開啟 & 本地安裝字型

- **配置**：防火牆開啟、Playwright 設定如預設值、使用 [action-install-google-fonts](https://github.com/jim60105/action-install-google-fonts) 安裝字型在 runner 本地
- **結果**：字體成功載入

![PR #4 截圖](https://github.com/user-attachments/assets/6ef7b186-ea03-45e8-859f-7f9d5d7672b3)

---

## GitHub Coding Agent 搭配 Playwright 正確顯示字型指南

GitHub Coding Agent 內建的 Playwright MCP Server 預設只能存取本機資源（localhost 或 127.0.0.1），這導致需要從 Google Fonts 等外部來源載入字型的網頁無法正確渲染。以下是幾種解決方案：

### 背景知識

- **防火牆設定不會套用在 MCP 上**：Coding Agent 的防火牆設定只適用於透過 Bash 工具啟動的程序，不適用於 MCP 伺服器。
- **Playwright 預設限制**：Playwright MCP Server 預設只能存取 localhost 或 127.0.0.1 上的資源。

### 方案一：修改 Playwright MCP Server 設定 (`--allowed-hosts *`)

**設定方式**：在 `.github/copilot/mcp.json` 中設定 Playwright 的 `--allowed-hosts *` 參數

```json
{
  "mcpServers": {
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "@anthropic-ai/mcp-server-playwright@latest",
        "--allowed-hosts",
        "*"
      ]
    }
  }
}
```

**優點**：
- 設定簡單，只需修改配置檔即可
- 能夠載入所有外部資源（Google Fonts、CDN 等）
- 適合需要完整外部網路存取的場景

**缺點**：
- 開放所有外部存取可能帶來安全風險
- 需要關閉防火牆設定才能使 Playwright 存取外部資源
- 可能載入不必要的外部資源，增加載入時間

### 方案二：使用 action-install-google-fonts 安裝本地字型

**設定方式**：在 Copilot setup step 中使用 [jim60105/action-install-google-fonts](https://github.com/jim60105/action-install-google-fonts) Action 安裝字型

```yaml
# .github/copilot/setup-steps.yml
- name: Install fonts
  uses: jim60105/action-install-google-fonts@v1
  with:
    fonts: |
      Noto Sans TC
      Iansui
      Noto Color Emoji
```

**優點**：
- 可保持防火牆設定開啟，安全性較高
- 字型安裝在 runner 本地，不需要網路存取即可渲染
- Playwright 使用預設設定即可正常運作
- 字型載入更快速穩定，不受網路影響
- 適合需要特定字型且注重安全性的場景

**缺點**：
- 需要事先知道並指定所需的字型名稱
- 僅支援 Ubuntu runner（不支援 Windows 或 macOS）
- 若需要新增字型，需修改配置檔

### 方案比較

| 面向 | 方案一：修改 Playwright 設定 | 方案二：安裝本地字型 |
|-----|---------------------------|-------------------|
| 安全性 | 較低（開放外部存取） | 較高（保持預設限制） |
| 設定複雜度 | 簡單 | 中等 |
| 網路依賴 | 每次渲染都需要 | 只需安裝時 |
| 適用場景 | 開發測試、需要完整外部存取 | 生產環境、安全優先 |
| Runner 相容性 | 所有 | 僅 Ubuntu |

### 建議

對於大多數使用情況，**方案二（安裝本地字型）是推薦的做法**，原因如下：

1. **安全性優先**：保持防火牆和 Playwright 的預設安全設定
2. **穩定性**：本地字型不受網路波動影響
3. **效能**：減少外部資源載入時間

若你的使用場景需要存取多種外部資源，或無法事先確定所需字型，則可考慮**方案一**。

---

## 授權條款

本專案採用 [AGPL-3.0](LICENSE) 授權條款。