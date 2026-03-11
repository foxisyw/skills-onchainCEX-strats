---
name: stablecoin-depeg-arbitrage
description: >
  Exploits temporary stablecoin peg deviations between CEX and DEX venues. Monitors USDT, USDC,
  and DAI price deviations from $1.0000, classifies depeg type (liquidity vs solvency), evaluates
  Curve 3pool imbalance ratios, and recommends buy-cheap-sell-at-peg arbitrage when safe.
  Analysis and recommendation only — does NOT execute trades.
  Trigger phrases include: "depeg", "脫鉤", "穩定幣套利", "stablecoin arb", "USDT depeg",
  "USDC depeg", "DAI depeg", "peg deviation", "穩定幣脫錨", "Curve 3pool", "三池比例",
  "stablecoin peg", "穩定幣匯率", "stablecoin discount", "穩定幣折價", "depeg arbitrage",
  "peg recovery", "搬穩定幣", "stablecoin spread", "穩定幣價差", "USDT-USDC spread".
  Do NOT use for: trade execution, non-stablecoin CEX-DEX arbitrage (use cex-dex-arbitrage),
  funding rate arbitrage (use funding-rate-arbitrage), basis trading (use basis-trading),
  yield comparison (use yield-optimizer), pure price checks (use price-feed-aggregator).
  Requires: okx-trade-mcp (CEX spot data) + OnchainOS CLI (DEX data for Curve/Uniswap quotes).
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

# stablecoin-depeg-arbitrage

CEX-DEX stablecoin peg exploitation skill (Skill 9) for the Onchain x CEX Strats system. Monitors major stablecoins (USDT, USDC, DAI) for peg deviations, classifies the root cause of any depeg (liquidity stress vs solvency crisis), evaluates Curve 3pool ratio imbalances, and produces risk-assessed arbitrage recommendations for buying discounted stablecoins and selling at par. Integrates OKX CEX spot data, OnchainOS DEX pricing, and Curve pool analytics to identify safe peg-recovery opportunities.

**Core thesis:** When a stablecoin temporarily trades below $1.00 due to liquidity stress (NOT solvency risk), buying the discounted stablecoin and waiting for — or actively swapping to capture — peg recovery is a high-probability, low-risk trade. The key challenge is correctly distinguishing liquidity depeg from solvency depeg.

---

## 1. Role

**Stablecoin depeg arbitrage analyst** — identifies, classifies, and evaluates stablecoin peg deviation opportunities.

This skill is responsible for:
- Monitoring USDT, USDC, and DAI prices across CEX and DEX venues for peg deviations
- Classifying depeg events as LIQUIDITY (safe to arb) or SOLVENCY (do not arb)
- Monitoring Curve 3pool balance ratios to detect pool stress and imbalance signals
- Computing profitability of depeg arbitrage after all costs (CEX fees, DEX gas, slippage)
- Producing risk-assessed recommendations with clear safety labels and solvency check results
- Continuous monitoring with threshold-based alerts when depeg reaches actionable levels

This skill does **NOT**:
- Execute any trades (read-only, analysis only — never sends orders or signs transactions)
- Place orders on CEX or sign onchain transactions
- Manage positions, portfolios, or account state
- Handle non-stablecoin CEX-DEX arbitrage (delegate to `cex-dex-arbitrage`)
- Handle funding rate analysis (delegate to `funding-rate-arbitrage`)
- Handle basis/futures analysis (delegate to `basis-trading`)
- Compare DeFi yields (delegate to `yield-optimizer`)

**Key Principle:** Every depeg opportunity MUST pass the Solvency Check Procedure (Section 9.2) before any recommendation is produced. If solvency cannot be confirmed, the opportunity is BLOCKED with a clear explanation. This is non-negotiable — the downside of arbing a solvency depeg is total loss.

---

## 2. Language

Match the user's language. Default: Traditional Chinese (繁體中文).

Technical labels may remain in English regardless of language:
- `depeg`, `peg`, `spread`, `bid`, `ask`, `PnL`, `slippage`, `gas`, `gwei`, `Curve`, `3pool`
- `USDT`, `USDC`, `DAI` (always English abbreviations)
- Timestamps always displayed in UTC

Examples:
- User writes "scan for stablecoin depeg" --> respond in English
- User writes "穩定幣有冇脫鉤" --> respond in Traditional Chinese
- User writes "USDT 而家幾多錢" --> respond in Traditional Chinese
- User writes "稳定币脱锚了吗" --> respond in Simplified Chinese

---

## 3. Account Safety

| Rule | Detail |
|------|--------|
| Default mode | Demo (`okx-DEMO-simulated-trading`) |
| Mode display | Every output header shows `[DEMO]` or `[LIVE]` |
| Read-only | This skill performs **zero** write operations — no trades, no transfers, no approvals |
| Recommendation header | Always show `[RECOMMENDATION ONLY — 不會自動執行]` |
| Live switch | Requires explicit user confirmation (see Account Safety Protocol in Section 12) |

Even in `[LIVE]` mode, this skill only reads market data and produces recommendations. There is no risk of accidental execution.

---

## 4. Pre-flight (Machine-Executable Checklist)

Run these checks **in order** before any command. BLOCK at any step halts execution.

| # | Check | Command / Tool | Success Criteria | Failure Action |
|---|-------|---------------|-----------------|----------------|
| 1 | okx-trade-mcp connected | `system_get_capabilities` (DEMO or LIVE server) | `authenticated: true`, `modules` includes `"market"` | BLOCK — output `MCP_NOT_CONNECTED`. Tell user to verify `~/.okx/config.toml` and restart MCP server. |
| 2 | okx-trade-mcp mode | `system_get_capabilities` -> `mode` field | Returns `"demo"` or `"live"` matching expected mode | WARN — display actual mode in header. If user requested live but got demo, surface mismatch. |
| 3 | USDT-USDC instrument accessible | `market_get_ticker(instId: "USDT-USDC")` | Returns valid ticker with `last`, `bidPx`, `askPx` | BLOCK — output `INSTRUMENT_NOT_FOUND`. Primary stablecoin pair unavailable. |
| 4 | OnchainOS CLI installed | `which onchainos` (Bash) | Exit code 0, returns a valid path | WARN — DEX data unavailable. CEX-only mode (reduced accuracy). Label outputs `[CEX-ONLY]`. |
| 5 | OnchainOS CLI functional | `onchainos dex-market price --chain ethereum --token 0xdac17f958d2ee523a2206206994597c13d831ec7` | Returns valid JSON with `priceUsd` field | WARN — DEX data unavailable. CEX-only mode. |

### Pre-flight Decision Tree

```
Check 1 FAIL -> BLOCK (cannot proceed without CEX data)
Check 1 PASS -> Check 2
  Check 2 mismatch -> WARN + continue
  Check 2 PASS -> Check 3
    Check 3 FAIL -> BLOCK (primary instrument USDT-USDC required)
    Check 3 PASS -> Check 4
      Check 4 FAIL -> WARN: DEX unavailable, CEX-only mode
      Check 4 PASS -> Check 5
        Check 5 FAIL -> WARN: DEX unavailable, CEX-only mode
        Check 5 PASS -> ALL SYSTEMS GO
```

Unlike `cex-dex-arbitrage`, this skill CAN operate in CEX-only mode (stablecoin cross-pairs exist on OKX). However, DEX data (especially Curve pool ratios) significantly improves accuracy and safety assessment.

---

## 5. Skill Routing Matrix

| User Need | Use THIS Skill? | Delegate To |
|-----------|----------------|-------------|
| "USDT 脫鉤咗" / "USDT depeg" | Yes — `scan` | — |
| "穩定幣價差幾多" / "stablecoin spread" | Yes — `scan` | — |
| "USDC 而家安唔安全" / "is USDC safe to hold" | Yes — `evaluate` | — |
| "Curve 3pool 比例正唔正常" / "Curve 3pool ratio" | Yes — `curve-ratio` | — |
| "持續監控 USDT 價格" / "monitor USDT peg" | Yes — `monitor` | — |
| "穩定幣套利機會" / "stablecoin arb opportunity" | Yes — `evaluate` | — |
| "USDT 折價值唔值得買" / "should I buy discounted USDT" | Yes — `evaluate` | — |
| "搬穩定幣" / "stablecoin arb" | Yes — `scan` (default for ambiguous stablecoin arb intent) | — |
| "ETH 套利" / "ETH arb" | No | `cex-dex-arbitrage` |
| "資金費率" / "funding rate" | No | `funding-rate-arbitrage` |
| "基差交易" / "basis trade" | No | `basis-trading` |
| "邊度收益最好" / "best yield" | No | `yield-optimizer` |
| "BTC 而家幾錢" / "BTC price" | No | `price-feed-aggregator.snapshot` |
| "幫我買" / "execute trade" | No | Refuse — no skill executes trades |
| "呢個幣安唔安全" / "is this token safe" (non-stablecoin) | No | GoPlus MCP directly |

---

## 6. Command Index

| Command | Function | Read/Write | Description |
|---------|----------|-----------|-------------|
| `scan` | Multi-stablecoin peg deviation scan | Read | Scan USDT, USDC, DAI across CEX and DEX for peg deviations, classify depeg type, flag opportunities |
| `evaluate` | Single-stablecoin deep analysis | Read | Full depeg classification, solvency check, Curve pool analysis, cost breakdown, profitability calc |
| `monitor` | Continuous peg monitoring | Read | Poll stablecoin prices at intervals and alert when deviation exceeds threshold |
| `curve-ratio` | Curve 3pool balance analysis | Read | Fetch and analyze Curve 3pool USDT/USDC/DAI balance ratios for stress signals |

---

## 7. Parameter Reference

### 7.1 Command: `scan`

Scan major stablecoins across CEX and DEX venues for peg deviations. Returns a ranked summary of all deviations with depeg classification.

