---
name: cex-dex-arbitrage
description: >
  Detects and evaluates price differentials between OKX CEX and onchain DEX venues.
  Comprehensive arbitrage opportunity scanning, single-pair evaluation, continuous monitoring,
  and historical spread backtesting. Analysis and recommendation only — does NOT execute trades.
  Trigger phrases include: "arbitrage", "套利", "CEX-DEX", "price difference", "價差套利",
  "搬磚", "arb scan", "spread hunting", "搵差價", "跨場所套利", "arb opportunity",
  "DEX 同 CEX 差幾多", "有冇得搬", "搬磚機會".
  Do NOT use for: trade execution, funding rate arbitrage (use funding-rate-arbitrage),
  basis trading (use basis-trading), yield comparison (use yield-optimizer),
  pure price checks without arb context (use price-feed-aggregator).
  Requires: okx-trade-mcp (market data) + OnchainOS CLI (DEX data) + GoPlus MCP (token security).
allowed-tools: >
  okx-DEMO-simulated-trading:market_get_ticker,
  okx-DEMO-simulated-trading:market_get_orderbook,
  okx-DEMO-simulated-trading:market_get_candles,
  okx-DEMO-simulated-trading:system_get_capabilities,
  okx-LIVE-real-money:market_get_ticker,
  okx-LIVE-real-money:market_get_orderbook,
  okx-LIVE-real-money:market_get_candles,
  okx-LIVE-real-money:system_get_capabilities
---

# cex-dex-arbitrage

Flagship strategy skill for the Onchain x CEX Strats system. Detects, evaluates, and monitors price differentials between OKX CEX and onchain DEX venues across multiple chains. Integrates GoPlus security checks, OnchainOS DEX liquidity analysis, and the profitability-calculator cost engine to produce ranked, risk-assessed arbitrage recommendations.

---

## 1. Role

**Primary strategy skill** — the most frequently used skill in the system.

This skill is responsible for:
- Scanning multiple asset/chain combinations for CEX-DEX price differentials
- Evaluating individual opportunities with full cost analysis and security checks
- Continuous spread monitoring with threshold-based alerts
- Historical spread backtesting over configurable lookback periods
- Producing ranked, actionable recommendations with safety labels

This skill does **NOT**:
- Execute any trades (read-only, analysis only)
- Place orders on CEX or sign onchain transactions
- Manage positions, portfolios, or account state
- Perform funding rate analysis (delegate to `funding-rate-arbitrage`)
- Perform basis/futures analysis (delegate to `basis-trading`)
- Compare DeFi yields (delegate to `yield-optimizer`)

**Key Principle:** Every recommendation passes through GoPlus security checks AND the profitability-calculator before being output. If either gate fails, the opportunity is suppressed with a clear explanation.

---

## 2. Language

Match the user's language. Default: Traditional Chinese (繁體中文).

Technical labels may remain in English regardless of language:
- `bps`, `spread`, `bid`, `ask`, `PnL`, `slippage`, `gas`, `gwei`, `taker`, `maker`
- Timestamps always displayed in UTC

Examples:
- User writes "scan for arb" --> respond in English
- User writes "幫我掃套利機會" --> respond in Traditional Chinese
- User writes "搬磚機會多唔多" --> respond in Cantonese-style Traditional Chinese
- User writes "有没有套利机会" --> respond in Simplified Chinese

---

## 3. Account Safety

| Rule | Detail |
|------|--------|
| Default mode | Demo (`okx-DEMO-simulated-trading`) |
| Mode display | Every output header shows `[DEMO]` or `[LIVE]` |
| Read-only | This skill performs **zero** write operations — no trades, no transfers, no approvals |
| Recommendation header | Always show `[RECOMMENDATION ONLY — 不會自動執行]` |
| Live switch | Requires explicit user confirmation per protocol in `references/safety-checks.md` |

Even in `[LIVE]` mode, this skill only reads market data and produces recommendations. There is no risk of accidental execution.

---

## 4. Pre-flight (Machine-Executable Checklist)

Run these checks **in order** before any command. BLOCK at any step halts execution.

| # | Check | Command / Tool | Success Criteria | Failure Action |
|---|-------|---------------|-----------------|----------------|
| 1 | okx-trade-mcp connected | `system_get_capabilities` (DEMO or LIVE server) | `authenticated: true`, `modules` includes `"market"` | BLOCK — output `MCP_NOT_CONNECTED`. Tell user to verify `~/.okx/config.toml` and restart MCP server. |
| 2 | okx-trade-mcp mode | `system_get_capabilities` → `mode` field | Returns `"demo"` or `"live"` matching expected mode | WARN — display actual mode in header. If user requested live but got demo, surface mismatch. |
| 3 | OnchainOS CLI installed | `which onchainos` (Bash) | Exit code 0, returns a valid path | BLOCK — DEX venue required for this skill. Tell user: `npx skills add okx/onchainos-skills` |
| 4 | OnchainOS CLI functional | `onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | Returns valid JSON with `priceUsd` field | BLOCK — DEX venue required. Suggest checking network connectivity. |
| 5 | GoPlus MCP available | `check_token_security` with a known-good token (e.g., WETH on Ethereum) | Returns valid security object | WARN — security checks unavailable. Proceed but label ALL outputs `[SECURITY UNCHECKED]`. |
| 6 | risk-limits.yaml loaded | Read `config/risk-limits.yaml` or fall back to defaults from `references/safety-checks.md` Section 2 | Valid YAML with `cex-dex-arbitrage` key | INFO — using default risk limits (max_trade_size=$2,000, min_liquidity_usd=$100,000). |

### Pre-flight Decision Tree

```
Check 1 FAIL → BLOCK (cannot proceed without CEX data)
Check 1 PASS → Check 2
  Check 2 mismatch → WARN + continue
  Check 2 PASS → Check 3
    Check 3 FAIL → BLOCK (DEX venue mandatory for CEX-DEX arbitrage)
    Check 3 PASS → Check 4
      Check 4 FAIL → BLOCK (DEX venue mandatory)
      Check 4 PASS → Check 5
        Check 5 FAIL → WARN: security unchecked, continue with labels
        Check 5 PASS → Check 6
          Check 6 FAIL → INFO: using default limits
          Check 6 PASS → ALL SYSTEMS GO
```

Unlike `price-feed-aggregator`, this skill **cannot** fall back to CEX-only mode — both CEX and DEX venues are mandatory for arbitrage detection.

---

## 5. Skill Routing Matrix

| User Need | Use THIS Skill? | Delegate To |
|-----------|----------------|-------------|
| "有冇套利機會" / "find arb opportunities" | Yes — `scan` | — |
| "ETH 喺 Base 同 OKX 有冇價差" / "ETH arb on Base" | Yes — `evaluate` | — |
| "持續監控 BTC 價差" / "monitor BTC spread" | Yes — `monitor` | — |
| "上個月 SOL 套利機會多唔多" / "backtest SOL arb" | Yes — `backtest` | — |
| "搬磚" / "搵差價" / "arb scan" | Yes — `scan` (default command for ambiguous arb intent) | — |
| "BTC 而家幾錢" / "what's the BTC price" | No | `price-feed-aggregator.snapshot` |
| "CEX 同 DEX 價差幾多" (no arb context, just price comparison) | No | `price-feed-aggregator.spread` |
| "呢個交易賺唔賺" / "is this profitable" | No | `profitability-calculator.estimate` |
| "呢個幣安唔安全" / "is this token safe" | No | GoPlus MCP directly |
| "資金費率套利" / "funding rate arb" | No | `funding-rate-arbitrage` |
| "基差交易" / "basis trade" | No | `basis-trading` |
| "邊度收益最好" / "best yield" | No | `yield-optimizer` |
| "聰明錢買咗乜" / "smart money" | No | `smart-money-tracker` |
| "幫我買" / "execute trade" | No | Refuse — no skill executes trades |
| "我嘅帳戶餘額" / "my balance" | No | okx-trade-mcp `account_*` directly |

---

## 6. Command Index

| Command | Function | Read/Write | Description |
|---------|----------|-----------|-------------|
| `scan` | Multi-asset, multi-chain arb scanner | Read | Scan multiple assets across multiple chains for profitable CEX-DEX price differentials |
| `evaluate` | Single-opportunity deep analysis | Read | Full cost breakdown, security check, and profitability analysis for one specific opportunity |
| `monitor` | Continuous spread monitoring | Read | Poll prices at intervals and alert when spread exceeds threshold |
| `backtest` | Historical spread analysis | Read | Analyze historical spread patterns over a lookback period for one asset/chain |

---

## 7. Parameter Reference

### 7.1 Command: `scan`

Scan multiple assets across multiple chains for profitable CEX-DEX price differentials. Returns a ranked list of opportunities filtered by minimum spread and liquidity thresholds.

```bash
cex-dex-arbitrage scan --assets BTC,ETH,SOL --chains ethereum,base,solana --min-spread-bps 50 --min-liquidity-usd 100000
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--assets` | string[] | No | `["BTC","ETH","SOL"]` | Any valid CEX-listed symbol | Uppercase, comma-separated, max 20 items. Each must have an OKX spot instrument. |
| `--chains` | string[] | No | `["ethereum"]` | `ethereum`, `solana`, `base`, `arbitrum`, `bsc`, `polygon`, `xlayer` | Must be supported (see `references/chain-reference.md`). |
| `--min-spread-bps` | number | No | `50` | — | Min: 1, Max: 1000. Only show opportunities above this spread. |
| `--min-liquidity-usd` | number | No | `100000` | — | Min: 1000, Max: 100,000,000. DEX liquidity floor for the token. |
| `--size-usd` | number | No | `1000` | — | Min: 100, Max: 10,000 (hard cap from risk-limits). Trade size for cost estimation. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`, `VIP1`, `VIP2`, `VIP3`, `VIP4`, `VIP5` | OKX fee tier for cost calculation. |
| `--skip-security` | boolean | No | `false` | `true`, `false` | Skip GoPlus checks (NOT recommended). Output will be labeled `[SECURITY UNCHECKED]`. |

