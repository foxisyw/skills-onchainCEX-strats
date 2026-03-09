---
name: basis-trading
description: >
  Spot-futures basis capture strategy. Buy spot, sell delivery futures, earn the basis premium
  as it converges to zero at expiry. Pure CEX strategy -- both legs on OKX.
  Trigger phrases include: "basis", "基差", "cash and carry", "期現套利", "futures premium",
  "期貨溢價", "term structure", "期限結構", "contango", "backwardation", "delivery futures",
  "交割合約", "到期", "roll", "展期".
  Do NOT use for: trade execution, perpetual swap funding (use funding-rate-arbitrage),
  onchain token security checks, or yield farming (use yield-optimizer).
  Requires: okx-trade-mcp (CEX-only strategy -- OnchainOS not needed).
allowed-tools: >
  okx-DEMO-simulated-trading:market_get_ticker,
  okx-DEMO-simulated-trading:market_get_instruments,
  okx-DEMO-simulated-trading:market_get_candles,
  okx-DEMO-simulated-trading:market_get_funding_rate,
  okx-DEMO-simulated-trading:system_get_capabilities,
  okx-LIVE-real-money:market_get_ticker,
  okx-LIVE-real-money:market_get_instruments,
  okx-LIVE-real-money:market_get_candles,
  okx-LIVE-real-money:market_get_funding_rate,
  okx-LIVE-real-money:system_get_capabilities
---

# basis-trading

Spot-futures basis capture skill for the Onchain x CEX Strats system. Identifies and evaluates cash-and-carry arbitrage opportunities by comparing spot prices against delivery futures prices on OKX. The strategy: **long spot + short delivery futures**, earning the basis premium as it converges to zero at settlement. Fully delta-neutral, fully collateralized.

Reuses: `Interest Information Aggregator/src/scrapers/futures_calculator.py` (implied rate and basis calculation logic).

---

## 1. Role

**Basis trade analyst** -- identifies and evaluates spot-futures basis capture opportunities across OKX delivery futures contracts.

This skill is responsible for:
- Scanning all OKX delivery futures contracts for basis premiums
- Computing annualized basis yield and net yield after fees
- Visualizing the futures term structure (contango/backwardation)
- Tracking basis convergence for open positions
- Evaluating roll decisions when contracts approach expiry

This skill does **NOT**:
- Execute any trades (analysis only, never sends orders)
- Handle perpetual swaps (delegate to `funding-rate-arbitrage`)
- Check onchain token security (CEX-only strategy, no onchain legs)
- Handle DeFi yield farming (delegate to `yield-optimizer`)
- Manage positions or portfolio state

---

## 2. Language

Match the user's language. Default: Traditional Chinese (繁體中文).

Metric labels may use English abbreviations regardless of language:
- `basis`, `contango`, `backwardation`, `DTE` (days to expiry), `ann. yield`, `bps`, `PnL`, `roll`
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
| Leverage | **1x ONLY** -- fully collateralized. No leverage allowed for basis trades. |

Even in `[LIVE]` mode, this skill only reads market data. There is no risk of accidental execution.

---

## 4. Pre-flight (Machine-Executable Checklist)

This is a **CEX-only** strategy. OnchainOS CLI is NOT required.

Run these checks **in order** before any command. BLOCK at any step halts execution.

| # | Check | Command / Tool | Success Criteria | Failure Action |
|---|-------|---------------|-----------------|----------------|
| 1 | okx-trade-mcp connected | `system_get_capabilities` (DEMO or LIVE server) | `authenticated: true`, `modules` includes `"market"` | BLOCK -- output `MCP_NOT_CONNECTED`. Tell user to verify `~/.okx/config.toml` and restart MCP server. |
| 2 | okx-trade-mcp mode | `system_get_capabilities` -> `mode` field | Returns `"demo"` or `"live"` matching expected mode | WARN -- display actual mode in header. If user requested live but got demo, surface mismatch. |
| 3 | FUTURES instruments accessible | `market_get_instruments(instType: "FUTURES")` | Returns non-empty array of delivery futures contracts | BLOCK -- output `INSTRUMENT_NOT_FOUND`. No delivery futures available. |
| 4 | Contract not expired | For specific asset: check `expTime` field from instruments | `expTime` > now + 14 days (default min) | BLOCK -- `FUTURES_EXPIRED`. Suggest next-expiry contract. |

### Pre-flight Decision Tree

```
Check 1 FAIL -> BLOCK (cannot proceed without CEX data)
Check 1 PASS -> Check 2
  Check 2 mismatch -> WARN + continue
  Check 2 PASS -> Check 3
    Check 3 FAIL -> BLOCK (no futures instruments)
    Check 3 PASS -> Check 4
      Check 4 FAIL -> BLOCK for that contract (suggest alternatives)
      Check 4 PASS -> ALL SYSTEMS GO
```

---

## 5. Skill Routing Matrix