```bash
stablecoin-depeg-arbitrage scan --stablecoins USDT,USDC,DAI --min-deviation-pct 0.3 --size-usd 100000
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--stablecoins` | string[] | No | `["USDT","USDC","DAI"]` | `USDT`, `USDC`, `DAI` | Uppercase, comma-separated, max 3 items. Must be supported stablecoins. |
| `--min-deviation-pct` | number | No | `0.3` | — | Min: 0.1, Max: 10.0. Only show deviations above this percentage from $1.0000. |
| `--size-usd` | number | No | `10000` | — | Min: 1000, Max: 500,000. Trade size for cost estimation. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`, `VIP1`, `VIP2`, `VIP3`, `VIP4`, `VIP5` | OKX fee tier for cost calculation. |
| `--include-curve` | boolean | No | `true` | `true`, `false` | Include Curve 3pool ratio analysis in scan. |

#### Return Schema

```yaml
DepegScanResult:
  timestamp: integer              # Unix ms when scan was assembled
  mode: string                    # "demo" or "live"
  scan_params:
    stablecoins: string[]         # Stablecoins scanned
    min_deviation_pct: number
    size_usd: number
    vip_tier: string
  stablecoins_scanned: integer    # Total stablecoins checked
  deviations_found: integer       # Stablecoins with deviation > min_deviation_pct
  results:
    - stablecoin: string          # e.g. "USDT"
      cex_price: string           # OKX USDT-USDC last price
      dex_price: string           # DEX price in USD
      deviation_pct: number       # Absolute deviation from $1.0000
      deviation_direction: string # "below_peg" or "above_peg"
      depeg_severity: string      # "NOISE", "MONITOR", "EVALUATE", "HIGH_ALERT", "DANGER"
      depeg_type: string          # "LIQUIDITY", "SOLVENCY", "UNKNOWN", "NONE"
      solvency_check: string      # "CONFIRMED_SAFE", "UNCONFIRMED", "SOLVENCY_RISK"
      curve_pool_pct: number      # % of this coin in Curve 3pool (ideal 33.3%)
      curve_stress: string        # "NORMAL", "WARNING", "ALERT", "DANGER", "EXTREME"
      estimated_net_profit_usd: number  # After all costs at size_usd
      risk_level: string          # "[SAFE]", "[WARN]", "[BLOCK]"
      warnings: string[]          # Any applicable warnings
  curve_3pool:                    # If include-curve is true
    usdt_pct: number
    usdc_pct: number
    dai_pct: number
    total_tvl_usd: number
    stress_level: string          # "NORMAL", "WARNING", "ALERT", "DANGER", "EXTREME"
```

#### Return Fields Detail

| Field | Type | Description |
|-------|------|-------------|
| `deviation_pct` | number | `abs(price - 1.0000) * 100`. E.g., price $0.997 -> deviation 0.3% |
| `depeg_severity` | string | Thresholded: < 0.3% NOISE, 0.3-0.5% MONITOR, 0.5-1.0% EVALUATE, 1.0-5.0% HIGH_ALERT, > 5.0% DANGER |
| `depeg_type` | string | Classified via Solvency Check Procedure. LIQUIDITY = safe to arb. SOLVENCY = DO NOT ARB. |
| `solvency_check` | string | CONFIRMED_SAFE: issuer backing verified. UNCONFIRMED: cannot verify. SOLVENCY_RISK: evidence of insolvency. |
| `curve_pool_pct` | number | Percentage of this stablecoin in Curve 3pool. > 40% = stress. > 60% = danger. |
| `estimated_net_profit_usd` | number | Net profit after CEX fees, DEX gas, slippage, withdrawal fees at given size. |

---

### 7.2 Command: `evaluate`

Deep analysis of a single stablecoin depeg opportunity. Performs full solvency check, Curve pool analysis, multi-venue price comparison, cost breakdown, and profitability calculation.

```bash
stablecoin-depeg-arbitrage evaluate --stablecoin USDT --size-usd 100000 --chain ethereum
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--stablecoin` | string | Yes | — | `USDT`, `USDC`, `DAI` | Uppercase, single stablecoin |
| `--size-usd` | number | No | `10000` | — | Min: 1000, Max: 500,000. Trade size for analysis. |
| `--chain` | string | No | `"ethereum"` | `ethereum`, `arbitrum`, `base` | Chain for DEX leg. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0` through `VIP5` | OKX fee tier. |
| `--target-stablecoin` | string | No | auto-select | `USDT`, `USDC`, `DAI` | The stablecoin to swap into. Auto-selects the one closest to $1.0000. |

#### Return Schema

```yaml
DepegEvaluateResult:
  timestamp: integer
  mode: string
  stablecoin: string
  target_stablecoin: string       # What to swap the cheap stablecoin into
  chain: string
  size_usd: number
  prices:
    cex:
      usdt_usdc: string           # USDT-USDC pair price
      usdc_usdt: string           # USDC-USDT pair price
      source: string              # "market_get_ticker"
      data_age_ms: integer
    dex:
      price: string               # DEX price
      source: string              # "dex-market price"
      data_age_ms: integer
    curve:
      implied_rate: string        # Curve pool implied exchange rate
      pool_balance_pct: number[]  # [USDT%, USDC%, DAI%]
  deviation:
    pct: number                   # Deviation from $1.0000
    direction: string             # "below_peg" or "above_peg"
    severity: string              # "NOISE" through "DANGER"
  solvency_assessment:
    depeg_type: string            # "LIQUIDITY" or "SOLVENCY" or "UNKNOWN"
    confidence: string            # "HIGH", "MEDIUM", "LOW"
    evidence: string[]            # List of evidence points
    issuer_status: string         # "BACKED", "UNVERIFIED", "RISK"
    cross_cex_check: string       # "CONSISTENT" or "DIVERGENT" (systemic check)
    multi_stablecoin_depeg: boolean # True if multiple stablecoins depegging (systemic crisis)
    recommendation: string        # "SAFE_TO_ARB", "CAUTION", "DO_NOT_ARB"
  profitability:
    gross_spread_usd: number
    cost_breakdown:
      cex_trading_fee: number
      dex_gas_cost: number
      dex_slippage_cost: number
      withdrawal_fee: number
    total_costs_usd: number
    net_profit_usd: number
    profit_to_cost_ratio: number
    return_pct: number            # net_profit / size_usd * 100
    is_profitable: boolean
  execution_plan:
    direction: string             # "Buy CEX -> Swap DEX" or "Buy DEX -> Sell CEX"
    buy_venue: string
    buy_price: string
    sell_venue: string
    sell_price: string
    steps: string[]               # Ordered execution steps
  risk:
    overall_score: integer        # 1-10
    solvency_risk: integer        # 1-10
    execution_risk: integer       # 1-10
    liquidity_risk: integer       # 1-10
    systemic_risk: integer        # 1-10
  recommended_action: string      # "PROCEED", "CAUTION", "DO NOT TRADE", "BLOCK"
  next_steps: string[]
```

---

### 7.3 Command: `monitor`

Continuously poll stablecoin prices across CEX and DEX and alert when deviation exceeds threshold.

```bash
stablecoin-depeg-arbitrage monitor --stablecoins USDT,USDC --threshold-pct 0.3 --duration-min 60 --check-interval-sec 30
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--stablecoins` | string[] | No | `["USDT","USDC","DAI"]` | `USDT`, `USDC`, `DAI` | Uppercase, comma-separated |
| `--threshold-pct` | number | No | `0.3` | — | Min: 0.1, Max: 5.0. Alert when deviation exceeds this %. |
| `--duration-min` | number | No | `60` | — | Min: 5, Max: 1440 (24h). Monitoring duration. |
| `--check-interval-sec` | number | No | `30` | — | Min: 10, Max: 300. Poll interval. |
| `--size-usd` | number | No | `10000` | — | Min: 1000, Max: 500,000. For profit estimation on alerts. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0` through `VIP5` | OKX fee tier. |
| `--include-curve` | boolean | No | `true` | `true`, `false` | Include Curve 3pool ratio in each check. |

#### Return Schema (per alert)

```yaml
DepegMonitorAlert:
  alert_type: string              # "DEPEG_THRESHOLD_CROSSED" or "CURVE_STRESS_DETECTED"
  timestamp: integer
  stablecoin: string
  current_price: string
  deviation_pct: number
  threshold_pct: number
  depeg_severity: string
  curve_pool_pct: number          # If include-curve
  curve_stress: string            # If include-curve
  estimated_net_profit: number    # Quick estimate at configured size_usd
  previous_deviation_pct: number
  deviation_change_pct: number
  checks_completed: integer
  checks_remaining: integer
  suggested_action: string        # e.g. "stablecoin-depeg-arbitrage evaluate --stablecoin USDT --size-usd 100000"
```

#### Monitor Summary Schema (on completion)

```yaml
DepegMonitorSummary:
  stablecoins: string[]
  duration_min: number
  total_checks: integer
  alerts_triggered: integer
  per_stablecoin:
    - stablecoin: string
      avg_deviation_pct: number
      max_deviation_pct: number
      min_deviation_pct: number
      std_dev_pct: number
      time_above_threshold_pct: number  # % of checks where deviation > threshold
      trend: string               # "depegging", "recovering", "stable", "volatile"
      sparkline: string           # e.g. "▁▁▂▃▅▇▅▃▂▁"
  curve_3pool_summary:
    avg_balance_pcts: number[]    # Average [USDT%, USDC%, DAI%] during period
    max_imbalance_pct: number     # Maximum single-coin deviation from 33.3%
```

---

### 7.4 Command: `curve-ratio`

Fetch and analyze the current Curve 3pool (USDT/USDC/DAI) balance ratios. Useful as a standalone stress indicator even without active depeg.

```bash
stablecoin-depeg-arbitrage curve-ratio
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--include-history` | boolean | No | `false` | `true`, `false` | Show historical balance ratio trend (7 days). |

#### Return Schema

```yaml
CurveRatioResult:
  timestamp: integer
  pool_name: string               # "Curve 3pool"
  pool_address: string            # Ethereum mainnet 3pool
  total_tvl_usd: number
  balances:
    - coin: string                # "USDT", "USDC", "DAI"
      balance_usd: number
      balance_pct: number         # e.g. 35.2
      deviation_from_ideal: number # e.g. +1.9 (ideal is 33.3%)
      stress_level: string        # "NORMAL", "WARNING", "ALERT", "DANGER", "EXTREME"
  overall_stress: string          # Max stress across all coins
  interpretation: string          # Human-readable summary
  history:                        # If include-history
    - date: string
      usdt_pct: number
      usdc_pct: number
      dai_pct: number
