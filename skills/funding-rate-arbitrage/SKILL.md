---
name: funding-rate-arbitrage
description: >
  Exploits funding rate differentials on perpetual swaps. When a perp has a high positive
  funding rate, short it and long spot to earn the funding payment (delta-neutral carry trade).
  Pure CEX strategy — both legs execute on OKX.
  Trigger phrases include: "funding rate", "資金費率", "carry trade", "套息", "利率差異",
  "funding arbitrage", "funding harvest", "做 carry", "資金費率套利", "funding income",
  "收資金費", "邊個幣 funding 最高", "持倉收息".
  Do NOT use for: trade execution, onchain token security checks, basis/delivery futures
  (use basis-trading), or yield farming (use yield-optimizer).
  Requires: okx-trade-mcp (CEX-only strategy — OnchainOS not needed).
allowed-tools: >
  okx-DEMO-simulated-trading:market_get_funding_rate,
  okx-DEMO-simulated-trading:market_get_ticker,
  okx-DEMO-simulated-trading:market_get_instruments,
  okx-DEMO-simulated-trading:market_get_candles,
  okx-DEMO-simulated-trading:system_get_capabilities,
  okx-LIVE-real-money:market_get_funding_rate,
  okx-LIVE-real-money:market_get_ticker,
  okx-LIVE-real-money:market_get_instruments,
  okx-LIVE-real-money:market_get_candles,
  okx-LIVE-real-money:system_get_capabilities
---

# funding-rate-arbitrage

Delta-neutral carry trade skill for the Onchain x CEX Strats system. Scans OKX perpetual swap funding rates, identifies high-yield carry opportunities, and projects income under multiple scenarios. When a perpetual swap has a high positive funding rate, the strategy is: **short the perp + long the spot** to earn the funding payment while remaining market-neutral.

Reuses patterns from: `Interest Information Aggregator/src/scrapers/hot_token_scraper.py` (funding rate scanning logic), `futures_calculator.py` (implied rate calculation).

---

## 1. Role

**Funding rate carry trade analyst** — identifies and evaluates delta-neutral funding harvesting opportunities.

This skill is responsible for:
- Scanning all OKX perpetual swaps for elevated funding rates
- Ranking opportunities by annualized yield
- Projecting carry income under bull/base/bear scenarios
- Computing break-even holding periods after entry/exit costs
- Monitoring existing carry positions for unwind signals

This skill does **NOT**:
- Execute any trades (analysis only, never sends orders)
- Check onchain token security (CEX-only strategy, no onchain legs)
- Handle basis/delivery futures (delegate to `basis-trading`)
- Handle DeFi yield farming (delegate to `yield-optimizer`)
- Manage positions or portfolio state

---

## 2. Language

Match the user's language. Default: Traditional Chinese (繁體中文).

Metric labels may use English abbreviations regardless of language:
- `funding rate`, `carry`, `annualized`, `bps`, `PnL`, `break-even`, `unwind`
- Timestamps always displayed in UTC

---

## 3. Account Safety

| Rule | Detail |
|------|--------|
| Default mode | Demo (`okx-DEMO-simulated-trading`) |
| Mode display | Every output header shows `[DEMO]` or `[LIVE]` |
| Read-only | This skill performs **zero** write operations -- no trades, no transfers |
| Recommendation header | Always show `[RECOMMENDATION ONLY -- 不會自動執行]` |
| Live switch | Requires explicit user confirmation (see Account Safety Protocol in Section 10) |

Even in `[LIVE]` mode, this skill only reads market data. There is no risk of accidental execution.

---

## 4. Pre-flight (Machine-Executable Checklist)

This is a **CEX-only** strategy. OnchainOS CLI is NOT required.

Run these checks **in order** before any command. BLOCK at any step halts execution.

| # | Check | Command / Tool | Success Criteria | Failure Action |
|---|-------|---------------|-----------------|----------------|
| 1 | okx-trade-mcp connected | `system_get_capabilities` (DEMO or LIVE server) | `authenticated: true`, `modules` includes `"market"` | BLOCK -- output `MCP_NOT_CONNECTED`. Tell user to verify `~/.okx/config.toml` and restart MCP server. |
| 2 | okx-trade-mcp mode | `system_get_capabilities` -> `mode` field | Returns `"demo"` or `"live"` matching expected mode | WARN -- display actual mode in header. If user requested live but got demo, surface mismatch. |
| 3 | SWAP instruments accessible | `market_get_instruments(instType: "SWAP")` | Returns non-empty array of perpetual swap instruments | BLOCK -- output `INSTRUMENT_NOT_FOUND`. Suggest checking API connectivity. |

### Pre-flight Decision Tree

```
Check 1 FAIL -> BLOCK (cannot proceed without CEX data)
Check 1 PASS -> Check 2
  Check 2 mismatch -> WARN + continue
  Check 2 PASS -> Check 3
    Check 3 FAIL -> BLOCK (no instruments available)
    Check 3 PASS -> ALL SYSTEMS GO
```

---

## 5. Skill Routing Matrix

| User Need | Use THIS Skill? | Delegate To |
|-----------|----------------|-------------|
| "邊個幣資金費率最高" / "highest funding rate" | Yes -- `scan` | -- |
| "ETH 做 carry 7 天可以賺幾多" / "ETH carry income" | Yes -- `evaluate` | -- |
| "ETH 而家資金費率幾多" / "current funding rate" | Yes -- `rates` | -- |
| "carry 持倉預估收入" / "projected carry income" | Yes -- `carry-pnl` | -- |
| "我嘅 carry 位仲值得繼續揸嗎" / "should I unwind" | Yes -- `unwind-check` | -- |
| "期貨基差幾多" / "basis trade" | No | `basis-trading` |
| "DeFi 收益比較" / "best yield" | No | `yield-optimizer` |
| "CEX 同 DEX 價差" / "CEX-DEX spread" | No | `cex-dex-arbitrage` |
| "呢個幣安唔安全" / "is this token safe" | No | GoPlus MCP directly |
| "幫我開倉" / "execute trade" | No | Refuse -- no skill executes trades |

---

## 6. Command Index