#### Return Schema

```yaml
ScanResult:
  timestamp: integer              # Unix ms — when the scan was completed
  mode: string                    # "demo" or "live"
  scan_params:
    assets: string[]              # Assets scanned
    chains: string[]              # Chains scanned
    min_spread_bps: number
    min_liquidity_usd: number
    size_usd: number
    vip_tier: string
  total_pairs_scanned: integer    # Total asset-chain combinations checked
  opportunities_found: integer    # Pairs passing all filters
  opportunities:
    - rank: integer               # 1 = best net profit
      asset: string               # e.g. "ETH"
      chain: string               # e.g. "arbitrum"
      cex_price: string           # OKX last price
      dex_price: string           # DEX price in USD
      gross_spread_bps: number    # Raw spread before costs
      direction: string           # "Buy CEX / Sell DEX" or "Buy DEX / Sell CEX"
      net_profit_usd: number      # After all costs at size_usd
      total_costs_usd: number     # Sum of all cost layers
      profit_to_cost_ratio: number
      security_status: string     # "SAFE", "WARN", "BLOCK", "UNCHECKED"
      security_warnings: string[] # Any GoPlus warnings
      liquidity_usd: number       # DEX pool liquidity
      data_age_ms: integer        # Max data age across venues
      confidence: string          # "high", "medium", "low"
  blocked_assets:                 # Assets that failed security or liquidity checks
    - asset: string
      chain: string
      reason: string              # e.g. "SECURITY_BLOCKED: honeypot detected"
```

#### Return Fields Detail

| Field | Type | Description |
|-------|------|-------------|
| `total_pairs_scanned` | integer | Number of asset-chain combinations evaluated |
| `opportunities_found` | integer | Opportunities with net_profit > 0 and spread > min_spread_bps |
| `opportunities[].rank` | integer | Ranked by net_profit_usd descending |
| `opportunities[].gross_spread_bps` | number | Unsigned spread: `abs(cex - dex) / min(cex, dex) * 10000` |
| `opportunities[].direction` | string | Which venue is cheaper: buy there, sell at the other |
| `opportunities[].net_profit_usd` | number | From profitability-calculator after ALL costs |
| `opportunities[].security_status` | string | Aggregated GoPlus result for the token on that chain |
| `blocked_assets` | array | Assets filtered out due to security or liquidity failures |

---

### 7.2 Command: `evaluate`

Deep analysis of a single CEX-DEX arbitrage opportunity. Performs GoPlus security audit, OnchainOS liquidity check, orderbook depth analysis, DEX swap quote, and full profitability calculation.

```bash
cex-dex-arbitrage evaluate --asset ETH --chain ethereum --size-usd 1000
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | — | Any valid CEX-listed symbol | Uppercase, single asset |
| `--chain` | string | No | `"ethereum"` | `ethereum`, `solana`, `base`, `arbitrum`, `bsc`, `polygon`, `xlayer` | Must be supported chain |
| `--size-usd` | number | No | `1000` | — | Min: 100, Max: 10,000 (hard cap). Trade size for analysis. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0` through `VIP5` | OKX fee tier |
| `--contract-address` | string | No | auto-resolve | — | Explicit DEX token address. If omitted, resolved via `dex-token search`. |
| `--include-withdrawal` | boolean | No | `true` | `true`, `false` | Include OKX withdrawal fee in cost |
| `--order-type` | string | No | `"market"` | `"market"`, `"limit"` | Affects fee rate (taker vs maker) |

#### Return Schema

```yaml
EvaluateResult:
  timestamp: integer
  mode: string
  asset: string
  chain: string
  size_usd: number
  contract_address: string        # Resolved or provided token address
  prices:
    cex:
      last: string
      bid: string
      ask: string
      source: string              # "market_get_ticker"
      data_age_ms: integer
    dex:
      price: string
      source: string              # "dex-market price"
      data_age_ms: integer
  spread:
    gross_spread_bps: number
    direction: string
    buy_venue: string
    sell_venue: string
  security:                       # GoPlus results
    status: string                # "SAFE", "WARN", "BLOCK"
    is_honeypot: boolean
    buy_tax_pct: number
    sell_tax_pct: number
    is_open_source: boolean
    is_proxy: boolean
    top10_holder_pct: number
    warnings: string[]
  liquidity:
    dex_liquidity_usd: number
    price_impact_pct: number      # From dex-swap quote at size_usd
    cex_depth_at_size_usd: number # Available depth within 10 bps of best price
  profitability:                  # From profitability-calculator
    gross_spread_usd: number
    cost_breakdown:
      cex_trading_fee: number
      dex_gas_cost: number
      dex_slippage_cost: number
      cex_slippage_cost: number
      withdrawal_fee: number
    total_costs_usd: number
    net_profit_usd: number
    profit_to_cost_ratio: number
    min_spread_for_breakeven_bps: number
    is_profitable: boolean
    confidence: string
  risk:
    overall_score: integer        # 1-10
    execution_risk: integer
    market_risk: integer
    contract_risk: integer
    liquidity_risk: integer
  recommended_action: string      # "PROCEED", "CAUTION", "DO NOT TRADE"
  next_steps: string[]
```

---

### 7.3 Command: `monitor`

Continuously poll prices across CEX and DEX and alert when the spread exceeds a user-defined threshold.

```bash
cex-dex-arbitrage monitor --assets BTC,ETH --chains arbitrum,base --threshold-bps 30 --duration-min 60 --check-interval-sec 15
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--assets` | string[] | Yes | — | Any valid symbol | Uppercase, comma-separated, max 10 |
| `--chains` | string[] | No | `["ethereum"]` | See chain reference | Must be supported |
| `--threshold-bps` | number | No | `30` | — | Min: 5, Max: 500 |
| `--duration-min` | number | No | `30` | — | Min: 1, Max: 1440 (24h) |
| `--check-interval-sec` | number | No | `15` | — | Min: 5, Max: 300 |
| `--size-usd` | number | No | `1000` | — | Min: 100, Max: 10,000. For cost estimation on alerts. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0` through `VIP5` | OKX fee tier |

#### Return Schema (per alert)

```yaml
MonitorAlert:
  alert_type: "ARB_SPREAD_THRESHOLD_CROSSED"
  timestamp: integer
  asset: string
  chain: string
  current_spread_bps: number
  threshold_bps: number
  direction: string
  cex_price: string
  dex_price: string
  estimated_net_profit: number    # Quick estimate at configured size_usd
  previous_spread_bps: number
  spread_change_bps: number
  checks_completed: integer
  checks_remaining: integer
  suggested_action: string        # e.g. "cex-dex-arbitrage evaluate --asset ETH --chain arbitrum --size-usd 1000"