```

---

## 8. Operation Flow

### Step 1: Intent Recognition

Parse user message to extract command, parameters, and context.

| Element | Extraction Logic | Fallback |
|---------|-----------------|----------|
| Command | Map to `scan` / `evaluate` / `monitor` / `curve-ratio` based on keywords | Default: `scan` |
| Stablecoins | Extract stablecoin names: "USDT", "USDC", "DAI" | Default: `["USDT","USDC","DAI"]` |
| Size | Look for USD amounts: "$100,000", "10萬", "100K" | Default: `10000` |
| Min deviation | Look for percentage: "0.5%", "50 bps" | Default: `0.3` |
| Chain | Look for chain names: "Ethereum", "Arbitrum", "Base" | Default: `"ethereum"` |
| VIP tier | Look for "VIP1", "VIP2" etc. | Default: `"VIP0"` |

**Keyword-to-command mapping:**

| Keywords | Command |
|----------|---------|
| "掃描", "scan", "脫鉤", "depeg", "穩定幣", "stablecoin", "peg check", "掃一掃" | `scan` |
| "評估", "evaluate", "analyze", "分析", "值唔值得", "should I buy", "safe to arb" | `evaluate` |
| "監控", "monitor", "watch", "alert", "通知", "持續", "keep an eye", "睇住" | `monitor` |
| "Curve", "3pool", "三池", "pool ratio", "池子比例", "balance ratio" | `curve-ratio` |

**Ambiguous intent resolution:**

| Input Pattern | Resolved Command | Reasoning |
|---------------|-----------------|-----------|
| "穩定幣有冇脫鉤" (generic) | `scan` | Broad depeg check -> scan all stablecoins |
| "USDT 脫鉤咗 0.5%" | `evaluate --stablecoin USDT` | Specific stablecoin -> evaluate |
| "USDT 而家買得過嗎" | `evaluate --stablecoin USDT` | Purchase intent for specific coin -> evaluate |
| "幫我睇住 USDT" | `monitor --stablecoins USDT` | "睇住" = monitoring intent |
| "Curve 3pool 幾多" | `curve-ratio` | Explicit Curve mention -> curve-ratio |
| "搬穩定幣" | `scan` | Generic stablecoin arb -> scan first |

---

### Step 2: Peg Deviation Detection

For each stablecoin, collect prices from multiple venues.

#### 2a. CEX Price Collection

```
For each stablecoin:

  1. market_get_ticker(instId: "USDT-USDC")
     -> Extract: last, bidPx, askPx, ts
     -> USDT price in USDC terms (if < 1.0000, USDT is below peg relative to USDC)

  2. market_get_ticker(instId: "USDC-USDT")
     -> Extract: last, bidPx, askPx, ts
     -> USDC price in USDT terms (cross-check)

  Derive individual prices:
    usdt_price = USDT-USDC last price     # e.g. 0.9970 means USDT at $0.997
    usdc_price = 1 / USDC-USDT last price # or direct if pair exists
    dai_price  = derive from USDT-USDC + DAI-USDT if available
```

#### 2b. DEX Price Collection

```
For each stablecoin:

  onchainos dex-market price --chain ethereum --token {contract_address}
  -> Extract: priceUsd

  Contract addresses (Ethereum):
    USDT: 0xdac17f958d2ee523a2206206994597c13d831ec7
    USDC: 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48
    DAI:  0x6b175474e89094c44da98b954eedeac495271d0f
```

#### 2c. Deviation Calculation

```
For each stablecoin:

  deviation_pct = abs(price - 1.0000) * 100
  direction = "below_peg" if price < 1.0000 else "above_peg"

  Classification:
    deviation < 0.3%:   severity = "NOISE"       -> action: IGNORE
    0.3% <= dev < 0.5%: severity = "MONITOR"     -> action: MONITOR (opportunity forming)
    0.5% <= dev < 1.0%: severity = "EVALUATE"    -> action: EVALUATE (likely profitable)
    1.0% <= dev < 5.0%: severity = "HIGH_ALERT"  -> action: HIGH ALERT (strong opportunity)
    dev >= 5.0%:        severity = "DANGER"       -> action: DANGER (solvency risk — investigate FIRST)
```

**Worked Example: USDT at $0.997**
```
price = 0.997
deviation_pct = abs(0.997 - 1.0000) * 100 = 0.3%
direction = "below_peg"
severity = "MONITOR"
```

**Worked Example: USDC at $0.985**
```
price = 0.985
deviation_pct = abs(0.985 - 1.0000) * 100 = 1.5%
direction = "below_peg"
severity = "HIGH_ALERT"
```

---

### Step 3: Depeg Type Classification (CRITICAL SAFETY STEP)

**This is the most important step in the entire skill.** Misclassifying a solvency depeg as a liquidity depeg can result in total loss of capital.

#### 3a. Classification Criteria

```
LIQUIDITY DEPEG (safe to arbitrage):
  ✓ Issuer has confirmed full backing (proof-of-reserves available)
  ✓ Depeg caused by temporary sell pressure (large redemptions, market panic)
  ✓ Curve pool rebalancing over time (imbalance increasing but reversible)
  ✓ No smart contract vulnerabilities reported
  ✓ Other CEXes show similar mild depeg (market-wide, not exchange-specific)

  Historical examples:
    USDC March 2023 (SVB panic): dropped to $0.87, recovered to $1.00 in 3 days
    USDT June 2022 (market panic): dropped to $0.95, recovered in hours
    DAI March 2020 (Black Thursday): dropped to $0.96, recovered in days

SOLVENCY DEPEG (DO NOT ARBITRAGE — EVER):
  ✗ Issuer insolvency or hack announced
  ✗ Protocol exploit detected (smart contract vulnerability)
  ✗ Backing assets frozen or seized (legal action)
  ✗ Depeg > 5% with no clear liquidity explanation
  ✗ Multiple stablecoins depegging simultaneously (systemic crisis)

  Historical examples:
    UST May 2022: algorithmic stablecoin collapse, dropped from $1.00 to $0.10 to ~$0.00
    HUSD: issuer insolvency, dropped and never recovered
```

#### 3b. Solvency Check Decision Tree

```
START — Is the depeg > 5%?
  |
  YES -> PRESUME SOLVENCY RISK. Investigate cause before proceeding.
  |      Check news, issuer statements, protocol status.
  |      If cause is CONFIRMED liquidity: downgrade to CAUTION, allow with warnings.
  |      If cause UNKNOWN: BLOCK — "Cannot confirm safety. Wait for clarity."
  |      If solvency issue detected: BLOCK — "Solvency risk confirmed. DO NOT ARB."
  |
  NO -> Continue to sub-checks:
    |
    +- Check 1: Issuer status
    |   Is the issuer (Tether/Circle/MakerDAO) operational?
    |   Has any hack, exploit, or freeze been reported?
    |   YES (hack/freeze) -> BLOCK: SOLVENCY_RISK
    |   NO -> Continue
    |
    +- Check 2: Multi-stablecoin check
    |   Are OTHER stablecoins ALSO depegging?
    |   YES (systemic) -> BLOCK: SYSTEMIC_RISK — "Multiple stablecoins depegging = systemic crisis"
    |   NO -> Continue
    |
    +- Check 3: Smart contract check
    |   Any reported exploits on the stablecoin contract?
    |   YES -> BLOCK: CONTRACT_EXPLOITED
    |   NO -> Continue
    |
    +- Check 4: Cross-exchange check
    |   Is the depeg visible on multiple CEXes, or only one?
    |   Only one CEX -> WARN: might be exchange-specific issue (withdrawal freeze, etc.)
    |   Multiple CEXes -> Continue (market-wide = more likely liquidity depeg)
    |
    +- Check 5: Curve 3pool ratio
    |   Is the depegged coin's pool % > 60%?
    |   YES -> WARN: Significant pool stress. May take longer to recover.
    |   > 70% -> DANGER: Extreme stress. Exercise extreme caution.
    |   NO -> Continue
    |
    ALL CHECKS PASSED -> Classify as LIQUIDITY DEPEG
    -> solvency_check = "CONFIRMED_SAFE"
    -> depeg_type = "LIQUIDITY"
    -> recommendation = "SAFE_TO_ARB" (with standard caveats)
```

---

### Step 4: Curve 3Pool Ratio Analysis

```
Query: DeFiLlama /pools endpoint for Curve 3pool
  Returns: coins[], balances[], totalSupply
  Calculate: pct[i] = balance[i] / totalSupply * 100

Ideal balance: [33.3%, 33.3%, 33.3%] USDT/USDC/DAI

Stress thresholds (per coin):
  30-36%:    NORMAL — within expected range
  36-40%:    WARNING — mild stress, one coin being sold into pool
  40-50%:    ALERT — significant depeg pressure
  50-60%:    DANGER — strong depeg, large outflows of other coins
  60-70%:    EXTREME — severe stress (happened in USDT 2023 depeg)
  > 70%:     CRITICAL — unprecedented, do NOT assume recovery

Interpretation:
  When a coin's % is HIGH: that coin is being SOLD into the pool (people want to get rid of it)
  -> That coin is likely below peg
  When a coin's % is LOW: that coin is being BOUGHT from the pool
  -> That coin is likely above peg (or the other coins are below peg)
```

**Worked Example: USDT Stress**
```
Balances: USDT 55%, USDC 25%, DAI 20%
Total TVL: $2.1B

USDT at 55%: ALERT — significant sell pressure on USDT
  Deviation from ideal: +21.7% (55% - 33.3%)
  Interpretation: Market participants are dumping USDT for USDC/DAI
  Signal: USDT likely below peg

USDC at 25%: Low — being bought
  Deviation: -8.3% (25% - 33.3%)
  Interpretation: Flight to quality — USDC perceived as safer

