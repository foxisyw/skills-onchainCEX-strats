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
| Live switch | Requires explicit user confirmation per protocol in `references/safety-checks.md` |

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
   -> Note: Use funding rate history from account_get_bills type=8 if available

5. profitability-calculator.estimate (internal call)
   -> TradeLeg[]: spot buy + perp short
   -> Returns: total entry/exit costs
```

**Important:** OKX returns all values as strings. Always `parseFloat()` before arithmetic.

### Step 3: Compute

#### Annualized Funding Rate

> Reference: `references/formulas.md` Section 3 -- Annualized Funding Rate

```
annualized_pct = parseFloat(funding_rate_per_8h) * 3 * 365 * 100

Example:
  funding_rate_per_8h = "0.000150"  (0.015% per 8h)
  annualized_pct = 0.000150 * 3 * 365 * 100
                 = 16.43%
```

#### Daily Income

> Reference: `references/formulas.md` Section 3 -- Daily Income Projection

```
daily_income = size_usd * parseFloat(funding_rate_per_8h) * 3

Example:
  size_usd = 10,000
  funding_rate_per_8h = 0.000150
  daily_income = 10000 * 0.000150 * 3 = $4.50
```

#### Estimated Daily Income per $10,000

```
estimated_daily_income_per_10k = 10000 * parseFloat(funding_rate_per_8h) * 3
```

#### Entry/Exit Cost Calculation

> Reference: `references/fee-schedule.md` Section 7, Pattern B: Funding Harvest

```
Entry:
  spot_buy_fee    = size_usd * spot_taker_rate  (fee-schedule.md)
  perp_short_fee  = size_usd * swap_taker_rate  (fee-schedule.md)
  spot_slippage   = estimated from orderbook or benchmark (~1 bps)
  perp_slippage   = estimated from orderbook or benchmark (~1 bps)
  total_entry     = spot_buy_fee + perp_short_fee + spot_slippage + perp_slippage

Exit (symmetric):
  total_exit = total_entry  (same legs, reversed)

Total round-trip = total_entry + total_exit

Example (VIP1, $50,000):
  spot_buy_fee    = 50000 * 0.0007  = $35.00
  perp_short_fee  = 50000 * 0.00045 = $22.50
  spot_slippage   = 50000 * 1/10000 = $5.00
  perp_slippage   = 50000 * 1/10000 = $5.00
  total_entry     = $67.50
  total_exit      = $67.50
  total_roundtrip = $135.00
```

#### Break-Even Holding Period

> Reference: `references/formulas.md` Section 3 -- Break-Even Holding Period

```
break_even_days = total_roundtrip / daily_income

Example:
  total_roundtrip = $135.00
  daily_income    = $22.50
  break_even_days = 135 / 22.50 = 6.0 days
```

#### Net Carry (Annualized)

> Reference: `references/formulas.md` Section 3 -- Net Carry

```
net_carry = annualized_funding - borrow_rate - annualized_trading_fees

annualized_trading_fees = (total_roundtrip / size_usd) * (365 / hold_days) * 100

Example:
  annualized_funding = 16.43%
  borrow_rate = 0% (own capital)
  annualized_trading_fees = (135 / 50000) * (365 / 30) * 100 = 3.29%
  net_carry = 16.43% - 0% - 3.29% = 13.14%
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

Use output templates from `references/output-templates.md`:

- **Header:** Global Header Template (skill icon: Clock) with mode
- **Body:** Funding Rate Display Template (Section 10) or scan results table
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

All formulas are canonical. For full derivations and worked examples, see `references/formulas.md` Section 3.

### Annualized Funding Rate

```
annualized_funding = funding_rate_per_8h * 3 * 365

Worked Example:
  Rate: 0.0150% per 8h = 0.000150
  Annualized: 0.000150 * 3 * 365 = 0.16425 = 16.43%
```

### Daily Income

```
daily_income = position_size * funding_rate_per_8h * 3

Worked Example:
  Position: $100,000 | Rate: 0.0150%
  Daily: $100,000 * 0.000150 * 3 = $45.00
```

### Break-Even Holding Period

```
break_even_days = (entry_cost + exit_cost) / daily_income

Worked Example:
  Entry: $67.50 | Exit: $67.50 | Daily income: $22.50
  Break-even: $135.00 / $22.50 = 6.0 days
```

### Net Carry

```
net_carry = annualized_funding - borrow_rate - annualized_trading_fees

Worked Example:
  Funding: 16.43% | Borrow: 5.00% | Fees: 0.14%
  Net carry: 16.43% - 5.00% - 0.14% = 11.29%
```

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