```

#### Monitor Summary Schema (on completion)

```yaml
MonitorSummary:
  asset: string
  chain: string
  duration_min: number
  total_checks: integer
  alerts_triggered: integer
  spread_stats:
    avg_bps: number
    max_bps: number
    min_bps: number
    std_dev_bps: number
    trend: string                 # "widening", "narrowing", "stable", "volatile"
    sparkline: string             # e.g. "▁▂▃▅▇▅▃▂"
  best_opportunity:               # Peak spread moment
    timestamp: integer
    spread_bps: number
    direction: string
```

---

### 7.4 Command: `backtest`

Historical spread analysis over a lookback period. Uses CEX candle data and DEX kline data to reconstruct historical spread patterns.

```bash
cex-dex-arbitrage backtest --asset SOL --chain solana --lookback-days 30 --granularity 1H --min-spread-bps 20
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | — | Any valid symbol | Uppercase, single asset |
| `--chain` | string | No | `"ethereum"` | See chain reference | Must be supported |
| `--lookback-days` | number | No | `7` | — | Min: 1, Max: 90 (OKX data retention limit) |
| `--granularity` | string | No | `"1H"` | `5m`, `15m`, `1H`, `4H`, `1D` | Must match both CEX and DEX candle support |
| `--min-spread-bps` | number | No | `20` | — | Min: 1, Max: 1000. Count opportunities above this. |
| `--size-usd` | number | No | `1000` | — | For cost estimation in backtest profitability. |

#### Return Schema

```yaml
BacktestResult:
  asset: string
  chain: string
  lookback_days: number
  granularity: string
  data_points: integer            # Number of candle periods analyzed
  spread_stats:
    avg_bps: number
    max_bps: number
    min_bps: number
    std_dev_bps: number
    median_bps: number
  opportunity_count: integer      # Periods where spread > min_spread_bps
  opportunity_pct: number         # % of periods with opportunity
  profitable_count: integer       # Periods where spread > breakeven (after costs)
  profitable_pct: number
  estimated_total_profit: number  # Sum of net profits across all profitable periods
  avg_profit_per_opp: number
  best_period:
    timestamp: integer
    spread_bps: number
    est_profit_usd: number
  worst_period:
    timestamp: integer
    spread_bps: number
  spread_distribution:            # Histogram buckets
    - range_bps: string           # e.g. "0-10"
      count: integer
      pct: number
  trend:
    direction: string             # "widening", "narrowing", "stable"
    sparkline: string
  time_of_day_analysis:           # When do spreads tend to be widest
    - hour_utc: integer
      avg_spread_bps: number
```

---

## 8. Operation Flow

### Step 1: Intent Recognition

Parse user message to extract command, parameters, and context.

| Element | Extraction Logic | Fallback |
|---------|-----------------|----------|
| Command | Map to `scan` / `evaluate` / `monitor` / `backtest` based on keywords | Default: `scan` |
| Assets | Extract token symbols (BTC, ETH, SOL, ARB, OP...) | Default: `["BTC","ETH","SOL"]` |
| Chains | Look for chain names: "Ethereum", "Base", "Arbitrum", "Solana", "以太坊", "Arb" | Default: `["ethereum"]` |
| Size | Look for USD amounts: "$1000", "1000 USDT", "一千" | Default: `1000` |
| Min spread | Look for bps values: "30 bps", "50 基點" | Default: `50` (scan), `0` (evaluate) |
| VIP tier | Look for "VIP1", "VIP2" etc. | Default: `"VIP0"` |

**Keyword-to-command mapping:**

| Keywords | Command |
|----------|---------|
| "掃描", "scan", "搵", "找", "搬磚", "arb scan", "套利機會", "有冇得搬", "掃一掃" | `scan` |
| "評估", "evaluate", "analyze", "分析", "詳細", "detailed", "呢個機會", "值唔值得" | `evaluate` |
| "監控", "monitor", "watch", "alert", "通知", "持續", "提醒", "keep an eye" | `monitor` |
| "回測", "backtest", "歷史", "history", "過去", "上個月", "previously", "how often" | `backtest` |

**Ambiguous intent resolution:**

| Input Pattern | Resolved Command | Reasoning |
|---------------|-----------------|-----------|
| "搬磚" (generic) | `scan` | Broad arb intent → scan with defaults |
| "ETH 有冇得搬" (single asset, no chain) | `evaluate --asset ETH` | Single asset mentioned → evaluate |
| "ETH 喺 Base 上面同 OKX 有冇價差" | `evaluate --asset ETH --chain base` | Specific asset + chain → evaluate |
| "幫我睇住 BTC 價差" | `monitor --assets BTC` | "睇住" = monitoring intent |
| "上個月 SOL 套利機會多唔多" | `backtest --asset SOL --lookback-days 30` | "上個月" = historical intent |

---

### Step 2: Pre-Execution Safety Checks

For each candidate token (after extracting the list from Step 1), run the following checks **before any price analysis**:

#### 2a. Token Address Resolution

```
For each (asset, chain) pair:

  IF asset is native token on chain (ETH on ethereum/arbitrum/base, SOL on solana, BNB on bsc):
    → Use native address from references/chain-reference.md
    → SKIP GoPlus security check (native tokens are safe)

  ELSE:
    → onchainos dex-token search {ASSET} --chains {chain}
    → Pick result matching target chain with highest liquidity
    → If no results → BLOCK: TOKEN_NOT_FOUND for this asset/chain
    → Store: token_address_cache[{ASSET}:{CHAIN}] = address
```

#### 2b. GoPlus Security Check (for non-native tokens)

```
For each non-native token:

  1. GoPlus check_token_security(chain_id, contract_address)

  Decision logic (from references/goplus-tools.md):

  BLOCK conditions (any single trigger → remove from scan):
    - is_honeypot === "1"           → BLOCK("Honeypot detected — cannot sell")
    - buy_tax > 0.05 (5%)          → BLOCK("Buy tax {tax}% exceeds 5% limit")
    - sell_tax > 0.10 (10%)        → BLOCK("Sell tax {tax}% exceeds 10% limit")
    - is_open_source === "0"       → BLOCK("Contract source not verified")
    - can_take_back_ownership === "1" → BLOCK("Ownership reclaimable")
    - owner_change_balance === "1"  → BLOCK("Owner can modify balances")
    - slippage_modifiable === "1"  → BLOCK("Tax modifiable by owner")
    - cannot_sell_all === "1"      → BLOCK("Cannot sell entire balance")
    - top 10 non-contract holders > 80% → BLOCK("Concentrated ownership > 80%")

  WARN conditions (flag but continue):
    - is_proxy === "1"             → WARN("Upgradeable proxy contract")
    - buy_tax > 0.01 (1%)         → WARN("Buy tax > 1%")
    - sell_tax > 0.02 (2%)        → WARN("Sell tax > 2%")
    - is_mintable === "1"          → WARN("Token is mintable")
    - transfer_pausable === "1"   → WARN("Transfers can be paused")
    - top 10 non-contract holders > 50% → WARN("Concentrated ownership > 50%")
```

#### 2c. Liquidity Validation

```
For each token that passed security:

  onchainos dex-token price-info {contract_address} --chain {chain}
  → Extract: liquidityUsd

  IF liquidityUsd < min_liquidity_usd:
    → BLOCK: INSUFFICIENT_LIQUIDITY
    → Message: "DEX 流動性 ${actual} 低於最低要求 ${required}"
```

#### 2d. Data Staleness Check

```
For all data sources:

  data_age_ms = Date.now() - source_timestamp

  Arbitrage mode thresholds (stricter than analysis mode):
    WARN if data_age_ms > 3000 (3 seconds)
    BLOCK if data_age_ms > 5000 (5 seconds)

  On BLOCK:
    → Refetch data once
    → If still stale after refetch → output DATA_STALE error
```

---

### Step 3: Data Collection & Computation

This is the core processing step. For each asset/chain pair that passed Step 2:

#### 3a. Price Snapshot