DAI at 20%: Low — being bought
  Deviation: -13.3% (20% - 33.3%)
```

---

### Step 5: Profitability Calculation

```
Given:
  cheap_stablecoin_price = 0.997    # e.g. USDT
  target_stablecoin_price = 1.0000  # e.g. USDC
  size_usd = 100,000
  vip_tier = VIP0

Step 5a: Calculate gross spread
  gross_spread = abs(cheap_price - target_price) / min(cheap_price, target_price)
  gross_spread = abs(0.997 - 1.0000) / 0.997 = 0.003009 = 0.3009%
  gross_spread_usd = size_usd * gross_spread = 100,000 * 0.003009 = $300.90

Step 5b: Calculate all costs

  Direction A: Buy USDT cheap on OKX CEX, transfer to DEX, swap for USDC on Curve
    cex_trading_fee = size_usd * taker_fee_rate
                    = 100,000 * 0.0008 (VIP0 taker)
                    = $80.00

    withdrawal_fee (USDT to Ethereum ERC-20) = 3.0 USDT = $3.00

    dex_gas_cost (Ethereum Curve swap) = ~$15.00 (typical at 30 gwei)

    dex_slippage (Curve stablecoin swap at $100K)
      Curve 3pool stablecoin swaps have very low slippage:
      ~0.01-0.05% for $100K on a healthy pool
      slippage_cost = 100,000 * 0.0003 = $30.00 (conservative)

    Curve protocol fee = 0.04% of swap = 100,000 * 0.0004 = $40.00

    total_costs = 80.00 + 3.00 + 15.00 + 30.00 + 40.00 = $168.00

  Direction B: Buy USDT cheap on OKX CEX, swap to USDC directly on OKX (if USDT-USDC pair)
    cex_trading_fee (buy USDT, instant, using USDC-USDT pair) = size_usd * taker_fee_rate
                    = 100,000 * 0.0008 = $80.00
    withdrawal_fee = $0 (no transfer needed, stay on CEX)
    dex_gas = $0 (no onchain tx)
    slippage (CEX orderbook) = ~$10.00

    total_costs = 80.00 + 0.00 + 0.00 + 10.00 = $90.00

Step 5c: Net profit
  Direction A (CEX buy -> DEX swap):
    net_profit = gross_spread_usd - total_costs = 300.90 - 168.00 = +$132.90
    return_pct = 132.90 / 100,000 * 100 = 0.133%
    profit_to_cost = 132.90 / 168.00 = 0.79x

  Direction B (CEX-only, USDT-USDC pair):
    net_profit = gross_spread_usd - total_costs = 300.90 - 90.00 = +$210.90
    return_pct = 210.90 / 100,000 * 100 = 0.211%
    profit_to_cost = 210.90 / 90.00 = 2.34x

  -> Direction B is superior for this size. CEX-only execution wins when:
     a) OKX has the direct stablecoin pair (USDT-USDC exists on OKX)
     b) Spread on OKX orderbook is close to DEX spread
     c) Savings on gas + withdrawal + Curve fee > CEX slippage difference
```

---

### Step 6: Output & Recommend

Format using output templates (see Section 13). The output structure varies by command:

#### For `scan`:
1. Global header (skill name, mode, timestamp, data sources)
2. Peg deviation summary table (all scanned stablecoins)
3. Depeg classification for each deviating stablecoin
4. Curve 3pool ratio snapshot (if included)
5. Opportunities with profitability estimate
6. Risk gauge for highest-deviation stablecoin
7. Next steps suggestions
8. Disclaimer

#### For `evaluate`:
1. Global header
2. Price comparison (CEX vs DEX vs Curve implied rate)
3. Deviation analysis (percentage, severity, direction)
4. Solvency assessment (full classification with evidence)
5. Curve 3pool analysis
6. Cost breakdown table (both directions if applicable)
7. Net profit summary
8. Risk gauge (4-dimension breakdown)
9. Execution plan (step-by-step)
10. Recommended action
11. Next steps
12. Disclaimer

#### For `monitor`:
1. Monitor start confirmation (parameters, schedule)
2. Per-alert output when deviation threshold crossed
3. Monitor completion summary with statistics and sparklines

#### For `curve-ratio`:
1. Global header
2. Pool balance table with stress levels
3. Historical trend (if requested)
4. Interpretation summary
5. Disclaimer

**Suggested follow-up actions (vary by result):**

| Result | Suggested Actions |
|--------|-------------------|
| Scan: no deviation | "穩定幣價格正常。設定監控等待脫鉤事件 -> `stablecoin-depeg-arbitrage monitor`" |
| Scan: deviation found | "詳細評估 -> `stablecoin-depeg-arbitrage evaluate --stablecoin {COIN}`" |
| Evaluate: profitable + safe | "如滿意，手動在 OKX 執行"; "檢查不同規模 -> 調整 `--size-usd`" |
| Evaluate: solvency risk | "DO NOT TRADE. Wait for issuer statement." |
| Evaluate: not profitable | "價差不足以覆蓋成本。設定監控等更大脫鉤 -> `monitor`" |
| Monitor: alert triggered | "立即評估 -> `stablecoin-depeg-arbitrage evaluate --stablecoin {COIN}`" |
| Curve-ratio: stress detected | "進一步調查 -> `stablecoin-depeg-arbitrage evaluate --stablecoin {STRESSED_COIN}`" |

---

## 9. Safety Checks (Per Operation)

### 9.1 Pre-Trade Safety Checklist

Every command runs through these checks **in order** before producing a recommendation. A BLOCK at any step halts the pipeline immediately.

| # | Check | Tool | BLOCK Threshold | WARN Threshold | Error Code |
|---|-------|------|----------------|----------------|------------|
| 1 | MCP connectivity | `system_get_capabilities` | Server not reachable | — | `MCP_NOT_CONNECTED` |
| 2 | Authentication | `system_get_capabilities` | `authenticated: false` | — | `AUTH_FAILED` |
| 3 | Data freshness | Internal timestamp comparison | > 10s stale | > 5s stale | `DATA_STALE` |
| 4 | Solvency check: issuer status | News/announcement check | Hack/exploit/insolvency reported | Unverified backing | `SOLVENCY_RISK` |
| 5 | Solvency check: multi-stablecoin | Cross-check all 3 stablecoins | Multiple depegging simultaneously | — | `SYSTEMIC_RISK` |
| 6 | Solvency check: depeg magnitude | Price deviation calculation | > 5% without confirmed cause | > 3% with unverified cause | `DEPEG_DANGER` |
| 7 | Curve 3pool stress | DeFiLlama pool query | One coin > 70% of pool | One coin > 50% of pool | `CURVE_EXTREME_STRESS` |
| 8 | Liquidity depth (DEX) | OnchainOS `dex-token price-info` | `liquidityUsd < $1,000,000` | `liquidityUsd < $10,000,000` | `INSUFFICIENT_LIQUIDITY` |
| 9 | Price impact | OnchainOS `dex-swap quote` | `priceImpactPercent > 0.5%` | `priceImpactPercent > 0.2%` | `PRICE_IMPACT_HIGH` |
| 10 | Net profitability | Internal calculation | `net_profit <= 0` | `profit_to_cost_ratio < 1.5` | `NOT_PROFITABLE` |

### 9.2 Solvency Check Procedure (MANDATORY)

**Before ANY depeg trade recommendation, check ALL of these:**

```
Step 1: News search
  Query: "{stablecoin} hack", "{stablecoin} insolvency", "{stablecoin} exploit"
  Source: Available news/search tools, or prompt user to verify
  If hack/exploit/insolvency found -> BLOCK: SOLVENCY_RISK

Step 2: Issuer proof-of-reserves
  USDT (Tether): Check tether.to/en/transparency for reserve attestation
  USDC (Circle): Check circle.com/en/transparency for reserve breakdown
  DAI (MakerDAO): Check daistats.com for collateral ratio (must be > 100%)
  If proof-of-reserves unavailable or outdated (> 7 days) -> WARN: UNVERIFIED_BACKING

Step 3: Cross-exchange check
  Check if depeg is visible on OTHER major CEXes (Binance, Coinbase, Bybit)
  If yes (multiple CEXes show depeg) -> likely market-wide (could be liquidity OR solvency)
  If only one CEX -> WARN: exchange-specific issue (possible withdrawal freeze)

Step 4: Multi-stablecoin check
  Are MULTIPLE stablecoins (USDT + USDC, or USDT + DAI, etc.) depegging simultaneously?
  If yes -> BLOCK: SYSTEMIC_RISK — "Multiple stablecoins depegging = systemic crypto crisis.
           This is NOT a standard arbitrage opportunity. DO NOT TRADE."

Step 5: Magnitude check
  If depeg > 5% AND cause unknown -> BLOCK: "Depeg exceeds 5% without confirmed cause.
  Assume solvency risk until proven otherwise. Wait for issuer statement."

BLOCK CONDITIONS SUMMARY:
  - Issuer hack/exploit reported                    -> BLOCK
  - Smart contract vulnerability found              -> BLOCK
  - Depeg > 5% without clear confirmed cause        -> BLOCK
  - Multiple stablecoins depegging simultaneously   -> BLOCK
  - Issuer insolvency announced                     -> BLOCK
  - Backing assets frozen by regulators             -> BLOCK
```

### 9.3 Check Execution Flow

```
START
  |
  +- Check 1-2: Infrastructure    -- BLOCK? -> Output error, STOP
  |
  +- Check 3: Freshness           -- BLOCK? -> Refetch, retry once -> BLOCK? -> STOP
  |
  +- Check 4-6: Solvency          -- BLOCK? -> Output SOLVENCY_RISK/SYSTEMIC_RISK, STOP
  |                                   WARN?  -> Attach solvency warning labels, CONTINUE
  |
  +- Check 7: Curve stress         -- BLOCK? -> Output CURVE_EXTREME_STRESS, STOP
  |                                   WARN?  -> Attach pool stress warning, CONTINUE
  |
  +- Check 8-9: Liquidity         -- BLOCK? -> Output INSUFFICIENT_LIQUIDITY, STOP
  |                                   WARN?  -> Attach liquidity note, CONTINUE
  |
  +- Check 10: Profitability      -- BLOCK? -> Output NOT_PROFITABLE, STOP
  |                                   WARN?  -> Include with thin-margin warning
  |
  +- ALL PASSED -> Produce recommendation with accumulated WARNs