| Command | Function | Read/Write | Description |
|---------|----------|-----------|-------------|
| `scan` | Scan top funding rates | Read | Fetch all perpetual swap funding rates, rank by annualized yield, return top N |
| `evaluate` | Deep profitability analysis | Read | Full cost/income projection for a specific asset over a holding period |
| `rates` | Current & historical rates | Read | Show current, realized, and predicted funding rates with trend sparkline |
| `carry-pnl` | Scenario-based P&L projection | Read | Project income under bull/base/bear funding rate scenarios |
| `unwind-check` | Unwind evaluation | Read | Assess whether an existing carry position should continue or close |

---

## 7. Parameter Reference

### 7.1 Command: `scan`

Scan all OKX perpetual swaps and rank by annualized funding rate yield.

```bash
funding-rate-arbitrage scan --top-n 10 --min-annualized-pct 10
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--top-n` | integer | No | `10` | -- | Min: 1, Max: 50. Positive integer. |
| `--min-annualized-pct` | number | No | `10` | -- | Min: 0, Max: 1000. Minimum annualized yield % to include. |
| `--inst-type` | string | No | `"SWAP"` | `SWAP` | Only perpetual swaps supported. Fixed value. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | Used for cost estimation in daily income calc. |

#### Return Schema

```yaml
FundingRateScanResult:
  timestamp: integer              # Unix ms when scan was assembled
  vip_tier: string                # VIP tier used for cost calculations
  total_scanned: integer          # Total number of perpetual swaps checked
  results:
    - instId: string              # e.g. "BTC-USDT-SWAP"
      asset: string               # e.g. "BTC"
      current_rate: string        # e.g. "0.000150" (0.015% per 8h)
      realized_rate: string       # Last settled funding rate
      next_funding_rate: string   # Predicted next rate (may be empty)
      annualized_pct: number      # current_rate * 3 * 365 * 100
      next_funding_time: integer  # Unix ms of next settlement
      direction: string           # "positive" (shorts receive) or "negative" (longs receive)
      recommended_position: string  # "short_perp_long_spot" or "long_perp_short_spot"
      estimated_daily_income_per_10k: number  # USD income per $10,000 position per day
      rate_trend_7d: string       # Sparkline of last 7 days: "▃▄▅▅▆▆▇▇"
      rate_stability: string      # "stable", "volatile", "trending_up", "trending_down"
      risk_level: string          # "[SAFE]", "[WARN]", or "[BLOCK]"
      warnings: string[]          # Any applicable warnings
```

#### Return Fields Detail

| Field | Type | Description |
|-------|------|-------------|
| `current_rate` | string | Current period funding rate. **String -- parse to float.** Positive = longs pay shorts. |
| `realized_rate` | string | Last settled (realized) rate. Compare with current to detect rate flips. |
| `annualized_pct` | number | Annualized yield: `parseFloat(current_rate) * 3 * 365 * 100` |
| `direction` | string | `"positive"` means shorts receive funding (shorts pay longs). `"negative"` is the reverse. |
| `recommended_position` | string | For positive rates: short perp + long spot. For negative: long perp + short spot (rare). |
| `estimated_daily_income_per_10k` | number | `10000 * parseFloat(current_rate) * 3` -- daily income on $10K position |
| `rate_stability` | string | Based on standard deviation of last 21 funding intervals (7 days) |

---

### 7.2 Command: `evaluate`

Deep analysis of a specific funding carry opportunity with full cost/income projection.

```bash
funding-rate-arbitrage evaluate --asset ETH --size-usd 5000 --hold-days 7 --vip-tier VIP1
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any valid OKX perp asset | Uppercase. Must have a corresponding `-USDT-SWAP` instrument. |
| `--size-usd` | number | Yes | -- | -- | Min: 100, Max: 50,000 (hard cap). Trade size in USD. |
| `--hold-days` | integer | No | `7` | -- | Min: 1, Max: 90. Expected holding period. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | Used for fee calculation. |
| `--leverage` | number | No | `1` | -- | Min: 1, Max: 2 (hard cap per safety rules). |
| `--borrow-rate-annual` | number | No | `0` | -- | Min: 0. Annualized borrow cost if using margin. |

#### Return Schema

```yaml
FundingEvaluateResult:
  timestamp: integer
  asset: string
  instId: string                   # e.g. "ETH-USDT-SWAP"
  size_usd: number
  hold_days: integer
  vip_tier: string
  leverage: number

  current_funding:
    rate_per_8h: string            # Current funding rate
    realized_rate: string          # Last settled rate
    annualized_pct: number         # Annualized as percentage
    next_payment_time: integer     # Unix ms
    trend_7d: string               # Sparkline

  cost_analysis:
    entry_cost:
      spot_buy_fee: number         # size_usd * spot_taker_rate
      perp_short_fee: number       # size_usd * swap_taker_rate
      spot_slippage: number        # Estimated from orderbook
      perp_slippage: number        # Estimated from orderbook
      total_entry: number          # Sum of above
    exit_cost:
      spot_sell_fee: number
      perp_close_fee: number
      spot_slippage: number
      perp_slippage: number
      total_exit: number
    total_roundtrip: number        # entry + exit
    borrow_cost: number            # size_usd * borrow_rate * hold_days / 365

  income_projection:
    daily_income: number           # size_usd * funding_rate * 3
    total_income: number           # daily_income * hold_days
    net_profit: number             # total_income - total_roundtrip - borrow_cost
    annualized_net_yield_pct: number  # (net_profit / size_usd) * (365 / hold_days) * 100
    break_even_days: number        # total_roundtrip / daily_income

  risk_assessment:
    risk_score: integer            # 1-10
    risk_gauge: string             # e.g. "▓▓▓░░░░░░░"
    risk_label: string             # "LOW RISK", "MODERATE", etc.
    warnings: string[]

  is_profitable: boolean
  confidence: string               # "high", "medium", "low"
```

---

### 7.3 Command: `rates`

Show current and historical funding rates for a specific perpetual swap.

```bash
funding-rate-arbitrage rates --asset BTC --lookback-intervals 24
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any valid OKX perp asset | Uppercase. Must have `-USDT-SWAP`. |
| `--lookback-intervals` | integer | No | `24` | -- | Min: 1, Max: 270 (90 days * 3/day). Number of 8h intervals. |