```
1. price-feed-aggregator.snapshot(
     assets=[asset],
     venues=["okx-cex","okx-dex"],
     chains=[chain],
     mode="arb"
   )
   → Returns PriceSnapshot with synchronized CEX and DEX prices

   Internally this calls:
     CEX: market_get_ticker({ instId: "{ASSET}-USDT" })
       → Extract: last, bidPx, askPx, ts
     DEX: onchainos dex-market price --chain {chain} --token {address}
       → Extract: priceUsd
```

#### 3b. Spread Calculation

```
2. For each asset where PriceSnapshot is available:

   spread_bps = abs(cex_price - dex_price) / min(cex_price, dex_price) * 10000

   Formula reference: references/formulas.md Section 1

   IF spread_bps < min_spread_bps:
     → Skip this asset (below threshold)

   IF spread_bps > 5000 (50%):
     → BLOCK: PRICE_ANOMALY
     → Likely: wrong token address, stale DEX pool, or low liquidity

   Determine direction:
     IF cex_price < dex_price:
       direction = "Buy CEX / Sell DEX"
       buy_venue = "okx-cex"
       sell_venue = "okx-dex:{chain}"
     ELSE:
       direction = "Buy DEX / Sell CEX"
       buy_venue = "okx-dex:{chain}"
       sell_venue = "okx-cex"
```

#### 3c. Depth & Slippage Analysis

```
3. For each asset with spread > min_spread_bps:

   a. CEX orderbook depth:
      market_get_orderbook(instId="{ASSET}-USDT", sz="400")
      → Walk bids (for sell) or asks (for buy) to compute:
        - Available depth at trade size
        - Volume-weighted average price (VWAP)
        - price_impact_bps = (vwap - best_price) / best_price * 10000

   b. DEX executable price + price impact:
      onchainos dex-swap quote \
        --from {native_address} \
        --to {token_address} \
        --amount {amount_in_minimal_units} \
        --chain {chain}
      → Extract: toTokenAmount, priceImpactPercent, estimatedGas

      Note: amount conversion:
        amount_wei = size_usd / native_price * 10^native_decimals
        (e.g., $1000 / $3412.50 * 10^18 for ETH)
```

#### 3d. Profitability Calculation

```
4. Construct TradeLeg[] and delegate to profitability-calculator:

   IF direction == "Buy CEX / Sell DEX":
     legs = [
       {
         venue: "okx-cex",
         asset: "{ASSET}",
         side: "buy",
         size_usd: size_usd,
         order_type: "market",
         vip_tier: vip_tier,
         withdrawal_required: true
       },
       {
         venue: "okx-dex",
         chain: "{chain}",
         asset: "{ASSET}",
         side: "sell",
         size_usd: size_usd,
         order_type: "market"
       }
     ]

   IF direction == "Buy DEX / Sell CEX":
     legs = [
       {
         venue: "okx-dex",
         chain: "{chain}",
         asset: "{ASSET}",
         side: "buy",
         size_usd: size_usd,
         order_type: "market"
       },
       {
         venue: "okx-cex",
         asset: "{ASSET}",
         side: "sell",
         size_usd: size_usd,
         order_type: "market",
         vip_tier: vip_tier,
         deposit_required: true
       }
     ]

   profitability-calculator.estimate(legs)
   → Returns ProfitabilityResult with:
     - net_profit_usd
     - total_costs_usd
     - cost_breakdown (cex_trading_fee, gas_cost, slippage_cost, withdrawal_fee, etc.)
     - profit_to_cost_ratio
     - min_spread_for_breakeven_bps
     - is_profitable
     - confidence
```

#### 3e. Ranking & Filtering

```
5. Rank all opportunities by net_profit_usd descending

6. Filter: only include opportunities where:
   - net_profit_usd > 0
   - security_status != "BLOCK"
   - min_net_profit threshold met (default $5 from risk-limits)

7. Attach risk scores:
   - execution_risk: based on slippage, data age, volume
   - market_risk: based on asset volatility (24h range)
   - contract_risk: from GoPlus results
   - liquidity_risk: based on DEX liquidity vs trade size
   - overall_risk: weighted average
```

---

### Step 4: Output & Recommend

Format using templates from `references/output-templates.md`. The output structure varies by command:

#### For `scan`:
1. Global header (skill name, mode, timestamp, data sources)
2. Scan summary (parameters, total scanned, opportunities found)
3. Opportunity table (ranked, with key metrics per row)
4. Blocked assets section (if any failed security/liquidity)
5. Risk gauge for top opportunity
6. Next steps suggestions
7. Disclaimer

#### For `evaluate`:
1. Global header
2. Price comparison (CEX vs DEX with bid/ask)
3. Spread analysis (gross, direction)
4. Security check results (GoPlus findings)
5. Cost breakdown table (from profitability-calculator)
6. Net profit summary
7. Risk gauge (4-dimension breakdown)
8. Recommended action
9. Next steps
10. Disclaimer

#### For `monitor`:
1. Monitor start confirmation (parameters, schedule)
2. Per-alert output when threshold crossed
3. Monitor completion summary with statistics

#### For `backtest`:
1. Global header
2. Summary statistics (avg/max/min/stddev spread)
3. Opportunity frequency analysis
4. Spread distribution histogram
5. Time-of-day analysis
6. Trend assessment
7. Sparkline visualization
8. Disclaimer

**Suggested follow-up actions (vary by result):**

| Result | Suggested Actions |
|--------|-------------------|
| Scan: 0 opportunities | "暫無套利機會。設定監控等待價差擴大 → `cex-dex-arbitrage monitor`" |
| Scan: opportunities found | "詳細評估最佳機會 → `cex-dex-arbitrage evaluate --asset {TOP} --chain {CHAIN}`" |
| Evaluate: profitable | "如滿意，手動在 2-3 分鐘內執行兩腿交易"; "檢查不同規模 → `profitability-calculator sensitivity`" |
| Evaluate: not profitable | "減小規模或切換 L2 降低 Gas"; "設定監控等待更大價差 → `cex-dex-arbitrage monitor`" |
| Monitor: alert triggered | "立即評估 → `cex-dex-arbitrage evaluate --asset {ASSET} --chain {CHAIN}`" |
| Backtest: frequent opportunities | "設定實時監控 → `cex-dex-arbitrage monitor`" |

---

## 9. Safety Checks (Per Operation)

All 11 checks from `references/safety-checks.md` applied, with cex-dex-arbitrage-specific thresholds.

| # | Check | Tool | BLOCK Threshold | WARN Threshold | Error Code |
|---|-------|------|----------------|----------------|------------|
| 1 | MCP connectivity | `system_get_capabilities` | Server not reachable | — | `MCP_NOT_CONNECTED` |
| 2 | Authentication | `system_get_capabilities` | `authenticated: false` | — | `AUTH_FAILED` |
| 3 | Data freshness | Internal timestamp comparison | > 5s stale (arb-specific) | > 3s stale | `DATA_STALE` |
| 4 | Token honeypot | GoPlus `check_token_security` | `is_honeypot === "1"` | — | `SECURITY_BLOCKED` |
| 5 | Token tax rate | GoPlus `check_token_security` | buy > 5% OR sell > 10% | buy > 1% | `SECURITY_BLOCKED` |
| 6 | Holder concentration | GoPlus `check_token_security` | Top 10 non-contract > 80% | Top 10 > 50% | `SECURITY_BLOCKED` |
| 7 | Contract verified | GoPlus `check_token_security` | `is_open_source === "0"` | — | `SECURITY_BLOCKED` |
| 8 | Liquidity depth | OnchainOS `dex-token price-info` | `liquidityUsd < $100,000` | `liquidityUsd < $500,000` | `INSUFFICIENT_LIQUIDITY` |
| 9 | Price impact | OnchainOS `dex-swap quote` | `priceImpactPercent > 1%` (stricter than default 2%) | `priceImpactPercent > 0.5%` | `PRICE_IMPACT_HIGH` |
| 10 | Gas vs profit ratio | `onchain-gateway gas` + calculation | Gas cost > 30% of gross spread | Gas cost > 10% of spread | `GAS_TOO_HIGH` |
| 11 | Net profitability | `profitability-calculator` | `net_profit <= 0` | `profit_to_cost_ratio < 2` | `NOT_PROFITABLE` |