```

### 9.4 Stablecoin Depeg Risk Limits

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| max_trade_size | $50,000 | $500,000 | Maximum USD per depeg arb trade |
| min_deviation_pct | 0.3% | — | Minimum deviation to consider (below = noise) |
| max_safe_deviation_pct | 5.0% | — | Above this, presume solvency risk |
| min_liquidity_usd | $1,000,000 | — | Minimum DEX pool liquidity |
| max_slippage_pct | 0.5% | 1.0% | Maximum acceptable slippage for stablecoin swaps |
| max_price_age_sec | 10 | 10 | Maximum age of price data (relaxed vs arb mode) |
| curve_danger_pct | 60% | — | Curve pool % above which = DANGER |
| curve_extreme_pct | 70% | — | Curve pool % above which = EXTREME/BLOCK |

---

## 10. Key Formulas (Inlined)

### 10.1 Peg Deviation

```
deviation_pct = abs(price - 1.0000) * 100
```

| Variable | Definition |
|----------|-----------|
| `price` | Current price of stablecoin in USD (or USDC terms) |
| `deviation_pct` | Unsigned percentage deviation from $1.0000 |

**Worked Example**
```
USDT price = $0.9950 (on OKX, USDT-USDC pair)

deviation_pct = abs(0.9950 - 1.0000) * 100
              = 0.0050 * 100
              = 0.50%

Severity: EVALUATE (0.5% threshold)
Direction: below_peg
```

### 10.2 Gross Spread (Stablecoin Pair)

```
gross_spread_pct = abs(cheap_price - target_price) / min(cheap_price, target_price) * 100
gross_spread_usd = size_usd * gross_spread_pct / 100
```

**Worked Example**
```
cheap_price (USDT) = 0.9950
target_price (USDC) = 1.0000
size_usd = 100,000

gross_spread_pct = abs(0.9950 - 1.0000) / 0.9950 * 100
                 = 0.005025 * 100 = 0.5025%

gross_spread_usd = 100,000 * 0.5025 / 100 = $502.50
```

### 10.3 CEX-Only Depeg Arb Net Profit

```
net_profit = gross_spread_usd - cex_trading_fee - cex_slippage
return_pct = net_profit / size_usd * 100
```

**Worked Example (CEX-Only, USDT-USDC pair on OKX)**
```
size_usd = 100,000
gross_spread_usd = $502.50  (from 10.2)

cex_trading_fee = 100,000 * 0.0008 (VIP0 taker) = $80.00
cex_slippage = $10.00 (estimated from orderbook depth at $100K)

net_profit = 502.50 - 80.00 - 10.00 = +$412.50
return_pct = 412.50 / 100,000 * 100 = 0.413%
profit_to_cost = 412.50 / 90.00 = 4.58x

Verdict: PROFITABLE ✓
```

### 10.4 CEX-to-DEX Depeg Arb Net Profit

```
net_profit = gross_spread_usd - cex_fee - withdrawal_fee - dex_gas - dex_slippage - curve_fee
return_pct = net_profit / size_usd * 100
```

**Worked Example (Buy USDT on OKX -> Swap to USDC on Curve via Ethereum)**
```
size_usd = 100,000
gross_spread_usd = $502.50

cex_fee = 100,000 * 0.0008 = $80.00
withdrawal_fee (USDT ERC-20) = $3.00
dex_gas (Ethereum, 30 gwei, Curve swap ~150K gas) = $15.00
dex_slippage (Curve, $100K stablecoin) = $30.00
curve_fee (0.04%) = 100,000 * 0.0004 = $40.00

total_costs = 80.00 + 3.00 + 15.00 + 30.00 + 40.00 = $168.00
net_profit = 502.50 - 168.00 = +$334.50
return_pct = 334.50 / 100,000 * 100 = 0.335%
profit_to_cost = 334.50 / 168.00 = 1.99x

Verdict: PROFITABLE ✓ (but CEX-only route is better if orderbook has depth)
```

### 10.5 CEX-to-DEX via L2 Net Profit

**Worked Example (Buy USDT on OKX -> Swap to USDC on Curve via Arbitrum)**
```
size_usd = 100,000
gross_spread_usd = $502.50

cex_fee = 100,000 * 0.0008 = $80.00
withdrawal_fee (USDT to Arbitrum) = $0.10
dex_gas (Arbitrum, ~0.2 gwei) = $0.30
dex_slippage (Curve Arbitrum, $100K) = $20.00
curve_fee (0.04%) = $40.00

total_costs = 80.00 + 0.10 + 0.30 + 20.00 + 40.00 = $140.40
net_profit = 502.50 - 140.40 = +$362.10
return_pct = 362.10 / 100,000 * 100 = 0.362%
profit_to_cost = 362.10 / 140.40 = 2.58x

Verdict: PROFITABLE ✓ (L2 route saves ~$28 vs Ethereum mainnet)
```

### 10.6 Curve 3Pool Ratio Deviation

```
deviation_from_ideal = pool_pct - 33.333
```

**Worked Example**
```
Pool balances: USDT 55.0%, USDC 25.0%, DAI 20.0%

USDT deviation = 55.0 - 33.333 = +21.667% (OVERWEIGHT — being dumped)
USDC deviation = 25.0 - 33.333 = -8.333%  (UNDERWEIGHT — being bought)
DAI deviation  = 20.0 - 33.333 = -13.333% (UNDERWEIGHT — being bought)

USDT stress level: ALERT (> 40%)
USDC stress level: NORMAL (within 30-36%)
DAI stress level: NORMAL (within 30-36%, though 20% is actually below range — coin is in high demand)
```

### 10.7 Minimum Deviation for Breakeven

```
min_deviation_pct = total_costs / size_usd * 100
```

**Worked Example (CEX-Only)**
```
total_costs = $90.00 (from 10.3 example)
size_usd = 100,000

min_deviation_pct = 90.00 / 100,000 * 100 = 0.090%

Any deviation above 0.09% from peg is profitable via CEX-only route at $100K.
```

**Worked Example (CEX-to-DEX via Ethereum)**
```
total_costs = $168.00
size_usd = 100,000

min_deviation_pct = 168.00 / 100,000 * 100 = 0.168%
```

### 10.8 Profit-to-Cost Ratio

```
profit_to_cost = net_profit / total_costs
```

**Interpretation:**
- < 1.0x: Costs exceed profit — do not trade
- 1.0x - 1.5x: Marginal — high execution risk
- 1.5x - 3.0x: Acceptable — proceed with caution
- 3.0x+: Attractive — high confidence trade

### 10.9 Signal Decay for Stablecoin Depeg

Stablecoin depeg signals decay SLOWER than regular arb signals because peg recovery often takes hours to days, not minutes.

```
signal_strength = initial_strength * exp(-decay_rate * time_elapsed_hours)
```

**Typical Decay Rates (Stablecoin Depeg):**

| Signal Type | Decay Rate (per hour) | Half-Life |
|------------|----------------------|-----------|
| Stablecoin depeg < 1% | 0.10 | ~6.9 hours |
| Stablecoin depeg 1-5% | 0.05 | ~13.9 hours |
| Stablecoin depeg > 5% | 0.02 | ~34.7 hours |
| Curve pool imbalance | 0.03 | ~23.1 hours |

**Worked Example**
```
initial_strength = 0.95  (strong depeg signal, USDT at $0.985)
decay_rate = 0.05 (1-5% depeg range)
time_elapsed_hours = 6

signal_strength = 0.95 * exp(-0.05 * 6)
               = 0.95 * 0.7408
               = 0.704

Signal still viable after 6 hours (stablecoin depegs persist longer than token arb spreads).
```

---

## 11. Fee Schedule (Inlined)

### 11.1 OKX Spot Trading Fees

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
- For depeg arb cost estimation, assume taker fees (speed matters during depeg events)
- Stablecoin pairs (USDT-USDC) use standard spot fee schedule

### 11.2 OKX Withdrawal Fees (Stablecoin-Relevant)

**USDT Withdrawals:**

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Ethereum (ERC-20) | 3.0 USDT | 10.0 USDT |
| Tron (TRC-20) | 1.0 USDT | 0.1 USDT |
| Arbitrum One | 0.1 USDT | 0.1 USDT |
| Optimism | 0.1 USDT | 0.1 USDT |
| Base | 0.1 USDT | 0.1 USDT |
| Solana | 1.0 USDT | 1.0 USDT |
| BSC (BEP-20) | 0.3 USDT | 10.0 USDT |
| Polygon | 0.8 USDT | 0.1 USDT |

**USDC Withdrawals:**

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Ethereum (ERC-20) | 3.0 USDC | 10.0 USDC |
| Arbitrum One | 0.1 USDC | 0.1 USDC |
| Optimism | 0.1 USDC | 0.1 USDC |
| Base | 0.1 USDC | 0.1 USDC |
| Solana | 1.0 USDC | 1.0 USDC |
| BSC (BEP-20) | 0.3 USDC | 10.0 USDC |

**DAI Withdrawals:**

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Ethereum (ERC-20) | 5.0 DAI | 10.0 DAI |
| Arbitrum One | 0.5 DAI | 1.0 DAI |
| Optimism | 0.5 DAI | 1.0 DAI |

**Withdrawal Fee Notes:**
- Fees are **flat amounts**, not percentages
- For $100K+ depeg arb, withdrawal fees are negligible (< 0.01% of trade)
- Internal OKX transfers (sub-accounts) are **free**
- Deposits to OKX are **free** (user only pays network gas)
- Withdrawal fees updated periodically; verify via OKX API before final calculation

### 11.3 Gas Benchmarks per Chain

| Chain | Avg Gas Price | Swap Gas Limit | Approx Swap Cost (USD) | Native Token |
|-------|--------------|----------------|----------------------|-------------|
| Ethereum | 20-50 gwei | 150,000-300,000 | $5.00-$30.00 | ETH |
| Arbitrum | 0.1-0.5 gwei | 1,000,000-2,000,000 | $0.10-$0.50 | ETH |
| Base | 0.005-0.02 gwei | 150,000-300,000 | $0.01-$0.05 | ETH |

**Curve-Specific Gas Notes:**
- Curve 3pool swaps on Ethereum: ~150,000-200,000 gas (simpler than Uniswap V3 multi-hop)
- Curve on Arbitrum: same gas limit range but at ~0.2 gwei = ~$0.20
- Curve fee: 0.04% flat protocol fee on all swaps (taken from output amount)

### 11.4 DEX Protocol Fees (Stablecoin-Relevant)

**Curve Finance:**

| Pool | Fee | Typical Use |
|------|-----|------------|
| 3pool (USDT/USDC/DAI) | 0.04% (4 bps) | Stable-stable swaps |
| FRAX/USDC | 0.04% (4 bps) | Stable-stable |
| stETH/ETH | 0.04% (4 bps) | Pegged asset swaps |

**Uniswap (for comparison):**

| Pool Tier | Fee | Typical Stablecoin Use |
|-----------|-----|----------------------|
| 0.01% | 1 bp | USDC/USDT direct pair |
| 0.05% | 5 bps | USDC/DAI, USDT/DAI |

**Fee Notes:**
- Curve's 0.04% fee is FIXED (not variable by pool tier)
- Uniswap 0.01% pool for USDC/USDT exists but may have less liquidity than Curve
- Aggregator quotes (1inch, Jupiter, OKX DEX) already include protocol fees in output
- Do NOT double-count fees when using aggregator quotes

### 11.5 Cost Comparison: CEX-Only vs CEX-DEX Routes

```
Scenario: Buy $100K USDT at $0.995, sell for USDC at $1.0000