| User Need | Use THIS Skill? | Delegate To |
|-----------|----------------|-------------|
| "BTC 期貨基差幾多" / "BTC futures basis" | Yes -- `scan` or `curve` | -- |
| "期限結構" / "term structure" | Yes -- `curve` | -- |
| "260627 到期嗰張值唔值得做基差" / "evaluate specific contract" | Yes -- `evaluate` | -- |
| "我嘅基差倉位追蹤" / "track basis position" | Yes -- `track` | -- |
| "快到期要唔要 roll" / "should I roll" | Yes -- `roll` | -- |
| "資金費率幾多" / "funding rate" | No | `funding-rate-arbitrage` |
| "CEX 同 DEX 價差" / "CEX-DEX spread" | No | `cex-dex-arbitrage` |
| "DeFi 收益比較" / "best yield" | No | `yield-optimizer` |
| "呢個幣安唔安全" / "is this token safe" | No | GoPlus MCP directly |
| "幫我開倉" / "execute trade" | No | Refuse -- no skill executes trades |

---

## 6. Command Index

| Command | Function | Read/Write | Description |
|---------|----------|-----------|-------------|
| `scan` | Scan basis across assets | Read | Fetch spot + all delivery futures, compute basis yield, rank by annualized net yield |
| `evaluate` | Deep analysis of a single contract | Read | Full cost/yield projection for a specific futures contract vs spot |
| `curve` | Term structure visualization | Read | Display all available expiries for one asset as a yield curve |
| `track` | Position convergence tracking | Read | Monitor how basis is converging for an existing position |
| `roll` | Roll evaluation | Read | Compare current position yield vs rolling to next expiry contract |

---

## 7. Parameter Reference

### 7.1 Command: `scan`

Scan all OKX delivery futures and rank by annualized net basis yield.

```bash
basis-trading scan --assets BTC,ETH --min-basis-bps 100 --contract-types delivery
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--assets` | string[] | No | All available | Any valid OKX futures underlying | Uppercase, comma-separated. If omitted, scan all assets with delivery futures. |
| `--min-basis-bps` | number | No | `50` | -- | Min: 0, Max: 5000. Minimum basis in bps to include. |
| `--min-annualized-pct` | number | No | `3` | -- | Min: 0. Minimum annualized net yield to include. |
| `--contract-types` | string | No | `"delivery"` | `delivery` | Only delivery futures. Fixed value. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | Used for net yield calculation. |
| `--min-days-to-expiry` | integer | No | `14` | -- | Min: 7 (hard cap). Filter out contracts with fewer DTE. |

#### Return Schema

```yaml
BasisScanResult:
  timestamp: integer               # Unix ms when scan was assembled
  vip_tier: string
  total_contracts_scanned: integer
  results:
    - instId: string               # e.g. "BTC-USD-260327"
      asset: string                # e.g. "BTC"
      spot_price: number           # Current spot price
      futures_price: number        # Current futures price
      basis_pct: number            # (futures - spot) / spot * 100
      basis_bps: number            # basis_pct * 100
      annualized_yield_pct: number # basis_pct * (360 / days_to_expiry) * 100
      net_yield_pct: number        # annualized_yield - annualized_fees - margin_cost
      days_to_expiry: integer
      expiry_date: string          # ISO date: "2026-03-27"
      contract_value: string       # From instrument spec
      term_structure_position: string  # "near", "mid", "far"
      risk_level: string           # "[SAFE]", "[WARN]", "[BLOCK]"
      warnings: string[]
```

#### Return Fields Detail

| Field | Type | Description |
|-------|------|-------------|
| `spot_price` | number | Current spot price from `market_get_ticker("{ASSET}-USDT")` |
| `futures_price` | number | Current delivery futures price from `market_get_ticker(instId)` |
| `basis_pct` | number | `(futures_price - spot_price) / spot_price * 100`. Positive = contango. |
| `annualized_yield_pct` | number | `basis_pct * (360 / days_to_expiry)`. Uses 360-day convention. |
| `net_yield_pct` | number | After deducting annualized entry/exit fees and margin cost. |
| `days_to_expiry` | integer | Calendar days until settlement: `(expTime - now) / 86400000` |
| `term_structure_position` | string | `"near"` (<30d), `"mid"` (30-90d), `"far"` (>90d) |

---

### 7.2 Command: `evaluate`

Deep analysis of a specific delivery futures contract vs spot.

```bash
basis-trading evaluate --asset ETH --contract ETH-USD-260627 --size-usd 10000 --vip-tier VIP1
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any valid OKX futures underlying | Uppercase. |
| `--contract` | string | No | -- | -- | Specific instId (e.g. "BTC-USD-260327"). If omitted, auto-select nearest with best yield. |
| `--size-usd` | number | No | `10,000` | -- | Min: 100, Max: 100,000 (hard cap). |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | Fee tier. |
| `--borrow-rate-annual` | number | No | `0` | -- | Min: 0. Margin borrow cost (0 if fully funded). |

#### Return Schema

```yaml
BasisEvaluateResult:
  timestamp: integer
  asset: string
  instId: string                    # e.g. "ETH-USD-260627"
  size_usd: number
  vip_tier: string

  market_data:
    spot_price: number
    futures_price: number
    spot_instId: string             # e.g. "ETH-USDT"
    futures_instId: string

  basis_analysis:
    basis_pct: number
    basis_bps: number
    basis_usd: number               # basis_pct * size_usd
    annualized_yield_pct: number
    days_to_expiry: integer
    expiry_date: string

  cost_analysis:
    entry_cost:
      spot_buy_fee: number
      futures_short_fee: number
      spot_slippage: number
      futures_slippage: number
      total_entry: number
    exit_cost:
      spot_sell_fee: number          # At expiry, sell spot. Futures settle automatically.
      total_exit: number
    total_cost: number               # entry + exit
    borrow_cost: number              # size_usd * borrow_rate * days_to_expiry / 365
    annualized_fee_drag_pct: number  # (total_cost / size_usd) * (360 / days_to_expiry) * 100

  net_yield:
    net_yield_pct: number            # annualized_yield - fee_drag - margin_cost
    net_profit_usd: number           # basis_usd - total_cost - borrow_cost
    break_even_basis_bps: number     # Min basis needed to cover costs
    profit_to_cost_ratio: number

  risk_assessment:
    risk_score: integer              # 1-10
    risk_gauge: string
    risk_label: string
    warnings: string[]

  is_profitable: boolean
  confidence: string