#### Return Schema

```yaml
FundingRatesResult:
  timestamp: integer
  asset: string
  instId: string
  current:
    rate_per_8h: string
    realized_rate: string
    next_funding_rate: string
    annualized_pct: number
    next_payment_time: integer
    countdown: string              # e.g. "2h 15m"
  history:
    - time: integer                # Unix ms of settlement
      rate: string                 # Settled rate
      cumulative: string           # Running cumulative from most recent
  summary:
    avg_rate_8h: number            # Mean over lookback period
    max_rate_8h: number
    min_rate_8h: number
    std_dev: number
    positive_pct: number           # % of intervals with positive rate
    trend: string                  # "rising", "falling", "stable", "volatile"
    sparkline: string              # e.g. "▃▄▅▅▆▆▇▇"
```

---

### 7.4 Command: `carry-pnl`

Project carry trade income under multiple funding rate scenarios.

```bash
funding-rate-arbitrage carry-pnl --asset ETH --size-usd 10000 --hold-days 30 --vip-tier VIP1
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any valid OKX perp asset | Uppercase. |
| `--size-usd` | number | Yes | -- | -- | Min: 100, Max: 50,000. |
| `--hold-days` | integer | No | `30` | -- | Min: 1, Max: 90. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | Fee tier. |
| `--scenarios` | string | No | `"auto"` | `auto`, `custom` | `auto` uses bull/base/bear derived from history. |

#### Return Schema

```yaml
CarryPnlResult:
  timestamp: integer
  asset: string
  size_usd: number
  hold_days: integer

  scenarios:
    bull:
      funding_rate_8h: number      # 75th percentile of historical rates
      daily_income: number
      total_income: number
      net_profit: number           # After entry/exit costs
      annualized_yield_pct: number
    base:
      funding_rate_8h: number      # Median historical rate
      daily_income: number
      total_income: number
      net_profit: number
      annualized_yield_pct: number
    bear:
      funding_rate_8h: number      # 25th percentile of historical rates
      daily_income: number
      total_income: number
      net_profit: number
      annualized_yield_pct: number

  cost_summary:
    total_roundtrip: number
    break_even_days_base: number   # Using base scenario rate

  warnings: string[]
```

---

### 7.5 Command: `unwind-check`

Evaluate whether an existing carry position should continue or be closed.

```bash
funding-rate-arbitrage unwind-check --asset ETH --entry-rate 0.000150 --held-days 5 --size-usd 10000
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any valid OKX perp asset | Uppercase. |
| `--entry-rate` | number | Yes | -- | -- | The funding rate (per 8h) when position was opened. |
| `--held-days` | integer | Yes | -- | -- | Min: 0. Days the position has been held. |
| `--size-usd` | number | Yes | -- | -- | Position size in USD. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | For exit cost calculation. |

#### Return Schema

```yaml
UnwindCheckResult:
  timestamp: integer
  asset: string
  instId: string

  position_summary:
    size_usd: number
    held_days: integer
    entry_rate: number
    current_rate: number
    rate_change_pct: number        # (current - entry) / entry * 100

  estimated_income_earned: number  # Approximate funding collected so far
  exit_cost: number                # Cost to close both legs now
  estimated_net_pnl: number        # income_earned - entry_cost - exit_cost

  forward_outlook:
    current_annualized_pct: number
    rate_trend: string             # "rising", "falling", "stable", "volatile"
    rate_trend_sparkline: string
    consecutive_sign_flips: integer  # How many recent intervals flipped sign
    projected_7d_income: number    # At current rate

  recommendation: string           # "CONTINUE", "UNWIND", "REDUCE"
  recommendation_reason: string    # Explanation
  confidence: string

  warnings: string[]
```

---

## 8. Operation Flow

### Step 1: Intent Recognition

Parse user message to extract:

| Element | Extraction Logic | Fallback |
|---------|-----------------|----------|
| Command | Map to `scan` / `evaluate` / `rates` / `carry-pnl` / `unwind-check` | Default: `scan` |
| Asset | Extract token symbol (BTC, ETH, SOL...) | For `scan`: not required. For others: **ask user**. |
| Size | Look for "$X", "X USDT", "X 美金" | Default: $10,000 |
| Hold days | Look for "X 天", "X days", "一週/一個月" | Default: 7 |
| VIP tier | Look for "VIP0"..."VIP5" | Default: VIP0 |

**Keyword-to-command mapping:**

| Keywords | Command |
|----------|---------|
| "掃描", "scan", "排名", "邊個最高", "top funding", "highest rate" | `scan` |
| "評估", "evaluate", "分析", "賺幾多", "profitable", "值唔值得" | `evaluate` |
| "費率", "rates", "而家幾多", "current rate", "歷史", "history" | `rates` |
| "收入", "income", "scenario", "情景", "projection", "預測" | `carry-pnl` |
| "平倉", "unwind", "繼續揸", "close", "keep holding", "要唔要走" | `unwind-check` |

### Step 2: Data Collection

#### For `scan` command:

```
1. market_get_instruments(instType: "SWAP")
   -> Extract all instIds ending in "-USDT-SWAP"
   -> Build candidate list

2. For each candidate instId:
   market_get_funding_rate(instId)
   -> Extract: fundingRate, realizedRate, fundingTime, nextFundingRate

3. For top candidates (after filtering by min_annualized_pct):
   market_get_ticker(instId)
   -> Extract: last (for position size calculation)
   -> Also fetch spot ticker: market_get_ticker("{ASSET}-USDT")
```

**Rate limit awareness:** `market_get_funding_rate` is limited to 10 req/s. For a full scan of ~100+ instruments, batch carefully and add delays between groups. Cap at 50 instruments per scan cycle.

#### For `evaluate` command:

