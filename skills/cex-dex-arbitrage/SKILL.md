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
| Live switch | Requires explicit user confirmation (see Account Safety Protocol in Section 9) |

Even in `[LIVE]` mode, this skill only reads market data and produces recommendations. There is no risk of accidental execution.

---

## 4. Pre-flight (Machine-Executable Checklist)

Run these checks **in order** before any command. BLOCK at any step halts execution.

| # | Check | Command / Tool | Success Criteria | Failure Action |
|---|-------|---------------|-----------------|----------------|
| 1 | okx-trade-mcp connected | `system_get_capabilities` (DEMO or LIVE server) | `authenticated: true`, `modules` includes `"market"` | BLOCK — output `MCP_NOT_CONNECTED`. Tell user to verify `~/.okx/config.toml` and restart MCP server. |
| 2 | okx-trade-mcp mode | `system_get_capabilities` -> `mode` field | Returns `"demo"` or `"live"` matching expected mode | WARN — display actual mode in header. If user requested live but got demo, surface mismatch. |
| 3 | OnchainOS CLI installed | `which onchainos` (Bash) | Exit code 0, returns a valid path | BLOCK — DEX venue required for this skill. Tell user: `npx skills add okx/onchainos-skills` |
| 4 | OnchainOS CLI functional | `onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | Returns valid JSON with `priceUsd` field | BLOCK — DEX venue required. Suggest checking network connectivity. |
| 5 | GoPlus MCP available | `check_token_security` with a known-good token (e.g., WETH on Ethereum) | Returns valid security object | WARN — security checks unavailable. Proceed but label ALL outputs `[SECURITY UNCHECKED]`. |
| 6 | risk-limits.yaml loaded | Read `config/risk-limits.yaml` or fall back to defaults from safety checks | Valid YAML with `cex-dex-arbitrage` key | INFO — using default risk limits (max_trade_size=$2,000, min_liquidity_usd=$100,000). |

### Pre-flight Decision Tree

```
Check 1 FAIL -> BLOCK (cannot proceed without CEX data)
Check 1 PASS -> Check 2
  Check 2 mismatch -> WARN + continue
  Check 2 PASS -> Check 3
    Check 3 FAIL -> BLOCK (DEX venue mandatory for CEX-DEX arbitrage)
    Check 3 PASS -> Check 4
      Check 4 FAIL -> BLOCK (DEX venue mandatory)
      Check 4 PASS -> Check 5
        Check 5 FAIL -> WARN: security unchecked, continue with labels
        Check 5 PASS -> Check 6
          Check 6 FAIL -> INFO: using default limits
          Check 6 PASS -> ALL SYSTEMS GO
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
| `--chains` | string[] | No | `["ethereum"]` | `ethereum`, `solana`, `base`, `arbitrum`, `bsc`, `polygon`, `xlayer` | Must be supported (see Chain Reference in this document). |
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
| "搬磚" (generic) | `scan` | Broad arb intent -> scan with defaults |
| "ETH 有冇得搬" (single asset, no chain) | `evaluate --asset ETH` | Single asset mentioned -> evaluate |
| "ETH 喺 Base 上面同 OKX 有冇價差" | `evaluate --asset ETH --chain base` | Specific asset + chain -> evaluate |
| "幫我睇住 BTC 價差" | `monitor --assets BTC` | "睇住" = monitoring intent |
| "上個月 SOL 套利機會多唔多" | `backtest --asset SOL --lookback-days 30` | "上個月" = historical intent |

---

### Step 2: Pre-Execution Safety Checks

For each candidate token (after extracting the list from Step 1), run the following checks **before any price analysis**:

#### 2a. Token Address Resolution

```
For each (asset, chain) pair:

  IF asset is native token on chain (ETH on ethereum/arbitrum/base, SOL on solana, BNB on bsc):
    -> Use native address from Chain Reference (Section 16)
    -> SKIP GoPlus security check (native tokens are safe)

  ELSE:
    -> onchainos dex-token search {ASSET} --chains {chain}
    -> Pick result matching target chain with highest liquidity
    -> If no results -> BLOCK: TOKEN_NOT_FOUND for this asset/chain
    -> Store: token_address_cache[{ASSET}:{CHAIN}] = address
```

#### 2b. GoPlus Security Check (for non-native tokens)

```
For each non-native token:

  1. GoPlus check_token_security(chain_id, contract_address)

  Decision logic (from GoPlus Decision Matrix — Section 17):

  BLOCK conditions (any single trigger -> remove from scan):
    - is_honeypot === "1"           -> BLOCK("Honeypot detected — cannot sell")
    - buy_tax > 0.05 (5%)          -> BLOCK("Buy tax {tax}% exceeds 5% limit")
    - sell_tax > 0.10 (10%)        -> BLOCK("Sell tax {tax}% exceeds 10% limit")
    - is_open_source === "0"       -> BLOCK("Contract source not verified")
    - can_take_back_ownership === "1" -> BLOCK("Ownership reclaimable")
    - owner_change_balance === "1"  -> BLOCK("Owner can modify balances")
    - slippage_modifiable === "1"  -> BLOCK("Tax modifiable by owner")
    - cannot_sell_all === "1"      -> BLOCK("Cannot sell entire balance")
    - top 10 non-contract holders > 80% -> BLOCK("Concentrated ownership > 80%")

  WARN conditions (flag but continue):
    - is_proxy === "1"             -> WARN("Upgradeable proxy contract")
    - buy_tax > 0.01 (1%)         -> WARN("Buy tax > 1%")
    - sell_tax > 0.02 (2%)        -> WARN("Sell tax > 2%")
    - is_mintable === "1"          -> WARN("Token is mintable")
    - transfer_pausable === "1"   -> WARN("Transfers can be paused")
    - top 10 non-contract holders > 50% -> WARN("Concentrated ownership > 50%")
```

#### 2c. Liquidity Validation

```
For each token that passed security:

  onchainos dex-token price-info {contract_address} --chain {chain}
  -> Extract: liquidityUsd

  IF liquidityUsd < min_liquidity_usd:
    -> BLOCK: INSUFFICIENT_LIQUIDITY
    -> Message: "DEX 流動性 ${actual} 低於最低要求 ${required}"
```

#### 2d. Data Staleness Check

```
For all data sources:

  data_age_ms = Date.now() - source_timestamp

  Arbitrage mode thresholds (stricter than analysis mode):
    WARN if data_age_ms > 3000 (3 seconds)
    BLOCK if data_age_ms > 5000 (5 seconds)

  On BLOCK:
    -> Refetch data once
    -> If still stale after refetch -> output DATA_STALE error
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
   -> Returns PriceSnapshot with synchronized CEX and DEX prices

   Internally this calls:
     CEX: market_get_ticker({ instId: "{ASSET}-USDT" })
       -> Extract: last, bidPx, askPx, ts
     DEX: onchainos dex-market price --chain {chain} --token {address}
       -> Extract: priceUsd
```

#### 3b. Spread Calculation