```

---

### 7.3 Command: `curve`

Display the futures term structure for a single asset -- all available expiries plotted as a yield curve.

```bash
basis-trading curve --asset BTC
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any valid OKX futures underlying | Uppercase. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | For net yield calculation. |
| `--include-perp` | boolean | No | `true` | `true`, `false` | Include perpetual swap basis for comparison. |

#### Return Schema

```yaml
TermStructureResult:
  timestamp: integer
  asset: string
  spot_price: number
  vip_tier: string

  contracts:
    - instId: string
      futures_price: number
      basis_pct: number
      annualized_yield_pct: number
      net_yield_pct: number
      days_to_expiry: integer
      expiry_date: string
      bar_visual: string            # e.g. "████████████████░░░░"
  perp:                              # Only if include-perp=true
    instId: string
    price: number
    basis_pct: number
    funding_rate_annualized: number  # From market_get_funding_rate

  shape: string                      # "CONTANGO", "BACKWARDATION", "FLAT", "INVERTED"
  sparkline: string                  # Yield curve sparkline: "▃▅▆▇"

  recommendation: string             # Which contract offers best risk/reward
```

---

### 7.4 Command: `track`

Track basis convergence for an existing position.

```bash
basis-trading track --asset BTC --contract BTC-USD-260327 --entry-basis-pct 0.82 --size-usd 50000
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any valid OKX futures underlying | Uppercase. |
| `--contract` | string | Yes | -- | -- | Specific instId. |
| `--entry-basis-pct` | number | Yes | -- | -- | Basis % when position was opened. |
| `--size-usd` | number | Yes | -- | -- | Position size in USD. |

#### Return Schema

```yaml
BasisTrackResult:
  timestamp: integer
  asset: string
  instId: string
  size_usd: number

  entry_basis_pct: number
  current_basis_pct: number
  basis_change_pct: number          # current - entry (should decrease toward 0)
  convergence_pct: number           # How much of the basis has been captured

  days_to_expiry: integer
  expected_remaining_pnl: number    # current_basis * size_usd
  total_expected_pnl: number        # entry_basis * size_usd

  convergence_sparkline: string     # Basis over time trending toward 0

  status: string                    # "ON_TRACK", "WIDENING" (basis increasing), "INVERTED"
  warnings: string[]
```

---

### 7.5 Command: `roll`

Evaluate whether to roll an expiring position to the next contract.

```bash
basis-trading roll --asset BTC --current-contract BTC-USD-260327 --next-contract BTC-USD-260627 --size-usd 50000 --vip-tier VIP1
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | -- | Any valid OKX futures underlying | Uppercase. |
| `--current-contract` | string | Yes | -- | -- | Expiring contract instId. |
| `--next-contract` | string | No | -- | -- | Target contract for roll. If omitted, auto-select next available. |
| `--size-usd` | number | Yes | -- | -- | Position size in USD. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | Fee tier. |

#### Return Schema

```yaml
RollEvaluateResult:
  timestamp: integer
  asset: string

  current_contract:
    instId: string
    days_to_expiry: integer
    current_basis_pct: number
    annualized_yield_pct: number
    net_yield_pct: number

  next_contract:
    instId: string
    days_to_expiry: integer
    basis_pct: number
    annualized_yield_pct: number
    net_yield_pct: number

  roll_cost:
    close_futures_fee: number       # Close current short
    open_futures_fee: number        # Open new short
    slippage: number
    total_roll_cost: number
    roll_cost_bps: number

  comparison:
    current_remaining_income: number  # From current contract until expiry
    next_net_income: number          # From next contract until its expiry (net of roll cost)
    yield_pickup_pct: number         # net_yield difference
    break_even_days: number          # roll_cost / daily_yield_difference

  recommendation: string             # "ROLL", "HOLD_TO_EXPIRY", "CLOSE"
  recommendation_reason: string
  warnings: string[]
```

---

## 8. Operation Flow

### Step 1: Intent Recognition

Parse user message to extract:

| Element | Extraction Logic | Fallback |
|---------|-----------------|----------|
| Command | Map to `scan` / `evaluate` / `curve` / `track` / `roll` | Default: `scan` |
| Asset | Extract token symbol (BTC, ETH...) | For `scan`: all. For others: **ask user**. |
| Contract | Look for instId patterns like "260327", "260627" | Auto-select best yield if omitted. |
| Size | Look for "$X", "X USDT", "X 美金" | Default: $10,000 |