> Cross-reference: `references/safety-checks.md` Section 2 -- funding-rate-arbitrage limits.

### Hard Limits

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| max_trade_size | $5,000 | $50,000 | Maximum USD value per position |
| max_leverage | 2x | 5x | Maximum leverage for the hedge leg |
| max_concurrent_positions | 2 | 5 | Maximum simultaneous carry positions |
| min_annualized_yield | 5% | -- | Minimum APY to recommend |
| review_period_days | 7 | -- | Re-evaluate positions every N days |
| max_funding_rate_std_dev | 3 | -- | Skip if historical funding volatility too high |

### BLOCK Conditions

| Condition | Action | Error Code |
|-----------|--------|------------|
| Leverage requested > 2x (default) or > 5x (hard cap) | BLOCK. Cap at limit and inform user. | `LEVERAGE_EXCEEDED` |
| Trade size > $50,000 | BLOCK. Cap at limit. | `TRADE_SIZE_EXCEEDED` |
| Funding settlement already passed (fundingTime < now) | BLOCK. Wait for next interval refresh. | `DATA_STALE` |
| MCP server unreachable | BLOCK. | `MCP_NOT_CONNECTED` |

### WARN Conditions

| Condition | Warning | Action |
|-----------|---------|--------|
| Rate flipped sign in last 24h | `[WARN] 資金費率在過去 24 小時內曾翻轉方向` | Compare `current_rate` sign vs `realized_rate` sign. If different, flag. |
| Break-even > 30 days | `[WARN] 盈虧平衡需超過 30 天，風險較高` | Display prominently. |
| Funding rate std_dev > 0.015% | `[WARN] 資金費率波動較大，實際收入可能大幅偏離預估` | Show historical range. |
| Hold without review > 7 days | `[WARN] 建議每 7 天重新評估持倉` | Recommend running `unwind-check`. |
| Annualized yield < 5% | `[WARN] 年化收益低於 5%，扣除成本後利潤空間有限` | Suggest looking at higher-yield assets. |
| 3 consecutive intervals with flipped funding | `[WARN] 資金費率已連續 3 個區間反向，強烈建議平倉` | Recommend immediate unwind. |

---

## 11. Error Codes & Recovery

| Code | Condition | User Message (ZH) | User Message (EN) | Recovery |
|------|-----------|-------------------|-------------------|----------|
| `MCP_NOT_CONNECTED` | okx-trade-mcp unreachable | MCP 伺服器無法連線。請確認 okx-trade-mcp 是否正在運行。 | MCP server unreachable. Check if okx-trade-mcp is running. | Verify config, restart server. |
| `INSTRUMENT_NOT_FOUND` | No SWAP instrument for the asset | 找不到 {asset} 的永續合約。請確認幣種名稱。 | No perpetual swap found for {asset}. Verify the symbol. | Suggest checking OKX supported instruments. |
| `DATA_STALE` | Funding data timestamp expired | 資金費率數據已過期，正在重新獲取... | Funding rate data stale. Refetching... | Auto-retry once, then error. |
| `LEVERAGE_EXCEEDED` | Requested leverage > limit | 槓桿倍數 {requested}x 超過上限 {limit}x | Leverage {requested}x exceeds limit {limit}x | Cap at limit and inform user. |
| `TRADE_SIZE_EXCEEDED` | Trade size > hard cap | 交易金額 ${amount} 超過上限 ${limit} | Trade size ${amount} exceeds limit ${limit} | Cap at limit and inform user. |
| `RATE_LIMITED` | API rate limit hit | API 請求頻率超限，{wait}秒後重試 | API rate limit reached. Retrying in {wait}s. | Wait 1s, retry up to 3x. |

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

## 13. Output Templates

### 13.1 Scan Output

```
══════════════════════════════════════════
  Funding Rate Scanner -- SCAN
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:30 UTC
  Data sources: OKX REST (perpetual swaps)
══════════════════════════════════════════

── 資金費率掃描結果: Top {COUNT} ──────────

  #{RANK}  {ASSET}-USDT-SWAP
  ├─ 當前費率:      {RATE} / 8h ({ANNUALIZED}% ann.)
  ├─ 上次結算費率:  {REALIZED}
  ├─ 方向:          {DIRECTION} -> {RECOMMENDED_POSITION}
  ├─ 每日收入/$10K: +${DAILY_10K}
  ├─ 7 日趨勢:      {SPARKLINE} ({STABILITY})
  └─ 風險:          {RISK_LEVEL}

  ...

──────────────────────────────────────────
  已掃描 {TOTAL_SCANNED} 個永續合約
  篩選條件: 年化 >= {MIN_PCT}%
```