```
1. market_get_funding_rate(instId: "{ASSET}-USDT-SWAP")
   -> Current rate, realized rate, next payment time

2. market_get_ticker(instId: "{ASSET}-USDT")
   -> Spot price for entry calculation

3. market_get_ticker(instId: "{ASSET}-USDT-SWAP")
   -> Perp price for position sizing

4. market_get_candles(instId: "{ASSET}-USDT-SWAP", bar: "8H", limit: "21")
   -> Historical funding rate proxied from candle data (for trend analysis)

5. profitability-calculator.estimate (internal call)
   -> TradeLeg[]: spot buy + perp short
   -> Returns: total entry/exit costs
```

**Important:** OKX returns all values as strings. Always `parseFloat()` before arithmetic.

### Step 3: Compute

#### Annualized Funding Rate

```
annualized_funding = funding_rate_per_8h * 3 * 365

Variable Definitions:
  funding_rate_per_8h  = Single funding payment rate (e.g., 0.01% = 0.0001)
  3                    = Funding payments per day (every 8 hours)
  365                  = Days per year

Worked Example:
  funding_rate_per_8h = 0.0150%  (= 0.000150)
  annualized_funding = 0.000150 * 3 * 365
                     = 0.16425
                     = 16.43%

Caveats:
  - Funding rates are variable and reset every 8h; annualization assumes constant rate.
  - Use a rolling average (e.g., 7-day) for more stable projections.
  - Negative funding means shorts pay longs; positive means longs pay shorts.
```

#### Daily Income

```
daily_income = position_size * funding_rate_per_8h * 3

Worked Example:
  position_size = $100,000
  funding_rate_per_8h = 0.0150%  (= 0.000150)
  daily_income = 100000 * 0.000150 * 3
               = $45.00
```

#### Estimated Daily Income per $10,000

```
estimated_daily_income_per_10k = 10000 * parseFloat(funding_rate_per_8h) * 3
```

#### Entry/Exit Cost Calculation

```
Entry:
  spot_buy_fee    = size_usd * spot_taker_rate
  perp_short_fee  = size_usd * swap_taker_rate
  spot_slippage   = estimated from orderbook or benchmark (~1 bps)
  perp_slippage   = estimated from orderbook or benchmark (~1 bps)
  total_entry     = spot_buy_fee + perp_short_fee + spot_slippage + perp_slippage

Exit (symmetric):
  total_exit = total_entry  (same legs, reversed)

Total round-trip = total_entry + total_exit
```

**Fee rates by VIP tier (for this calculation):**

| Tier | Spot Taker | Swap Taker |
|------|-----------|-----------|
| VIP0 | 0.080% | 0.050% |
| VIP1 | 0.070% | 0.045% |
| VIP2 | 0.060% | 0.040% |
| VIP3 | 0.050% | 0.035% |
| VIP4 | 0.040% | 0.030% |
| VIP5 | 0.035% | 0.025% |

**Worked Example (VIP1, $50,000):**
```
spot_buy_fee    = 50000 * 0.0007  = $35.00
perp_short_fee  = 50000 * 0.00045 = $22.50
spot_slippage   = 50000 * 1/10000 = $5.00
perp_slippage   = 50000 * 1/10000 = $5.00
total_entry     = $67.50
total_exit      = $67.50
total_roundtrip = $135.00
```

**Pattern B: Funding Harvest Cost Template (VIP1, $100,000):**
```
Position size: $100,000

Spot buy (VIP1 taker):        $70.00  (7.0 bps)
Perp short (VIP1 taker):      $45.00  (4.5 bps)
                               ------
Entry cost:                   $115.00  (11.5 bps)
Exit cost (same):             $115.00  (11.5 bps)
Round-trip:                   $230.00  (23.0 bps)
```

#### Break-Even Holding Period

```
break_even_days = (entry_cost + exit_cost) / daily_income
               = total_roundtrip / daily_income

Variable Definitions:
  entry_cost   = Total cost to enter the position (CEX fees + slippage)
  exit_cost    = Total cost to exit the position
  daily_income = Projected daily income from funding

Worked Example:
  entry_cost = $80.00
  exit_cost  = $80.00
  daily_income = $45.00
  break_even_days = (80 + 80) / 45
                  = 3.56 days

Caveats:
  - If funding rate drops or flips, break-even extends.
  - Monitor funding rate trend — declining trend suggests exiting before break-even may be optimal.
```

#### Net Carry (Annualized)

```
net_carry = annualized_funding - borrow_rate - annualized_trading_fees

annualized_trading_fees = (total_roundtrip / size_usd) * (365 / hold_days) * 100

Variable Definitions:
  annualized_funding       = Expected annualized funding income
  borrow_rate              = Annualized cost of borrowing margin (if applicable)
  annualized_trading_fees  = Entry + exit fee annualized

Worked Example:
  annualized_funding = 16.43%
  borrow_rate = 5.00%  (USDT borrow on OKX)
  annualized_trading_fees = 0.14%  (VIP1 taker round-trip: 0.07% * 2)
  net_carry = 16.43% - 5.00% - 0.14%
            = 11.29%

Caveats:
  - Borrow rate may be zero if using own capital with no margin borrowing.
  - For delta-neutral funding harvest (spot long + perp short), no directional risk
    but still exposed to funding rate flipping negative.
```

#### Scenario Generation (for carry-pnl)

```
From historical funding rates (last 90 intervals = 30 days):
  bull_rate = 75th percentile
  base_rate = median (50th percentile)
  bear_rate = 25th percentile

For each scenario:
  daily_income = size_usd * scenario_rate * 3
  total_income = daily_income * hold_days
  net_profit   = total_income - total_roundtrip - borrow_cost
  annualized_yield = (net_profit / size_usd) * (365 / hold_days) * 100
```

### Step 4: Format Output & Suggest Next Steps

Use output templates (see Section 13):

- **Header:** Global Header Template with mode
- **Body:** Funding Rate Display Template or scan results table
- **Footer:** Next Steps Template

Suggested follow-up actions (vary by result):

| Situation | Suggested Actions |
|-----------|-------------------|
| Scan shows high-yield opportunities | "深入評估 -> `funding-rate-arbitrage evaluate --asset {ASSET}`" |
| Evaluate shows profitable carry | "查看情景分析 -> `funding-rate-arbitrage carry-pnl --asset {ASSET}`" |
| Rate declining or flipped | "評估平倉 -> `funding-rate-arbitrage unwind-check --asset {ASSET}`" |
| Break-even > 30 days | "風險較高，考慮較短持倉期或尋找更高費率資產" |