**Keyword-to-command mapping:**

| Keywords | Command |
|----------|---------|
| "掃描", "scan", "邊個基差最大", "best basis", "機會" | `scan` |
| "評估", "evaluate", "值唔值得", "profitable", "分析" | `evaluate` |
| "期限結構", "term structure", "curve", "yield curve", "曲線" | `curve` |
| "追蹤", "track", "convergence", "收斂", "而家基差" | `track` |
| "展期", "roll", "到期", "expiry", "要唔要 roll" | `roll` |

### Step 2: Data Collection

#### For `scan` command:

```
1. market_get_instruments(instType: "FUTURES")
   -> Extract all delivery futures contracts
   -> Parse: instId, uly (underlying), expTime, ctVal, settleCcy

2. For each unique underlying asset:
   market_get_ticker(instId: "{ASSET}-USDT")
   -> Extract: last (spot price)

3. For each futures contract:
   market_get_ticker(instId)
   -> Extract: last (futures price)

4. Calculate basis, annualized yield, net yield for each
5. Filter by min_basis_bps and min_days_to_expiry
6. Sort by net_yield_pct descending
```

#### For `curve` command:

```
1. market_get_instruments(instType: "FUTURES", uly: "{ASSET}-USD" or "{ASSET}-USDT")
   -> All delivery contracts for this asset

2. market_get_ticker(instId: "{ASSET}-USDT")
   -> Spot price

3. For each futures contract:
   market_get_ticker(instId)
   -> Futures price

4. If include-perp=true:
   market_get_ticker(instId: "{ASSET}-USDT-SWAP")
   market_get_funding_rate(instId: "{ASSET}-USDT-SWAP")
   -> Perp price + funding rate for comparison

5. Build term structure: sort contracts by expiry date
```

**Important:** OKX returns all values as strings. Always `parseFloat()` before arithmetic.

### Step 3: Compute

#### Basis Percentage

> Reference: `references/formulas.md` Section 4 -- Basis Percentage

```
basis_pct = (futures_price - spot_price) / spot_price

Example:
  futures_price = 88,250.00  (BTC-USD-260627)
  spot_price    = 87,500.00
  basis_pct = (88250 - 87500) / 87500 = 0.00857 = 0.857%
```

#### Days to Expiry

```
days_to_expiry = (parseFloat(expTime) - Date.now()) / 86400000

Round down to integer. Filter out contracts with DTE < min_days_to_expiry.
```

#### Annualized Basis Yield

> Reference: `references/formulas.md` Section 4 -- Annualized Basis Yield

```
annualized_yield = basis_pct * (360 / days_to_expiry)

Example:
  basis_pct = 0.857%
  days_to_expiry = 110
  annualized_yield = 0.00857 * (360 / 110) = 0.02804 = 2.80%

Note: Uses 360-day convention (money market standard).
```

#### Net Yield After Fees and Margin Cost

> Reference: `references/formulas.md` Section 4 -- Net Yield After Fees and Margin Cost

```
annualized_entry_exit_fees = (entry_fee_bps + exit_fee_bps) / 10000 * (360 / days_to_expiry)
margin_cost = borrow_rate * margin_utilization  (0 if fully funded)
net_yield = annualized_yield - annualized_entry_exit_fees - margin_cost

Entry fees (basis trade):
  spot_buy (taker)     = spot_taker_rate  (from fee-schedule.md)
  futures_short (taker) = futures_taker_rate  (from fee-schedule.md)
  entry_fee_bps = (spot_taker_rate + futures_taker_rate) * 10000

Exit fees (at expiry):
  Futures settle automatically (no exit fee for futures leg)
  spot_sell (taker) = spot_taker_rate
  exit_fee_bps = spot_taker_rate * 10000

Example (VIP1):
  entry_fee_bps = (0.0007 + 0.00045) * 10000 = 11.5 bps
  exit_fee_bps  = 0.0007 * 10000 = 7.0 bps
  days_to_expiry = 110

  annualized_fees = (11.5 + 7.0) / 10000 * (360/110) = 0.00185 * 3.2727 = 0.605%
  annualized_yield = 2.80%
  margin_cost = 0% (fully funded)
  net_yield = 2.80% - 0.605% - 0% = 2.195%
```

#### Roll Yield Comparison

> Reference: `references/formulas.md` Section 4 -- Roll Yield

```
For roll decision, compare annualized net yields:
  current_net_yield = current_annualized - current_fees
  next_net_yield    = next_annualized - next_fees - roll_cost_annualized

roll_cost_annualized = (roll_cost / size_usd) * (360 / next_days_to_expiry)

If next_net_yield > current_net_yield: recommend ROLL
If next_net_yield <= current_net_yield: recommend HOLD_TO_EXPIRY
```

### Step 4: Format Output

Use output templates from `references/output-templates.md`:

- **Header:** Global Header Template (skill icon: Calendar) with mode
- **Body:** Term Structure Template (Section 6) for `curve` command
- **Footer:** Next Steps Template

---

## 9. Key Formulas

All formulas are canonical. For full derivations and worked examples, see `references/formulas.md` Section 4.

### Basis Percentage

```
basis_pct = (futures_price - spot_price) / spot_price

Positive = contango (futures > spot) -- normal market, earnable premium
Negative = backwardation (futures < spot) -- unusual, investigate
```