**Example (filled):**

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
  ├─ 當前費率:      +0.0350% / 8h (38.33% ann.)
  ├─ 上次結算費率:  +0.0320%
  ├─ 方向:          positive -> 做空永續 + 做多現貨
  ├─ 每日收入/$10K: +$10.50
  ├─ 7 日趨勢:      ▃▄▅▆▆▇▇█ (trending_up)
  └─ 風險:          [SAFE]

  #2  SOL-USDT-SWAP
  ├─ 當前費率:      +0.0250% / 8h (27.38% ann.)
  ├─ 上次結算費率:  +0.0230%
  ├─ 方向:          positive -> 做空永續 + 做多現貨
  ├─ 每日收入/$10K: +$7.50
  ├─ 7 日趨勢:      ▅▅▆▅▆▆▇▆ (stable)
  └─ 風險:          [SAFE]

  #3  ETH-USDT-SWAP
  ├─ 當前費率:      +0.0150% / 8h (16.43% ann.)
  ├─ 上次結算費率:  +0.0142%
  ├─ 方向:          positive -> 做空永續 + 做多現貨
  ├─ 每日收入/$10K: +$4.50
  ├─ 7 日趨勢:      ▃▄▅▅▆▆▇▇ (rising)
  └─ 風險:          [SAFE]

  #4  BTC-USDT-SWAP
  ├─ 當前費率:      +0.0120% / 8h (13.14% ann.)
  ├─ 上次結算費率:  +0.0115%
  ├─ 方向:          positive -> 做空永續 + 做多現貨
  ├─ 每日收入/$10K: +$3.60
  ├─ 7 日趨勢:      ▅▅▄▅▅▅▆▅ (stable)
  └─ 風險:          [SAFE]

  #5  PEPE-USDT-SWAP
  ├─ 當前費率:      +0.0480% / 8h (52.56% ann.)
  ├─ 上次結算費率:  +0.0120%
  ├─ 方向:          positive -> 做空永續 + 做多現貨
  ├─ 每日收入/$10K: +$14.40
  ├─ 7 日趨勢:      ▁▂▂▃▃▅▆█ (volatile)
  └─ 風險:          [WARN] 費率波動大，過去 24h 曾翻轉

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
  All recommendations require manual review and execution.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

### 13.2 Evaluate Output

> Uses Funding Rate Display Template from `references/output-templates.md` Section 10.

```
══════════════════════════════════════════
  Funding Rate Arbitrage -- EVALUATE
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:35 UTC
  Data sources: OKX REST
══════════════════════════════════════════

── ETH 資金費率套利評估 ───────────────────

  ETH-USDT-SWAP
  Current Rate:  +0.0150% per 8h  (16.43% ann.)
  Next Payment:  2026-03-09 16:00 UTC (1h 25m)
  Trend (7d):    ▃▄▅▅▆▆▇▇  (rising)

── 建倉成本 (VIP1, $50,000) ───────────────

  Leg 1 -- OKX Spot (Buy ETH)
  ├─ Trading Fee:    $35.00      (7.0 bps)
  └─ Slippage Est:   $5.00       (1.0 bps)

  Leg 2 -- OKX Perp (Short ETH-USDT-SWAP)
  ├─ Trading Fee:    $22.50      (4.5 bps)
  └─ Slippage Est:   $5.00       (1.0 bps)

  ──────────────────────────────────────
  Entry Cost:        $67.50      (13.5 bps)
  Exit Cost:         $67.50      (13.5 bps)
  Total Round-Trip:  $135.00     (27.0 bps)

── 收入預估 (持倉 7 天) ──────────────────

  每日收入:        +$22.50
  7 天總收入:      +$157.50
  扣除成本後淨利:  +$22.50
  年化淨收益:       9.4%
  盈虧平衡天數:    6.0 天

── Risk Assessment ─────────────────────────

  Overall Risk:  ▓▓▓░░░░░░░  3/10
                 MODERATE-LOW

  Breakdown:
  ├─ Execution Risk:   ▓▓░░░░░░░░  2/10
  │                    Both legs on OKX; deep liquidity
  ├─ Funding Risk:     ▓▓▓░░░░░░░  3/10
  │                    Rate rising trend; 0 sign flips in 7d
  └─ Holding Risk:     ▓▓▓░░░░░░░  3/10
                       7-day hold is within review period

  Result: [PROFITABLE]

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 查看情景分析 (bull/base/bear):
     funding-rate-arbitrage carry-pnl --asset ETH --size-usd 50000 --hold-days 30
  2. 比較其他幣種費率:
     funding-rate-arbitrage scan --top-n 10 --min-annualized-pct 10
  3. 如已開倉，定期檢查:
     funding-rate-arbitrage unwind-check --asset ETH --entry-rate 0.000150 --held-days 7 --size-usd 50000

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

## 14. Conversation Examples

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
   -> { fundingRate: "0.000150", realizedRate: "0.000142", fundingTime: "...", ... }
4. Filter: annualized_pct >= 10
5. Sort descending by annualized_pct
6. For top 10: market_get_ticker(instId) for income calculation
```