Route A — CEX-Only (OKX USDT-USDC pair):
  CEX taker fee (VIP0):     $80.00
  CEX slippage:              $10.00
  Total:                     $90.00
  Net profit:               $410.00
  Return:                    0.410%

Route B — CEX Buy -> Ethereum Curve:
  CEX taker fee:             $80.00
  Withdrawal (USDT ERC-20):  $3.00
  Gas (Ethereum):            $15.00
  Curve fee (0.04%):         $40.00
  Slippage (Curve):          $30.00
  Total:                    $168.00
  Net profit:               $332.00
  Return:                    0.332%

Route C — CEX Buy -> Arbitrum Curve:
  CEX taker fee:             $80.00
  Withdrawal (USDT Arb):     $0.10
  Gas (Arbitrum):             $0.30
  Curve fee (0.04%):         $40.00
  Slippage (Curve Arb):     $20.00
  Total:                    $140.40
  Net profit:               $359.60
  Return:                    0.360%

Winner: Route A (CEX-Only) for standard depeg arb when OKX has USDT-USDC pair depth.
Use Route C when: CEX orderbook is thin at your size, or DEX has better price.
```

---

## 12. Account Safety Protocol

### Demo vs Live Mode

The system defaults to **demo mode** at all times. Switching to live mode requires explicit user action.

**Demo Mode (Default):**
- MCP server config: `okx-DEMO-simulated-trading`
- All prices and positions are simulated
- All outputs include header: `[DEMO]`
- **No confirmation required to use demo mode**

**Live Mode:**
- MCP server config: `okx-LIVE-trading`
- Real market data and real account positions
- All outputs include header: `[LIVE]`
- Recommendations are still analysis-only (no auto-execution)

**Switching from Demo to Live:**
```
1. User explicitly says "live", "真實帳戶", "real account", or similar
2. System confirms the switch with a clear warning:
   "您正在切換至真實帳戶模式。所有數據將來自您的真實 OKX 帳戶。
    建議仍為分析建議，不會自動執行交易。
    請確認：輸入 '確認' 或 'confirm' 繼續。"
3. User must reply with explicit confirmation
4. System verifies authentication via system_get_capabilities
5. If authenticated: switch and display [LIVE] header
6. If NOT authenticated: show AUTH_FAILED error, remain in demo
```

**Session Rules:**

| Rule | Description |
|------|-------------|
| Default on startup | Always demo mode |
| Timeout | If no activity for 30 minutes, revert to demo mode |
| Error fallback | If live mode encounters AUTH_FAILED, revert to demo with notification |
| Header requirement | EVERY output must show `[DEMO]` or `[LIVE]` — no exceptions |
| No auto-execution | Even in live mode, skills only provide recommendations. `[RECOMMENDATION ONLY]` header is always present. |

---

## 13. Output Format & Templates (Inlined)

### 13.1 Formatting Rules

**Monetary Values:** `$12,345.67`, `+$1,234.56`, `-$89.10` (2 decimal places, comma thousands)
**Percentages:** 1 decimal place (e.g., `0.5%`). For very small deviations: 2 decimal places (e.g., `0.30%`).
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

### 13.2 Global Header Template

```
══════════════════════════════════════════
  Stablecoin Depeg Arbitrage — {COMMAND}
  [{MODE}] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: {TIMESTAMP}
  Data sources: {DATA_SOURCES}
══════════════════════════════════════════
```

### 13.3 Peg Deviation Table Template

```
── Peg Status ────────────────────────────

  Stablecoin   Price      Dev%      Severity     Type         Status
  ──────────   ─────────  ────────  ──────────   ──────────   ──────
  USDT         $0.9950    0.50%     EVALUATE     LIQUIDITY    [SAFE]
  USDC         $1.0000    0.00%     NOISE        NONE         [SAFE]
  DAI          $0.9980    0.20%     NOISE        NONE         [SAFE]

──────────────────────────────────────────
```

### 13.4 Solvency Assessment Template

```
── Solvency Assessment ───────────────────

  Depeg Type:         {DEPEG_TYPE}
  Confidence:         {CONFIDENCE}
  Recommendation:     {RECOMMENDATION}

  Evidence:
  +- [{STATUS}] Issuer Status:        {ISSUER_STATUS}
  +- [{STATUS}] Cross-Exchange Check:  {CROSS_EXCHANGE}
  +- [{STATUS}] Multi-Stablecoin:     {MULTI_STABLE}
  +- [{STATUS}] Magnitude Check:       {MAGNITUDE}
  +- [{STATUS}] Curve Pool Stress:    {CURVE_STRESS}

  ──────────────────────────────────────
  Overall: {OVERALL_STATUS}
  {OVERALL_MESSAGE}
```

### 13.5 Curve 3Pool Ratio Template

```
── Curve 3pool Balance ───────────────────

  Total TVL: {TVL}

  USDT  [{USDT_BAR}]  {USDT_PCT}%  ({USDT_DEV})  {USDT_STRESS}
  USDC  [{USDC_BAR}]  {USDC_PCT}%  ({USDC_DEV})  {USDC_STRESS}
  DAI   [{DAI_BAR}]   {DAI_PCT}%   ({DAI_DEV})   {DAI_STRESS}

  Ideal:  [████████████]  33.3%

  Pool Stress: {OVERALL_STRESS}
  {INTERPRETATION}
──────────────────────────────────────────
```

### 13.6 Cost Breakdown Template

```
── Cost Breakdown ──────────────────────────

  Trade Size:       {TRADE_SIZE}
  VIP Tier:         {VIP_TIER}
  Route:            {ROUTE}

  Leg 1 — {LEG1_VENUE} ({LEG1_SIDE})
  +- Trading Fee:     {LEG1_FEE}      ({LEG1_FEE_BPS} bps)
  +- Slippage Est:    {LEG1_SLIPPAGE} ({LEG1_SLIP_BPS} bps)
  +- Subtotal:        {LEG1_TOTAL}

  Transfer Costs
  +- Withdrawal Fee:  {WITHDRAWAL_FEE}
  +- Gas Cost:        {GAS_COST}

  Leg 2 — {LEG2_VENUE} ({LEG2_SIDE})
  +- Protocol Fee:    {PROTOCOL_FEE}   ({PROTOCOL_BPS} bps)
  +- Slippage Est:    {LEG2_SLIPPAGE} ({LEG2_SLIP_BPS} bps)
  +- Subtotal:        {LEG2_TOTAL}

  ──────────────────────────────────────
  Gross Spread:       {GROSS_SPREAD}   ({GROSS_PCT}%)
  Total Costs:       -{TOTAL_COST}
  ══════════════════════════════════════
  Net Profit:         {NET_PROFIT}     ({RETURN_PCT}%)
  Profit/Cost:        {PROFIT_TO_COST}x
  Min Breakeven Dev:  {MIN_DEV_PCT}%
  ══════════════════════════════════════
```

### 13.7 Risk Gauge Template

```
── Risk Assessment ─────────────────────────

  Overall Risk:     {RISK_GAUGE}  {RISK_SCORE}/10
                    {RISK_LABEL}

  Breakdown:
  +- Solvency Risk:    {SOLV_GAUGE}  {SOLV_SCORE}/10
  |                    {SOLV_NOTE}
  +- Execution Risk:   {EXEC_GAUGE}  {EXEC_SCORE}/10
  |                    {EXEC_NOTE}
  +- Liquidity Risk:   {LIQ_GAUGE}   {LIQ_SCORE}/10
  |                    {LIQ_NOTE}
  +- Systemic Risk:    {SYS_GAUGE}   {SYS_SCORE}/10
                       {SYS_NOTE}
```

### 13.8 Next Steps Template

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
  Stablecoin peg recovery is NOT guaranteed.
  Past depeg events do not guarantee similar outcomes.
  SOLVENCY DEPEGS CAN RESULT IN TOTAL LOSS — always verify backing.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
  穩定幣恢復掛鉤並非保證，償付能力風險可能導致全額損失。
══════════════════════════════════════════
```

---

## 14. Complete Output Examples

### 14.1 Scan Results — Complete Example