### Annualized Basis Yield

```
annualized_yield = basis_pct * (360 / days_to_expiry)

Uses 360-day convention (money market standard).
```

### Net Yield

```
net_yield = annualized_yield - annualized_fee_drag - margin_cost

annualized_fee_drag = (entry_fee_bps + exit_fee_bps) / 10000 * (360 / days_to_expiry)
margin_cost = borrow_rate_annual * margin_utilization
```

### Break-Even Basis

```
break_even_basis_bps = total_cost / size_usd * 10000

Minimum basis (in bps) needed for the trade to break even after all costs.
```

### Roll Yield

```
Compare net annualized yields:
  near_annualized = near_basis_pct * (360 / near_DTE) - near_fees
  far_annualized  = far_basis_pct * (360 / far_DTE) - far_fees - roll_cost_ann

Recommendation based on net yield comparison.
```

---

## 10. Safety Checks

> Cross-reference: `references/safety-checks.md` Section 2 -- basis-trading limits.

### Hard Limits

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| max_trade_size | $10,000 | $100,000 | Maximum USD value per basis trade |
| max_leverage | 1x | 1x | **No leverage** -- fully collateralized only |
| min_days_to_expiry | 14 | 7 | Minimum days until futures expiry |
| min_annualized_basis | 3% | -- | Minimum annualized basis yield to recommend |
| max_concurrent_positions | 2 | 3 | Maximum simultaneous basis positions |

### BLOCK Conditions

| Condition | Action | Error Code |
|-----------|--------|------------|
| Leverage requested > 1x | BLOCK. Basis trades must be fully collateralized. | `LEVERAGE_EXCEEDED` |
| Trade size > $100,000 | BLOCK. Cap at limit. | `TRADE_SIZE_EXCEEDED` |
| Days to expiry < 7 (hard cap) | BLOCK. Contract too close to settlement. | `FUTURES_EXPIRED` |
| Futures contract already expired | BLOCK. | `FUTURES_EXPIRED` |
| MCP server unreachable | BLOCK. | `MCP_NOT_CONNECTED` |

### WARN Conditions

| Condition | Warning | Action |
|-----------|---------|--------|
| Days to expiry < 14 (default min) | `[WARN] 距到期不足 14 天，滑點及結算風險較高` | Show prominently. Allow override if DTE >= 7. |
| Basis is negative (backwardation) | `[WARN] 基差為負 (backwardation)，非正常情況，需調查原因` | Investigate: market stress, supply squeeze, etc. |
| Net yield < 3% annualized | `[WARN] 年化淨收益低於 3%，扣除成本後利潤空間有限` | Suggest looking at longer-dated or other assets. |
| Net yield < 0% (costs exceed basis) | `[WARN] 扣除費用後淨收益為負，不建議開倉` | Show full cost breakdown. |
| Both legs not on same exchange | N/A -- this skill is OKX-only | No cross-venue risk by design. |

---

## 11. Error Codes & Recovery

| Code | Condition | User Message (ZH) | User Message (EN) | Recovery |
|------|-----------|-------------------|-------------------|----------|
| `MCP_NOT_CONNECTED` | okx-trade-mcp unreachable | MCP 伺服器無法連線。請確認 okx-trade-mcp 是否正在運行。 | MCP server unreachable. Check if okx-trade-mcp is running. | Verify config, restart server. |
| `INSTRUMENT_NOT_FOUND` | No FUTURES instrument for the asset | 找不到 {asset} 的交割期貨合約。 | No delivery futures found for {asset}. | Suggest checking OKX supported instruments. |
| `FUTURES_EXPIRED` | Contract expired or < min DTE | 期貨合約已到期或距到期不足 {min_days} 天 | Futures contract expired or < {min_days} days to expiry | Suggest next-expiry contract. |
| `LEVERAGE_EXCEEDED` | Leverage > 1x requested | 基差交易僅允許 1x 槓桿（全額保證金） | Basis trading allows 1x leverage only (fully collateralized) | Inform user this is a hard rule. |
| `TRADE_SIZE_EXCEEDED` | Trade size > hard cap | 交易金額 ${amount} 超過上限 ${limit} | Trade size ${amount} exceeds limit ${limit} | Cap at limit. |
| `DATA_STALE` | Market data too old | 市場數據已過期，正在重新獲取... | Market data stale. Refetching... | Auto-retry once, then error. |
| `RATE_LIMITED` | API rate limit hit | API 請求頻率超限，{wait}秒後重試 | API rate limit reached. Retrying in {wait}s. | Wait 1s, retry up to 3x. |

---

## 12. Cross-Skill Integration

### Input: What This Skill Consumes

| Source Skill | Data | When |
|-------------|------|------|
| `price-feed-aggregator` | Spot price (`PriceSnapshot.venues[].price`) | Used for basis calculation in evaluate |
| `profitability-calculator` | `ProfitabilityResult.total_costs_usd` | Entry/exit cost for net yield calc |

### Output: What This Skill Produces

| Consuming Skill | Data Provided | Usage |
|----------------|---------------|-------|
| `profitability-calculator` | TradeLeg[] with spot + futures legs, `hold_days` | Full cost analysis |
| AGENTS.md Chain C | BasisScanResult -> profitability-calculator -> curve -> Output | Full basis trading pipeline |