**Output:** (See Section 13.1 filled example above)

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
   -> { fundingRate: "0.000150", realizedRate: "0.000142", fundingTime: "1741536000000" }
3. market_get_ticker(instId: "ETH-USDT")
   -> { last: "3412.50", bidPx: "3412.10", askPx: "3412.90" }
4. market_get_ticker(instId: "ETH-USDT-SWAP")
   -> { last: "3413.20", bidPx: "3413.00", askPx: "3413.40" }
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

**Output:** (See Section 13.2 filled example above)

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

**Tool Calls:**

```
1. system_get_capabilities -> { authenticated: true, mode: "demo" }
2. market_get_funding_rate(instId: "ETH-USDT-SWAP")
   -> { fundingRate: "0.000160", realizedRate: "0.000155" }
3. market_get_candles(instId: "ETH-USDT-SWAP", bar: "8H", limit: "21")
   -> (for trend analysis over last 7 days)
```

**Formatted Output:**

```
══════════════════════════════════════════
  Funding Rate Arbitrage -- UNWIND CHECK
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:40 UTC
══════════════════════════════════════════

── ETH Carry 持倉評估 ─────────────────────

  持倉規模:     $10,000.00
  已持有:       5 天
  建倉費率:     +0.0150% / 8h
  當前費率:     +0.0160% / 8h (+6.7%)
  費率趨勢:     ▃▄▅▅▆▆▇▇ (rising)

── 已實現收益估算 ─────────────────────────

  預估已收資金費:  +$22.50  (5天 * $4.50/天)
  建倉成本:        -$13.50  (VIP0 entry)
  預估平倉成本:    -$13.50
  預估淨 PnL:      -$4.50   (尚未回本)

── 前瞻展望 ───────────────────────────────

  當前年化收益:    17.52%
  未來 7 天預估:   +$33.60 (at current rate)
  連續反向區間:    0 次 (費率一直為正)

── 建議 ───────────────────────────────────

  建議: CONTINUE

  原因: 費率持續上升 (+6.7% vs 建倉時)，無翻轉跡象。
  再持有 1-2 天即可完全覆蓋進出成本 (盈虧平衡 ~6.0 天)。
  建議下次審視: 2026-03-12 (建倉 7 天時)

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 持續持倉，7 天時重新評估:
     funding-rate-arbitrage unwind-check --asset ETH --entry-rate 0.000150 --held-days 7 --size-usd 10000
  2. 查看情景分析:
     funding-rate-arbitrage carry-pnl --asset ETH --size-usd 10000 --hold-days 30
  3. 掃描是否有更高費率機會:
     funding-rate-arbitrage scan --top-n 5

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

---

## 15. Implementation Notes

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

For consecutive flip detection (unwind-check), analyze the last N intervals from candle/bill data.

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

## 16. Reference Files

| File | Relevance to This Skill |
|------|------------------------|
| `references/formulas.md` | Funding rate formulas: Sections 3 (annualized, net carry, daily income, break-even) |
| `references/fee-schedule.md` | OKX spot + swap fee tiers (Sections 1); Pattern B: Funding Harvest cost template (Section 7) |
| `references/mcp-tools.md` | `market_get_funding_rate`, `market_get_ticker`, `market_get_instruments` specs |
| `references/output-templates.md` | Funding Rate Display Template (Section 10), Global Header, Next Steps |
| `references/safety-checks.md` | Pre-trade checklist, per-strategy risk limits (funding-rate-arbitrage section) |

---

## 17. Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2026-03-09 | 1.0.0 | Initial release. 5 commands: scan, evaluate, rates, carry-pnl, unwind-check. |