---

## 9. Key Formulas

All formulas are canonical with full derivations and worked examples inlined above in Section 8 Step 3.

### Quick Reference

| Formula | Expression | Example |
|---------|-----------|---------|
| Annualized Funding | `rate_per_8h * 3 * 365` | 0.000150 * 3 * 365 = 16.43% |
| Daily Income | `position_size * rate_per_8h * 3` | $100K * 0.000150 * 3 = $45.00 |
| Break-Even Days | `total_roundtrip / daily_income` | $135.00 / $22.50 = 6.0 days |
| Net Carry | `annualized_funding - borrow_rate - ann_fees` | 16.43% - 5.00% - 0.14% = 11.29% |

### Rate Stability (Standard Deviation)

```
rate_stability = std_dev(last_21_funding_rates)

Interpretation:
  std_dev < 0.005%  -> "stable"
  std_dev < 0.015%  -> "moderate"
  std_dev >= 0.015% -> "volatile"
```

---

## 10. Safety Checks

### 10.1 Pre-Trade Safety Checklist

Every skill runs through these checks **in order** before producing a recommendation. A BLOCK at any step halts the pipeline immediately.

| # | Check | Tool | BLOCK Threshold | WARN Threshold | Error Code |
|---|-------|------|----------------|----------------|------------|
| 1 | MCP connectivity | `system_get_capabilities` | Server not reachable | — | `MCP_NOT_CONNECTED` |
| 2 | Authentication | `system_get_capabilities` | `authenticated: false` | — | `AUTH_FAILED` |
| 3 | Data freshness | Internal timestamp comparison | > 60s stale | > 30s stale | `DATA_STALE` |
| 4 | Funding settlement | Check `fundingTime` vs `now` | `fundingTime < now` (already passed) | — | `DATA_STALE` |

**Check Execution Flow:**
```
START
  |
  +- Check 1-2: Infrastructure   -- BLOCK? -> Output error, STOP
  |
  +- Check 3: Freshness          -- BLOCK? -> Refetch data, retry once -> BLOCK? -> STOP
  |
  +- Check 4: Settlement timing  -- BLOCK? -> Wait for next interval refresh
  |
  +- ALL PASSED -> Output recommendation with any accumulated WARNs
```

**Special case for funding-rate-arbitrage:** Verify funding settlement hasn't passed (fundingTime must be in the future).

### 10.2 funding-rate-arbitrage Risk Limits

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| max_trade_size | $5,000 | $50,000 | Maximum USD value per position |
| max_leverage | 2x | 5x | Maximum leverage for the hedge leg |
| max_concurrent_positions | 2 | 5 | Maximum simultaneous carry positions |
| min_annualized_yield | 5% | — | Minimum APY to recommend |
| review_period_days | 7 | — | Re-evaluate positions every N days |
| max_funding_rate_std_dev | 3 | — | Skip if historical funding volatility too high |

### 10.3 BLOCK Conditions

| Condition | Action | Error Code |
|-----------|--------|------------|
| Leverage requested > 2x (default) or > 5x (hard cap) | BLOCK. Cap at limit and inform user. | `LEVERAGE_EXCEEDED` |
| Trade size > $50,000 | BLOCK. Cap at limit. | `TRADE_SIZE_EXCEEDED` |
| Funding settlement already passed (fundingTime < now) | BLOCK. Wait for next interval refresh. | `DATA_STALE` |
| MCP server unreachable | BLOCK. | `MCP_NOT_CONNECTED` |

### 10.4 WARN Conditions

| Condition | Warning | Action |
|-----------|---------|--------|
| Rate flipped sign in last 24h | `[WARN] 資金費率在過去 24 小時內曾翻轉方向` | Compare `current_rate` sign vs `realized_rate` sign. If different, flag. |
| Break-even > 30 days | `[WARN] 盈虧平衡需超過 30 天，風險較高` | Display prominently. |
| Funding rate std_dev > 0.015% | `[WARN] 資金費率波動較大，實際收入可能大幅偏離預估` | Show historical range. |
| Hold without review > 7 days | `[WARN] 建議每 7 天重新評估持倉` | Recommend running `unwind-check`. |
| Annualized yield < 5% | `[WARN] 年化收益低於 5%，扣除成本後利潤空間有限` | Suggest looking at higher-yield assets. |
| 3 consecutive intervals with flipped funding | `[WARN] 資金費率已連續 3 個區間反向，強烈建議平倉` | Recommend immediate unwind. |

### 10.5 Account Safety Protocol

The system defaults to **demo mode** at all times.

**Demo Mode (Default):**
- MCP server config: `okx-DEMO-simulated-trading`
- All outputs include header: `[DEMO]`
- No real funds at risk. No confirmation required.

**Live Mode:**
- MCP server config: `okx-LIVE-trading`
- All outputs include header: `[LIVE]`
- Recommendations are still analysis-only (no auto-execution)

**Switching from Demo to Live:**
```
1. User explicitly says "live", "真實帳戶", "real account", or similar
2. System confirms with warning and asks for "確認" or "confirm"
3. User must reply with explicit confirmation
4. System verifies authentication via system_get_capabilities
5. If authenticated: switch and display [LIVE] header
6. If NOT authenticated: show AUTH_FAILED error, remain in demo
```

**Session Rules:**
- Default on startup: Always demo mode
- Timeout: Revert to demo after 30 minutes idle
- Header requirement: EVERY output must show `[DEMO]` or `[LIVE]`
- No auto-execution: `[RECOMMENDATION ONLY]` header always present

---

## 11. Error Codes & Recovery

