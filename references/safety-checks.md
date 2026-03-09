# Safety Checks — Consolidated Procedures

This document defines all safety checks, risk limits, error codes, and security protocols that apply across every skill in the Onchain x CEX Strats project.

> **Cardinal rule:** No recommendation is ever output without passing the Pre-Trade Safety Checklist.
> If any check returns BLOCK, the entire recommendation is suppressed and the user sees an error.

---

## Table of Contents

1. [Pre-Trade Safety Checklist](#1-pre-trade-safety-checklist)
2. [Per-Strategy Risk Limits](#2-per-strategy-risk-limits)
3. [Error Code Catalog](#3-error-code-catalog)
4. [Security Tool Delegation Map](#4-security-tool-delegation-map)
5. [Account Safety Protocol](#5-account-safety-protocol)

---

## 1. Pre-Trade Safety Checklist

Every skill runs through these checks **in order** before producing a recommendation. A BLOCK at any step halts the pipeline immediately.

| # | Check | Tool | BLOCK Threshold | WARN Threshold | Error Code |
|---|-------|------|----------------|----------------|------------|
| 1 | MCP connectivity | `system_get_capabilities` | Server not reachable | — | `MCP_NOT_CONNECTED` |
| 2 | Authentication | `system_get_capabilities` | `authenticated: false` | — | `AUTH_FAILED` |
| 3 | Data freshness | Internal timestamp comparison | > 60s stale (> 5s for arbitrage skills) | > 30s stale | `DATA_STALE` |
| 4 | Token honeypot | GoPlus `check_token_security` | `is_honeypot === "1"` | — | `SECURITY_BLOCKED` |
| 5 | Token tax rate | GoPlus `check_token_security` | buy > 5% OR sell > 10% | buy > 1% | `SECURITY_BLOCKED` |
| 6 | Holder concentration | GoPlus `check_token_security` | Top 10 non-contract holders > 80% | Top 10 > 50% | `SECURITY_BLOCKED` |
| 7 | Contract verified | GoPlus `check_token_security` | `is_open_source === "0"` | — | `SECURITY_BLOCKED` |
| 8 | Liquidity depth | OnchainOS `dex-token price-info` | `liquidityUsd < $100,000` | `liquidityUsd < $500,000` | `INSUFFICIENT_LIQUIDITY` |
| 9 | Price impact | OnchainOS `dex-swap quote` | `priceImpactPercent > 2%` | `priceImpactPercent > 0.5%` | `INSUFFICIENT_LIQUIDITY` |
| 10 | Gas vs profit ratio | `onchain-gateway gas` + calculation | Gas cost > 30% of gross spread | Gas cost > 10% of spread | `NOT_PROFITABLE` |
| 11 | Net profitability | `profitability-calculator` | `net_profit <= 0` | `profit_to_cost_ratio < 2` | `NOT_PROFITABLE` |

### Check Execution Flow

```
START
  │
  ├─ Check 1-2: Infrastructure   ── BLOCK? → Output error, STOP
  │
  ├─ Check 3: Freshness          ── BLOCK? → Refetch data, retry once → BLOCK? → STOP
  │
  ├─ Check 4-7: Token Security   ── BLOCK? → Output security alert, STOP
  │                                  WARN?  → Attach warning labels, CONTINUE
  │
  ├─ Check 8-9: Liquidity        ── BLOCK? → Output liquidity warning, STOP
  │                                  WARN?  → Attach liquidity note, CONTINUE
  │
  ├─ Check 10-11: Profitability  ── BLOCK? → Output "not profitable", STOP
  │                                  WARN?  → Attach margin warning, CONTINUE
  │
  └─ ALL PASSED → Output recommendation with any accumulated WARNs
```

### Special Cases by Skill

| Skill | Additional Checks | Modified Thresholds |
|-------|-------------------|-------------------|
| cex-dex-arbitrage | Data freshness BLOCK at 5s (not 60s) | Price impact BLOCK at 1% (not 2%) |
| funding-rate-arbitrage | Verify funding settlement hasn't passed | — |
| basis-trading | Verify futures haven't expired | Min 14 days to expiry |
| yield-optimizer | DeFiLlama TVL check (min $10M) | — |
| smart-money-tracker | Signal age < 30 min | ALWAYS require human approval |

---

## 2. Per-Strategy Risk Limits

Default risk limits for each strategy. Users may override via `config/risk-limits.example.yaml`, but cannot exceed the hard caps defined here.

### cex-dex-arbitrage

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| max_trade_size | $2,000 | $10,000 | Maximum USD value per arbitrage trade |
| max_concurrent_arbs | 3 | 5 | Maximum simultaneous open arbitrage positions |
| min_net_profit | $5 | — | Minimum net profit after all costs to recommend |
| max_price_age_sec | 5 | 5 | Maximum age of price data in seconds |
| min_liquidity_usd | $100,000 | — | Minimum DEX liquidity to consider |
| max_slippage_pct | 0.5% | 2% | Maximum acceptable slippage |

### funding-rate-arbitrage

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| max_trade_size | $5,000 | $50,000 | Maximum USD value per position |
| max_leverage | 2x | 5x | Maximum leverage for the hedge leg |
| max_concurrent_positions | 2 | 5 | Maximum simultaneous carry positions |
| min_annualized_yield | 5% | — | Minimum APY to recommend |
| review_period_days | 7 | — | Re-evaluate positions every N days |
| max_funding_rate_std_dev | 3 | — | Skip if historical funding volatility is too high |

### basis-trading

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| max_trade_size | $10,000 | $100,000 | Maximum USD value per basis trade |
| max_leverage | 1x | 1x | **No leverage** — spot-futures basis only |
| min_days_to_expiry | 14 | 7 | Minimum days until futures expiry |
| min_annualized_basis | 3% | — | Minimum annualized basis yield to recommend |
| max_concurrent_positions | 2 | 3 | Maximum simultaneous basis positions |

### yield-optimizer

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| max_single_protocol_pct | 25% | 50% | Maximum allocation to any single DeFi protocol |
| min_tvl_usd | $10,000,000 | $1,000,000 | Minimum protocol TVL to consider |
| min_yield_advantage | 1% | — | Minimum yield improvement to recommend switching |
| max_gas_cost_pct | 5% | — | Gas cost as % of first-year yield must be below this |
| approved_protocol_types | lending, staking, LP | — | Allowed protocol categories |

### smart-money-tracker

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| max_copy_size | $1,000 | $5,000 | Maximum USD value per copy trade |
| require_human_approval | **ALWAYS** | **ALWAYS** | Cannot be overridden — every copy plan needs confirmation |
| max_signal_age_min | 30 | 60 | Maximum age of signal in minutes |
| min_wallet_pnl_pct | 50% | — | Minimum tracked wallet historical PnL to follow |
| min_wallet_win_rate | 60% | — | Minimum tracked wallet win rate |
| blocked_wallet_types | — | — | Wallets flagged by GoPlus `check_address_security` |

---

## 3. Error Code Catalog

Complete catalog of error codes, conditions, user-facing messages, and recovery actions.

| Code | Condition | User Message (ZH) | User Message (EN) | Recovery Action |
|------|-----------|-------------------|-------------------|-----------------|
| `MCP_NOT_CONNECTED` | MCP server unreachable | MCP 伺服器無法連線，請檢查 okx-trade-mcp 是否啟動 | MCP server unreachable. Check if okx-trade-mcp is running. | Verify MCP config, restart server |
| `AUTH_FAILED` | API key invalid or expired | API 認證失敗，請檢查 OKX API 金鑰設定 | API authentication failed. Check OKX API key configuration. | Update `~/.okx/config.toml` |
| `DATA_STALE` | Price/market data too old | 市場數據已過期（超過 {threshold} 秒），正在重新獲取... | Market data stale (> {threshold}s). Refetching... | Auto-retry fetch, then fail if still stale |
| `SECURITY_BLOCKED` | GoPlus check failed (honeypot, tax, ownership) | 安全檢查未通過：{reason}。此代幣存在風險，不建議操作。 | Security check failed: {reason}. This token is flagged as risky. | Do not proceed. Show specific GoPlus findings. |
| `SECURITY_WARN` | GoPlus check returned warnings | 安全提醒：{warnings}。請自行評估風險。 | Security notice: {warnings}. Assess risk before proceeding. | Display warnings, continue with labels |
| `SECURITY_UNAVAILABLE` | GoPlus API unreachable | 安全檢查服務暫時無法使用，請謹慎操作 | Security check service unavailable. Proceed with extreme caution. | Retry once, then output WARN and continue |
| `INSUFFICIENT_LIQUIDITY` | DEX liquidity below threshold | 鏈上流動性不足（{actual}），最低要求 {required} | Onchain liquidity insufficient ({actual}), minimum required: {required} | Show liquidity details, suggest smaller size |
| `PRICE_IMPACT_HIGH` | Swap price impact exceeds limit | 預估價格影響 {impact}% 過高（上限 {limit}%） | Estimated price impact {impact}% exceeds limit ({limit}%) | Suggest reducing trade size or splitting |
| `NOT_PROFITABLE` | Net P&L is zero or negative | 扣除所有成本後淨利潤為負（{net_pnl}），不建議執行 | Net profit is negative after all costs ({net_pnl}). Not recommended. | Show full cost breakdown |
| `MARGIN_TOO_THIN` | Profit-to-cost ratio < 2 | 利潤空間偏薄（利潤/成本比 = {ratio}），風險較高 | Thin margin (profit/cost ratio = {ratio}). Higher risk. | Display sensitivity analysis |
| `GAS_TOO_HIGH` | Gas cost exceeds % of spread | Gas 費用佔價差 {pct}%，超過上限 {limit}% | Gas cost is {pct}% of spread, exceeding {limit}% limit | Wait for lower gas or switch chains |
| `RATE_LIMITED` | API rate limit hit | API 請求頻率超限，{wait}秒後重試 | API rate limit reached. Retrying in {wait}s. | Auto-retry with backoff |
| `INSTRUMENT_NOT_FOUND` | Invalid instId or token address | 找不到交易對 {instId}，請確認名稱是否正確 | Instrument {instId} not found. Verify the identifier. | Suggest similar instruments |
| `LEVERAGE_EXCEEDED` | Requested leverage > limit | 槓桿倍數 {requested}x 超過上限 {limit}x | Leverage {requested}x exceeds limit {limit}x | Cap at limit and inform user |
| `POSITION_LIMIT` | Max concurrent positions reached | 已達持倉上限（{current}/{max}），請先關閉現有倉位 | Position limit reached ({current}/{max}). Close existing positions first. | Show current open positions |
| `TRADE_SIZE_EXCEEDED` | Trade size > per-strategy max | 交易金額 ${amount} 超過上限 ${limit} | Trade size ${amount} exceeds limit ${limit} | Cap at limit and inform user |
| `FUTURES_EXPIRED` | Futures contract has expired or < min days | 期貨合約已到期或距到期不足 {min_days} 天 | Futures contract expired or < {min_days} days to expiry | Suggest next-expiry contract |
| `SIGNAL_TOO_OLD` | Smart money signal age > threshold | 信號已超過 {age} 分鐘，可能已失效 | Signal is {age} minutes old. May no longer be actionable. | Show current token price vs signal price |
| `HUMAN_APPROVAL_REQUIRED` | Copy trade needs confirmation | 此操作需要您的確認才能繼續 | This action requires your explicit approval to proceed. | Wait for user confirmation |

---

## 4. Security Tool Delegation Map

Security checks are delegated to specialized external tools. The skills do NOT implement their own security logic.

| Check Category | Primary Tool | Fallback | Notes |
|---------------|-------------|----------|-------|
| **Token honeypot/tax/ownership** | GoPlus `check_token_security` | None (BLOCK if unavailable and token is unknown) | Mandatory for all onchain tokens |
| **Token holder concentration** | GoPlus `check_token_security` (holders array) | OnchainOS `dex-token holders` | GoPlus provides is_contract flag for filtering |
| **Contract verification** | GoPlus `check_token_security` (is_open_source) | None | Unverified = BLOCK |
| **Approval/allowance risk** | GoPlus `check_approval_security` | None | Used when evaluating DeFi protocol interactions |
| **Wallet/address screening** | GoPlus `check_address_security` | None | Used by smart-money-tracker before following a wallet |
| **Phishing URL detection** | GoPlus `check_phishing_site` | None | Used when outputting any external links |
| **DEX liquidity depth** | OnchainOS `dex-token price-info` | OnchainOS `dex-swap quote` (price impact as proxy) | liquidityUsd is the primary metric |
| **Price impact estimation** | OnchainOS `dex-swap quote` | None | priceImpactPercent at the intended trade size |
| **Gas cost estimation** | OnchainOS `onchain-gateway gas` + `gas-limit` | None | Calculate gas_cost = gasPrice * gasLimit |
| **Transaction simulation** | OnchainOS `onchain-gateway simulate` | None | Optional pre-execution check for complex interactions |
| **CEX market data integrity** | okx-trade-mcp `market_get_ticker` timestamp | None | Compare `ts` to current time for staleness |
| **CEX account status** | okx-trade-mcp `system_get_capabilities` | None | Check authenticated + mode |

### When to Use Each Tool

```
User mentions a TOKEN ADDRESS:
  → GoPlus check_token_security (ALWAYS)
  → OnchainOS dex-token price-info (for liquidity)
  → OnchainOS dex-swap quote (for price impact at trade size)

User mentions a WALLET ADDRESS (to follow/track):
  → GoPlus check_address_security (ALWAYS)

User mentions a DeFi PROTOCOL URL:
  → GoPlus check_phishing_site

Any ONCHAIN TRANSACTION is being evaluated:
  → OnchainOS onchain-gateway gas (for cost)
  → OnchainOS onchain-gateway simulate (optional validation)

Any CEX DATA is being used:
  → okx-trade-mcp system_get_capabilities (session start)
  → Timestamp freshness check (every data fetch)
```

---

## 5. Account Safety Protocol

### Demo vs Live Mode

The system defaults to **demo mode** at all times. Switching to live mode requires explicit user action.

#### Demo Mode (Default)

- MCP server config: `okx-DEMO-simulated-trading`
- All prices and positions are simulated
- All outputs include header: `[DEMO]`
- All tools function identically to live mode
- No real funds at risk
- **No confirmation required to use demo mode**

#### Live Mode

- MCP server config: `okx-LIVE-trading`
- Real market data and real account positions
- All outputs include header: `[LIVE]`
- Recommendations are still analysis-only (no auto-execution)

#### Switching from Demo to Live

The following protocol MUST be followed when a user requests live mode:

```
1. User explicitly says "live", "真實帳戶", "real account", or similar
2. System confirms the switch with a clear warning:

   ⚠️ 您正在切換至真實帳戶模式。
   - 所有數據將來自您的真實 OKX 帳戶
   - 建議仍為分析建議，不會自動執行交易
   - 請確認：輸入 "確認" 或 "confirm" 繼續

3. User must reply with explicit confirmation
4. System verifies authentication via system_get_capabilities
5. If authenticated: switch and display [LIVE] header
6. If NOT authenticated: show AUTH_FAILED error, remain in demo
```

#### Switching from Live to Demo

- Can be done at any time without confirmation
- User says "demo", "模擬", "switch to demo", or similar
- System immediately switches and displays `[DEMO]` header

#### Session Rules

| Rule | Description |
|------|-------------|
| Default on startup | Always demo mode |
| Timeout | If no activity for 30 minutes, revert to demo mode |
| Error fallback | If live mode encounters AUTH_FAILED, revert to demo with notification |
| Header requirement | EVERY output must show `[DEMO]` or `[LIVE]` — no exceptions |
| No auto-execution | Even in live mode, skills only provide recommendations. The `[RECOMMENDATION ONLY]` header is always present. |

### Output Header Format

Every skill output begins with this header block:

```
[DEMO] [RECOMMENDATION ONLY — 不會自動執行]
```

or

```
[LIVE] [RECOMMENDATION ONLY — 不會自動執行]
```

This header is non-negotiable and cannot be suppressed by any configuration.

### API Key Security

| Rule | Description |
|------|-------------|
| Storage | API keys stored in `~/.okx/config.toml` only — never in project files |
| Permissions | OKX API keys should have **read-only** permissions where possible |
| Rotation | Recommend key rotation every 90 days |
| IP whitelist | Recommend enabling IP whitelist on OKX API settings |
| Never log | API keys must never appear in logs, outputs, or error messages |
| Never commit | `.okx/` and any `*.toml` with credentials must be in `.gitignore` |