### Check Execution Flow

```
START
  │
  ├─ Check 1-2: Infrastructure   ── BLOCK? → Output error, STOP
  │
  ├─ Check 3: Freshness          ── BLOCK? → Refetch data, retry once → BLOCK? → STOP
  │
  ├─ Check 4-7: Token Security   ── BLOCK? → Move asset to blocked_assets list, CONTINUE to next asset
  │                                  WARN?  → Attach warning labels, CONTINUE
  │
  ├─ Check 8-9: Liquidity        ── BLOCK? → Move to blocked_assets, CONTINUE to next asset
  │                                  WARN?  → Attach liquidity note, CONTINUE
  │
  ├─ Check 10-11: Profitability  ── BLOCK? → Exclude from results, CONTINUE to next asset
  │                                  WARN?  → Include with margin warning
  │
  └─ ALL PASSED → Include in ranked results with accumulated WARNs
```

---

## 10. Error Codes & Recovery

| Code | Condition | User Message (ZH) | User Message (EN) | Recovery |
|------|-----------|-------------------|-------------------|----------|
| `MCP_NOT_CONNECTED` | okx-trade-mcp server unreachable | MCP 伺服器無法連線。請確認 okx-trade-mcp 是否正在運行。 | MCP server unreachable. Check if okx-trade-mcp is running. | Verify `~/.okx/config.toml`, restart server |
| `AUTH_FAILED` | API key invalid or expired | API 認證失敗。請檢查 OKX API 金鑰設定。 | API authentication failed. Check OKX API key config. | Update config.toml |
| `DATA_STALE` | Price data > 5s old (after retry) | 市場數據已過期（{venue} 延遲 {age}ms，套利模式上限 5000ms）。 | Market data stale ({venue}: {age}ms, arb mode max 5000ms). | Auto-retry once, then fail |
| `SECURITY_BLOCKED` | GoPlus check failed | 安全檢查未通過：{reason}。此代幣存在風險，已從結果中排除。 | Security check failed: {reason}. Token excluded. | Show GoPlus findings, do not recommend |
| `SECURITY_UNAVAILABLE` | GoPlus API unreachable | 安全檢查服務暫時無法使用。所有結果標記為 [SECURITY UNCHECKED]。 | Security service unavailable. Results marked [SECURITY UNCHECKED]. | Retry once, then continue with warning labels |
| `INSUFFICIENT_LIQUIDITY` | DEX liquidity < $100,000 | {asset} 在 {chain} 的 DEX 流動性不足（${actual}，最低要求 ${required}）。 | {asset} DEX liquidity on {chain} insufficient (${actual}, min ${required}). | Suggest different chain or larger-liquidity token |
| `PRICE_IMPACT_HIGH` | Swap price impact > 1% | 預估價格影響 {impact}% 過高（套利模式上限 1%）。建議減小規模。 | Price impact {impact}% exceeds 1% arb limit. Reduce size. | Reduce trade size, try different chain |
| `PRICE_ANOMALY` | CEX vs DEX spread > 50% | {asset} 的 CEX 與 DEX 價格差異異常大（{spread}%），可能是地址解析錯誤。 | Price anomaly: {asset} CEX-DEX spread is {spread}%. Likely address error. | Verify token address, check DEX pool |
| `TOKEN_NOT_FOUND` | dex-token search returns 0 results | 在 {chain} 上找不到 {asset} 的代幣合約。請確認名稱及鏈名。 | Token {asset} not found on {chain}. Verify symbol and chain. | Try alternative chains, check symbol |
| `INSTRUMENT_NOT_FOUND` | OKX spot instrument missing | OKX 上找不到 {instId} 交易對。 | Instrument {instId} not found on OKX. | Suggest similar instruments |
| `NOT_PROFITABLE` | Net profit <= 0 after all costs | 扣除所有成本後淨利潤為負（{net_pnl}），不建議執行。 | Net profit negative after costs ({net_pnl}). Not recommended. | Show cost breakdown, suggest L2 or larger size |
| `MARGIN_TOO_THIN` | Profit-to-cost < 2.0 | 利潤空間偏薄（利潤/成本比 = {ratio}x），執行風險較高。 | Thin margin (profit/cost = {ratio}x). Higher execution risk. | Proceed with caution, show sensitivity analysis |
| `GAS_TOO_HIGH` | Gas > 30% of gross spread | Gas 費用佔價差 {pct}%，超過 30% 上限。考慮使用 L2 鏈。 | Gas is {pct}% of spread, exceeding 30% limit. Consider L2. | Wait for lower gas, switch to Arbitrum/Base |
| `RATE_LIMITED` | API rate limit hit after 3 retries | API 請求頻率超限，{wait} 秒後重試。 | API rate limit reached. Retrying in {wait}s. | Exponential backoff: 1s, 2s, 4s |
| `TRADE_SIZE_EXCEEDED` | Size > $10,000 hard cap | 交易金額 ${amount} 超過套利上限 $10,000。 | Trade size ${amount} exceeds arb limit $10,000. | Cap at $10,000 and inform user |

---

## 11. Cross-Skill Integration Contracts

### Input: What This Skill Consumes

| Source Skill / Tool | Data Consumed | Schema | Usage |
|---------------------|--------------|--------|-------|
| `price-feed-aggregator.snapshot` | Synchronized CEX + DEX prices | `PriceSnapshot[]` | Spread calculation in Step 3a |
| GoPlus `check_token_security` | Token security audit results | GoPlus response object | Safety gate in Step 2b |
| OnchainOS `dex-token search` | Token contract address resolution | `{address, symbol, chain, decimals}` | Address resolution in Step 2a |
| OnchainOS `dex-token price-info` | DEX liquidity depth | `{liquidityUsd, priceUsd, ...}` | Liquidity validation in Step 2c |
| OnchainOS `dex-swap quote` | Executable DEX price + impact | `{toTokenAmount, priceImpactPercent, estimatedGas}` | Price impact check in Step 3c |
| `profitability-calculator.estimate` | Net P&L after all costs | `ProfitabilityResult` | Go/no-go decision in Step 3d |

### Output: What This Skill Produces

| Output | Consumer | Schema | Handoff |
|--------|----------|--------|---------|
| Ranked opportunity list | User (formatted output) | `ScanResult` | Displayed directly |
| Single evaluation | User (formatted output) | `EvaluateResult` | Displayed directly |
| Monitor alerts | User (formatted output) | `MonitorAlert` | Displayed in real-time |
| Backtest statistics | User (formatted output) | `BacktestResult` | Displayed directly |

### Data Flow Diagram (Full Pipeline — Chain A from AGENTS.md)

```
price-feed-aggregator.snapshot(assets, venues=["okx-cex","okx-dex"], chains, mode="arb")
  │
  │  PriceSnapshot[]
  ▼
cex-dex-arbitrage.scan / evaluate
  │
  ├─ GoPlus check_token_security(chain_id, contract_address)
  │    → BLOCK / WARN / SAFE
  │
  ├─ OnchainOS dex-token price-info → liquidityUsd check
  │    → BLOCK if < min_liquidity_usd
  │
  ├─ OnchainOS dex-swap quote → priceImpactPercent check
  │    → BLOCK if > 1%
  │
  ├─ market_get_orderbook → CEX depth analysis
  │
  │  Constructs TradeLeg[]
  ▼
profitability-calculator.estimate(legs)
  │
  │  ProfitabilityResult
  ▼
cex-dex-arbitrage applies go/no-go:
  │  - is_profitable == true?
  │  - profit_to_cost_ratio >= 1.5?
  │  - security_status != BLOCK?
  ▼
Output: Ranked opportunities with safety checks + cost breakdowns
```

---

## 12. Output Format

### 12.1 Scan Results — Complete Example