```
2. For each asset where PriceSnapshot is available:

   spread_bps = abs(cex_price - dex_price) / min(cex_price, dex_price) * 10000

   IF spread_bps < min_spread_bps:
     -> Skip this asset (below threshold)

   IF spread_bps > 5000 (50%):
     -> BLOCK: PRICE_ANOMALY
     -> Likely: wrong token address, stale DEX pool, or low liquidity

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
      -> Walk bids (for sell) or asks (for buy) to compute:
        - Available depth at trade size
        - Volume-weighted average price (VWAP)
        - price_impact_bps = (vwap - best_price) / best_price * 10000

   b. DEX executable price + price impact:
      onchainos dex-swap quote \
        --from {native_address} \
        --to {token_address} \
        --amount {amount_in_minimal_units} \
        --chain {chain}
      -> Extract: toTokenAmount, priceImpactPercent, estimatedGas

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
   -> Returns ProfitabilityResult with:
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

Format using output templates (see Section 18). The output structure varies by command:

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
| Scan: 0 opportunities | "暫無套利機會。設定監控等待價差擴大 -> `cex-dex-arbitrage monitor`" |
| Scan: opportunities found | "詳細評估最佳機會 -> `cex-dex-arbitrage evaluate --asset {TOP} --chain {CHAIN}`" |
| Evaluate: profitable | "如滿意，手動在 2-3 分鐘內執行兩腿交易"; "檢查不同規模 -> `profitability-calculator sensitivity`" |
| Evaluate: not profitable | "減小規模或切換 L2 降低 Gas"; "設定監控等待更大價差 -> `cex-dex-arbitrage monitor`" |
| Monitor: alert triggered | "立即評估 -> `cex-dex-arbitrage evaluate --asset {ASSET} --chain {CHAIN}`" |
| Backtest: frequent opportunities | "設定實時監控 -> `cex-dex-arbitrage monitor`" |

---

## 9. Safety Checks (Per Operation)

### 9.1 Pre-Trade Safety Checklist

Every skill runs through these checks **in order** before producing a recommendation. A BLOCK at any step halts the pipeline immediately.

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
  |
  +- Check 1-2: Infrastructure   -- BLOCK? -> Output error, STOP
  |
  +- Check 3: Freshness          -- BLOCK? -> Refetch data, retry once -> BLOCK? -> STOP
  |
  +- Check 4-7: Token Security   -- BLOCK? -> Move asset to blocked_assets list, CONTINUE to next asset
  |                                  WARN?  -> Attach warning labels, CONTINUE
  |
  +- Check 8-9: Liquidity        -- BLOCK? -> Move to blocked_assets, CONTINUE to next asset
  |                                  WARN?  -> Attach liquidity note, CONTINUE
  |
  +- Check 10-11: Profitability  -- BLOCK? -> Exclude from results, CONTINUE to next asset
  |                                  WARN?  -> Include with margin warning
  |
  +- ALL PASSED -> Include in ranked results with accumulated WARNs
```

### 9.2 cex-dex-arbitrage Risk Limits

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| max_trade_size | $2,000 | $10,000 | Maximum USD value per arbitrage trade |
| max_concurrent_arbs | 3 | 5 | Maximum simultaneous open arbitrage positions |
| min_net_profit | $5 | — | Minimum net profit after all costs to recommend |
| max_price_age_sec | 5 | 5 | Maximum age of price data in seconds |
| min_liquidity_usd | $100,000 | — | Minimum DEX liquidity to consider |
| max_slippage_pct | 0.5% | 2% | Maximum acceptable slippage |

### 9.3 Account Safety Protocol

#### Demo vs Live Mode

The system defaults to **demo mode** at all times. Switching to live mode requires explicit user action.

**Demo Mode (Default):**
- MCP server config: `okx-DEMO-simulated-trading`
- All prices and positions are simulated
- All outputs include header: `[DEMO]`
- No real funds at risk. No confirmation required.

**Live Mode:**
- MCP server config: `okx-LIVE-trading`
- Real market data and real account positions
- All outputs include header: `[LIVE]`
- Recommendations are still analysis-only (no auto-execution)

**Switching from Demo to Live:**
```
1. User explicitly says "live", "真實帳戶", "real account", or similar
2. System confirms the switch with a clear warning:
   You are switching to real account mode.
   - All data will come from your real OKX account
   - Recommendations remain analysis-only, no auto-execution
   - Please confirm: enter "確認" or "confirm" to continue
3. User must reply with explicit confirmation
4. System verifies authentication via system_get_capabilities
5. If authenticated: switch and display [LIVE] header
6. If NOT authenticated: show AUTH_FAILED error, remain in demo
```

**Session Rules:**
- Default on startup: Always demo mode
- Timeout: If no activity for 30 minutes, revert to demo mode
- Error fallback: If live mode encounters AUTH_FAILED, revert to demo with notification
- Header requirement: EVERY output must show `[DEMO]` or `[LIVE]` — no exceptions
- No auto-execution: Even in live mode, skills only provide recommendations. The `[RECOMMENDATION ONLY]` header is always present.

---

## 10. Error Codes & Recovery

| Code | Condition | User Message (ZH) | User Message (EN) | Recovery |
|------|-----------|-------------------|-------------------|----------|
| `MCP_NOT_CONNECTED` | okx-trade-mcp server unreachable | MCP 伺服器無法連線。請確認 okx-trade-mcp 是否正在運行。 | MCP server unreachable. Check if okx-trade-mcp is running. | Verify `~/.okx/config.toml`, restart server |
| `AUTH_FAILED` | API key invalid or expired | API 認證失敗。請檢查 OKX API 金鑰設定。 | API authentication failed. Check OKX API key config. | Update config.toml |
| `DATA_STALE` | Price data > 5s old (after retry) | 市場數據已過期（{venue} 延遲 {age}ms，套利模式上限 5000ms）。 | Market data stale ({venue}: {age}ms, arb mode max 5000ms). | Auto-retry once, then fail |
| `SECURITY_BLOCKED` | GoPlus check failed | 安全檢查未通過：{reason}。此代幣存在風險，已從結果中排除。 | Security check failed: {reason}. Token excluded. | Show GoPlus findings, do not recommend |
| `SECURITY_UNAVAILABLE` | GoPlus API unreachable | 安全檢查服務暫時無法使用。所有結果標記為 [SECURITY UNCHECKED]。 | Security service unavailable. Results marked [SECURITY UNCHECKED]. | Retry once, then continue with warning labels |
| `SECURITY_WARN` | GoPlus check returned warnings | 安全提醒：{warnings}。請自行評估風險。 | Security notice: {warnings}. Assess risk before proceeding. | Display warnings, continue with labels |
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
| `LEVERAGE_EXCEEDED` | Requested leverage > limit | 槓桿倍數 {requested}x 超過上限 {limit}x | Leverage {requested}x exceeds limit {limit}x | Cap at limit and inform user |
| `POSITION_LIMIT` | Max concurrent positions reached | 已達持倉上限（{current}/{max}），請先關閉現有倉位 | Position limit reached ({current}/{max}). Close existing positions first. | Show current open positions |
| `FUTURES_EXPIRED` | Futures contract has expired or < min days | 期貨合約已到期或距到期不足 {min_days} 天 | Futures contract expired or < {min_days} days to expiry | Suggest next-expiry contract |
| `HUMAN_APPROVAL_REQUIRED` | Copy trade needs confirmation | 此操作需要您的確認才能繼續 | This action requires your explicit approval to proceed. | Wait for user confirmation |

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
  |
  |  PriceSnapshot[]
  v
cex-dex-arbitrage.scan / evaluate
  |
  +- GoPlus check_token_security(chain_id, contract_address)
  |    -> BLOCK / WARN / SAFE
  |
  +- OnchainOS dex-token price-info -> liquidityUsd check
  |    -> BLOCK if < min_liquidity_usd
  |
  +- OnchainOS dex-swap quote -> priceImpactPercent check
  |    -> BLOCK if > 1%
  |
  +- market_get_orderbook -> CEX depth analysis
  |
  |  Constructs TradeLeg[]
  v
profitability-calculator.estimate(legs)
  |
  |  ProfitabilityResult
  v