| Code | Condition | User Message (ZH) | User Message (EN) | Recovery |
|------|-----------|-------------------|-------------------|----------|
| `MCP_NOT_CONNECTED` | okx-trade-mcp unreachable | MCP 伺服器無法連線。請確認 okx-trade-mcp 是否正在運行。 | MCP server unreachable. Check if okx-trade-mcp is running. | Verify config, restart server. |
| `AUTH_FAILED` | API key invalid or expired | API 認證失敗，請檢查 OKX API 金鑰設定 | API authentication failed. Check OKX API key configuration. | Update `~/.okx/config.toml` |
| `INSTRUMENT_NOT_FOUND` | No SWAP instrument for the asset | 找不到 {asset} 的永續合約。請確認幣種名稱。 | No perpetual swap found for {asset}. Verify the symbol. | Suggest checking OKX supported instruments. |
| `DATA_STALE` | Funding data timestamp expired | 資金費率數據已過期，正在重新獲取... | Funding rate data stale. Refetching... | Auto-retry once, then error. |
| `LEVERAGE_EXCEEDED` | Requested leverage > limit | 槓桿倍數 {requested}x 超過上限 {limit}x | Leverage {requested}x exceeds limit {limit}x | Cap at limit and inform user. |
| `TRADE_SIZE_EXCEEDED` | Trade size > hard cap | 交易金額 ${amount} 超過上限 ${limit} | Trade size ${amount} exceeds limit ${limit} | Cap at limit and inform user. |
| `RATE_LIMITED` | API rate limit hit | API 請求頻率超限，{wait}秒後重試 | API rate limit reached. Retrying in {wait}s. | Wait 1s, retry up to 3x. |
| `NOT_PROFITABLE` | Net P&L is zero or negative | 扣除所有成本後淨利潤為負（{net_pnl}），不建議執行 | Net profit is negative after all costs ({net_pnl}). Not recommended. | Show full cost breakdown |
| `MARGIN_TOO_THIN` | Profit-to-cost ratio < 2 | 利潤空間偏薄（利潤/成本比 = {ratio}），風險較高 | Thin margin (profit/cost ratio = {ratio}). Higher risk. | Display sensitivity analysis |
| `POSITION_LIMIT` | Max concurrent positions reached | 已達持倉上限（{current}/{max}），請先關閉現有倉位 | Position limit reached ({current}/{max}). Close existing positions first. | Show current open positions |
| `SECURITY_BLOCKED` | GoPlus check failed | 安全檢查未通過：{reason}。此代幣存在風險，不建議操作。 | Security check failed: {reason}. This token is flagged as risky. | Do not proceed. |
| `SECURITY_UNAVAILABLE` | GoPlus API unreachable | 安全檢查服務暫時無法使用，請謹慎操作 | Security check service unavailable. Proceed with extreme caution. | Retry once, then WARN and continue |
| `HUMAN_APPROVAL_REQUIRED` | Copy trade needs confirmation | 此操作需要您的確認才能繼續 | This action requires your explicit approval to proceed. | Wait for user confirmation |

---

## 12. Cross-Skill Integration

### Input: What This Skill Consumes

| Source Skill | Data | When |
|-------------|------|------|
| `price-feed-aggregator` | Spot price (`PriceSnapshot.venues[].price`) | Used for entry price in evaluate |
| `profitability-calculator` | `ProfitabilityResult.total_costs_usd` | Entry/exit cost for break-even calc |

### Output: What This Skill Produces

| Consuming Skill | Data Provided | Usage |
|----------------|---------------|-------|
| `profitability-calculator` | TradeLeg[] with spot + swap legs, `funding_rate`, `funding_intervals` | Full cost analysis |
| AGENTS.md Chain B | FundingRateScanResult -> profitability-calculator -> carry-pnl -> Output | Full funding harvest pipeline |

### Data Flow (Chain B from AGENTS.md)

```
1. funding-rate-arbitrage.scan
   |  -> Fetches funding rates from OKX
   |  -> Compares with borrow rates
   v
2. profitability-calculator.estimate
   |  -> Entry/exit cost analysis
   v
3. funding-rate-arbitrage.carry-pnl
   |  -> Projected income with scenarios
   v
4. Output: Ranked carry trades with projections
```

---

## 13. Output Templates (Inlined)

### 13.1 Formatting Rules

**Monetary Values:** `$12,345.67`, `+$1,234.56`, `-$89.10` (2 decimal places, comma thousands)
**Percentages:** 1 decimal place (e.g., `12.5%`). APY/APR: 2 decimal places.
**Basis Points:** Integer only (e.g., `21 bps`)
**Risk Levels:** `[SAFE]`, `[WARN]`, `[BLOCK]`
**Timestamps:** `YYYY-MM-DD HH:MM UTC` (always UTC)

**Risk Gauge:**
```
1/10:  ▓░░░░░░░░░      LOW RISK
3/10:  ▓▓▓░░░░░░░      MODERATE-LOW
5/10:  ▓▓▓▓▓░░░░░      MODERATE
7/10:  ▓▓▓▓▓▓▓░░░      ELEVATED
9/10:  ▓▓▓▓▓▓▓▓▓░      HIGH RISK
```

**Sparklines:** Characters: `▁▂▃▄▅▆▇█` (8-24 data points)

### 13.2 Global Header Template

```
══════════════════════════════════════════
  {SKILL_NAME}
  [{MODE}] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: {TIMESTAMP}
  Data sources: {DATA_SOURCES}
══════════════════════════════════════════
```

### 13.3 Funding Rate Display Template

```
── Funding Rate Overview ───────────────────

  {ASSET}-USDT-SWAP
  Current Rate:  {CURRENT_RATE} per 8h  ({ANNUALIZED} ann.)
  Next Payment:  {NEXT_TIME} ({COUNTDOWN})
  Trend (7d):    {SPARKLINE}  ({TREND_LABEL})

  ── Rate History (last 8 payments) ────────
  Time              Rate         Cumulative
  ──────────────────────────────────────────
  {T1}              {R1}         {C1}
  ...

  ── Harvest Projection ────────────────────
  Position Size:   {POSITION_SIZE}
  Daily Income:    {DAILY_INCOME}
  Entry Cost:      {ENTRY_COST}
  Break-Even:      {BREAK_EVEN_DAYS} days
  30-Day Proj:     {THIRTY_DAY_INCOME} (at current rate)
```