```
══════════════════════════════════════════
  CEX-DEX Arbitrage Scanner — SCAN
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:30 UTC
  Data sources: OKX CEX + OKX DEX (Arbitrum, Base)
══════════════════════════════════════════

── 掃描參數 ──────────────────────────────

  資產:         BTC, ETH, SOL
  鏈:           arbitrum, base
  最低價差:     50 bps
  最低流動性:   $100,000
  分析規模:     $1,000.00
  VIP 等級:     VIP0

── Opportunities Found: 2 ────────────────

  #1  ETH-USDT (Arbitrum)
  ├─ Spread:      68 bps (0.68%)
  ├─ Direction:    Buy CEX / Sell DEX
  ├─ CEX Price:   $3,412.50
  ├─ DEX Price:   $3,435.70
  ├─ Est. Profit: +$2.94  (after costs)
  ├─ Costs:       $3.86
  ├─ Profit/Cost: 0.76x
  ├─ Liquidity:   $45.2M
  ├─ Confidence:  HIGH
  └─ Risk:        [SAFE]

  #2  SOL-USDT (Base)
  ├─ Spread:      52 bps (0.52%)
  ├─ Direction:    Buy DEX / Sell CEX
  ├─ CEX Price:   $145.50
  ├─ DEX Price:   $144.74
  ├─ Est. Profit: +$1.40  (after costs)
  ├─ Costs:       $3.80
  ├─ Profit/Cost: 0.37x
  ├─ Liquidity:   $12.8M
  ├─ Confidence:  HIGH
  └─ Risk:        [WARN] 利潤空間偏薄

──────────────────────────────────────────
  Scanned: 6 pairs | Found: 2 opportunities

── Blocked Assets ────────────────────────

  (none)

── Risk Assessment (Top Opportunity) ─────

  Overall Risk:  ▓▓▓░░░░░░░  3/10
                 MODERATE-LOW

  Breakdown:
  ├─ Execution Risk:   ▓▓░░░░░░░░  2/10
  │                    Stable spread; deep orderbook
  ├─ Market Risk:      ▓▓▓░░░░░░░  3/10
  │                    ETH 24h range 2.4%; moderate vol
  ├─ Smart Contract:   ▓▓░░░░░░░░  2/10
  │                    Native ETH — no contract risk
  └─ Liquidity Risk:   ▓▓░░░░░░░░  2/10
                       $45.2M DEX liquidity; deep orderbook

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 詳細評估最佳機會:
     cex-dex-arbitrage evaluate --asset ETH --chain arbitrum --size-usd 1000
  2. 增大規模查看盈利變化:
     profitability-calculator sensitivity --variable size_usd --range '[500,5000]'
  3. 設定持續監控:
     cex-dex-arbitrage monitor --assets ETH,SOL --chains arbitrum,base --threshold-bps 50

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Past spreads do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

---

### 12.2 Evaluate Results — Complete Example

```
══════════════════════════════════════════
  CEX-DEX Arbitrage Scanner — EVALUATE
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:35 UTC
  Data sources: OKX CEX + OKX DEX (Arbitrum) + GoPlus
══════════════════════════════════════════

── ETH CEX-DEX 套利評估 ─────────────────

  Trade Size:       $5,000.00
  VIP Tier:         VIP0
  Contract:         0xeeee...eeee (Native ETH)

── 價格比較 ──────────────────────────────

  OKX CEX
  ├─ Last:     $3,412.50
  ├─ Bid:      $3,412.10
  ├─ Ask:      $3,412.90
  └─ Data Age: 2s [SAFE]

  OKX DEX (Arbitrum)
  ├─ Price:    $3,435.70
  ├─ Impact:   0.08% (at $5K swap quote)
  └─ Data Age: 3s [SAFE]

── 價差分析 ──────────────────────────────

  Gross Spread:     68 bps (0.68%)
  Direction:        Buy CEX / Sell DEX
  Buy at:           OKX CEX @ $3,412.90 (ask)
  Sell at:          OKX DEX @ $3,435.70

── Safety Checks ───────────────────────────

  [  SAFE  ] Token Security
              Native ETH — no contract risk

  [  SAFE  ] Liquidity Depth
              DEX: $45.2M | CEX: $2.1M within 10 bps

  [  SAFE  ] Price Impact
              DEX impact 0.08% at $5K (limit: 1%)

  [  SAFE  ] Price Freshness
              All data < 3 seconds old

  [  SAFE  ] Gas Conditions
              Arbitrum: 0.2 gwei — optimal

  ──────────────────────────────────────
  Overall: SAFE

── Cost Breakdown ──────────────────────────

  Leg 1 — OKX CEX (Buy)
  ├─ Trading Fee:    $4.00      (8.0 bps, VIP0 taker)
  ├─ Slippage Est:   $0.75      (1.5 bps, from orderbook)
  └─ Subtotal:       $4.75

  Leg 2 — OKX DEX Arbitrum (Sell)
  ├─ Gas Cost:       $0.30      (0.2 gwei, Arbitrum)
  ├─ Slippage Est:   $0.40      (0.8 bps, from swap quote)
  └─ Subtotal:       $0.70

  Transfer Costs
  ├─ Withdrawal Fee: $0.34      (0.0001 ETH to Arbitrum)
  └─ Bridge Fee:     $0.00      (direct withdrawal)

  ──────────────────────────────────────
  Gross Spread:      +$34.00    (68 bps)
  Total Costs:       -$5.79     (11.6 bps)
  ══════════════════════════════════════
  Net Profit:        +$28.21    (56.4 bps)
  Profit/Cost:       4.87x
  Min Breakeven:     12 bps
  Confidence:        HIGH
  ══════════════════════════════════════

  Result: [PROFITABLE]

── Risk Assessment ─────────────────────────

  Overall Risk:  ▓▓▓░░░░░░░  3/10
                 MODERATE-LOW

  Breakdown:
  ├─ Execution Risk:   ▓▓░░░░░░░░  2/10
  │                    Both venues liquid; spread stable
  ├─ Market Risk:      ▓▓▓░░░░░░░  3/10
  │                    ETH vol moderate; 2% move unlikely in 3-min window
  ├─ Smart Contract:   ▓░░░░░░░░░  1/10
  │                    Native ETH — zero contract risk
  └─ Liquidity Risk:   ▓▓░░░░░░░░  2/10
                       Deep orderbook + $45.2M DEX liquidity

── Recommended Action ──────────────────────

  [PROCEED] — 利潤空間充裕 (4.87x)，風險偏低

  執行建議:
  1. 在 OKX CEX 以市價買入 ~1.465 ETH ($5,000)
  2. 提幣至 Arbitrum 錢包 (預計 1-5 分鐘)
  3. 在 DEX 賣出 ETH 為 USDT
  4. 需在 2-3 分鐘內完成兩腿以鎖定價差

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 查看不同規模的盈利敏感度:
     profitability-calculator sensitivity --variable size_usd --range '[1000,10000]'
  2. 查看最低價差要求:
     profitability-calculator min-spread --venue-a okx-cex --venue-b okx-dex --chain arbitrum --asset ETH --size-usd 5000
  3. 設定持續監控:
     cex-dex-arbitrage monitor --assets ETH --chains arbitrum --threshold-bps 50

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Past spreads do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

---

## 13. Conversation Examples

### Example 1: "幫我掃描一下有冇套利機會" (Cantonese — scan with defaults)

**User:**
> 幫我掃描一下有冇套利機會

**Intent Recognition:**
- Command: `scan` (keyword: "掃描", "套利")
- Assets: not specified → default `["BTC","ETH","SOL"]`
- Chains: not specified → default `["ethereum"]`
- Min spread: not specified → default `50 bps`

**Tool Calls (in order):**

```
1. system_get_capabilities
   → { authenticated: true, mode: "demo" }

2. For BTC:
   a. market_get_ticker({ instId: "BTC-USDT" })
      → { last: "87234.5", bidPx: "87234.1", askPx: "87234.9", ts: "..." }
   b. onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
      → { priceUsd: "87210.2" }  (using ETH price for the chain, then resolve WBTC)
   c. onchainos dex-token search "WBTC" --chains ethereum
      → { address: "0x2260fac5e5542a773aa44fbcfedf7c193bc2c599", ... }
   d. onchainos dex-market price --chain ethereum --token 0x2260fac5e5542a773aa44fbcfedf7c193bc2c599
      → { priceUsd: "87180.0" }
   e. Spread = abs(87234.5 - 87180.0) / 87180.0 * 10000 = 6.3 bps → BELOW 50 bps threshold, skip

3. For ETH:
   a. market_get_ticker({ instId: "ETH-USDT" })
      → { last: "3412.50", ... }
   b. onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
      → { priceUsd: "3410.80" }
   c. Spread = 5.0 bps → BELOW threshold, skip

4. For SOL:
   a. market_get_ticker({ instId: "SOL-USDT" })
      → { last: "145.50", ... }
   b. onchainos dex-token search "SOL" --chains ethereum
      → Resolve wSOL or wrapped SOL on Ethereum
   c. Low liquidity on Ethereum for wrapped SOL → skip
```