cex-dex-arbitrage applies go/no-go:
  |  - is_profitable == true?
  |  - profit_to_cost_ratio >= 1.5?
  |  - security_status != BLOCK?
  v
Output: Ranked opportunities with safety checks + cost breakdowns
```

---

## 12. Key Formulas (Inlined)

### 12.1 CEX-DEX Spread (basis points)

```
spread_bps = abs(cex_price - dex_price) / min(cex_price, dex_price) * 10000
```

| Variable | Definition |
|----------|-----------|
| `cex_price` | Mid-price on CEX orderbook (OKX), or best bid/ask depending on direction |
| `dex_price` | Quote price returned by DEX aggregator (e.g., Jupiter, 1inch, OKX DEX API) for the given size |
| `spread_bps` | Unsigned spread in basis points (1 bp = 0.01%) |

**Worked Example**

```
cex_price = 3,412.50  (ETH-USDT mid on OKX)
dex_price = 3,419.80  (Jupiter quote, selling ETH for USDT on Solana)

spread_bps = abs(3412.50 - 3419.80) / min(3412.50, 3419.80) * 10000
           = 7.30 / 3412.50 * 10000
           = 21.4 bps
```

**Caveats**
- Always use the same quote direction (both buy or both sell) for a fair comparison.
- DEX price already includes DEX protocol fee and price impact for the quoted size.
- CEX price should use bid for sell, ask for buy — not mid — when evaluating executable spread.

### 12.2 Effective Price After Slippage

```
effective_price = quoted_price * (1 + slippage_bps / 10000)   # for buys
effective_price = quoted_price * (1 - slippage_bps / 10000)   # for sells
```

### 12.3 Price Impact from Orderbook Depth

```
price_impact_bps = (execution_avg_price - best_price) / best_price * 10000

execution_avg_price = sum(level_price_i * level_qty_i) / sum(level_qty_i)
    for levels consumed until order_size is filled
```

**Worked Example (Selling 10 ETH)**

```
Orderbook bids:
  Level 1: 3,412.50 x 3 ETH
  Level 2: 3,412.00 x 4 ETH
  Level 3: 3,411.20 x 5 ETH

execution_avg_price = (3412.50*3 + 3412.00*4 + 3411.20*3) / (3+4+3)
                    = 34119.10 / 10
                    = 3,411.91

price_impact_bps = (3412.50 - 3411.91) / 3412.50 * 10000
                 = 1.7 bps
```

### 12.4 Cost Calculations

**CEX Trading Fee:**
```
cex_fee = size_usd * fee_rate
```

**DEX Gas Cost (EVM):**
```
gas_cost_usd = gas_price_gwei * gas_limit * native_token_price_usd / 1e9
```

**DEX Gas Cost (Solana):**
```
gas_cost_usd = (base_fee_lamports + priority_fee_lamports) * sol_price / 1e9
```

**Slippage Cost:**
```
slippage_cost_usd = size_usd * estimated_slippage_bps / 10000
```

**Total Cost (one direction):**
```
total_cost = cex_fee + dex_gas_cost + dex_protocol_fee + slippage_cost + bridge_fee + withdrawal_fee
```

**Net Profit:**
```
net_profit = gross_spread_usd - total_cost
gross_spread_usd = size_usd * spread_bps / 10000
```

**Worked Example (CEX-DEX arb, one direction)**
```
cex_fee (VIP1 taker)    = $35.00
dex_gas (Arbitrum)       =  $0.30
dex_protocol_fee         =  $0.00  (included in quote)
slippage (CEX)           = $15.00
withdrawal_fee (USDT)    =  $1.00
bridge_fee               =  $0.00  (same chain)

total_cost = 35.00 + 0.30 + 0.00 + 15.00 + 1.00 + 0.00
           = $51.30
```

### 12.5 Profit-to-Cost Ratio

```
profit_to_cost = net_profit / total_cost
```

**Interpretation:**
- < 1.0x: Costs exceed profit — do not trade
- 1.0x - 1.5x: Marginal — high execution risk, consider skipping
- 1.5x - 3.0x: Acceptable — proceed with caution
- 3.0x+: Attractive — high confidence trade

### 12.6 Signal Decay

Arb signals decay rapidly:

```
signal_strength = initial_strength * exp(-decay_rate * time_elapsed_minutes)
```

**Typical Decay Rates:**

| Signal Type | Decay Rate | Half-Life |
|------------|------------|-----------|
| CEX-DEX arb spread | 0.15 | ~4.6 min |
| Funding rate anomaly | 0.01 | ~69 min |
| Basis mispricing | 0.005 | ~139 min |
| Smart money signal | 0.02 | ~35 min |

**Worked Example**
```
initial_strength = 0.90  (strong arb signal)
decay_rate = 0.15
time_elapsed_minutes = 5

signal_strength = 0.90 * exp(-0.15 * 5)
               = 0.90 * 0.4724
               = 0.425