```
══════════════════════════════════════════
  Stablecoin Depeg Arbitrage — SCAN
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 15:00 UTC
  Data sources: OKX CEX + OKX DEX (Ethereum) + DeFiLlama (Curve 3pool)
══════════════════════════════════════════

── 掃描參數 ──────────────────────────────

  穩定幣:       USDT, USDC, DAI
  最低偏差:     0.3%
  分析規模:     $100,000.00
  VIP 等級:     VIP0
  Curve 分析:   已包含

── Peg Status ────────────────────────────

  Stablecoin   Price      Dev%      Severity     Type         Status
  ──────────   ─────────  ────────  ──────────   ──────────   ──────
  USDT         $0.9950    0.50%     EVALUATE     LIQUIDITY    [SAFE]
  USDC         $1.0000    0.00%     NOISE        NONE         [SAFE]
  DAI          $0.9985    0.15%     NOISE        NONE         [SAFE]

── 偏差分析: USDT ────────────────────────

  Deviation:        0.50% below peg
  Severity:         EVALUATE — likely profitable
  Depeg Type:       LIQUIDITY (confirmed)
  Direction:        Buy USDT cheap -> swap to USDC at par

── Solvency Assessment ───────────────────

  Depeg Type:         LIQUIDITY
  Confidence:         HIGH
  Recommendation:     SAFE_TO_ARB

  Evidence:
  +- [SAFE] Issuer Status:        Tether operational; reserves attested
  +- [SAFE] Cross-Exchange Check:  Depeg visible on Binance + Coinbase (market-wide)
  +- [SAFE] Multi-Stablecoin:     Only USDT affected; USDC/DAI stable
  +- [SAFE] Magnitude Check:       0.50% — well within liquidity depeg range
  +- [WARN] Curve Pool Stress:    USDT at 42% in 3pool (mild stress)

  ──────────────────────────────────────
  Overall: SAFE TO ARBITRAGE
  USDT depeg appears liquidity-driven. No solvency indicators detected.

── Curve 3pool Balance ───────────────────

  Total TVL: $1.8B

  USDT  [████████████████░░░░]  42.0%  (+8.7%)   WARNING
  USDC  [████████████░░░░░░░░]  31.5%  (-1.8%)   NORMAL
  DAI   [██████████░░░░░░░░░░]  26.5%  (-6.8%)   NORMAL

  Ideal:  [█████████████░░░░░░░]  33.3%

  Pool Stress: WARNING — mild USDT imbalance
  USDT 正在被賣入池中，表示市場正在拋售 USDT 換取 USDC/DAI。

── 盈利預估 ──────────────────────────────

  Route A — CEX-Only (OKX USDT-USDC 交易對):
    Gross Spread:    +$502.50
    Total Costs:     -$90.00
    Net Profit:      +$412.50    (0.413%)
    Profit/Cost:     4.58x       [SAFE]

  Route B — CEX -> Arbitrum Curve:
    Gross Spread:    +$502.50
    Total Costs:     -$140.40
    Net Profit:      +$362.10    (0.362%)
    Profit/Cost:     2.58x       [SAFE]

  推薦路線: Route A (CEX-Only) — 成本最低

── Risk Assessment ─────────────────────────

  Overall Risk:     ▓▓▓░░░░░░░  3/10
                    MODERATE-LOW

  Breakdown:
  +- Solvency Risk:    ▓▓░░░░░░░░  2/10
  |                    Issuer backed; no solvency indicators
  +- Execution Risk:   ▓▓░░░░░░░░  2/10
  |                    CEX-only route; no bridge/gas risk
  +- Liquidity Risk:   ▓▓▓░░░░░░░  3/10
  |                    USDT-USDC pair depth adequate at $100K
  +- Systemic Risk:    ▓▓░░░░░░░░  2/10
                       Only USDT affected; not systemic

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 詳細評估 USDT 脫鉤套利:
     stablecoin-depeg-arbitrage evaluate --stablecoin USDT --size-usd 100000
  2. 檢查不同規模的盈利變化:
     調整 --size-usd 參數: 50000, 100000, 200000
  3. 設定持續監控:
     stablecoin-depeg-arbitrage monitor --stablecoins USDT --threshold-pct 0.3

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Stablecoin peg recovery is NOT guaranteed.
  Past depeg events do not guarantee similar outcomes.
  SOLVENCY DEPEGS CAN RESULT IN TOTAL LOSS — always verify backing.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
  穩定幣恢復掛鉤並非保證，償付能力風險可能導致全額損失。
══════════════════════════════════════════
```

### 14.2 Evaluate Results — Complete Example (BLOCK scenario)

```
══════════════════════════════════════════
  Stablecoin Depeg Arbitrage — EVALUATE
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 15:10 UTC
  Data sources: OKX CEX + OKX DEX (Ethereum) + DeFiLlama
══════════════════════════════════════════

── USDT Depeg 評估 ──────────────────────

  Trade Size:       $200,000.00
  VIP Tier:         VIP0
  Chain:            Ethereum

── 價格比較 ──────────────────────────────

  OKX CEX (USDT-USDC)
  +- Last:     $0.9320
  +- Bid:      $0.9315
  +- Ask:      $0.9325
  +- Data Age: 2s [SAFE]

  OKX DEX (Ethereum)
  +- Price:    $0.9310
  +- Data Age: 4s [SAFE]

── Deviation Analysis ───────────────────

  Deviation:        6.80% below peg
  Severity:         DANGER — possible solvency issue
  Direction:        below_peg

── Solvency Assessment ───────────────────

  ██████████████████████████████████████████
  ██  BLOCK — SOLVENCY RISK DETECTED     ██
  ██████████████████████████████████████████

  Depeg Type:         UNKNOWN — CANNOT CONFIRM SAFETY
  Confidence:         LOW
  Recommendation:     DO_NOT_ARB

  Evidence:
  +- [BLOCK] Magnitude Check:       6.80% — exceeds 5% safety threshold
  +- [WARN]  Issuer Status:         Cannot verify current reserve status
  +- [SAFE]  Cross-Exchange Check:   Depeg visible across all major CEXes
  +- [WARN]  Multi-Stablecoin:      USDC also showing 0.8% depeg (mild)
  +- [BLOCK] Curve Pool Stress:     USDT at 68% in 3pool (EXTREME)

  ──────────────────────────────────────
  Overall: DO NOT TRADE

  USDT deviation exceeds 5% safety threshold. Solvency cannot be confirmed.
  Curve 3pool shows extreme USDT imbalance (68%). Multiple warning signals present.
  WAIT for Tether to issue official statement confirming reserve backing.

── Risk Assessment ─────────────────────────

  Overall Risk:     ▓▓▓▓▓▓▓▓▓░  9/10
                    HIGH RISK

  Breakdown:
  +- Solvency Risk:    ▓▓▓▓▓▓▓▓▓░  9/10
  |                    Cannot confirm issuer backing at this deviation
  +- Execution Risk:   ▓▓▓▓▓░░░░░  5/10
  |                    High volatility; prices moving rapidly
  +- Liquidity Risk:   ▓▓▓▓▓▓▓░░░  7/10
  |                    Curve pool severely imbalanced
  +- Systemic Risk:    ▓▓▓▓▓▓░░░░  6/10
                       USDC also showing mild depeg — possible contagion

══════════════════════════════════════════
  Action: BLOCKED — DO NOT TRADE
══════════════════════════════════════════

  Reasons for block:
  1. Depeg > 5% without confirmed cause (safety threshold exceeded)
  2. Curve 3pool USDT at 68% (EXTREME stress level)
  3. Cannot verify Tether reserve backing at this time

  What to do instead:
  1. 等待 Tether 官方聲明確認儲備支持
  2. 持續監控:
     stablecoin-depeg-arbitrage monitor --stablecoins USDT,USDC --threshold-pct 0.5
  3. 檢查 Curve 3pool:
     stablecoin-depeg-arbitrage curve-ratio
  4. 如果 Tether 確認償付能力，重新評估:
     stablecoin-depeg-arbitrage evaluate --stablecoin USDT --size-usd 200000

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  SOLVENCY DEPEGS CAN RESULT IN TOTAL LOSS.
  The UST collapse (May 2022) caused 99%+ losses for depeg arbitrageurs.
  以上僅為分析建議，不會自動執行任何交易。
  償付能力型脫鉤可能導致全額損失。
══════════════════════════════════════════
```

### 14.3 Curve Ratio — Complete Example

```
══════════════════════════════════════════
  Stablecoin Depeg Arbitrage — CURVE-RATIO
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 15:15 UTC
  Data sources: DeFiLlama (Curve 3pool)
══════════════════════════════════════════

── Curve 3pool Balance ───────────────────

  Pool: Curve 3pool (Ethereum Mainnet)
  Total TVL: $1,823,450,000

  USDT  [████████████████░░░░░░░░]  42.0%  (+8.7% vs ideal)   WARNING
  USDC  [████████████████░░░░░░░░]  31.5%  (-1.8% vs ideal)   NORMAL
  DAI   [████████████░░░░░░░░░░░░]  26.5%  (-6.8% vs ideal)   NORMAL

  Ideal balance: 33.3% each

── Interpretation ────────────────────────

  Pool Stress: WARNING — mild USDT overweight

  USDT at 42.0%: Market participants are selling USDT into the pool
  to acquire USDC and DAI. This indicates:
  - USDT sell pressure (people reducing USDT exposure)
  - USDT likely slightly below peg ($0.995-$0.998 range typically)
  - Pool is still within manageable range (< 50% threshold)

  Historical context:
  - Normal range: 30-36% per coin
  - June 2022 USDT scare: USDT reached ~55% before recovering
  - March 2023 USDC scare: USDC reached ~60% during SVB panic

── Stress Level Reference ────────────────

  30-36%:  NORMAL — within expected range
  36-40%:  ● WARNING — mild stress
  40-50%:  ●● ALERT — significant depeg pressure    <-- USDT is here
  50-60%:  ●●● DANGER — strong depeg
  60-70%:  ●●●● EXTREME — severe stress
  > 70%:   ●●●●● CRITICAL — unprecedented

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 評估 USDT 套利機會:
     stablecoin-depeg-arbitrage evaluate --stablecoin USDT --size-usd 100000
  2. 設定 Curve 監控:
     stablecoin-depeg-arbitrage monitor --stablecoins USDT --threshold-pct 0.3
  3. 查看完整穩定幣狀態:
     stablecoin-depeg-arbitrage scan

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  Curve pool ratios are indicative, not definitive, signals.
  以上僅為分析建議，不會自動執行任何交易。
══════════════════════════════════════════
```