**Formatted Output:**

```
══════════════════════════════════════════
  CEX-DEX Arbitrage Scanner — SCAN
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════

── 掃描結果 ──────────────────────────────

  掃描: 3 資產 x 1 鏈 = 3 組合
  最低價差: 50 bps | 最低流動性: $100,000

  未發現符合條件的套利機會。
  BTC: 6 bps | ETH: 5 bps | SOL: 流動性不足

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 降低最低價差閾值重新掃描:
     cex-dex-arbitrage scan --min-spread-bps 20
  2. 嘗試 L2 鏈（通常價差更大）:
     cex-dex-arbitrage scan --chains arbitrum,base,solana
  3. 設定監控等待價差擴大:
     cex-dex-arbitrage monitor --assets BTC,ETH,SOL --chains ethereum --threshold-bps 30

  ── Disclaimer ────────────────────────────
  以上僅為分析建議，不會自動執行任何交易。
══════════════════════════════════════════
```

---

### Example 2: "ETH 喺 Base 鏈上面同 OKX 有冇價差" (Cantonese — evaluate specific)

**User:**
> ETH 喺 Base 鏈上面同 OKX 有冇價差

**Intent Recognition:**
- Command: `evaluate` (specific asset + chain)
- Asset: ETH
- Chain: Base
- Size: not specified → default $1,000

**Tool Calls (in order):**

```
1. system_get_capabilities
   → { authenticated: true, mode: "demo" }

2. market_get_ticker({ instId: "ETH-USDT" })
   → { last: "3412.50", bidPx: "3412.10", askPx: "3412.90", ts: "..." }

3. onchainos dex-market price --chain base --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
   → { priceUsd: "3419.80" }

4. Spread = abs(3412.50 - 3419.80) / 3412.50 * 10000 = 21.4 bps

5. (ETH is native — skip GoPlus security check)

6. onchainos dex-token price-info 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee --chain base
   → { liquidityUsd: "28500000", ... }

7. onchainos dex-swap quote \
     --from 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee \
     --to 0x833589fcd6edb6e08f4c7c32d4f71b54bda02913 \
     --amount 293000000000000000 \
     --chain base
   → { toTokenAmount: "1002340000", priceImpactPercent: "0.01", estimatedGas: "200000" }

8. market_get_orderbook({ instId: "ETH-USDT", sz: "400" })
   → Walk asks to estimate slippage at $1,000 buy → ~1 bps

9. onchainos onchain-gateway gas --chain base
   → { gasPriceGwei: "0.008" }

10. profitability-calculator.estimate([
      { venue: "okx-cex", asset: "ETH", side: "buy", size_usd: 1000, vip_tier: "VIP0", withdrawal_required: true },
      { venue: "okx-dex", chain: "base", asset: "ETH", side: "sell", size_usd: 1000 }
    ])
   → {
       net_profit_usd: 0.54,
       total_costs_usd: 1.60,
       cost_breakdown: { cex_trading_fee: 0.80, gas_cost: 0.005, slippage: 0.10, withdrawal_fee: 0.14 },
       profit_to_cost_ratio: 0.34,
       is_profitable: true,
       confidence: "high"
     }
```

**Formatted Output:**

```
══════════════════════════════════════════
  CEX-DEX Arbitrage Scanner — EVALUATE
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════

── ETH CEX-DEX 價差分析 (Base) ──────────

  OKX CEX:     $3,412.50  (Bid: $3,412.10 / Ask: $3,412.90)
  OKX DEX:     $3,419.80  (Base)
  Gross Spread: 21 bps (0.21%)
  Direction:    Buy CEX / Sell DEX

── Cost Breakdown ($1,000) ─────────────

  | 項目               | 金額   | bps |
  |--------------------|--------|-----|
  | CEX Taker 費       | $0.80  | 8.0 |
  | CEX 滑點 (1 bps)   | $0.10  | 1.0 |
  | DEX Gas (Base)     | $0.01  | 0.1 |
  | 提幣費 (ETH Base)  | $0.14  | 1.4 |
  | **總成本**         | **$1.05** | **10.5** |
  | **毛利差**         | +$2.14 | 21.4 |
  | **淨利潤**         | **+$1.09** | **10.9** |

  Profit/Cost: 1.04x | Breakeven: 11 bps
  Result: [PROFITABLE] (利潤空間偏薄)

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 增大規模攤薄固定成本:
     cex-dex-arbitrage evaluate --asset ETH --chain base --size-usd 5000
  2. 比較不同鏈:
     cex-dex-arbitrage evaluate --asset ETH --chain arbitrum --size-usd 1000
  3. 設定監控等待更大價差:
     cex-dex-arbitrage monitor --assets ETH --chains base --threshold-bps 30

  以上僅為分析建議，不會自動執行任何交易。
══════════════════════════════════════════
```

---

### Example 3: "持續監控 BTC 價差，超過 30bps 提醒我" (Traditional Chinese — monitor)

**User:**
> 持續監控 BTC 價差，超過 30bps 提醒我

**Intent Recognition:**
- Command: `monitor` (keyword: "持續監控", "提醒")
- Assets: BTC
- Threshold: 30 bps
- Duration: not specified → default 30 min
- Chains: not specified → default ethereum

**Tool Calls (initial setup):**

```
1. system_get_capabilities → { authenticated: true, mode: "demo" }
2. Resolve BTC address on ethereum: onchainos dex-token search "WBTC" --chains ethereum
   → 0x2260fac5e5542a773aa44fbcfedf7c193bc2c599
3. Verify liquidity: onchainos dex-token price-info ... → { liquidityUsd: "320000000" }
```

**Monitor Start Output:**

```
══════════════════════════════════════════
  CEX-DEX Arbitrage Scanner — MONITOR
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════

── 監控設定 ──────────────────────────────

  資產:       BTC (WBTC on Ethereum)
  場所:       OKX CEX vs OKX DEX (Ethereum)
  閾值:       30 bps
  持續時間:   30 分鐘 (至 15:05 UTC)
  檢查間隔:   每 15 秒
  預計檢查:   120 次

  當前價差:   6 bps — 低於閾值，持續監控中...

══════════════════════════════════════════
```

**Alert Output (when threshold crossed):**

```
══════════════════════════════════════════
  ALERT: ARB SPREAD OPPORTUNITY
  2026-03-09 14:52 UTC
══════════════════════════════════════════

  BTC CEX-DEX 價差超過閾值

  ├─ 當前價差:     38 bps (0.38%)
  ├─ 閾值:         30 bps
  ├─ 上次價差:     22 bps (+16 bps in 15s)
  ├─ OKX CEX:     $87,234.50
  ├─ OKX DEX:     $87,565.80 (Ethereum)
  ├─ 方向:         Buy CEX / Sell DEX
  ├─ 預估淨利:    +$1.80 (at $1,000)
  └─ 已檢查:       68/120 次

  ── 建議操作 ────────────────────────────
  cex-dex-arbitrage evaluate --asset BTC --chain ethereum --size-usd 2000

══════════════════════════════════════════
```

---

### Example 4: "上個月 SOL 套利機會多唔多" (Cantonese — backtest)

**User:**
> 上個月 SOL 套利機會多唔多

**Intent Recognition:**
- Command: `backtest` (keyword: "上個月", historical intent)
- Asset: SOL
- Lookback: 30 days
- Chain: solana (inferred from SOL)

**Tool Calls:**

```
1. system_get_capabilities → OK
2. market_get_candles({ instId: "SOL-USDT", bar: "1H", limit: "100" })
   → Paginate to get 720 hourly candles (30 days)
3. onchainos dex-market kline 11111111111111111111111111111111 --chain solana --bar 1H
   → DEX hourly candles for same period
4. For each candle pair: compute spread_bps
5. profitability-calculator.min-spread(venue-a=okx-cex, venue-b=okx-dex, chain=solana, asset=SOL, size-usd=1000)
   → min_breakeven_bps = 10 bps
```