Signal has decayed from 0.90 to 0.43 in 5 minutes — opportunity may be fading.
```

Implications:
- Prices fetched > 5 minutes ago are unreliable for arb decisions
- Monitor alerts should be acted on within 2-3 minutes
- Signal strength below 0.3 generally means the opportunity has likely been arbitraged away

---

## 13. Fee Schedule (Inlined)

### 13.1 OKX Spot Trading Fees

| Tier | 30d Volume (USD) | Maker | Taker |
|------|------------------|-------|-------|
| VIP0 | < 5M | 0.060% | 0.080% |
| VIP1 | >= 5M | 0.040% | 0.070% |
| VIP2 | >= 10M | 0.030% | 0.060% |
| VIP3 | >= 20M | 0.020% | 0.050% |
| VIP4 | >= 100M | 0.015% | 0.040% |
| VIP5 | >= 200M | 0.010% | 0.035% |

**Quick Reference: Round-Trip Taker Fees (bps)**

| Tier | Spot RT |
|------|---------|
| VIP0 | 16.0 bps |
| VIP1 | 14.0 bps |
| VIP2 | 12.0 bps |
| VIP3 | 10.0 bps |
| VIP4 | 8.0 bps |
| VIP5 | 7.0 bps |

**Fee Notes:**
- **Maker** = limit order that adds liquidity to the orderbook (not immediately matched)
- **Taker** = market order or limit order that immediately matches
- For cost estimation, assume taker fees unless the skill specifically uses limit orders

### 13.2 OKX Withdrawal Fees

**ETH Withdrawals:**

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Ethereum (ERC-20) | 0.00035 ETH | 0.001 ETH |
| Arbitrum One | 0.0001 ETH | 0.0001 ETH |
| Optimism | 0.00004 ETH | 0.0001 ETH |
| Base | 0.00004 ETH | 0.0001 ETH |
| zkSync Era | 0.000065 ETH | 0.0001 ETH |

**USDT Withdrawals:**

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Ethereum (ERC-20) | 3.0 USDT | 10.0 USDT |
| Tron (TRC-20) | 1.0 USDT | 0.1 USDT |
| Polygon | 0.8 USDT | 0.1 USDT |
| Arbitrum One | 0.1 USDT | 0.1 USDT |
| Optimism | 0.1 USDT | 0.1 USDT |
| Base | 0.1 USDT | 0.1 USDT |
| Solana | 1.0 USDT | 1.0 USDT |
| BSC (BEP-20) | 0.3 USDT | 10.0 USDT |
| Avalanche C-Chain | 1.0 USDT | 1.0 USDT |

**USDC Withdrawals:**

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Ethereum (ERC-20) | 3.0 USDC | 10.0 USDC |
| Arbitrum One | 0.1 USDC | 0.1 USDC |
| Optimism | 0.1 USDC | 0.1 USDC |
| Base | 0.1 USDC | 0.1 USDC |
| Solana | 1.0 USDC | 1.0 USDC |
| BSC (BEP-20) | 0.3 USDC | 10.0 USDC |

**SOL Withdrawals:**

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Solana | 0.008 SOL | 0.1 SOL |

**BTC Withdrawals:**

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Bitcoin (BTC) | 0.0001 BTC | 0.001 BTC |

**Other Major Assets:**

| Asset | Network | Fee | Min Withdrawal |
|-------|---------|-----|---------------|
| BNB | BSC (BEP-20) | 0.0005 BNB | 0.01 BNB |
| MATIC | Polygon | 0.1 MATIC | 0.1 MATIC |
| AVAX | Avalanche C-Chain | 0.01 AVAX | 0.1 AVAX |
| ARB | Arbitrum One | 0.1 ARB | 1.0 ARB |
| OP | Optimism | 0.1 OP | 1.0 OP |

**Withdrawal Fee Notes:**
- Withdrawal fees are **flat amounts**, not percentages.
- Fees are updated periodically based on network conditions.
- Always verify current fees via OKX API before calculating costs.
- Internal transfers between OKX accounts (sub-accounts) are **free**.
- Deposits to OKX are **free** (user only pays network gas).

### 13.3 Gas Benchmarks per Chain

| Chain | Avg Gas Price | Swap Gas Limit | Approx Cost (USD) | Native Token |
|-------|--------------|----------------|-------------------|-------------|
| Ethereum | 20-50 gwei | 150,000-300,000 | $5.00-$30.00 | ETH |
| Arbitrum | 0.1-0.5 gwei | 1,000,000-2,000,000 | $0.10-$0.50 | ETH |
| Base | 0.005-0.02 gwei | 150,000-300,000 | $0.01-$0.05 | ETH |
| Optimism | 0.005-0.02 gwei | 150,000-300,000 | $0.01-$0.05 | ETH |
| Polygon | 30-100 gwei | 150,000-300,000 | $0.01-$0.05 | MATIC |
| BSC | 3-5 gwei | 150,000-300,000 | $0.10-$0.30 | BNB |
| Avalanche | 25-30 nAVAX | 150,000-300,000 | $0.02-$0.10 | AVAX |
| Solana | N/A (lamports) | N/A (compute units) | $0.001-$0.01 | SOL |
| X Layer | 0.01-0.05 gwei | 150,000-300,000 | $0.001-$0.01 | OKB |

**Cost Efficiency by Chain (for $10,000 trade):**

| Chain | Gas as % of Trade | Practical Minimum Trade |
|-------|-------------------|------------------------|
| Ethereum | 0.05-0.30% | $5,000+ |
| Arbitrum | 0.001-0.005% | $100+ |
| Base | 0.0001-0.0005% | $50+ |
| Solana | 0.00001-0.0001% | $10+ |
| BSC | 0.001-0.003% | $100+ |
| Polygon | 0.0001-0.0005% | $50+ |

### 13.4 DEX Protocol Fees

**Uniswap (Ethereum, Arbitrum, Base, Polygon, Optimism, BSC):**

| Pool Tier | Fee | Typical Use |
|-----------|-----|------------|
| 0.01% | 1 bp | Stable-stable pairs (USDC/USDT) |
| 0.05% | 5 bps | Stable pairs, high-correlation pairs |
| 0.30% | 30 bps | Standard pairs (ETH/USDC, WBTC/ETH) |
| 1.00% | 100 bps | Exotic / low-liquidity pairs |

**Jupiter (Solana):**
- Jupiter itself charges **no protocol fee** (aggregator layer)
- Underlying pool fees vary: Orca 0.01%-2.00%, Raydium 0.25%, Meteora 0.01%-1.00%
- Route optimization selects the best fee-adjusted path automatically

**DEX Fee Notes:**
- Aggregator quotes already include DEX fees in the output amount. Do not double-count.
- When comparing CEX vs DEX prices, the DEX price is already net of protocol fees.

### 13.5 Pattern A: CEX-DEX Arbitrage Cost Template

```
Trade size: $50,000

CEX buy (VIP1 taker):         $35.00  (7.0 bps)
CEX withdrawal (ETH to Arb):   $0.34  (0.0001 ETH)
DEX sell gas (Arbitrum):        $0.30
DEX slippage:                  $10.00  (est. 2 bps)
                               ------
Total one-way cost:            $45.64  (9.1 bps)
```

---

## 14. Output Format & Templates (Inlined)

### 14.1 Formatting Rules

**Monetary Values:** `$12,345.67`, `+$1,234.56`, `-$89.10` (2 decimal places, comma thousands)
**Percentages:** 1 decimal place (e.g., `12.5%`). APY/APR: 2 decimal places.
**Basis Points:** Integer only (e.g., `21 bps`, `145 bps`)
**Risk Levels:** `[SAFE]`, `[WARN]`, `[BLOCK]`
**Timestamps:** `YYYY-MM-DD HH:MM UTC` (always UTC)

**Risk Gauge (Visual):**
```
1/10:  ▓░░░░░░░░░      LOW RISK
2/10:  ▓▓░░░░░░░░      LOW RISK
3/10:  ▓▓▓░░░░░░░      MODERATE-LOW
4/10:  ▓▓▓▓░░░░░░      MODERATE-LOW
5/10:  ▓▓▓▓▓░░░░░      MODERATE
6/10:  ▓▓▓▓▓▓░░░░      MODERATE
7/10:  ▓▓▓▓▓▓▓░░░      ELEVATED
8/10:  ▓▓▓▓▓▓▓▓░░      ELEVATED
9/10:  ▓▓▓▓▓▓▓▓▓░      HIGH RISK
10/10: ▓▓▓▓▓▓▓▓▓▓      HIGH RISK
```

**Sparklines:** Characters: `▁▂▃▄▅▆▇█` (8-24 data points)

**Section Separators:**
- Major sections: `══════════════════════════════════════════`
- Sub-sections: `── Section Name ──────────────────────────`
- Minor: `──────────────────────────────────────────`

### 14.2 Global Header Template

```
══════════════════════════════════════════
  {SKILL_NAME}
  [{MODE}] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: {TIMESTAMP}
  Data sources: {DATA_SOURCES}
══════════════════════════════════════════
```

### 14.3 Opportunity Table Template

```
── Opportunities Found: {COUNT} ────────────────────

  #{RANK}  {PAIR} ({CHAIN})
  +- Spread:      {SPREAD_BPS} bps ({SPREAD_PCT}%)
  +- Direction:    {DIRECTION}
  +- Est. Profit:  {NET_PROFIT}  (after costs)
  +- Costs:        {TOTAL_COST}
  +- Profit/Cost:  {PROFIT_TO_COST}x
  +- Signal Age:   {SIGNAL_AGE}
  +- Confidence:   {CONFIDENCE}
  +- Risk:         {RISK_LEVEL}

──────────────────────────────────────────
  Showing top {COUNT} of {TOTAL_SCANNED} pairs scanned