### 13.4 Next Steps Template

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
  Past funding rates do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
  過去資金費率不代表未來表現。
══════════════════════════════════════════
```

---

## 14. MCP Tool Reference (Inlined)

### 14.1 market_get_funding_rate

**Purpose:** Retrieve the current and predicted funding rate for perpetual swap instruments.

**Parameters:**

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | — | string | Perpetual swap instrument ID. **Must** end in `-SWAP` (e.g. `"BTC-USDT-SWAP"`). |

**Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| fundingRate | string | Current period funding rate (e.g. `"0.0001"` = 0.01%) |
| realizedRate | string | Last settled (realized) funding rate |
| fundingTime | string | Next funding settlement time in ms since epoch |
| nextFundingRate | string | Predicted next funding rate (may be empty) |
| instId | string | Instrument ID echo |

**Rate Limit:** 10 requests/second.

**Example Response:**
```json
{
  "fundingRate": "0.00015",
  "realizedRate": "0.00012",
  "fundingTime": "1741536000000",
  "nextFundingRate": "0.00018",
  "instId": "BTC-USDT-SWAP"
}
```

### 14.2 market_get_ticker

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

### 14.3 market_get_instruments

**Purpose:** List available trading instruments with contract specifications.

**Parameters:**

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instType | Yes | — | string | Instrument type: `SPOT`, `SWAP`, `FUTURES`, `OPTION` |
| instId | No | — | string | Filter by specific instrument ID |
| uly | No | — | string | Filter by underlying (e.g. `"BTC-USDT"`) |

**Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| instId | string | Instrument ID |
| instType | string | Instrument type |
| settleCcy | string | Settlement currency |
| ctVal | string | Contract value |
| lever | string | Maximum leverage available |
| tickSz | string | Tick size |
| lotSz | string | Lot size |
| minSz | string | Minimum order size |

**Rate Limit:** 20 requests/second.

### 14.4 market_get_candles

**Purpose:** Retrieve OHLCV candlestick data.

**Parameters:**

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | — | string | Instrument ID |
| bar | No | `"1m"` | string | Interval: `1m`, `5m`, `15m`, `30m`, `1H`, `4H`, `1D`, `1W` |
| limit | No | `100` | string | Number of candles. Max `100`. |

**Rate Limit:** 20 requests/second.

### 14.5 system_get_capabilities

**Purpose:** Health check.
**Parameters:** None.
**Returns:** `{ authenticated, mode, modules, serverVersion, timestamp }`

### 14.6 Important MCP Notes

- **String values** — Every numeric value from OKX is a string. Always `parseFloat()` before arithmetic.
- **Timestamps** — All in milliseconds since Unix epoch.
- **Data retention** — ~3 months max.
- **Demo mode** — When `mode: "demo"`, all data is simulated.
- **Error responses** — On failure, check for `error` field before processing.

### 14.7 Rate Limits Summary

| Tool | Rate Limit | Notes |
|------|-----------|-------|
| market_get_funding_rate | 10 req/s | Batch in groups of 8, 1s delay between batches |
| market_get_ticker | 20 req/s | Safe to batch multiple assets |
| market_get_instruments | 20 req/s | Single call, cache for session |
| market_get_candles | 20 req/s | Paginate with limit=100 for history |

---

## 15. Fee Schedule (Inlined)

### 15.1 OKX Spot Trading Fees

| Tier | 30d Volume (USD) | Maker | Taker |
|------|------------------|-------|-------|
| VIP0 | < 5M | 0.060% | 0.080% |
| VIP1 | >= 5M | 0.040% | 0.070% |
| VIP2 | >= 10M | 0.030% | 0.060% |
| VIP3 | >= 20M | 0.020% | 0.050% |
| VIP4 | >= 100M | 0.015% | 0.040% |
| VIP5 | >= 200M | 0.010% | 0.035% |

### 15.2 USDT-Margined Swap (Perpetual) Fees

| Tier | 30d Volume (USD) | Maker | Taker |
|------|------------------|-------|-------|
| VIP0 | < 5M | 0.020% | 0.050% |
| VIP1 | >= 5M | 0.015% | 0.045% |
| VIP2 | >= 10M | 0.010% | 0.040% |
| VIP3 | >= 20M | 0.008% | 0.035% |
| VIP4 | >= 100M | 0.005% | 0.030% |
| VIP5 | >= 200M | 0.002% | 0.025% |

### 15.3 Quick Reference: Round-Trip Taker Fees (bps)

| Tier | Spot RT | Swap RT |
|------|---------|---------|
| VIP0 | 16.0 bps | 10.0 bps |
| VIP1 | 14.0 bps | 9.0 bps |
| VIP2 | 12.0 bps | 8.0 bps |
| VIP3 | 10.0 bps | 7.0 bps |
| VIP4 | 8.0 bps | 6.0 bps |
| VIP5 | 7.0 bps | 5.0 bps |

**Fee Notes:**
- Maker = limit order that adds liquidity. Taker = market order or immediately matched.
- For cost estimation, assume taker fees unless specifically using limit orders.
- Fees are deducted from the received asset (spot) or from margin (derivatives).

---

## 16. Conversation Examples

### Example 1: Scan for Top Funding Rates

**User:**
> 而家邊個幣資金費率最高？

**Intent Recognition:**
- Command: `scan` (ranking funding rates)
- Assets: all (scan mode)
- Top N: 10 (default)
- Min annualized: 10% (default)

**Tool Calls:**

```
1. system_get_capabilities -> { authenticated: true, mode: "demo" }
2. market_get_instruments(instType: "SWAP")
   -> [{ instId: "BTC-USDT-SWAP" }, { instId: "ETH-USDT-SWAP" }, ...]
3. For each instId (batched, respecting 10 req/s limit):
   market_get_funding_rate(instId)
4. Filter: annualized_pct >= 10
5. Sort descending by annualized_pct
6. For top 10: market_get_ticker(instId) for income calculation
```

**Output:**

```
══════════════════════════════════════════
  Funding Rate Scanner -- SCAN
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:30 UTC
  Data sources: OKX REST (perpetual swaps)
══════════════════════════════════════════