**Formatted Output:**

```
══════════════════════════════════════════
  CEX-DEX Arbitrage Scanner — BACKTEST
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════

── SOL CEX-DEX 回測分析 (Solana) ────────

  回測期間:     2026-02-07 至 2026-03-09 (30 天)
  粒度:         1 小時
  數據點:       720 個

── 價差統計 ──────────────────────────────

  平均價差:     12.4 bps
  最大價差:     142 bps (2026-02-28 03:00 UTC)
  最小價差:     0 bps
  標準差:       18.7 bps
  中位數:       8 bps

── 機會分析 ──────────────────────────────

  價差 > 20 bps 次數:     148 次 (20.6% 時間)
  價差 > 50 bps 次數:      38 次 (5.3% 時間)
  盈虧平衡價差:            10 bps (at $1,000, Solana)
  盈利次數 (> 10 bps):    312 次 (43.3% 時間)
  預估累計淨利:           +$186.40 (312 次 x avg +$0.60)
  平均每次利潤:           +$0.60

── 價差分布 ──────────────────────────────

  0-10 bps:   ████████████████  408 (56.7%)
  10-20 bps:  ████████          164 (22.8%)
  20-50 bps:  █████             110 (15.3%)
  50-100 bps: █                  28 (3.9%)
  100+ bps:   ▏                  10 (1.4%)

── 時段分析 ──────────────────────────────

  價差最大時段: 00:00-06:00 UTC (亞洲早盤)
  價差最小時段: 14:00-18:00 UTC (歐美重疊)

── 趨勢 ─────────────────────────────────

  方向:   stable (標準差波動為主)
  走勢:   ▃▅▇▃▂▅▇▅▃▂▄▆▇▃▂▅▃▂▄▅ (volatile)

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. SOL 套利機會相當頻繁（43% 時間可盈利）
  2. 設定實時監控捕捉高價差時段:
     cex-dex-arbitrage monitor --assets SOL --chains solana --threshold-bps 30 --duration-min 360
  3. 嘗試更大規模提高單次利潤:
     cex-dex-arbitrage evaluate --asset SOL --chain solana --size-usd 5000

  以上僅為分析建議，不會自動執行任何交易。
══════════════════════════════════════════
```

---

### Example 5: "Scan for arb on Arbitrum and Base, $5000 size, VIP1" (English — scan with params)

**User:**
> Scan for arb on Arbitrum and Base, $5000 size, VIP1

**Intent Recognition:**
- Command: `scan`
- Assets: not specified → default `["BTC","ETH","SOL"]`
- Chains: `["arbitrum","base"]`
- Size: $5,000
- VIP: VIP1

**Tool Calls:**

```
1. system_get_capabilities → OK

2. For each (asset, chain) pair — 6 combinations:
   a. Fetch CEX ticker (3 calls — one per asset)
   b. Resolve DEX token address (if needed)
   c. Fetch DEX price (6 calls — per asset per chain)
   d. GoPlus check for non-native tokens
   e. Liquidity check
   f. Orderbook depth analysis
   g. DEX swap quote
   h. profitability-calculator.estimate for pairs with spread > 50 bps

3. Rank results by net_profit_usd descending
```

**Output follows the scan template from Section 12.1, with VIP1 fee rates (7 bps taker spot) applied to cost calculations.**

---

## 14. Implementation Notes

### Token Address Resolution

- **Native tokens** (ETH, SOL, BNB, etc.): Use the hardcoded native address from `references/chain-reference.md`. No `dex-token search` needed.
  - EVM chains: `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee`
  - Solana: `11111111111111111111111111111111`

- **Wrapped tokens** (WBTC on Ethereum, wSOL on Solana): Resolve via `dex-token search`. Cache the result for the session:
  ```
  token_address_cache["BTC:ethereum"] = "0x2260fac5e5542a773aa44fbcfedf7c193bc2c599"
  ```

- **Non-native tokens**: Always resolve via `dex-token search`, pick the result on the target chain with the highest liquidity.

### CEX instId Mapping

- Spot: `{ASSET}-USDT` (e.g., `BTC-USDT`, `ETH-USDT`, `SOL-USDT`)
- This skill only uses spot instruments. Perpetuals and futures are handled by `funding-rate-arbitrage` and `basis-trading`.

### Amount Conversion for DEX Swap Quotes

When calling `dex-swap quote`, amounts must be in minimal units:

```
amount_minimal = (size_usd / asset_price_usd) * 10^decimals

Examples:
  $1,000 of ETH at $3,412.50:
    qty = 1000 / 3412.50 = 0.29304 ETH
    amount_wei = 0.29304 * 10^18 = 293040000000000000

  $1,000 of SOL at $145.50:
    qty = 1000 / 145.50 = 6.8729 SOL
    amount_lamports = 6.8729 * 10^9 = 6872900000
```

**Critical: BSC USDT has 18 decimals** (not 6). Always check `references/chain-reference.md` Section 4 for correct decimals.

### Rate Limiting

| Tool | Rate Limit | Strategy |
|------|-----------|----------|
| `market_get_ticker` | 20 req/s | Safe to batch multiple assets in parallel |
| `market_get_orderbook` | 20 req/s | Only fetch for assets with spread > min_spread_bps |
| `market_get_candles` | 20 req/s | Paginate with 100 candles per call for backtest |
| `onchainos dex-market price` | Internal OKX limits | Add 1s delay if throttled |
| `onchainos dex-swap quote` | Internal OKX limits | Only call for assets passing spread + security filters |
| GoPlus `check_token_security` | ~100/day (free tier) | Cache results for 5 minutes within session |
| `account_*` tools | 1 req/s | Not used by this skill (read-only market data) |

### Multi-Chain Scanning Strategy

When `--chains` contains multiple chains, iterate chains **sequentially** and parallelize **within** each chain:

```
For chains = [arbitrum, base, solana]:

  Chain: arbitrum
    ├─ Parallel: fetch CEX tickers (BTC, ETH, SOL)
    ├─ Parallel: fetch DEX prices (BTC, ETH, SOL on arbitrum)
    ├─ Sequential: GoPlus checks (for non-native tokens)
    └─ Sequential: profitability-calculator (for qualifying pairs)

  Chain: base
    ├─ (same structure)
    ...

  Chain: solana
    ├─ (same structure)
    ...
```

This approach prevents hitting rate limits while maximizing throughput within each chain.

### Signal Decay

Arb signals decay rapidly. From `references/formulas.md` Section 7:

```
signal_strength = initial_strength * exp(-0.15 * time_elapsed_minutes)
Half-life: ~4.6 minutes
```

Implications:
- Prices fetched > 5 minutes ago are unreliable for arb decisions
- Monitor alerts should be acted on within 2-3 minutes
- Evaluate results include data age warnings when approaching staleness

### OKX String-to-Number Convention

All OKX API values are returned as **strings**. Always parse to `float` before any arithmetic:

```
WRONG:  "87234.5" - "87200.0" → NaN or string concat
RIGHT:  parseFloat("87234.5") - parseFloat("87200.0") → 34.5
```

This applies to: `last`, `bidPx`, `askPx`, `ts`, `vol24h`, all orderbook levels, all candle values.

---

## 15. Reference Files

| File | Relevance to This Skill |
|------|------------------------|
| `references/formulas.md` | Spread calculation (Section 1), cost formulas (Section 2), signal decay (Section 7) |
| `references/fee-schedule.md` | OKX VIP tiers, withdrawal fees, gas benchmarks, DEX protocol fees |
| `references/mcp-tools.md` | okx-trade-mcp tool reference (ticker, orderbook, candles) |
| `references/onchainos-tools.md` | OnchainOS CLI commands (dex-market price, dex-token search, dex-swap quote, gas) |
| `references/goplus-tools.md` | GoPlus security check tools and decision matrix |
| `references/chain-reference.md` | Supported chains, native addresses, instId formats, token addresses |
| `references/safety-checks.md` | Pre-trade checklist, risk limits, error catalog |
| `references/output-templates.md` | Header, opportunity table, cost breakdown, risk gauge, next steps templates |

---

## 16. Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2026-03-09 | 1.0.0 | Initial release. 4 commands: scan, evaluate, monitor, backtest. Full GoPlus + profitability-calculator integration. |