```

### 14.4 Cost Breakdown Template

```
── Cost Breakdown ──────────────────────────

  Trade Size:    {TRADE_SIZE}
  VIP Tier:      {VIP_TIER}

  Leg 1 -- {LEG1_VENUE} ({LEG1_SIDE})
  +- Trading Fee:    {LEG1_FEE}      ({LEG1_FEE_BPS} bps)
  +- Slippage Est:   {LEG1_SLIPPAGE} ({LEG1_SLIP_BPS} bps)
  +- Subtotal:       {LEG1_TOTAL}

  Leg 2 -- {LEG2_VENUE} ({LEG2_SIDE})
  +- Trading Fee:    {LEG2_FEE}      ({LEG2_FEE_BPS} bps)
  +- Gas Cost:       {LEG2_GAS}
  +- Slippage Est:   {LEG2_SLIPPAGE} ({LEG2_SLIP_BPS} bps)
  +- Subtotal:       {LEG2_TOTAL}

  Transfer Costs
  +- Withdrawal Fee: {WITHDRAWAL_FEE}
  +- Bridge Fee:     {BRIDGE_FEE}

  ──────────────────────────────────────
  Gross Spread:      {GROSS_SPREAD}   ({GROSS_BPS} bps)
  Total Costs:      -{TOTAL_COST}     ({COST_BPS} bps)
  ══════════════════════════════════════
  Net Profit:        {NET_PROFIT}     ({NET_BPS} bps)
  Profit/Cost:       {PROFIT_TO_COST}x
  ══════════════════════════════════════
```

### 14.5 Safety Check Results Template

```
── Safety Checks ───────────────────────────

  [  {STATUS}  ] {CHECK_NAME}
                  {CHECK_DETAIL}
  ...

  ──────────────────────────────────────
  Overall: {OVERALL_STATUS}
  {OVERALL_MESSAGE}
```

### 14.6 Risk Gauge Template

```
── Risk Assessment ─────────────────────────

  Overall Risk:  {RISK_GAUGE}  {RISK_SCORE}/10
                 {RISK_LABEL}

  Breakdown:
  +- Execution Risk:   {EXEC_GAUGE}  {EXEC_SCORE}/10
  |                    {EXEC_NOTE}
  +- Market Risk:      {MKT_GAUGE}   {MKT_SCORE}/10
  |                    {MKT_NOTE}
  +- Smart Contract:   {SC_GAUGE}    {SC_SCORE}/10
  |                    {SC_NOTE}
  +- Liquidity Risk:   {LIQ_GAUGE}   {LIQ_SCORE}/10
                       {LIQ_NOTE}
```

### 14.7 Next Steps Template

```
══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  {STEP_1}
  {STEP_2}
  {STEP_3}

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Past spreads/yields do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

---

## 15. Scan Results — Complete Example

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
  +- Spread:      68 bps (0.68%)
  +- Direction:    Buy CEX / Sell DEX
  +- CEX Price:   $3,412.50
  +- DEX Price:   $3,435.70
  +- Est. Profit: +$2.94  (after costs)
  +- Costs:       $3.86
  +- Profit/Cost: 0.76x
  +- Liquidity:   $45.2M
  +- Confidence:  HIGH
  +- Risk:        [SAFE]

  #2  SOL-USDT (Base)
  +- Spread:      52 bps (0.52%)
  +- Direction:    Buy DEX / Sell CEX
  +- CEX Price:   $145.50
  +- DEX Price:   $144.74
  +- Est. Profit: +$1.40  (after costs)
  +- Costs:       $3.80
  +- Profit/Cost: 0.37x
  +- Liquidity:   $12.8M
  +- Confidence:  HIGH
  +- Risk:        [WARN] 利潤空間偏薄

──────────────────────────────────────────
  Scanned: 6 pairs | Found: 2 opportunities

── Blocked Assets ────────────────────────

  (none)

── Risk Assessment (Top Opportunity) ─────

  Overall Risk:  ▓▓▓░░░░░░░  3/10
                 MODERATE-LOW

  Breakdown:
  +- Execution Risk:   ▓▓░░░░░░░░  2/10
  |                    Stable spread; deep orderbook
  +- Market Risk:      ▓▓▓░░░░░░░  3/10
  |                    ETH 24h range 2.4%; moderate vol
  +- Smart Contract:   ▓▓░░░░░░░░  2/10
  |                    Native ETH — no contract risk
  +- Liquidity Risk:   ▓▓░░░░░░░░  2/10
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

## 15.2 Evaluate Results — Complete Example

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
  +- Last:     $3,412.50
  +- Bid:      $3,412.10
  +- Ask:      $3,412.90
  +- Data Age: 2s [SAFE]

  OKX DEX (Arbitrum)
  +- Price:    $3,435.70
  +- Impact:   0.08% (at $5K swap quote)
  +- Data Age: 3s [SAFE]

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
  +- Trading Fee:    $4.00      (8.0 bps, VIP0 taker)
  +- Slippage Est:   $0.75      (1.5 bps, from orderbook)
  +- Subtotal:       $4.75

  Leg 2 — OKX DEX Arbitrum (Sell)
  +- Gas Cost:       $0.30      (0.2 gwei, Arbitrum)
  +- Slippage Est:   $0.40      (0.8 bps, from swap quote)
  +- Subtotal:       $0.70

  Transfer Costs
  +- Withdrawal Fee: $0.34      (0.0001 ETH to Arbitrum)
  +- Bridge Fee:     $0.00      (direct withdrawal)

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
  +- Execution Risk:   ▓▓░░░░░░░░  2/10
  |                    Both venues liquid; spread stable
  +- Market Risk:      ▓▓▓░░░░░░░  3/10
  |                    ETH vol moderate; 2% move unlikely in 3-min window
  +- Smart Contract:   ▓░░░░░░░░░  1/10
  |                    Native ETH — zero contract risk
  +- Liquidity Risk:   ▓▓░░░░░░░░  2/10
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

## 16. Chain Reference (Inlined)

### 16.1 Supported Chains

| Chain | chainIndex (OKX) | Native Token | Native Token Address (DEX) | RPC Unit | Block Time | Explorer |
|-------|------------------|-------------|---------------------------|----------|------------|----------|
| Ethereum | 1 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~12s | etherscan.io |
| BSC | 56 | BNB | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~3s | bscscan.com |
| Polygon | 137 | MATIC | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~2s | polygonscan.com |
| Arbitrum One | 42161 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~0.25s | arbiscan.io |
| Optimism | 10 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~2s | optimistic.etherscan.io |
| Base | 8453 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~2s | basescan.org |
| Avalanche C-Chain | 43114 | AVAX | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | nAVAX | ~2s | snowscan.xyz |
| Solana | 501 | SOL | `11111111111111111111111111111111` | lamports | ~0.4s | solscan.io |
| X Layer | 196 | OKB | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~3s | oklink.com/xlayer |

### 16.2 CRITICAL: Solana Native Token Address

```
CORRECT:   11111111111111111111111111111111
WRONG:     So11111111111111111111111111111111111111112  (this is wSOL, the wrapped version)
```

- Use `11111111111111111111111111111111` (the System Program ID) when referring to native SOL in OKX DEX API calls.
- `So11111111111111111111111111111111111111112` is Wrapped SOL (wSOL) — a different asset used within DeFi protocols.

### 16.3 EVM Native Token Address

All EVM-compatible chains use the same placeholder address for native tokens:
```
0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
```
Applies to: ETH (Ethereum, Arbitrum, Optimism, Base), BNB (BSC), MATIC (Polygon), AVAX (Avalanche), OKB (X Layer).

### 16.4 OKX instId Formats