### Data Flow (Chain C from AGENTS.md)

```
1. basis-trading.scan
   |  -> Fetches spot + futures prices from OKX
   |  -> Calculates annualized basis
   v
2. profitability-calculator.estimate
   |  -> Trading fee impact
   v
3. basis-trading.curve
   |  -> Term structure visualization
   v
4. Output: Basis opportunities with yield curves
```

---

## 13. Output Templates

### 13.1 Scan Output

```
══════════════════════════════════════════
  Basis Trading Scanner -- SCAN
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: {TIMESTAMP}
  Data sources: OKX REST (spot + delivery futures)
══════════════════════════════════════════

── 基差掃描結果 ───────────────────────────

  #{RANK}  {INST_ID}
  ├─ 到期日:        {EXPIRY_DATE} ({DTE} 天)
  ├─ 現貨價格:      ${SPOT_PRICE}
  ├─ 期貨價格:      ${FUTURES_PRICE}
  ├─ 基差:          {BASIS_PCT}% ({BASIS_BPS} bps)
  ├─ 年化收益:      {ANN_YIELD}%
  ├─ 淨收益 (VIP{TIER}): {NET_YIELD}%
  └─ 風險:          {RISK_LEVEL}

  ...

──────────────────────────────────────────
  已掃描 {TOTAL} 張合約
  篩選: 基差 >= {MIN_BPS} bps, DTE >= {MIN_DTE} 天
```

### 13.2 Term Structure / Curve Output

> Uses Term Structure Template from `references/output-templates.md` Section 6.

```
══════════════════════════════════════════
  Basis Trading -- TERM STRUCTURE
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:30 UTC
  Data sources: OKX REST
══════════════════════════════════════════

── BTC 期貨期限結構 ───────────────────────

  現貨價格: $87,500.00

  合約           | 到期日      | 期貨價格      | 基差     | 年化收益   | 淨收益
  ─────────────────────────────────────────────────────────────────────────────
  BTC-USD-260327 | 2026-03-27 | $87,750.00   | +0.286% | 5.72%     | 4.10%  ████████████████░░░░
  BTC-USD-260627 | 2026-06-27 | $88,250.00   | +0.857% | 2.80%     | 2.20%  ████████░░░░░░░░░░░░
  BTC-USD-260925 | 2026-09-25 | $89,100.00   | +1.829% | 3.30%     | 2.86%  ██████████░░░░░░░░░░
  SWAP (perp)    | --         | $87,520.00   | +0.023% | 16.43%*   | --

  * 永續合約「年化」為資金費率年化，非基差收益。

  Term Structure Shape: CONTANGO
  Curve: ▃▅▆▇

── 淨收益 (VIP1 費率) ─────────────────────

  BTC-USD-260327:  4.10%  (盈虧平衡: 3.2 天)
  BTC-USD-260627:  2.20%  (盈虧平衡: 4.1 天)
  BTC-USD-260925:  2.86%  (盈虧平衡: 3.6 天)

── 建議 ───────────────────────────────────

  BTC-USD-260327 提供最高年化淨收益 4.10%，但距到期僅 18 天。
  BTC-USD-260925 提供穩定 2.86% 年化，適合較長期持倉。
  考慮風險/回報平衡，BTC-USD-260627 (110 天, 2.20%) 為中間選擇。

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 深入評估特定合約:
     basis-trading evaluate --asset BTC --contract BTC-USD-260327 --size-usd 50000
  2. 查看 ETH 期限結構:
     basis-trading curve --asset ETH
  3. 比較 funding rate carry:
     funding-rate-arbitrage evaluate --asset BTC --size-usd 50000 --hold-days 18

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Past basis levels do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
  過去基差水平不代表未來表現。
══════════════════════════════════════════
```

### 13.3 Roll Output

```
══════════════════════════════════════════
  Basis Trading -- ROLL EVALUATION
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════

── Roll 評估: BTC ─────────────────────────

  當前合約: BTC-USD-260327
  ├─ 距到期: 18 天
  ├─ 剩餘基差: +0.12%
  ├─ 年化淨收益: 2.40%
  └─ 預估剩餘收入: +$60.00

  目標合約: BTC-USD-260627
  ├─ 距到期: 110 天
  ├─ 基差: +0.857%
  ├─ 年化淨收益: 2.20%
  └─ 預估淨收入: +$385.00 (扣除 roll 成本)

── Roll 成本 ──────────────────────────────

  平倉現有期貨空單:   $22.50 (4.5 bps)
  開倉新期貨空單:     $22.50 (4.5 bps)
  滑點估計:           $10.00 (2.0 bps)
  ──────────────────────────────────────
  Roll 總成本:        $55.00 (11.0 bps)

── 比較 ───────────────────────────────────

  | 選項 | 預估淨收入 | 年化淨收益 |
  |------|-----------|-----------|
  | 持有至到期 (18d) | +$60.00 | 2.40% |
  | Roll 至 260627 (110d) | +$385.00 | 2.20% |
  | 平倉 | -$27.50 (exit cost) | -- |

  建議: HOLD_TO_EXPIRY

  原因: 當前合約年化淨收益 (2.40%) 高於 roll 後 (2.20%)，
  且僅剩 18 天。建議持有至結算，屆時再評估新倉位。

══════════════════════════════════════════
```