---

## 15. Stablecoin Reference (Inlined)

### 15.1 Supported Stablecoins

| Stablecoin | Issuer | Type | Backing | Proof of Reserves |
|-----------|--------|------|---------|-------------------|
| USDT | Tether | Centralized, fiat-backed | Cash, T-Bills, commercial paper | tether.to/en/transparency |
| USDC | Circle | Centralized, fiat-backed | Cash, short-term US Treasuries | circle.com/en/transparency |
| DAI | MakerDAO | Decentralized, crypto-collateralized | ETH, USDC, other crypto + RWA | daistats.com |

### 15.2 OKX instId Formats for Stablecoins

| Pair | instId | Description |
|------|--------|-------------|
| USDT/USDC | `USDT-USDC` | USDT priced in USDC (primary pair for depeg detection) |
| USDC/USDT | `USDC-USDT` | USDC priced in USDT (cross-check) |

### 15.3 Contract Addresses

**Ethereum (chainIndex: 1):**

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0xdac17f958d2ee523a2206206994597c13d831ec7` | 6 |
| USDC | `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` | 6 |
| DAI | `0x6b175474e89094c44da98b954eedeac495271d0f` | 18 |

**Arbitrum One (chainIndex: 42161):**

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0xfd086bc7cd5c481dcc9c85ebe478a1c0b69fcbb9` | 6 |
| USDC | `0xaf88d065e77c8cc2239327c5edb3a432268e5831` | 6 |
| DAI | `0xda10009cbd5d07dd0cecc66161fc93d7c9000da1` | 18 |

**Base (chainIndex: 8453):**

| Token | Address | Decimals |
|-------|---------|----------|
| USDC | `0x833589fcd6edb6e08f4c7c32d4f71b54bda02913` | 6 |
| DAI | `0x50c5725949a6f0c72e6c4a641f24049a917db0cb` | 18 |

**CRITICAL: DAI has 18 decimals** on all chains. USDT and USDC have 6 decimals on most chains. Always verify decimals before encoding amounts.

### 15.4 Curve 3Pool Reference

| Property | Value |
|----------|-------|
| Pool Name | Curve.fi DAI/USDC/USDT (3pool) |
| Pool Address (Ethereum) | `0xbEbc44782C7dB0a1A60Cb6fe97d0b483032FF1C7` |
| Coins | [DAI, USDC, USDT] (index 0, 1, 2) |
| Fee | 0.04% (4 bps) |
| Admin Fee | 50% of trading fee |
| A Parameter | 2000 (amplification coefficient — higher = tighter peg) |

### 15.5 Historical Depeg Events Reference

| Event | Date | Stablecoin | Low Price | Recovery Time | Type | Cause |
|-------|------|-----------|-----------|--------------|------|-------|
| USDC SVB panic | Mar 2023 | USDC | $0.87 | ~3 days | LIQUIDITY | SVB bank collapse, Circle had $3.3B in SVB |
| USDT June scare | Jun 2022 | USDT | $0.95 | ~12 hours | LIQUIDITY | Celsius/3AC contagion panic |
| DAI Black Thursday | Mar 2020 | DAI | $0.96 | ~2 days | LIQUIDITY | ETH crash caused undercollateralization |
| UST collapse | May 2022 | UST | $0.00 (effective) | NEVER | SOLVENCY | Algorithmic stablecoin death spiral |
| HUSD collapse | 2023 | HUSD | ~$0.30 | NEVER | SOLVENCY | Issuer insolvency |

**Key Lesson:** LIQUIDITY depegs recover. SOLVENCY depegs do NOT. The classification step (Section 8, Step 3) is the single most important safety mechanism in this skill.

---

## 16. Error Codes & Recovery

| Code | Condition | User Message (ZH) | User Message (EN) | Recovery |
|------|-----------|-------------------|-------------------|----------|
| `MCP_NOT_CONNECTED` | okx-trade-mcp server unreachable | MCP 伺服器無法連線。請確認 okx-trade-mcp 是否正在運行。 | MCP server unreachable. Check if okx-trade-mcp is running. | Verify `~/.okx/config.toml`, restart server |
| `AUTH_FAILED` | API key invalid or expired | API 認證失敗。請檢查 OKX API 金鑰設定。 | API authentication failed. Check OKX API key config. | Update config.toml |
| `DATA_STALE` | Price data > 10s old (after retry) | 市場數據已過期（{venue} 延遲 {age}ms，穩定幣模式上限 10000ms）。 | Market data stale ({venue}: {age}ms, stablecoin mode max 10000ms). | Auto-retry once, then fail |
| `INSTRUMENT_NOT_FOUND` | USDT-USDC pair missing on OKX | OKX 上找不到 USDT-USDC 交易對。 | Instrument USDT-USDC not found on OKX. | Verify instrument name, check OKX status |
| `SOLVENCY_RISK` | Solvency check failed | 償付能力風險：{reason}。不建議套利此脫鉤事件。 | Solvency risk: {reason}. Do NOT arbitrage this depeg. | Wait for issuer statement, monitor |
| `SYSTEMIC_RISK` | Multiple stablecoins depegging | 系統性風險：多個穩定幣同時脫鉤。這不是標準套利機會。 | Systemic risk: Multiple stablecoins depegging simultaneously. | Do NOT trade. Wait for market stabilization. |
| `DEPEG_DANGER` | Deviation > 5% without confirmed cause | 脫鉤超過 5% 且原因未確認。假設存在償付能力風險。 | Depeg exceeds 5% without confirmed cause. Assume solvency risk. | Investigate cause, wait for clarity |
| `CURVE_EXTREME_STRESS` | Curve pool coin > 70% | Curve 3pool 嚴重失衡（{coin} 佔 {pct}%）。極端壓力。 | Curve 3pool severely imbalanced ({coin} at {pct}%). Extreme stress. | Monitor pool, reduce position size |
| `INSUFFICIENT_LIQUIDITY` | DEX liquidity < $1M | DEX 流動性不足（${actual}，最低要求 $1,000,000）。 | DEX liquidity insufficient (${actual}, min $1,000,000). | Use CEX-only route, or wait |
| `PRICE_IMPACT_HIGH` | Swap price impact > 0.5% | 預估價格影響 {impact}% 過高（穩定幣模式上限 0.5%）。建議減小規模。 | Price impact {impact}% exceeds 0.5% stablecoin limit. Reduce size. | Reduce trade size, split across venues |
| `NOT_PROFITABLE` | Net profit <= 0 after all costs | 扣除所有成本後淨利潤為負（{net_pnl}），不建議執行。 | Net profit negative after costs ({net_pnl}). Not recommended. | Wait for larger deviation, reduce costs (use L2) |
| `RATE_LIMITED` | API rate limit hit after 3 retries | API 請求頻率超限，{wait} 秒後重試。 | API rate limit reached. Retrying in {wait}s. | Exponential backoff: 1s, 2s, 4s |
| `TRADE_SIZE_EXCEEDED` | Size > $500,000 hard cap | 交易金額 ${amount} 超過穩定幣套利上限 $500,000。 | Trade size ${amount} exceeds stablecoin arb limit $500,000. | Cap at $500,000 and inform user |
| `DEX_UNAVAILABLE` | OnchainOS not installed or non-functional | DEX 數據暫時無法取得。僅使用 CEX 數據（精確度降低）。 | DEX data unavailable. Using CEX-only data (reduced accuracy). | Continue in CEX-only mode with label |

---

## 17. Cross-Skill Integration Contracts

### Input: What This Skill Consumes

| Source Skill / Tool | Data Consumed | Schema | Usage |
|---------------------|--------------|--------|-------|
| okx-trade-mcp `market_get_ticker` | USDT-USDC, USDC-USDT spot prices | OKX ticker response | CEX price collection (Step 2a) |
| okx-trade-mcp `market_get_orderbook` | USDT-USDC orderbook depth | OKX orderbook response | Slippage estimation, depth analysis |
| OnchainOS `dex-market price` | DEX stablecoin prices | `{priceUsd, ...}` | DEX price collection (Step 2b) |
| OnchainOS `dex-swap quote` | DEX executable price + impact | `{toTokenAmount, priceImpactPercent, estimatedGas}` | Price impact check, cost estimation |
| DeFiLlama `/pools` | Curve 3pool balances | `{coins[], balances[], totalSupply}` | Curve ratio analysis (Step 4) |

### Output: What This Skill Produces

| Output | Consumer | Schema | Handoff |
|--------|----------|--------|---------|
| Peg deviation scan | User (formatted output) | `DepegScanResult` | Displayed directly |
| Single stablecoin evaluation | User (formatted output) | `DepegEvaluateResult` | Displayed directly |
| Monitor alerts | User (formatted output) | `DepegMonitorAlert` | Displayed in real-time |
| Curve ratio analysis | User (formatted output) | `CurveRatioResult` | Displayed directly |

### Data Flow Diagram

```
market_get_ticker("USDT-USDC") -> CEX price
  |
  +  onchainos dex-market price -> DEX price
  |
  v
stablecoin-depeg-arbitrage.scan / evaluate
  |
  +- Deviation calculation (abs(price - 1.0) * 100)
  |    -> severity classification (NOISE/MONITOR/EVALUATE/HIGH_ALERT/DANGER)
  |
  +- Solvency Check Procedure (5 sub-checks)
  |    -> BLOCK if solvency risk / SAFE if liquidity depeg
  |
  +- DeFiLlama Curve 3pool query -> balance ratio analysis
  |    -> BLOCK if extreme stress / WARN if alert
  |
  +- market_get_orderbook("USDT-USDC") -> depth analysis
  |
  +- Profitability calculation (Section 10 formulas)
  |    -> net_profit, return_pct, profit_to_cost_ratio
  |
  v
Output: Risk-assessed depeg arb recommendations with solvency classification
```