**Spot:** `{BASE}-{QUOTE}` (e.g., `BTC-USDT`, `ETH-USDT`, `SOL-USDT`)
**USDT-Margined Perpetual Swaps:** `{BASE}-USDT-SWAP` (e.g., `BTC-USDT-SWAP`)
**Coin-Margined Perpetual Swaps:** `{BASE}-USD-SWAP` (e.g., `BTC-USD-SWAP`)
**Futures:** `{BASE}-USD-{YYMMDD}` or `{BASE}-USDT-{YYMMDD}` (e.g., `BTC-USD-260327`)
**Options:** `{BASE}-USD-{YYMMDD}-{STRIKE}-{C|P}` (e.g., `BTC-USD-260327-90000-C`)

### 16.5 Major Token Contract Addresses

**Ethereum (chainIndex: 1):**

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0xdac17f958d2ee523a2206206994597c13d831ec7` | 6 |
| USDC | `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` | 6 |
| WETH | `0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2` | 18 |
| WBTC | `0x2260fac5e5542a773aa44fbcfedf7c193bc2c599` | 8 |
| DAI | `0x6b175474e89094c44da98b954eedeac495271d0f` | 18 |
| LINK | `0x514910771af9ca656af840dff83e8264ecf986ca` | 18 |
| UNI | `0x1f9840a85d5af5bf1d1762f925bdaddc4201f984` | 18 |

**Arbitrum One (chainIndex: 42161):**

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0xfd086bc7cd5c481dcc9c85ebe478a1c0b69fcbb9` | 6 |
| USDC | `0xaf88d065e77c8cc2239327c5edb3a432268e5831` | 6 |
| USDC.e (bridged) | `0xff970a61a04b1ca14834a43f5de4533ebddb5cc8` | 6 |
| WETH | `0x82af49447d8a07e3bd95bd0d56f35241523fbab1` | 18 |
| WBTC | `0x2f2a2543b76a4166549f7aab2e75bef0aefc5b0f` | 8 |
| ARB | `0x912ce59144191c1204e64559fe8253a0e49e6548` | 18 |

**Base (chainIndex: 8453):**

| Token | Address | Decimals |
|-------|---------|----------|
| USDC | `0x833589fcd6edb6e08f4c7c32d4f71b54bda02913` | 6 |
| USDbC (bridged) | `0xd9aaec86b65d86f6a7b5b1b0c42ffa531710b6ca` | 6 |
| WETH | `0x4200000000000000000000000000000000000006` | 18 |
| cbETH | `0x2ae3f1ec7f1f5012cfeab0185bfc7aa3cf0dec22` | 18 |

**Optimism (chainIndex: 10):**

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0x94b008aa00579c1307b0ef2c499ad98a8ce58e58` | 6 |
| USDC | `0x0b2c639c533813f4aa9d7837caf62653d097ff85` | 6 |
| WETH | `0x4200000000000000000000000000000000000006` | 18 |
| OP | `0x4200000000000000000000000000000000000042` | 18 |

**Polygon (chainIndex: 137):**

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0xc2132d05d31c914a87c6611c10748aeb04b58e8f` | 6 |
| USDC | `0x3c499c542cef5e3811e1192ce70d8cc03d5c3359` | 6 |
| WETH | `0x7ceb23fd6bc0add59e62ac25578270cff1b9f619` | 18 |
| WMATIC | `0x0d500b1d8e8ef31e21c99d1db9a6444d3adf1270` | 18 |

**BSC (chainIndex: 56):**

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0x55d398326f99059ff775485246999027b3197955` | 18 |
| USDC | `0x8ac76a51cc950d9822d68b83fe1ad97b32cd580d` | 18 |
| WBNB | `0xbb4cdb9cbd36b01bd1cbaebf2de08d9173bc095c` | 18 |
| WETH | `0x2170ed0880ac9a755fd29b2688956bd959f933f8` | 18 |

**Solana (chainIndex: 501):**

| Token | Mint Address | Decimals |
|-------|-------------|----------|
| USDT | `Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB` | 6 |
| USDC | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | 6 |
| wSOL | `So11111111111111111111111111111111111111112` | 9 |
| BONK | `DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263` | 5 |
| JUP | `JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN` | 6 |

### 16.6 Stablecoin Cross-Reference

**USDT Addresses:**

| Chain | Address | Decimals |
|-------|---------|----------|
| Ethereum | `0xdac17f958d2ee523a2206206994597c13d831ec7` | 6 |
| Arbitrum | `0xfd086bc7cd5c481dcc9c85ebe478a1c0b69fcbb9` | 6 |
| Optimism | `0x94b008aa00579c1307b0ef2c499ad98a8ce58e58` | 6 |
| Polygon | `0xc2132d05d31c914a87c6611c10748aeb04b58e8f` | 6 |
| BSC | `0x55d398326f99059ff775485246999027b3197955` | 18 |
| Solana | `Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB` | 6 |

**USDC Addresses (Native, not bridged):**

| Chain | Address | Decimals |
|-------|---------|----------|
| Ethereum | `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` | 6 |
| Arbitrum | `0xaf88d065e77c8cc2239327c5edb3a432268e5831` | 6 |
| Optimism | `0x0b2c639c533813f4aa9d7837caf62653d097ff85` | 6 |
| Base | `0x833589fcd6edb6e08f4c7c32d4f71b54bda02913` | 6 |
| Polygon | `0x3c499c542cef5e3811e1192ce70d8cc03d5c3359` | 6 |
| BSC | `0x8ac76a51cc950d9822d68b83fe1ad97b32cd580d` | 18 |
| Solana | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | 6 |

**CRITICAL: BSC USDT has 18 decimals** (not 6 like most chains). Always check decimals before encoding amounts.

### 16.7 Amount Encoding

Token amounts in API calls must be in the smallest unit (no decimals):
```
1 ETH    = 1000000000000000000  (18 decimals)
1 USDT   = 1000000              (6 decimals on most chains)
1 USDT   = 1000000000000000000  (18 decimals on BSC — WATCH OUT!)
1 BTC    = 100000000            (8 decimals for WBTC)
1 SOL    = 1000000000           (9 decimals)
```

### 16.8 Chain Selection Guide for CEX-DEX Arbitrage

| Priority | Chain | Reason |
|----------|-------|--------|
| 1 | Arbitrum | Low gas, deep liquidity, fast blocks |
| 2 | Base | Ultra-low gas, growing liquidity |
| 3 | Solana | Minimal gas, deep Jupiter liquidity |
| 4 | BSC | Low gas, decent liquidity |
| 5 | Polygon | Low gas, moderate liquidity |
| 6 | Ethereum | Only for very large trades ($100K+) where liquidity matters most |

### 16.9 Chain-Specific Gotchas

| Chain | Gotcha |
|-------|--------|
| Ethereum | High gas costs; unsuitable for small arb trades |
| Arbitrum | Gas limit values are much larger than L1 (1M-2M) but cost is low due to low gas price |
| Base | Very cheap but liquidity may be thinner for non-major pairs |
| Solana | Uses compute units + priority fees, not gas/gwei model |
| BSC | Fast blocks but more susceptible to MEV/sandwich attacks |
| Polygon | Gas spikes possible during high activity; use gas oracle |

---

## 17. GoPlus Security Reference (Inlined)

### 17.1 check_token_security Parameters

| Param | Required | Type | Description |
|-------|----------|------|-------------|
| chain_id | Yes | string | Blockchain identifier (see chain_id mapping below) |
| contract_address | Yes | string | Token contract address to audit |

### 17.2 Key Return Fields

| Field | Type | Description |
|-------|------|-------------|
| is_honeypot | string | `"0"` = safe, `"1"` = honeypot (cannot sell). **BLOCK if "1".** |
| buy_tax | string | Buy tax rate as decimal (e.g. `"0.05"` = 5%) |
| sell_tax | string | Sell tax rate as decimal (e.g. `"0.10"` = 10%) |
| is_mintable | string | `"0"` = fixed supply, `"1"` = owner can mint new tokens |
| can_take_back_ownership | string | `"0"` = safe, `"1"` = renounced ownership can be reclaimed |
| owner_change_balance | string | `"0"` = safe, `"1"` = owner can modify token balances |
| is_open_source | string | `"0"` = unverified source, `"1"` = verified/open source |
| is_proxy | string | `"0"` = not upgradeable, `"1"` = proxy/upgradeable contract |
| holder_count | string | Total number of unique token holders |
| holders | array | Top holders list with `address`, `balance`, `percent`, `is_contract` |
| slippage_modifiable | string | `"0"` = fixed, `"1"` = slippage/tax can be changed by owner |
| transfer_pausable | string | `"0"` = always transferable, `"1"` = transfers can be paused |
| cannot_sell_all | string | `"0"` = can sell all, `"1"` = cannot sell entire balance |
| personal_slippage_modifiable | string | `"0"` = no, `"1"` = can set different tax per address |

### 17.3 GoPlus Chain ID Mapping

| Chain | GoPlus chain_id |
|-------|----------------|
| Ethereum | `"1"` |
| BSC | `"56"` |
| Polygon | `"137"` |
| Arbitrum | `"42161"` |
| Base | `"8453"` |
| Avalanche | `"43114"` |
| Optimism | `"10"` |
| Solana | `"solana"` (string, not numeric) |

### 17.4 Decision Matrix

| GoPlus Field | BLOCK if | WARN if |
|-------------|----------|---------|
| is_honeypot | `=== "1"` | — |
| buy_tax | `> "0.05"` (5%) | `> "0.01"` (1%) |
| sell_tax | `> "0.10"` (10%) | `> "0.02"` (2%) |
| is_mintable | `=== "1"` AND contract < 7 days old | `=== "1"` |
| can_take_back_ownership | `=== "1"` | — |
| owner_change_balance | `=== "1"` | — |
| is_open_source | `=== "0"` | — |
| is_proxy | — | `=== "1"` |
| top 10 holders % | > 80% combined | > 50% combined |
| slippage_modifiable | `=== "1"` | — |
| transfer_pausable | — | `=== "1"` |
| cannot_sell_all | `=== "1"` | — |
| personal_slippage_modifiable | `=== "1"` | — |

### 17.5 How to Calculate Top 10 Holder Concentration

```
1. Sort holders[] by percent descending
2. Filter out known contract addresses (is_contract === 1) that are
   DEX pools or burn addresses (these are NOT concentration risk)