---

## 14. Conversation Examples

### Example 1: BTC Futures Basis Scan

**User:**
> BTC 嘅期貨基差幾多？

**Intent Recognition:**
- Command: `curve` (asking about basis for a specific asset)
- Asset: BTC
- VIP tier: VIP0 (default)

**Tool Calls:**

```
1. system_get_capabilities -> { authenticated: true, mode: "demo" }
2. market_get_instruments(instType: "FUTURES")
   -> Filter for BTC: [
        { instId: "BTC-USD-260327", expTime: "1743091200000" },
        { instId: "BTC-USD-260627", expTime: "1750982400000" },
        { instId: "BTC-USD-260925", expTime: "1758758400000" }
      ]
3. market_get_ticker(instId: "BTC-USDT")
   -> { last: "87500.0" }
4. For each futures contract:
   market_get_ticker(instId: "BTC-USD-260327") -> { last: "87750.0" }
   market_get_ticker(instId: "BTC-USD-260627") -> { last: "88250.0" }
   market_get_ticker(instId: "BTC-USD-260925") -> { last: "89100.0" }
5. market_get_ticker(instId: "BTC-USDT-SWAP") -> { last: "87520.0" }
6. market_get_funding_rate(instId: "BTC-USDT-SWAP")
   -> { fundingRate: "0.000120" }
```

**Computation:**

```
BTC-USD-260327: basis = (87750 - 87500) / 87500 = 0.286%, DTE = 18
  annualized = 0.286% * (360/18) = 5.72%
  fees (VIP0): entry=13.0 bps, exit=8.0 bps -> ann_fees = 21/10000 * 360/18 = 4.20%
  net = 5.72% - 4.20% = 1.52%

BTC-USD-260627: basis = (88250 - 87500) / 87500 = 0.857%, DTE = 110
  annualized = 0.857% * (360/110) = 2.80%
  fees: ann_fees = 21/10000 * 360/110 = 0.687%
  net = 2.80% - 0.687% = 2.11%

BTC-USD-260925: basis = (89100 - 87500) / 87500 = 1.829%, DTE = 200
  annualized = 1.829% * (360/200) = 3.29%
  fees: ann_fees = 21/10000 * 360/200 = 0.378%
  net = 3.29% - 0.378% = 2.91%
```

**Output:** (See Section 13.2 filled example above)

---

### Example 2: Evaluate a Specific Contract

**User:**
> 260627 到期嗰張 ETH 合約值唔值得做基差？$20,000 位，VIP1

**Intent Recognition:**
- Command: `evaluate`
- Asset: ETH
- Contract: ETH-USD-260627 (inferred from "260627")
- Size: $20,000
- VIP tier: VIP1

**Tool Calls:**

```
1. system_get_capabilities -> { authenticated: true, mode: "demo" }
2. market_get_instruments(instType: "FUTURES", uly: "ETH-USD")
   -> Find ETH-USD-260627: { expTime: "1750982400000", ctVal: "10", ... }
3. market_get_ticker(instId: "ETH-USDT")
   -> { last: "3412.50" }
4. market_get_ticker(instId: "ETH-USD-260627")
   -> { last: "3445.20" }
```

**Computation:**

```
basis_pct = (3445.20 - 3412.50) / 3412.50 = 0.958%
days_to_expiry = 110
annualized_yield = 0.958% * (360 / 110) = 3.13%

Entry (VIP1):
  spot_buy = 20000 * 0.0007 = $14.00
  futures_short = 20000 * 0.00045 = $9.00
  slippage = $2.00 + $2.00 = $4.00
  total_entry = $27.00

Exit (at expiry):
  futures settle automatically
  spot_sell = 20000 * 0.0007 = $14.00
  total_exit = $14.00

Total cost = $41.00
annualized_fee_drag = (41 / 20000) * (360/110) = 0.672%
net_yield = 3.13% - 0.672% = 2.46%

basis_usd = 0.00958 * 20000 = $191.60
net_profit = $191.60 - $41.00 = $150.60
break_even_basis_bps = 41 / 20000 * 10000 = 20.5 bps
profit_to_cost = 150.60 / 41.00 = 3.67x
```

**Formatted Output:**

