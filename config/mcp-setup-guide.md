# MCP 伺服器配置指南

> **Onchain x CEX Strats** 專案需要多個 MCP 伺服器協同運作。
> 本指南將逐步引導你完成安裝、配置及驗證。

---

## 前置條件

在開始之前，請確保你的環境滿足以下要求：

| 項目 | 最低版本 | 檢查指令 |
|------|---------|---------|
| Node.js | v18+ | `node -v` |
| npm | v9+ | `npm -v` |
| npx | 隨 npm 一起安裝 | `npx -v` |
| Git | 任意版本 | `git --version` |

如果 Node.js 版本不足，建議使用 [nvm](https://github.com/nvm-sh/nvm) 安裝：

```bash
nvm install 18
nvm use 18
```

---

## 1. okx-trade-mcp (必要)

這是核心交易伺服器，負責所有 CEX 端的下單、查詢餘額、取得行情等操作。

### 安裝

```bash
npm install -g okx-trade-mcp
```

### 配置 API 憑證

建立 OKX 配置目錄及檔案：

```bash
mkdir -p ~/.okx
```

編輯 `~/.okx/config.toml`，貼上以下內容並替換你的金鑰：

```toml
default_profile = "demo"

[profiles.demo]
api_key = "your-demo-api-key"
secret_key = "your-demo-secret-key"
passphrase = "your-demo-passphrase"
demo = true

[profiles.live]
api_key = "your-live-api-key"
secret_key = "your-live-secret-key"
passphrase = "your-live-passphrase"
```

> **重要安全提醒：**
> - `config.toml` 包含敏感金鑰，請確保檔案權限為 600：
>   ```bash
>   chmod 600 ~/.okx/config.toml
>   ```
> - **永遠不要**將此檔案提交到 Git。專案的 `.gitignore` 已排除 `~/.okx/`，但請自行確認。
> - 建議先使用 `demo` 模式測試所有策略，確認無誤後再切換到 `live`。

### 驗證安裝

```bash
okx market ticker BTC-USDT
```

預期輸出應包含 BTC-USDT 的最新價格、24 小時成交量等資訊。如果看到認證錯誤，請檢查 `config.toml` 中的金鑰是否正確。

### 獲取 API Key

1. 前往 [okx.com](https://www.okx.com) → 頂部選單 → **API** → **Create API Key**
2. 設定權限：
   - **Read**：查詢行情、餘額（所有策略必需）
   - **Trade**：下單（執行交易時必需）
   - **Withdraw**：提幣（通常不需要，請勿勾選除非你知道自己在做什麼）
3. 設定 IP 白名單（強烈建議）：僅允許你的伺服器 IP
4. **Demo Trading API**（模擬交易）：
   - 前往 [okx.com](https://www.okx.com) → 右上角 → **模擬交易** → **API**
   - Demo API 使用虛擬資金，非常適合測試策略
   - Demo 模式下的 API Key 與正式環境的 API Key 是分開的

---

## 2. OnchainOS CLI (必要)

OnchainOS 負責所有鏈上操作：DEX 價格查詢、代幣資訊、鏈上交易執行等。

### 安裝

方法一 — 透過 npx（推薦）：

```bash
npx skills add okx/onchainos-skills
```

方法二 — 透過安裝腳本：

```bash
curl -sSL https://raw.githubusercontent.com/okx/onchainos-skills/main/install.sh | sh
```

### 配置

設定環境變數。你可以直接 export，或寫入 shell 設定檔（`~/.zshrc` 或 `~/.bashrc`）：

```bash
export OKX_API_KEY="your-api-key"
export OKX_SECRET_KEY="your-secret-key"
export OKX_PASSPHRASE="your-passphrase"
```

或者在專案根目錄建立 `.env` 檔案：

```bash
# .env — 請勿提交到 Git
OKX_API_KEY=your-api-key
OKX_SECRET_KEY=your-secret-key
OKX_PASSPHRASE=your-passphrase
```

> **注意：** `.env` 檔案已在 `.gitignore` 中排除。請務必確認不會意外提交。

### 驗證安裝

```bash
onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
```

預期輸出：ETH 在各 DEX 上的當前價格。如果指令找不到，請確認 PATH 中包含 npx 安裝的全域二進位目錄。

---

## 3. GoPlus MCP (必要 — 安全檢查)

GoPlus 提供代幣安全分析：蜜罐偵測、賣出稅檢查、合約驗證等。**所有策略在交易前都會呼叫 GoPlus 進行安全檢查。**

### 獲取 API Key

1. 前往 [https://gopluslabs.io/security-api](https://gopluslabs.io/security-api)
2. 點擊 **Get Started** → 註冊免費帳戶
3. 進入 Dashboard → 複製你的 API Key
4. 免費版有速率限制（約 100 次/天），對一般使用已足夠

### 配置

將以下內容加入專案根目錄的 `.mcp.json`（如果尚未存在則建立之）：

```json
{
  "mcpServers": {
    "goplus-security": {
      "command": "npx",
      "args": ["-y", "goplus-mcp"],
      "env": {
        "GOPLUS_API_KEY": "your-goplus-api-key"
      }
    }
  }
}
```

### 驗證

在 Claude Desktop 或 Claude Code 中呼叫：

```
check_token_security("1", "0xdac17f958d2ee523a2206206994597c13d831ec7")
```

- 參數 `"1"` 代表 Ethereum 主網（chain_id = 1）
- 此地址為 USDT 合約地址
- 預期回傳：`is_honeypot: 0`, `buy_tax: "0"`, `sell_tax: "0"` 等安全數據

---

## 4. CoinGecko MCP (建議 — 市場數據備份)

CoinGecko 提供廣泛的市場數據，作為 OKX 行情的備份來源，也用於獲取市值、交易量排名等輔助資訊。

### 安裝

無需單獨安裝 — 透過 npx 自動執行。

### 獲取 API Key（可選）

1. 前往 [https://www.coingecko.com/en/api](https://www.coingecko.com/en/api)
2. **免費版**：有速率限制（約 10-30 req/min），不需要 API Key
3. **Demo 版**：需要註冊，提供更高的速率限制
4. **Pro 版**：需要付費訂閱（$129/月起），提供更高速率和更多端點

> **建議：** 如果你只是偶爾使用市場數據作為參考，免費版就夠了。如果策略頻繁查詢 CoinGecko（例如 Yield Optimizer 比較多個代幣），考慮升級。

### 配置

加入 `.mcp.json`：

```json
{
  "mcpServers": {
    "coingecko": {
      "command": "npx",
      "args": ["-y", "@coingecko/coingecko-mcp"],
      "env": {
        "COINGECKO_API_KEY": "your-key-or-leave-empty-for-free"
      }
    }
  }
}
```

如果使用免費版，可以將 `COINGECKO_API_KEY` 留空或省略 `env` 區塊。

---

## 5. DeFiLlama MCP (建議 — TVL/收益數據)

DeFiLlama 提供 DeFi 協議的 TVL（鎖倉量）、收益率、協議列表等數據。對 Yield Optimizer 策略尤其重要。

### 安裝

無需單獨安裝 — 透過 npx 自動執行。

### 配置

加入 `.mcp.json`：

```json
{
  "mcpServers": {
    "defillama": {
      "command": "npx",
      "args": ["-y", "mcp-defillama"]
    }
  }
}
```

**不需要 API Key。** DeFiLlama 是完全免費且開源的。

### 驗證

呼叫以下工具確認連接：

```
get_protocol_data("aave")
```

預期回傳 Aave 的 TVL、鏈分佈、歷史數據等。

---

## 6. Optional: MistTrack MCP (地址風險檢查)

MistTrack 是由慢霧（SlowMist）提供的地址風險評分工具。可以偵測與詐騙、洗錢、駭客相關的地址。

> **注意：** 這是付費 API，只有在你需要進階的地址風險分析時才需要。對大多數使用者來說，GoPlus 的免費安全檢查已足夠。

### 獲取 API Key

1. 前往 [https://misttrack.io](https://misttrack.io)
2. 點擊 **API** → 查看價格方案
3. 購買 API 存取權並獲取金鑰

### 配置

```bash
npm install -g misttrack
```

加入 `.mcp.json`：

```json
{
  "mcpServers": {
    "misttrack": {
      "command": "npx",
      "args": ["-y", "misttrack-mcp"],
      "env": {
        "MISTTRACK_API_KEY": "your-misttrack-api-key"
      }
    }
  }
}
```

---

## 7. Optional: DexScreener MCP

DexScreener 提供即時的 DEX 交易對數據、價格走勢和流動性資訊。對 CEX-DEX Arbitrage 策略有幫助。

**完全免費，無需 API Key。**

### 配置

加入 `.mcp.json`：

```json
{
  "mcpServers": {
    "dexscreener": {
      "command": "npx",
      "args": ["-y", "@opensvm/dexscreener-mcp-server"]
    }
  }
}
```

### 驗證

查詢任意交易對：

```
search_pairs("WETH USDC")
```

預期回傳各 DEX 上 WETH/USDC 交易對的價格和流動性。

---

## 8. Optional: Rug Munch MCP

Rug Munch 專注於偵測 Rug Pull 風險，分析代幣合約的可疑模式。

- **免費版：** 每日 10 次呼叫
- **付費版：** 更高額度

適合與 GoPlus 搭配使用，提供第二層安全驗證。

### 配置

請參考 [Rug Munch 官方文件](https://rugmunch.xyz) 獲取最新的 MCP 配置方式，並加入 `.mcp.json`。

---

## 完整 .mcp.json 範例

以下是包含所有已配置伺服器的完整 `.mcp.json` 範例。請根據你的需求取消註解或移除不需要的伺服器：

```json
{
  "mcpServers": {
    "goplus-security": {
      "command": "npx",
      "args": ["-y", "goplus-mcp"],
      "env": {
        "GOPLUS_API_KEY": "your-goplus-api-key"
      }
    },
    "coingecko": {
      "command": "npx",
      "args": ["-y", "@coingecko/coingecko-mcp"],
      "env": {
        "COINGECKO_API_KEY": ""
      }
    },
    "defillama": {
      "command": "npx",
      "args": ["-y", "mcp-defillama"]
    },
    "dexscreener": {
      "command": "npx",
      "args": ["-y", "@opensvm/dexscreener-mcp-server"]
    }
  }
}
```

> **注意：** `okx-trade-mcp` 和 `OnchainOS` 使用獨立的配置方式（分別是 `~/.okx/config.toml` 和環境變數），不在 `.mcp.json` 中配置。

---

## 驗證所有連接

完成所有配置後，依序執行以下驗證步驟，確保每個伺服器都正常運作：

### 預檢清單

| # | 伺服器 | 驗證指令/呼叫 | 預期結果 |
|---|--------|-------------|---------|
| 1 | okx-trade-mcp | `system_get_capabilities` | 回傳 `authenticated: true` |
| 2 | OnchainOS | `onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | 回傳 ETH 價格 |
| 3 | GoPlus | `check_token_security("1", "0xdac17f958d2ee523a2206206994597c13d831ec7")` | 回傳 USDT 安全數據 |
| 4 | CoinGecko | `simple_price(ids: "bitcoin", vs_currencies: "usd")` | 回傳 BTC 美元價格 |
| 5 | DeFiLlama | `get_protocol_data("aave")` | 回傳 Aave TVL 數據 |

### 快速驗證腳本

你也可以在終端機中快速驗證前兩個：

```bash
# 驗證 okx-trade-mcp
okx market ticker BTC-USDT && echo "okx-trade-mcp: OK" || echo "okx-trade-mcp: FAILED"

# 驗證 OnchainOS
onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee && echo "OnchainOS: OK" || echo "OnchainOS: FAILED"
```

其餘伺服器（GoPlus、CoinGecko、DeFiLlama）在 Claude Desktop / Claude Code 中透過 MCP 工具呼叫驗證。

---

## 常見問題

| 問題 | 解決方案 |
|------|---------|
| `okx-trade-mcp` 連接失敗 | 檢查 `~/.okx/config.toml` 中的 API Key 是否正確。確認 `default_profile` 指向正確的 profile。 |
| `okx-trade-mcp` 回傳 `50125` 錯誤 | 地區限制。部分 API 在特定地區不可用，嘗試 VPN 或確認你的 IP 在白名單中。 |
| OnchainOS 命令找不到 | 重新安裝：`npx skills add okx/onchainos-skills`。確認 PATH 包含全域 npm bin 目錄（`npm bin -g`）。 |
| GoPlus API 回傳 401 | API Key 過期或無效。前往 [gopluslabs.io](https://gopluslabs.io) 重新獲取。 |
| GoPlus 回傳空結果 | 該代幣可能尚未被 GoPlus 收錄。嘗試用已知代幣（如 USDT）確認 API 正常。 |
| CoinGecko 速率限制 (429) | 免費版限制嚴格（10-30 req/min）。減少請求頻率，或升級到 Pro 版。 |
| DeFiLlama 無回應 | DeFiLlama 偶爾會維護。等待幾分鐘後重試。這不會影響交易策略的執行。 |
| `.mcp.json` 語法錯誤 | 使用 JSON 校驗工具檢查：`cat .mcp.json \| python3 -m json.tool` |
| npx 下載超時 | 網路問題。嘗試設定 npm registry mirror：`npm config set registry https://registry.npmmirror.com`（中國大陸使用者）|
| Demo 模式無法下單 | 確認 `config.toml` 中 demo profile 設定了 `demo = true`，且使用的是 Demo API Key。 |

---

## 安全最佳實踐

1. **API Key 保管：** 所有 API Key 只存放在 `~/.okx/config.toml`、`.env` 或系統環境變數中，絕不硬編碼在程式碼或 skill 檔案中。
2. **最小權限原則：** API Key 只授予必要的權限。如果策略只需要讀取行情，不要授予 Trade 權限。
3. **IP 白名單：** 為 OKX API Key 設定 IP 白名單，限制只有你的伺服器可以使用。
4. **Demo 優先：** 新策略永遠先在 Demo 模式下測試。確認邏輯正確後再切換到 Live。
5. **定期輪換：** 每 90 天輪換一次 API Key，降低洩漏風險。
6. **監控異常：** 定期檢查 OKX 帳戶的 API 呼叫記錄，留意異常活動。