3. Sum the top 10 non-contract holder percentages
4. Compare against 80% (BLOCK) and 50% (WARN) thresholds
```

### 17.6 Decision Logic (Pseudocode)

```
security = check_token_security(chain_id, address)

// Hard blocks
if security.is_honeypot === "1":           return BLOCK("Honeypot detected")
if security.buy_tax > 0.05:               return BLOCK("Buy tax > 5%")
if security.sell_tax > 0.10:              return BLOCK("Sell tax > 10%")
if security.can_take_back_ownership === "1": return BLOCK("Ownership reclaimable")
if security.owner_change_balance === "1":  return BLOCK("Owner can change balances")
if security.is_open_source === "0":        return BLOCK("Unverified contract")
if security.slippage_modifiable === "1":   return BLOCK("Tax modifiable by owner")
if security.cannot_sell_all === "1":       return BLOCK("Cannot sell all tokens")
if top10HolderPct > 0.80:                 return BLOCK("Top 10 holders > 80%")

// Soft warnings
warnings = []
if security.buy_tax > 0.01:      warnings.push("Buy tax > 1%")
if security.sell_tax > 0.02:     warnings.push("Sell tax > 2%")
if security.is_mintable === "1":  warnings.push("Token is mintable")
if security.is_proxy === "1":    warnings.push("Upgradeable proxy contract")
if security.transfer_pausable === "1": warnings.push("Transfers can be paused")
if top10HolderPct > 0.50:       warnings.push("Top 10 holders > 50%")

if warnings.length > 0: return WARN(warnings)
return SAFE
```

### 17.7 GoPlus Integration Notes

- Always call before recommending any onchain token. No exceptions.
- Cache results for 5 minutes within a session.
- Solana tokens: Use `chain_id: "solana"` (string, not numeric).
- GoPlus API response wrapper: extract using `response.result["<contract_address>"]`.
- Free tier: ~100 requests/day. Implement caching aggressively.
- Fallback: If GoPlus is unreachable, surface `[WARN]` to user: "Security check unavailable — proceed with extreme caution." Never silently skip.

---

## 18. MCP Tool Reference (Inlined)

### 18.1 market_get_ticker

**Purpose:** Retrieve real-time ticker data for a single instrument.

**Parameters:**

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | — | string | Instrument ID (e.g. `"BTC-USDT"`, `"ETH-USDT-SWAP"`) |

**Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| last | string | Last traded price |
| bidPx | string | Best bid price |
| askPx | string | Best ask price |
| open24h | string | Opening price 24 hours ago |
| high24h | string | 24-hour high |
| low24h | string | 24-hour low |
| vol24h | string | 24-hour volume in base currency |
| ts | string | Data timestamp in milliseconds since epoch |

**Rate Limit:** 20 requests/second.

### 18.2 market_get_orderbook

**Purpose:** Retrieve order book depth (bids and asks) for a given instrument.

**Parameters:**

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | — | string | Instrument ID |
| sz | No | `5` | string | Book depth — number of price levels per side. Max `400`. |

**Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| asks | array | Array of ask levels, each `[price, size, num_orders]` (strings) |
| bids | array | Array of bid levels, each `[price, size, num_orders]` (strings) |
| ts | string | Snapshot timestamp in milliseconds since epoch |

**Rate Limit:** 20 requests/second.

### 18.3 market_get_candles

**Purpose:** Retrieve OHLCV candlestick data.

**Parameters:**

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | — | string | Instrument ID |
| bar | No | `"1m"` | string | Interval: `1m`, `5m`, `15m`, `30m`, `1H`, `4H`, `1D`, `1W` |
| after | No | — | string | Pagination cursor — returns data before this timestamp (ms) |
| before | No | — | string | Pagination cursor — returns data after this timestamp (ms) |
| limit | No | `100` | string | Number of candles. Max `100`. |

**Return:** Each candle is an array: `[ts, open, high, low, close, vol, volCcy, volCcyQuote, confirm]` (all strings).

**Rate Limit:** 20 requests/second.

### 18.4 system_get_capabilities

**Purpose:** Health check — verifies connectivity, auth, and available modules.
**Parameters:** None.
**Returns:** `{ authenticated, mode, modules, serverVersion, timestamp }`

### 18.5 Important MCP Notes

- **String values** — Every numeric value from OKX is a string. Always `parseFloat()` before arithmetic.
- **Timestamps** — All in milliseconds since Unix epoch.
- **Data retention** — ~3 months max.
- **Demo mode** — When `mode: "demo"`, all data is simulated but tools behave identically.

---

## 19. OnchainOS Tool Reference (Inlined)

### 19.1 dex-market price

**Purpose:** Get current price of a single token on a specific chain.

```bash
onchainos dex-market price --chain <chain> --token <address>
```

| Param | Required | Type | Description |
|-------|----------|------|-------------|
| --chain | Yes | string | Chain name (e.g. `ethereum`, `solana`, `base`) |
| --token | Yes | string | Token contract address |

**Returns:** `{ price, priceUsd, volume24h }`

### 19.2 dex-token search

**Purpose:** Search for tokens by name or symbol across chains.

```bash
onchainos dex-token search <query> [--chains <chains>]
```

| Param | Required | Type | Description |
|-------|----------|------|-------------|
| query | Yes | string | Token name or symbol (positional) |
| --chains | No | string | Comma-separated chain filter |

**Returns:** Array of `{ address, symbol, name, chain, decimals }`

### 19.3 dex-token info

**Purpose:** Get detailed metadata for a specific token.

```bash
onchainos dex-token info <address> --chain <chain>
```

**Returns:** `{ symbol, name, decimals, totalSupply, holderCount, isRiskToken, createTime, website, socialLinks }`

### 19.4 dex-token price-info

**Purpose:** Get price analytics including market cap, liquidity, and price change data.

```bash
onchainos dex-token price-info <address> --chain <chain>
```

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| marketCapUsd | string | Market capitalization in USD |
| liquidityUsd | string | Total DEX liquidity in USD |
| priceUsd | string | Current price in USD |
| priceChange24h | string | 24h price change % |
| volume24h | string | 24h volume in USD |

### 19.5 dex-swap quote

**Purpose:** Get a swap quote without executing — returns expected output, price impact, and safety flags. **Read-only, does not execute a trade.**

```bash
onchainos dex-swap quote --from <addr> --to <addr> --amount <minimal_units> --chain <chain>
```

| Param | Required | Type | Description |
|-------|----------|------|-------------|
| --from | Yes | string | Source token address |
| --to | Yes | string | Destination token address |
| --amount | Yes | string | Amount in **minimal units** (wei/lamports) |
| --chain | Yes | string | Chain name |

**Returns:**

| Field | Type | Description |
|-------|------|-------------|
| toTokenAmount | string | Expected output in minimal units |
| priceImpactPercent | string | Estimated price impact % |
| fromToken.isHoneyPot | boolean | Source token honeypot flag |
| toToken.isHoneyPot | boolean | Dest token honeypot flag |
| estimatedGas | string | Estimated gas units |

### 19.6 onchain-gateway gas

**Purpose:** Get current gas price for a chain.

```bash
onchainos onchain-gateway gas --chain <chain>
```

**Returns:** `{ gasPrice, gasPriceGwei, baseFee, priorityFee }`

### 19.7 OnchainOS Critical Notes

- **Lowercase addresses** — EVM contract addresses MUST be lowercase.
- **Minimal units** — All `--amount` values are in the token's smallest unit.
- **Read-only** — `dex-swap quote` does NOT execute a trade.
- **Rate limits** — Add 1s delay between calls if throttled.

---

## 20. Conversation Examples

### Example 1: "幫我掃描一下有冇套利機會" (Cantonese — scan with defaults)

**User:**
> 幫我掃描一下有冇套利機會

**Intent Recognition:**
- Command: `scan` (keyword: "掃描", "套利")
- Assets: not specified -> default `["BTC","ETH","SOL"]`
- Chains: not specified -> default `["ethereum"]`
- Min spread: not specified -> default `50 bps`

**Tool Calls (in order):**

```
1. system_get_capabilities
   -> { authenticated: true, mode: "demo" }