```
══════════════════════════════════════════
  Basis Trading -- EVALUATE
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:45 UTC
══════════════════════════════════════════

── ETH-USD-260627 基差評估 ─────────────────

  現貨價格:    $3,412.50  (ETH-USDT)
  期貨價格:    $3,445.20  (ETH-USD-260627)
  距到期:      110 天 (2026-06-27)
  持倉規模:    $20,000.00

── 基差分析 ───────────────────────────────

  基差:         +0.958% (+95.8 bps)
  年化收益:     3.13%
  基差收入:     +$191.60

── 成本分析 (VIP1) ────────────────────────

  建倉:
  ├─ 現貨買入 (VIP1 taker):    $14.00  (7.0 bps)
  ├─ 期貨做空 (VIP1 taker):    $9.00   (4.5 bps)
  └─ 滑點估計:                 $4.00   (2.0 bps)

  平倉 (到期結算):
  ├─ 期貨自動結算:              $0.00
  └─ 現貨賣出 (VIP1 taker):    $14.00  (7.0 bps)

  ──────────────────────────────────────
  總成本:         $41.00  (20.5 bps)
  淨收益 (年化):  2.46%
  淨利潤:         +$150.60
  利潤/成本比:    3.67x
  最小盈利基差:   20.5 bps

── Risk Assessment ─────────────────────────

  Overall Risk:  ▓▓░░░░░░░░  2/10
                 LOW RISK

  Breakdown:
  ├─ Execution Risk:    ▓▓░░░░░░░░  2/10
  │                     Both legs on OKX; 1x leverage
  ├─ Convergence Risk:  ▓░░░░░░░░░  1/10
  │                     Basis converges to 0 at settlement by design
  └─ Duration Risk:     ▓▓▓░░░░░░░  3/10
                        110-day hold; capital locked

  Result: [PROFITABLE]

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 查看 ETH 完整期限結構:
     basis-trading curve --asset ETH
  2. 比較 funding rate carry (同期間):
     funding-rate-arbitrage evaluate --asset ETH --size-usd 20000 --hold-days 110
  3. 如已開倉，追蹤基差收斂:
     basis-trading track --asset ETH --contract ETH-USD-260627 --entry-basis-pct 0.958 --size-usd 20000

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

---

### Example 3: Roll Evaluation

**User:**
> 我嘅 BTC 基差倉位快到期 (260327)，要唔要 roll 去下一張？$50,000 位

**Intent Recognition:**
- Command: `roll`
- Asset: BTC
- Current contract: BTC-USD-260327 (inferred)
- Size: $50,000

**Tool Calls:**

```
1. system_get_capabilities -> { authenticated: true, mode: "demo" }
2. market_get_instruments(instType: "FUTURES", uly: "BTC-USD")
   -> Find BTC-USD-260327 and BTC-USD-260627
3. market_get_ticker(instId: "BTC-USDT") -> { last: "87500.0" }
4. market_get_ticker(instId: "BTC-USD-260327") -> { last: "87605.0" }
5. market_get_ticker(instId: "BTC-USD-260627") -> { last: "88250.0" }
```

**Output:** (See Section 13.3 filled example above)

---

## 15. Implementation Notes

### Instrument ID Parsing

OKX delivery futures use the format `{ASSET}-{QUOTE}-{YYMMDD}`:
- `BTC-USD-260327` = BTC coin-margined, expires 2026-03-27
- `ETH-USDT-260627` = ETH USDT-margined, expires 2026-06-27

Parse expiry date from instId:
```
date_str = instId.split("-").pop()  // "260327"
year  = 2000 + parseInt(date_str.substring(0,2))  // 2026
month = parseInt(date_str.substring(2,4))          // 03
day   = parseInt(date_str.substring(4,6))          // 27
```

### Expiry Time Calculation

The `expTime` field from `market_get_instruments` gives the exact settlement timestamp in ms.

```
days_to_expiry = Math.floor((parseFloat(expTime) - Date.now()) / 86400000)
```

Filter out contracts where `days_to_expiry < min_days_to_expiry`.

### Exit Cost at Expiry

For basis trades, the futures leg settles automatically at expiry -- **no trading fee** for the futures close. Only the spot sell incurs a fee:

```
exit_cost = spot_sell_fee + spot_slippage
         = size_usd * spot_taker_rate + size_usd * estimated_slippage_bps / 10000
```

This is different from funding-rate-arbitrage where both legs must be actively closed.

### Bar Visualization for Term Structure

Generate proportional bars for the yield curve display:

```
max_yield = max(all net_yields)
for each contract:
  filled = round(net_yield / max_yield * 20)
  empty  = 20 - filled
  bar = "█" * filled + "░" * empty
```

### OKX String-to-Number Convention

All OKX API values are returned as **strings**. Always parse to `float` before arithmetic:

```
WRONG:  "88250.0" - "87500.0"  -> NaN or string concat
RIGHT:  parseFloat("88250.0") - parseFloat("87500.0") -> 750.0
```

### Rate Limit Awareness

| Tool | Rate Limit | Strategy |
|------|-----------|----------|
| `market_get_ticker` | 20 req/s | Safe to batch. One call per contract + one for spot. |
| `market_get_instruments` | 20 req/s | Single call, cache for session. |
| `market_get_candles` | 20 req/s | Used for historical basis tracking. |
| `market_get_funding_rate` | 10 req/s | Only when include-perp=true in curve. |

For a full scan across ~20 delivery contracts: ~25 ticker calls -- well within 20 req/s limit when batched.

---

## 16. Reference Files

| File | Relevance to This Skill |
|------|------------------------|
| `references/formulas.md` | Basis trading formulas: Section 4 (basis %, annualized yield, net yield, roll yield) |
| `references/fee-schedule.md` | OKX spot + futures fee tiers (Sections 1); Pattern C: Basis Trade cost template (Section 7) |
| `references/mcp-tools.md` | `market_get_ticker`, `market_get_instruments`, `market_get_funding_rate` specs |
| `references/output-templates.md` | Term Structure Template (Section 6), Global Header, Next Steps |
| `references/safety-checks.md` | Pre-trade checklist, per-strategy risk limits (basis-trading section) |

---

## 17. Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2026-03-09 | 1.0.0 | Initial release. 5 commands: scan, evaluate, curve, track, roll. |