── 資金費率掃描結果: Top 5 ────────────────

  #1  DOGE-USDT-SWAP
  +- 當前費率:      +0.0350% / 8h (38.33% ann.)
  +- 上次結算費率:  +0.0320%
  +- 方向:          positive -> 做空永續 + 做多現貨
  +- 每日收入/$10K: +$10.50
  +- 7 日趨勢:      ▃▄▅▆▆▇▇█ (trending_up)
  +- 風險:          [SAFE]

  #2  SOL-USDT-SWAP
  +- 當前費率:      +0.0250% / 8h (27.38% ann.)
  +- 上次結算費率:  +0.0230%
  +- 方向:          positive -> 做空永續 + 做多現貨
  +- 每日收入/$10K: +$7.50
  +- 7 日趨勢:      ▅▅▆▅▆▆▇▆ (stable)
  +- 風險:          [SAFE]

  ...

──────────────────────────────────────────
  已掃描 142 個永續合約
  篩選條件: 年化 >= 10%

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 深入評估 DOGE carry 交易:
     funding-rate-arbitrage evaluate --asset DOGE --size-usd 5000 --hold-days 7
  2. 查看 ETH 歷史資金費率:
     funding-rate-arbitrage rates --asset ETH --lookback-intervals 24
  3. 查看情景分析:
     funding-rate-arbitrage carry-pnl --asset SOL --size-usd 10000 --hold-days 30

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  以上僅為分析建議，不會自動執行任何交易。
══════════════════════════════════════════
```

---

### Example 2: Evaluate ETH Carry for 7 Days

**User:**
> ETH 做 carry 7 天可以賺幾多？$50,000 VIP1

**Intent Recognition:**
- Command: `evaluate`
- Asset: ETH
- Size: $50,000
- Hold days: 7
- VIP tier: VIP1

**Tool Calls:**
```
1. system_get_capabilities -> { authenticated: true, mode: "demo" }
2. market_get_funding_rate(instId: "ETH-USDT-SWAP")
3. market_get_ticker(instId: "ETH-USDT")
4. market_get_ticker(instId: "ETH-USDT-SWAP")
```

**Computation:**
```
annualized = 0.000150 * 3 * 365 * 100 = 16.43%
daily_income = 50000 * 0.000150 * 3 = $22.50
total_7d_income = 22.50 * 7 = $157.50

Entry cost (VIP1):
  spot_buy = 50000 * 0.0007 = $35.00
  perp_short = 50000 * 0.00045 = $22.50
  slippage = $5.00 + $5.00 = $10.00
  total_entry = $67.50

Exit cost = $67.50 (symmetric)
Round-trip = $135.00
Net profit = $157.50 - $135.00 = $22.50
Break-even = 135 / 22.50 = 6.0 days
```

**Output:** Full evaluate template with cost breakdown, income projection, risk gauge, and next steps.

---

### Example 3: Unwind Check

**User:**
> 我嘅 ETH carry 位已經揸咗 5 日，entry rate 係 0.015%，$10,000 位，仲值得繼續揸嗎？

**Intent Recognition:**
- Command: `unwind-check`
- Asset: ETH
- Entry rate: 0.000150 (0.015%)
- Held days: 5
- Size: $10,000

**Output:** Position summary, estimated income earned, exit cost, forward outlook with rate trend, recommendation (CONTINUE/UNWIND/REDUCE) with reasoning.

---

## 17. Implementation Notes

### Funding Rate Scan Optimization

To scan ~100+ perpetual swaps within rate limits (`market_get_funding_rate` = 10 req/s):

```
1. Fetch all SWAP instruments: market_get_instruments(instType: "SWAP")
2. Filter to USDT-margined only (instId ending "-USDT-SWAP")
3. Batch funding rate fetches in groups of 8 with 1s delay between batches
4. Cap at 50 instruments per scan (most active by volume)
5. For top candidates: fetch ticker for income calculation
```

### Rate Sign Flip Detection

Compare current and realized rates to detect flips:

```
current_sign  = parseFloat(fundingRate) >= 0 ? "positive" : "negative"
realized_sign = parseFloat(realizedRate) >= 0 ? "positive" : "negative"

if current_sign != realized_sign:
  -> Flag: "Rate flipped since last settlement"
  -> WARN level
```

### OKX String-to-Number Convention

All OKX API values are returned as **strings**. Always parse to `float` before arithmetic:

```
WRONG:  "0.000150" * 3 * 365  -> NaN or unexpected
RIGHT:  parseFloat("0.000150") * 3 * 365 -> 0.16425
```

### Rate Limit Awareness

| Tool | Rate Limit | Strategy |
|------|-----------|----------|
| `market_get_funding_rate` | 10 req/s | Batch in groups of 8, 1s delay between batches |
| `market_get_ticker` | 20 req/s | Safe to batch multiple assets |
| `market_get_instruments` | 20 req/s | Single call, cache for session |
| `market_get_candles` | 20 req/s | Paginate with limit=100 for history |

---

## 18. Reference Files

| File | Relevance to This Skill |
|------|------------------------|
| `references/formulas.md` | Funding rate formulas: Sections 3 (annualized, net carry, daily income, break-even) |
| `references/fee-schedule.md` | OKX spot + swap fee tiers (Sections 1); Pattern B: Funding Harvest cost template (Section 7) |
| `references/mcp-tools.md` | `market_get_funding_rate`, `market_get_ticker`, `market_get_instruments` specs |
| `references/output-templates.md` | Funding Rate Display Template (Section 10), Global Header, Next Steps |
| `references/safety-checks.md` | Pre-trade checklist, per-strategy risk limits (funding-rate-arbitrage section) |

> **Note:** All content from the above reference files has been inlined in this document (Sections 8-10, 13-15) for OpenClaw/Lark compatibility. The LLM can ONLY see this SKILL.md content and cannot read any other files.

---

## 19. Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2026-03-09 | 1.0.0 | Initial release. 5 commands: scan, evaluate, rates, carry-pnl, unwind-check. |
| 2026-03-09 | 1.1.0 | All external reference content inlined for OpenClaw/Lark self-contained deployment. |