2. For BTC:
   a. market_get_ticker({ instId: "BTC-USDT" })
   b. onchainos dex-token search "WBTC" --chains ethereum
      -> { address: "0x2260fac5e5542a773aa44fbcfedf7c193bc2c599", ... }
   c. onchainos dex-market price --chain ethereum --token 0x2260fac5e5542a773aa44fbcfedf7c193bc2c599
   d. Spread = 6.3 bps -> BELOW 50 bps threshold, skip

3. For ETH:
   a. market_get_ticker({ instId: "ETH-USDT" })
   b. onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
   c. Spread = 5.0 bps -> BELOW threshold, skip

4. For SOL:
   a. Resolve wSOL on Ethereum -> Low liquidity -> skip
```

**Output:** "未發現符合條件的套利機會" with suggestions to lower threshold, try L2 chains, or set up monitoring.

---

### Example 2: "ETH 喺 Base 鏈上面同 OKX 有冇價差" (Cantonese — evaluate specific)

**User:**
> ETH 喺 Base 鏈上面同 OKX 有冇價差

**Intent Recognition:**
- Command: `evaluate` (specific asset + chain)
- Asset: ETH
- Chain: Base
- Size: default $1,000

**Tool Calls:** system_get_capabilities -> market_get_ticker(ETH-USDT) -> dex-market price (Base, native ETH) -> spread calc -> dex-token price-info (liquidity) -> dex-swap quote (price impact) -> market_get_orderbook -> onchain-gateway gas -> profitability-calculator.estimate

**Output:** Full evaluate template with cost breakdown, safety checks, risk gauge, and next steps.

---

### Example 3: "持續監控 BTC 價差，超過 30bps 提醒我" (monitor)

**Intent:** `monitor --assets BTC --threshold-bps 30`

**Output:** Monitor start confirmation, then per-alert output when threshold crossed, then completion summary with spread statistics and sparkline.

---

### Example 4: "上個月 SOL 套利機會多唔多" (backtest)

**Intent:** `backtest --asset SOL --chain solana --lookback-days 30`

**Output:** Spread statistics, opportunity frequency analysis, spread distribution histogram, time-of-day analysis, trend sparkline.

---

## 21. Implementation Notes

### Token Address Resolution

- **Native tokens** (ETH, SOL, BNB, etc.): Use the hardcoded native address from Section 16.
  - EVM chains: `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee`
  - Solana: `11111111111111111111111111111111`

- **Wrapped tokens** (WBTC on Ethereum, wSOL on Solana): Resolve via `dex-token search`. Cache for session.

- **Non-native tokens**: Always resolve via `dex-token search`, pick the result on the target chain with the highest liquidity.

### CEX instId Mapping

- Spot: `{ASSET}-USDT` (e.g., `BTC-USDT`, `ETH-USDT`, `SOL-USDT`)
- This skill only uses spot instruments.

### Amount Conversion for DEX Swap Quotes

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

### Rate Limiting

| Tool | Rate Limit | Strategy |
|------|-----------|----------|
| `market_get_ticker` | 20 req/s | Safe to batch multiple assets in parallel |
| `market_get_orderbook` | 20 req/s | Only fetch for assets with spread > min_spread_bps |
| `market_get_candles` | 20 req/s | Paginate with 100 candles per call for backtest |
| `onchainos dex-market price` | Internal OKX limits | Add 1s delay if throttled |
| `onchainos dex-swap quote` | Internal OKX limits | Only call for assets passing spread + security filters |
| GoPlus `check_token_security` | ~100/day (free tier) | Cache results for 5 minutes within session |

### Multi-Chain Scanning Strategy

When `--chains` contains multiple chains, iterate chains sequentially and parallelize within each chain.

### OKX String-to-Number Convention

All OKX API values are returned as **strings**. Always parse to `float` before any arithmetic:
```
WRONG:  "87234.5" - "87200.0" -> NaN or string concat
RIGHT:  parseFloat("87234.5") - parseFloat("87200.0") -> 34.5
```

---

## 22. Reference Files

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

> **Note:** All content from the above reference files has been inlined in this document (Sections 9, 12-19) for OpenClaw/Lark compatibility. The LLM can ONLY see this SKILL.md content and cannot read any other files.

---

## 23. Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2026-03-09 | 1.0.0 | Initial release. 4 commands: scan, evaluate, monitor, backtest. Full GoPlus + profitability-calculator integration. |
| 2026-03-09 | 1.1.0 | All external reference content inlined for OpenClaw/Lark self-contained deployment. |
