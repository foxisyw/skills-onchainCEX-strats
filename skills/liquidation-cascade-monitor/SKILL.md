---
name: liquidation-cascade-monitor
description: >
  Monitors DeFi protocol liquidation thresholds and positions CEX perp shorts on OKX to profit
  from cascade price impact. Does NOT compete for onchain liquidation bonuses (Flashbots domain).
  Instead: detect when large DeFi positions approach liquidation -> short the collateral token
  on OKX perps to capture the cascade sell pressure.
  Trigger phrases include: "liquidation", "清算", "cascade", "連環清算", "health factor",
  "健康因子", "DeFi liquidation", "清算瀑布", "清算連鎖", "forced selling", "強制平倉",
  "liquidation wall", "清算牆", "underwater positions", "水下倉位", "清算風險",
  "cascade alert", "連鎖反應", "清算監控", "liq cascade", "清算級聯".
  Do NOT use for: trade execution, onchain liquidation bot operation, MEV extraction,
  Flashbots bundles, or basis trading (use basis-trading).
  Requires: okx-trade-mcp (CEX perp data), DeFiLlama (liquidation data), OnchainOS (whale monitoring).
allowed-tools: >
  okx-DEMO-simulated-trading:market_get_ticker,
  okx-DEMO-simulated-trading:market_get_orderbook,
  okx-DEMO-simulated-trading:market_get_funding_rate,
  okx-DEMO-simulated-trading:market_get_instruments,
  okx-DEMO-simulated-trading:system_get_capabilities,
  okx-LIVE-real-money:market_get_ticker,
  okx-LIVE-real-money:market_get_orderbook,
  okx-LIVE-real-money:market_get_funding_rate,
  okx-LIVE-real-money:market_get_instruments,
  okx-LIVE-real-money:system_get_capabilities
---

# liquidation-cascade-monitor

DeFi liquidation cascade detection and CEX perp positioning skill for the Onchain x CEX Strats system. Monitors health factors on major DeFi lending protocols (Aave V3, Compound V3, MakerDAO/Sky), detects when large positions approach liquidation thresholds, calculates expected cascade price impact, and recommends short positions on OKX perpetual swaps to capture the forced-sell pressure. **This skill does NOT compete for onchain liquidation bonuses** -- that domain requires sub-100ms latency, Flashbots integration, and dedicated MEV infrastructure. Instead, it identifies the *second-order effect*: the price impact of cascade liquidations, and positions accordingly on CEX.

Core insight: When $50M of ETH collateral faces liquidation on Aave V3, that ETH will be force-sold into DEX pools and aggregators, creating downward price pressure that propagates to CEX within seconds. This skill detects the setup *before* the cascade triggers and recommends a short perp position on OKX.

---

## 1. Role

**Liquidation cascade analyst** -- monitors DeFi lending protocol health factors, estimates cascade price impact, and recommends CEX perp positions to profit from forced-sell pressure.

This skill is responsible for:
- Scanning DeFi lending protocols for positions approaching liquidation thresholds
- Computing cascade price impact based on liquidatable collateral vs market depth
- Recommending short perp positions on OKX to capture the cascade drop
- Monitoring active cascade alerts for trigger confirmation or expiry
- Evaluating multi-protocol systemic risk scenarios

This skill does **NOT**:
- Execute any trades (analysis only, never sends orders)
- Operate onchain liquidation bots (not competing for liquidation bonus)
- Submit Flashbots bundles or MEV transactions
- Interact with DeFi smart contracts directly
- Guarantee that cascades will trigger (health factors can improve)
- Handle basis trading (delegate to `basis-trading`)
- Handle funding rate strategies (delegate to `funding-rate-arbitrage`)

### Strategic Rationale

| Approach | Latency Needed | Competition | This Skill? |
|----------|---------------|-------------|-------------|
| Onchain liquidation bot | <100ms | Extreme (Flashbots, MEV searchers) | NO |
| DEX front-running | <50ms | Extreme (MEV) | NO |
| CEX perp positioning on cascade impact | 1-30 min | Low-moderate | YES |
| Post-cascade mean reversion | 5-60 min | Low | Possible follow-up |

---

## 2. Language

Match the user's language. Default: Traditional Chinese (繁體中文).

Metric labels may use English abbreviations regardless of language:
- `health factor`, `HF`, `cascade`, `TVL`, `LTV`, `liquidation threshold`, `bps`, `PnL`
- `depth`, `impact`, `confidence`, `perp`, `SWAP`
- Timestamps always displayed in UTC

---

## 3. Account Safety

| Rule | Detail |
|------|--------|
| Default mode | Demo (`okx-DEMO-simulated-trading`) |
| Mode display | Every output header shows `[DEMO]` or `[LIVE]` |
| Read-only | This skill performs **zero** write operations -- no trades, no transfers |
| Recommendation header | Always show `[RECOMMENDATION ONLY -- 不會自動執行]` |
| Live switch | Requires explicit user confirmation (see Section 12 below) |
| Leverage | Max **5x** for cascade shorts. Higher leverage amplifies risk of false cascade. |
| Max hold time | **4 hours** hard cap. If cascade does not materialize, recommendation is to exit. |

Even in `[LIVE]` mode, this skill only reads market data. There is no risk of accidental execution.

### Account Safety Protocol (Inlined)

The system defaults to **demo mode** at all times. Switching to live mode requires explicit user action.

**Demo Mode (Default):**
- MCP server config: `okx-DEMO-simulated-trading`
- All prices and positions are simulated
- All outputs include header: `[DEMO]`
- **No confirmation required to use demo mode**

**Live Mode:**
- MCP server config: `okx-LIVE-real-money`
- Real market data and real account positions
- All outputs include header: `[LIVE]`
- Recommendations are still analysis-only (no auto-execution)

**Switching from Demo to Live:**
1. User explicitly says "live", "真實帳戶", "real account", or similar
2. System confirms the switch with a clear warning:
   - "您正在切換至真實帳戶模式。所有數據將來自您的真實 OKX 帳戶。建議仍為分析建議，不會自動執行交易。請確認：輸入 '確認' 或 'confirm' 繼續"
3. User must reply with explicit confirmation
4. System verifies authentication via `system_get_capabilities`
5. If authenticated: switch and display `[LIVE]` header
6. If NOT authenticated: show `AUTH_FAILED` error, remain in demo

**Session Rules:**

| Rule | Description |
|------|-------------|
| Default on startup | Always demo mode |
| Timeout | If no activity for 30 minutes, revert to demo mode |
| Error fallback | If live mode encounters AUTH_FAILED, revert to demo with notification |
| Header requirement | EVERY output must show `[DEMO]` or `[LIVE]` -- no exceptions |
| No auto-execution | Even in live mode, skills only provide recommendations. `[RECOMMENDATION ONLY]` header is always present. |

---

## 4. Pre-flight (Machine-Executable Checklist)

This is a **hybrid strategy** requiring both CEX data (OKX) and DeFi data (DeFiLlama / OnchainOS).

Run these checks **in order** before any command. BLOCK at any step halts execution.

| # | Check | Command / Tool | Success Criteria | Failure Action |
|---|-------|---------------|-----------------|----------------|
| 1 | okx-trade-mcp connected | `system_get_capabilities` (DEMO or LIVE server) | `authenticated: true`, `modules` includes `"market"` | BLOCK -- output `MCP_NOT_CONNECTED`. Tell user to verify `~/.okx/config.toml` and restart MCP server. |
| 2 | okx-trade-mcp mode | `system_get_capabilities` -> `mode` field | Returns `"demo"` or `"live"` matching expected mode | WARN -- display actual mode in header. If user requested live but got demo, surface mismatch. |
| 3 | SWAP instruments accessible | `market_get_instruments(instType: "SWAP")` | Returns non-empty array of perpetual swap contracts | BLOCK -- output `INSTRUMENT_NOT_FOUND`. No perpetual swaps available. |
| 4 | Target token has OKX perp | `market_get_instruments(instType: "SWAP", instId: "{TOKEN}-USDT-SWAP")` | Returns valid instrument | BLOCK -- `NO_PERP_AVAILABLE`. Cannot short token without OKX perp. |
| 5 | DeFiLlama API accessible | HTTP GET `https://api.llama.fi/protocols` | Returns JSON with protocol list | WARN -- degrade to OKX-only mode (no DeFi health factor data). |
| 6 | Orderbook depth available | `market_get_orderbook(instId: "{TOKEN}-USDT-SWAP")` | Returns non-empty `bids` and `asks` arrays | WARN -- cannot assess market depth. Use conservative sizing. |

### Pre-flight Decision Tree

```
Check 1 FAIL -> BLOCK (cannot proceed without CEX data)
Check 1 PASS -> Check 2
  Check 2 mismatch -> WARN + continue
  Check 2 PASS -> Check 3
    Check 3 FAIL -> BLOCK (no swap instruments)
    Check 3 PASS -> Check 4
      Check 4 FAIL -> BLOCK for that token (suggest alternatives with OKX perps)
      Check 4 PASS -> Check 5
        Check 5 FAIL -> WARN (degrade: no DeFi data, rely on secondary signals)
        Check 5 PASS -> Check 6
          Check 6 FAIL -> WARN (use conservative 0.3 confidence factor)
          Check 6 PASS -> ALL SYSTEMS GO
```

---

## 5. Skill Routing Matrix

| User Need | Use THIS Skill? | Delegate To |
|-----------|----------------|-------------|
| "清算監控" / "liquidation monitoring" | Yes -- `scan` or `monitor` | -- |
| "Health factor 幾多" / "what's the health factor" | Yes -- `evaluate` | -- |
| "會唔會連環清算" / "will there be a cascade" | Yes -- `evaluate` or `cascade-alert` | -- |
| "清算瀑布風險" / "cascade risk" | Yes -- `cascade-alert` | -- |
| "有冇大倉接近清算" / "any big positions near liquidation" | Yes -- `scan` | -- |
| "Aave 清算數據" / "Aave liquidation data" | Yes -- `scan --protocol aave-v3` | -- |
| "資金費率幾多" / "funding rate" | No | `funding-rate-arbitrage` |
| "CEX 同 DEX 價差" / "CEX-DEX spread" | No | `cex-dex-arbitrage` |
| "基差交易" / "basis trade" | No | `basis-trading` |
| "幫我開空單" / "open a short for me" | No | Refuse -- no skill executes trades |
| "Flashbots 清算" / "MEV liquidation bot" | No | Refuse -- not in scope |

---

## 6. Command Index

| Command | Function | Read/Write | Description |
|---------|----------|-----------|-------------|
| `scan` | Scan protocols for at-risk positions | Read | Fetch health factors from DeFi lending protocols, identify positions with HF < 1.10, rank by cascade impact |
| `evaluate` | Deep analysis of a specific cascade scenario | Read | Full impact analysis: liquidatable collateral, market depth, expected price drop, recommended position |
| `monitor` | Continuous monitoring mode | Read | Set up alert thresholds for specific tokens or protocols, report when HF crosses trigger levels |
| `cascade-alert` | Assess systemic cascade risk | Read | Multi-protocol analysis: correlated liquidations, domino effects, total forced-sell volume |

---

## 7. Parameter Reference

### 7.1 Command: `scan`

Scan DeFi lending protocols for positions approaching liquidation.

```bash
liquidation-cascade-monitor scan --protocols aave-v3,compound-v3,maker --min-collateral-usd 1000000 --max-health-factor 1.10
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--protocols` | string[] | No | `aave-v3,compound-v3,maker` | `aave-v3`, `compound-v3`, `maker`, `morpho`, `spark` | Comma-separated. Protocol identifiers matching DeFiLlama slugs. |
| `--min-collateral-usd` | number | No | `500,000` | -- | Min: 100,000. Positions below this are too small to cause meaningful cascade. |
| `--max-health-factor` | number | No | `1.10` | -- | Min: 0.5, Max: 2.0. Only show positions with HF below this value. |
| `--tokens` | string[] | No | All | Any token symbol | Filter: only show positions collateralized by these tokens. |
| `--chains` | string[] | No | `ethereum` | `ethereum`, `arbitrum`, `optimism`, `polygon`, `base`, `avalanche` | Chain filter. |
| `--sort-by` | string | No | `cascade_impact` | `cascade_impact`, `health_factor`, `collateral_usd` | Sort order for results. |

#### Return Schema

```yaml
CascadeScanResult:
  timestamp: integer               # Unix ms when scan was assembled
  protocols_scanned: string[]
  chains_scanned: string[]
  total_positions_found: integer
  results:
    - protocol: string             # e.g. "aave-v3"
      chain: string                # e.g. "ethereum"
      collateral_token: string     # e.g. "ETH"
      collateral_usd: number       # Total collateral value in USD
      borrow_token: string         # e.g. "USDC"
      borrow_usd: number           # Total borrow value in USD
      health_factor: number        # Current health factor
      liquidation_threshold: number # Protocol LTV threshold (e.g. 0.825)
      distance_to_liquidation_pct: number  # How far price must drop to trigger
      cascade_impact_pct: number   # Expected price impact if liquidated
      has_okx_perp: boolean        # Whether OKX has a perp for this token
      okx_perp_instId: string      # e.g. "ETH-USDT-SWAP" or "N/A"
      risk_level: string           # "[SAFE]", "[WARN]", "[BLOCK]"
      signal_strength: string      # "LOW", "MEDIUM", "HIGH", "CRITICAL"
      warnings: string[]
```

#### Return Fields Detail

| Field | Type | Description |
|-------|------|-------------|
| `health_factor` | number | `(collateral_value * liquidation_threshold) / borrow_value`. Below 1.0 = liquidatable. |
| `distance_to_liquidation_pct` | number | `(1 - (1 / health_factor)) * 100`. How much the collateral price must drop (%) for HF to reach 1.0. |
| `cascade_impact_pct` | number | `collateral_usd / (total_market_depth_usd * 2) * 100`. Estimated price drop from forced sell. |
| `signal_strength` | string | Based on cascade_impact: <1% = LOW, 1-2% = MEDIUM, 2-5% = HIGH, >5% = CRITICAL. |

---

### 7.2 Command: `evaluate`

Deep analysis of a specific cascade scenario with position recommendation.

```bash
liquidation-cascade-monitor evaluate --token ETH --protocol aave-v3 --collateral-at-risk-usd 5000000 --capital 50000 --leverage 3 --vip-tier VIP1
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--token` | string | Yes | -- | Any token with OKX perp | Uppercase. Must have corresponding `{TOKEN}-USDT-SWAP` on OKX. |
| `--protocol` | string | No | Auto-detect largest | `aave-v3`, `compound-v3`, `maker`, `morpho`, `spark` | Source protocol for liquidation data. |
| `--collateral-at-risk-usd` | number | No | Auto-fetch | -- | Override: manually specify at-risk collateral amount. |
| `--capital` | number | No | `10,000` | -- | Min: 100, Max: 100,000 (hard cap). Available capital for the position. |
| `--leverage` | number | No | `3` | -- | Min: 1, Max: 5 (hard cap). Leverage for the perp short. |
| `--vip-tier` | string | No | `"VIP0"` | `VIP0`..`VIP5` | OKX fee tier. |
| `--confidence` | string | No | `"moderate"` | `conservative`, `moderate`, `aggressive` | Maps to confidence_factor: 0.3, 0.5, 0.7. |
| `--max-hold-hours` | number | No | `4` | -- | Min: 1, Max: 24 (hard cap). Auto-exit recommendation time. |

#### Return Schema

```yaml
CascadeEvaluateResult:
  timestamp: integer
  token: string
  protocol: string
  mode: string                     # "DEMO" or "LIVE"

  defi_data:
    collateral_at_risk_usd: number
    health_factor: number
    liquidation_threshold: number
    distance_to_liquidation_pct: number
    liquidation_price: number      # Price at which HF reaches 1.0
    current_price: number          # Current token price

  cascade_analysis:
    forced_sell_amount_usd: number # = collateral_at_risk_usd (100% liquidated)
    market_depth_bid_usd: number   # Total bid liquidity within 2% of mid price
    cascade_impact_pct: number     # forced_sell / (depth * 2)
    expected_price_drop_usd: number # current_price * cascade_impact_pct
    signal_strength: string        # LOW / MEDIUM / HIGH / CRITICAL

  position_recommendation:
    direction: string              # "SHORT"
    instId: string                 # e.g. "ETH-USDT-SWAP"
    entry_price: number            # Current perp price
    position_size_usd: number      # Calculated per formula
    leverage: number
    margin_required_usd: number    # position_size / leverage
    stop_loss_price: number        # 2% above entry
    stop_loss_pct: number          # 2.0%
    take_profit_price: number      # cascade_impact * 0.5 below entry
    take_profit_pct: number
    max_hold_hours: number
    risk_reward_ratio: number      # take_profit_pct / stop_loss_pct

  cost_analysis:
    entry_fee_usd: number
    exit_fee_usd: number
    funding_cost_usd: number       # Estimated funding for hold period
    total_cost_usd: number
    break_even_drop_pct: number    # Min price drop to cover all costs

  scenarios:
    - label: string                # "Full Cascade", "Partial (50%)", "No Cascade"
      probability: string          # "30%", "40%", "30%"
      price_change_pct: number
      gross_pnl_usd: number
      net_pnl_usd: number         # After fees + funding

  risk_assessment:
    risk_score: integer            # 1-10
    risk_gauge: string
    risk_label: string
    warnings: string[]

  is_actionable: boolean
  confidence: string
```

---

### 7.3 Command: `monitor`

Set up continuous monitoring for specific tokens or protocols.

```bash
liquidation-cascade-monitor monitor --tokens ETH,WBTC,SOL --hf-alert 1.05 --refresh-minutes 15
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--tokens` | string[] | No | `ETH,WBTC,SOL,LINK,UNI` | Any token | Tokens to monitor for liquidation risk. |
| `--protocols` | string[] | No | `aave-v3,compound-v3,maker` | Protocol identifiers | Protocols to scan. |
| `--hf-alert` | number | No | `1.05` | -- | Min: 0.8, Max: 1.50. Alert when HF drops below this level. |
| `--min-collateral-usd` | number | No | `1,000,000` | -- | Min position size to trigger alert. |
| `--refresh-minutes` | number | No | `15` | -- | Min: 5, Max: 60. How often to re-check. |

#### Return Schema

```yaml
CascadeMonitorResult:
  timestamp: integer
  monitoring_config:
    tokens: string[]
    protocols: string[]
    hf_alert_threshold: number
    min_collateral_usd: number
    refresh_minutes: number

  current_status:
    - token: string
      protocol: string
      health_factor: number
      collateral_usd: number
      status: string               # "SAFE", "WATCH", "ALERT", "CRITICAL"
      hf_trend: string             # "IMPROVING", "STABLE", "DECLINING"
      hf_sparkline: string         # e.g. "▇▆▅▄▃" (declining)
      next_check: string           # ISO timestamp

  alerts:
    - token: string
      protocol: string
      alert_type: string           # "HF_BELOW_THRESHOLD", "RAPID_DECLINE", "CASCADE_IMMINENT"
      health_factor: number
      collateral_usd: number
      message: string
      suggested_action: string     # e.g. "Run: evaluate --token ETH --protocol aave-v3"
```

---

### 7.4 Command: `cascade-alert`

Assess systemic cascade risk across multiple protocols simultaneously.

```bash
liquidation-cascade-monitor cascade-alert --tokens ETH --include-correlated true
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--tokens` | string[] | No | All major tokens | Any token | Focus tokens. |
| `--include-correlated` | boolean | No | `true` | `true`, `false` | Include correlated assets (e.g., stETH when analyzing ETH). |
| `--scenario-drop-pct` | number | No | `10` | -- | Min: 1, Max: 50. Hypothetical price drop to stress-test. |

#### Return Schema

```yaml
CascadeAlertResult:
  timestamp: integer
  scenario: string                 # "ETH drops 10%"

  protocol_breakdown:
    - protocol: string
      token: string
      liquidatable_usd: number     # At the scenario drop level
      health_factors_breached: integer  # Number of positions that would liquidate
      pct_of_protocol_tvl: number

  aggregate:
    total_liquidatable_usd: number
    total_cascade_impact_pct: number
    systemic_risk_level: string    # "LOW", "MODERATE", "HIGH", "CRITICAL"
    domino_risk: boolean           # True if cascade from protocol A triggers cascade in protocol B

  correlated_assets:
    - token: string                # e.g. "stETH"
      correlation: number          # e.g. 0.98
      additional_liquidatable_usd: number

  recommendation:
    action: string                 # "MONITOR", "PREPARE_SHORT", "ACTIVE_SHORT", "AVOID"
    reasoning: string
    suggested_command: string      # e.g. "evaluate --token ETH --capital 20000"
```

---

## 8. Execution Flow

### Step 1: Intent Recognition

Parse user message to extract:

| Element | Extraction Logic | Fallback |
|---------|-----------------|----------|
| Command | Map to `scan` / `evaluate` / `monitor` / `cascade-alert` | Default: `scan` |
| Token | Extract token symbol (ETH, BTC, SOL...) | For `scan`: all. For `evaluate`: **ask user**. |
| Protocol | Look for protocol names (Aave, Compound, Maker...) | Default: all major protocols. |
| Capital | Look for "$X", "X USDT", "X 美金" | Default: $10,000 |
| Leverage | Look for "Xx", "X 倍" | Default: 3x |

**Keyword-to-command mapping:**

| Keywords | Command |
|----------|---------|
| "掃描", "scan", "邊個接近清算", "at-risk positions", "清算風險掃描" | `scan` |
| "評估", "evaluate", "分析清算影響", "cascade impact", "值唔值得做" | `evaluate` |
| "監控", "monitor", "追蹤", "watch", "通知我", "alert me" | `monitor` |
| "系統性風險", "cascade alert", "連環清算", "domino", "壓力測試" | `cascade-alert` |

### Step 2: Data Collection

#### For `scan` command:

```
1. DeFiLlama API: GET https://api.llama.fi/protocols
   -> Filter for lending protocols with liquidation data
   -> Extract: tvl, chainTvls, category == "Lending"

2. For each protocol, fetch liquidation data:
   DeFiLlama: GET https://api.llama.fi/v2/protocols/{protocol}/liquidations
   -> Extract positions where health_factor < max_health_factor
   -> Filter: collateral_usd >= min_collateral_usd

3. For each at-risk token, check OKX perp availability:
   market_get_instruments(instType: "SWAP")
   -> Check if {TOKEN}-USDT-SWAP exists

4. For tokens with OKX perps, get market depth:
   market_get_orderbook(instId: "{TOKEN}-USDT-SWAP")
   -> Sum bid liquidity within 2% of mid price

5. Calculate cascade_impact for each position
6. Sort by cascade_impact descending
7. Apply signal_strength labels
```

#### For `evaluate` command:

```
1. system_get_capabilities -> verify mode

2. Fetch DeFi health factor data:
   DeFiLlama liquidation API or Aave subgraph
   -> health_factor, collateral_value, borrow_value

3. market_get_ticker(instId: "{TOKEN}-USDT")
   -> Current spot price

4. market_get_ticker(instId: "{TOKEN}-USDT-SWAP")
   -> Current perp price

5. market_get_orderbook(instId: "{TOKEN}-USDT-SWAP")
   -> Bid-side depth for impact calculation

6. market_get_funding_rate(instId: "{TOKEN}-USDT-SWAP")
   -> Current funding rate for cost projection

7. Calculate:
   a. cascade_impact
   b. position_size
   c. stop_loss / take_profit
   d. cost breakdown
   e. scenario P&L
   f. risk score
```

#### For `cascade-alert` command:

```
1. system_get_capabilities -> verify mode

2. For each major protocol:
   DeFiLlama liquidation API
   -> All positions for target token(s)

3. Apply hypothetical price drop (scenario_drop_pct):
   new_health_factor = health_factor * (1 - scenario_drop_pct / 100)
   -> Positions where new_health_factor < 1.0 become liquidatable

4. Sum total_liquidatable_usd across all protocols

5. For correlated assets (e.g., stETH for ETH):
   Repeat step 3-4 with correlated token positions

6. market_get_orderbook for depth assessment

7. Calculate systemic cascade impact
8. Determine domino risk (Protocol A cascade triggers Protocol B)
```

**Important:** OKX returns all values as strings. Always `parseFloat()` before arithmetic.

### Step 3: Compute

(See Section 9 for all formulas with worked examples.)

### Step 4: Format Output

Use these output templates:

- **Header:** Global Header Template (skill icon: Warning Triangle) with mode
- **Body:** Cascade-specific templates per command
- **Footer:** Next Steps Template

(All templates are inlined in Section 13 below.)

---

## 9. Formulas (All Self-Contained with Worked Examples)

### 9.1 Health Factor

```
health_factor = (collateral_value_usd * liquidation_threshold) / borrow_value_usd

Variables:
  collateral_value_usd  = Current market value of deposited collateral
  liquidation_threshold = Protocol-specific LTV at which liquidation triggers
  borrow_value_usd      = Current value of outstanding borrows

When health_factor < 1.0 -> liquidation is triggered.
When health_factor = 1.0 -> exactly at liquidation boundary.
When health_factor > 1.0 -> position is healthy (higher = safer).

Protocol Liquidation Thresholds (reference):
  Aave V3 ETH:    0.860  (86% LTV)
  Aave V3 WBTC:   0.780  (78% LTV)
  Compound V3 ETH: 0.830  (83% LTV)
  Maker ETH-A:    0.667  (66.7% LTV, i.e. 150% collateralization ratio)
  Maker ETH-C:    0.588  (58.8% LTV, i.e. 170% collateralization ratio)

Worked Example:
  collateral: 1,500 ETH at $3,400 = $5,100,000
  liquidation_threshold: 0.860 (Aave V3 ETH)
  borrow: $3,800,000 USDC

  health_factor = (5,100,000 * 0.860) / 3,800,000
               = 4,386,000 / 3,800,000
               = 1.154

  This position is healthy (HF > 1.0) but within monitoring range (HF < 1.20).
```

### 9.2 Distance to Liquidation

```
distance_to_liquidation_pct = (1 - (1 / health_factor)) * 100

Variables:
  health_factor = Current health factor of the position

Interpretation: The collateral token must drop by this percentage for HF to reach 1.0.

Worked Example:
  health_factor = 1.154

  distance_to_liquidation_pct = (1 - (1 / 1.154)) * 100
                              = (1 - 0.8666) * 100
                              = 0.1334 * 100
                              = 13.34%

  ETH must drop 13.34% (from $3,400 to ~$2,946) for this position to be liquidated.
```

### 9.3 Liquidation Price

```
liquidation_price = current_price * (1 - distance_to_liquidation_pct / 100)

Worked Example:
  current_price = $3,400.00
  distance_to_liquidation_pct = 13.34%

  liquidation_price = 3400 * (1 - 0.1334)
                    = 3400 * 0.8666
                    = $2,946.44
```

### 9.4 Cascade Price Impact

```
cascade_price_impact_pct = forced_sell_amount_usd / (pool_liquidity_usd * 2) * 100

Variables:
  forced_sell_amount_usd = Total collateral that will be force-sold upon liquidation
  pool_liquidity_usd     = Total bid-side liquidity within 2% of mid price on the deepest venue

The "* 2" accounts for the AMM constant-product curve / orderbook bilateral depth.
This is a simplified model. Real impact depends on:
  - Liquidation split across multiple DEX pools
  - Aggregator routing efficiency
  - Time for arbitrageurs to rebalance
Actual impact is typically 40-60% of this estimate due to distributed liquidation.

Worked Example:
  forced_sell_amount = $5,000,000 (1,500 ETH liquidated)
  pool_liquidity (2% depth) = $50,000,000

  cascade_price_impact = 5,000,000 / (50,000,000 * 2) * 100
                       = 5,000,000 / 100,000,000 * 100
                       = 5.0%

  Expected cascade price drop: ~5.0%
  Conservative estimate (60% realized): ~3.0%
```

### 9.5 Signal Strength Classification

```
signal_strength:
  cascade_impact < 1.0%   -> "LOW"     (not worth trading)
  cascade_impact 1.0-2.0% -> "MEDIUM"  (monitor closely)
  cascade_impact 2.0-5.0% -> "HIGH"    (actionable signal)
  cascade_impact > 5.0%   -> "CRITICAL" (strong signal, but also higher risk)
```

### 9.6 Position Sizing

```
position_size_usd = min(max_position_size, cascade_impact_pct * capital * confidence_factor)

Variables:
  max_position_size  = capital * leverage (hard cap: $100,000)
  cascade_impact_pct = Expected price impact from forced sell (decimal, e.g. 0.05 for 5%)
  capital            = Available capital in USD
  confidence_factor  = 0.3 (conservative), 0.5 (moderate), 0.7 (aggressive)
  leverage           = 1-5x (default 3x)

The position is intentionally conservative: we don't bet the full expected impact.
At 0.5 confidence, we size as if the cascade will only produce 50% of the theoretical impact.

Worked Example (moderate confidence):
  cascade_impact_pct = 5.0% = 0.05
  capital = $50,000
  confidence_factor = 0.5
  leverage = 3x
  max_position_size = 50,000 * 3 = $150,000 -> capped at $100,000

  position_size = min(100000, 0.05 * 50000 * 0.5)
               = min(100000, 1250)
               = $1,250.00

  margin_required = 1250 / 3 = $416.67

  Note: Position size is deliberately small relative to capital.
  This reflects the speculative nature of cascade timing.
```

### 9.7 Stop Loss

```
stop_loss_price = entry_price * (1 + stop_loss_pct / 100)

For SHORT positions: stop loss is ABOVE entry price (price went up = loss).

Default stop_loss_pct: 2.0%

Rationale: If price moves 2% against us, the cascade thesis is likely wrong
(health factors may have been repaid, or market is rallying, which would
push health factors UP, not down).

Worked Example:
  entry_price = $3,400.00
  stop_loss_pct = 2.0%

  stop_loss_price = 3400 * (1 + 0.02)
                  = 3400 * 1.02
                  = $3,468.00

  Max loss = position_size * stop_loss_pct
           = 1250 * 0.02
           = $25.00
```

### 9.8 Take Profit

```
take_profit_price = entry_price * (1 - take_profit_pct / 100)

take_profit_pct = cascade_impact_pct * 0.5 * 100

For SHORT positions: take profit is BELOW entry price (price went down = profit).

We target 50% of the theoretical cascade impact. Reasoning:
  - Not all liquidations execute simultaneously
  - Arbitrageurs will partially absorb the impact
  - We want to capture the "easy" half of the move

Worked Example:
  entry_price = $3,400.00
  cascade_impact_pct = 5.0% (0.05)
  take_profit_pct = 5.0% * 0.5 = 2.5%

  take_profit_price = 3400 * (1 - 0.025)
                    = 3400 * 0.975
                    = $3,315.00

  Expected profit = position_size * take_profit_pct
                  = 1250 * 0.025
                  = $31.25

  Risk/Reward ratio = take_profit_pct / stop_loss_pct
                    = 2.5% / 2.0%
                    = 1.25x
```

### 9.9 Funding Cost Estimation

```
funding_cost_usd = position_size_usd * funding_rate_per_8h * (hold_hours / 8)

Variables:
  position_size_usd       = Notional size of the perp position
  funding_rate_per_8h     = Current OKX funding rate (settled every 8 hours)
  hold_hours              = Expected hold duration

Note: Funding is paid by longs to shorts when rate is positive,
and by shorts to longs when rate is negative.
For our SHORT position:
  - Positive funding rate = we RECEIVE funding (reduces cost)
  - Negative funding rate = we PAY funding (adds cost)

Worked Example:
  position_size = $1,250.00
  funding_rate = 0.0001 (0.01% per 8h) -- positive, so we receive
  hold_hours = 4

  funding_cost = 1250 * 0.0001 * (4 / 8)
               = 1250 * 0.0001 * 0.5
               = $0.0625

  Since rate is positive, we receive $0.06. Actual cost = -$0.06.
  Net effect: reduces our trading cost slightly.
```

### 9.10 Total Cost and Break-Even

```
total_cost_usd = entry_fee + exit_fee + abs(funding_cost)  [if funding is against us]

entry_fee = position_size * taker_rate
exit_fee  = position_size * taker_rate

break_even_drop_pct = (total_cost_usd / position_size_usd) * 100

Worked Example (VIP0):
  position_size = $1,250.00
  taker_rate (USDT-Margined Swap VIP0) = 0.050% = 0.0005

  entry_fee = 1250 * 0.0005 = $0.625
  exit_fee  = 1250 * 0.0005 = $0.625
  funding_cost = -$0.06 (we receive, so net benefit)

  total_cost = 0.625 + 0.625 + 0 = $1.25
  (funding received offsets, but we don't count it toward cost reduction for safety)

  break_even_drop_pct = (1.25 / 1250) * 100
                      = 0.10%

  Token price must drop at least 0.10% to break even after fees.
```

### 9.11 Scenario P&L Table

```
For each scenario, compute:
  gross_pnl = position_size * price_change_pct * direction_multiplier
  net_pnl   = gross_pnl - total_cost

  direction_multiplier: -1 for SHORT (profit when price drops)

Scenarios:
  1. Full Cascade:   price drops by cascade_impact_pct
  2. Partial (50%):  price drops by cascade_impact_pct * 0.5
  3. No Cascade:     price unchanged, exit at max_hold time
  4. Adverse (stop): price rises by stop_loss_pct, stopped out

Worked Example (position = $1,250, cascade_impact = 5.0%, cost = $1.25):

  Scenario 1: Full Cascade (-5.0%)
    gross_pnl = 1250 * 0.05 = +$62.50
    net_pnl   = 62.50 - 1.25 = +$61.25

  Scenario 2: Partial Cascade (-2.5%)
    gross_pnl = 1250 * 0.025 = +$31.25
    net_pnl   = 31.25 - 1.25 = +$30.00

  Scenario 3: No Cascade (0%)
    gross_pnl = $0
    net_pnl   = 0 - 1.25 = -$1.25

  Scenario 4: Adverse (+2.0%, stopped out)
    gross_pnl = 1250 * (-0.02) = -$25.00
    net_pnl   = -25.00 - 1.25 = -$26.25
```

---

## 10. OKX Fee Schedule (Inlined)

### USDT-Margined Swap / Perpetual Fees

| Tier | 30d Volume (USD) | Maker | Taker |
|------|------------------|-------|-------|
| VIP0 | < 5M | 0.020% | 0.050% |
| VIP1 | >= 5M | 0.015% | 0.045% |
| VIP2 | >= 10M | 0.010% | 0.040% |
| VIP3 | >= 20M | 0.008% | 0.035% |
| VIP4 | >= 100M | 0.005% | 0.030% |
| VIP5 | >= 200M | 0.002% | 0.025% |

### Fee Calculation Notes

- **Maker** = limit order that adds liquidity to the orderbook (not immediately matched)
- **Taker** = market order or limit order that immediately matches
- For **cost estimation**, assume taker fees (cascade trades require speed, not limit orders)
- Fees are deducted from margin (derivatives)
- Funding rate is settled every 8 hours on OKX perpetual swaps

### Fee Formula

```
fee_usd = notional_size_usd * fee_rate

# Round-trip cost (open + close)
roundtrip_fee_usd = notional_size_usd * (entry_fee_rate + exit_fee_rate)
roundtrip_fee_bps = (entry_fee_rate + exit_fee_rate) * 10000
```

### Quick Reference: Round-Trip Taker Fees (bps)

| Tier | Swap RT (open+close) | On $1,000 position | On $10,000 position |
|------|---------------------|--------------------|--------------------|
| VIP0 | 10.0 bps | $1.00 | $10.00 |
| VIP1 | 9.0 bps | $0.90 | $9.00 |
| VIP2 | 8.0 bps | $0.80 | $8.00 |
| VIP3 | 7.0 bps | $0.70 | $7.00 |
| VIP4 | 6.0 bps | $0.60 | $6.00 |
| VIP5 | 5.0 bps | $0.50 | $5.00 |

### Pattern: Cascade Short Cost Template

```
Position size: $1,250 (SHORT, 3x leverage, margin = $416.67)

Perp short entry (VIP0 taker):    $0.625  (5.0 bps)
Perp close (VIP0 taker):          $0.625  (5.0 bps)
Funding (4h, rate 0.01%):         -$0.063 (received)
                                   ------
Total cost:                        $1.19   (9.5 bps net)
Break-even price drop:             0.10%
```

---

## 11. MCP Tool Specifications (Inlined)

### market_get_ticker

**Purpose:** Retrieve real-time ticker data for a single instrument -- last price, bid/ask, 24h volume and price range.

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | -- | string | Instrument ID (e.g. `"ETH-USDT"`, `"ETH-USDT-SWAP"`) |

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

Rate Limit: 20 requests/second.

### market_get_orderbook

**Purpose:** Retrieve orderbook depth for an instrument -- bid/ask levels with size.

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | -- | string | Instrument ID (e.g. `"ETH-USDT-SWAP"`) |
| sz | No | `"1"` | string | Book depth. `"1"` = top-of-book, `"5"` = 5 levels, `"20"` or `"400"` for deep book. |

**Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| bids | array | Array of `[price, size, deprecated, numOrders]` sorted best-to-worst |
| asks | array | Array of `[price, size, deprecated, numOrders]` sorted best-to-worst |
| ts | string | Timestamp in ms |

**Depth Calculation:**
```
To compute bid-side liquidity within X% of mid:
  mid_price = (best_bid + best_ask) / 2
  threshold = mid_price * (1 - X / 100)
  depth_usd = sum(price * size for [price, size, _, _] in bids if price >= threshold)
```

Rate Limit: 20 requests/second. Use `sz: "400"` for depth calculations.

### market_get_funding_rate

**Purpose:** Retrieve the current and predicted funding rate for perpetual swap instruments.

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | -- | string | Perpetual swap instrument ID. **Must** end in `-SWAP` (e.g. `"ETH-USDT-SWAP"`). |

**Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| fundingRate | string | Current period funding rate (e.g. `"0.0001"` = 0.01%) |
| realizedRate | string | Last settled (realized) funding rate |
| fundingTime | string | Next funding settlement time in ms since epoch |
| nextFundingRate | string | Predicted next funding rate (may be empty) |
| instId | string | Instrument ID echo |

Rate Limit: 10 requests/second.

### market_get_instruments (SWAP)

**Purpose:** List available perpetual swap instruments with contract specifications.

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instType | Yes | -- | string | Instrument type. For cascade trading: `"SWAP"` |
| instId | No | -- | string | Filter by specific instrument ID |
| uly | No | -- | string | Filter by underlying |

**Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| instId | string | Instrument ID (e.g. `"ETH-USDT-SWAP"`) |
| instType | string | Instrument type |
| uly | string | Underlying index |
| settleCcy | string | Settlement currency |
| ctVal | string | Contract value (face value of one contract) |
| lever | string | Maximum leverage available |
| tickSz | string | Tick size (minimum price increment) |
| lotSz | string | Lot size (minimum order quantity increment) |
| minSz | string | Minimum order size |

Rate Limit: 20 requests/second.

### system_get_capabilities

**Purpose:** Check MCP server connectivity, authentication status, and available modules.

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| (none) | -- | -- | -- | No parameters needed |

**Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| authenticated | boolean | Whether API key is valid |
| mode | string | `"demo"` or `"live"` |
| modules | string[] | Available modules (e.g. `["market", "trade", "account"]`) |

**Important Notes:**
- Every numeric value from OKX is a **string**. Always use `parseFloat()` before arithmetic.
- All timestamps are in **milliseconds** since Unix epoch.
- Instrument ID format for perps: `{TOKEN}-USDT-SWAP` (e.g. `"ETH-USDT-SWAP"`).

---

## 12. Safety Checks

### Pre-Trade Safety Checklist

Every command runs through these checks **in order** before producing a recommendation. A BLOCK at any step halts the pipeline immediately.

| # | Check | Tool | BLOCK Threshold | WARN Threshold | Error Code |
|---|-------|------|----------------|----------------|------------|
| 1 | MCP connectivity | `system_get_capabilities` | Server not reachable | -- | `MCP_NOT_CONNECTED` |
| 2 | Authentication | `system_get_capabilities` | `authenticated: false` | -- | `AUTH_FAILED` |
| 3 | Data freshness | Internal timestamp comparison | > 60s stale | > 30s stale | `DATA_STALE` |
| 4 | OKX perp exists | `market_get_instruments` | No SWAP for target token | -- | `NO_PERP_AVAILABLE` |
| 5 | Cascade impact threshold | Cascade calculation | `cascade_impact < 1.0%` | `cascade_impact 1.0-2.0%` | `CASCADE_TOO_SMALL` |
| 6 | Health factor already critical | DeFi data | `health_factor < 0.5` | `health_factor < 0.8` | `CASCADE_IN_PROGRESS` |
| 7 | Multi-protocol systemic risk | Cross-protocol scan | -- | Multiple protocols cascading | `SYSTEMIC_RISK` |

### Cascade-Specific Safety Rules

| Check | Threshold | Action | Rationale |
|-------|-----------|--------|-----------|
| Cascade impact < 1.0% | SKIP | `[SAFE]` -- do NOT recommend a position | Too small to overcome fees and slippage |
| Cascade impact 1.0-2.0% | MEDIUM signal | `[WARN]` -- position with conservative sizing (0.3 factor) | Marginal opportunity, tight risk |
| Cascade impact 2.0-5.0% | HIGH signal | `[SAFE]` -- position with moderate sizing (0.5 factor) | Sweet spot for this strategy |
| Cascade impact > 5.0% | CRITICAL signal | `[WARN]` -- position with aggressive sizing (0.7 factor) but add caution | May indicate systemic event, higher tail risk |
| No OKX perp for token | BLOCK | `[BLOCK]` -- cannot hedge | Only trade tokens with OKX perpetual swaps |
| Health factor already < 0.5 | Late entry warning | `[WARN]` -- cascade likely in progress, may be too late | Price impact may already be priced in |
| Health factor already < 1.0 | Liquidation active | `[WARN]` -- liquidation is happening NOW | Enter only if impact not yet reflected in CEX price |
| Multiple protocols cascading | Systemic risk | REDUCE position size by 50% | Correlated cascades = unpredictable dynamics |
| Leverage > 5x requested | BLOCK | `[BLOCK]` -- hard cap at 5x | Cascade timing is uncertain; high leverage amplifies false-signal losses |
| Position size > $100,000 | BLOCK | `[BLOCK]` -- cap at limit | Risk management hard cap |
| Hold time > 24 hours | BLOCK | `[BLOCK]` -- cap at 24h | Cascade is a time-sensitive event; stale positions accumulate risk |

### Risk Scoring Matrix

| Factor | Weight | 1-2 (Low) | 3-5 (Moderate) | 6-8 (Elevated) | 9-10 (High) |
|--------|--------|-----------|----------------|-----------------|-------------|
| Cascade Certainty | 30% | HF < 1.02, large position, single protocol | HF 1.02-1.05, medium position | HF 1.05-1.10, small position | HF > 1.10, speculative |
| Market Depth | 25% | Deep book (>$100M 2% depth) | Moderate ($20-100M) | Thin ($5-20M) | Very thin (<$5M) |
| Timing Risk | 25% | HF declining rapidly | HF slowly declining | HF stable | HF improving |
| Execution Risk | 20% | Major token (BTC, ETH), tight spread | Mid-cap, moderate spread | Small-cap, wide spread | Exotic, illiquid |

```
risk_score = sum(factor_score * factor_weight) for each factor
  Round to nearest integer, clamp to [1, 10]

Worked Example:
  Cascade Certainty: 3 (HF = 1.04, decent size)     -> 3 * 0.30 = 0.90
  Market Depth: 2 (ETH, deep book)                   -> 2 * 0.25 = 0.50
  Timing Risk: 4 (HF slowly declining)               -> 4 * 0.25 = 1.00
  Execution Risk: 2 (ETH-USDT-SWAP, tight spread)    -> 2 * 0.20 = 0.40

  risk_score = 0.90 + 0.50 + 1.00 + 0.40 = 2.80 -> rounded to 3
  risk_label = "MODERATE-LOW"
  risk_gauge = "▓▓▓░░░░░░░ 3/10"
```

### BLOCK Conditions

| Condition | Action | Error Code |
|-----------|--------|------------|
| Leverage requested > 5x | BLOCK. Hard cap for cascade trades. | `LEVERAGE_EXCEEDED` |
| Position size > $100,000 | BLOCK. Cap at limit. | `TRADE_SIZE_EXCEEDED` |
| No OKX perp for target token | BLOCK. Cannot short without perp. | `NO_PERP_AVAILABLE` |
| MCP server unreachable | BLOCK. | `MCP_NOT_CONNECTED` |
| Cascade impact < 1.0% | BLOCK recommendation (still show data). | `CASCADE_TOO_SMALL` |
| Hold time > 24 hours | BLOCK. Cascade is time-sensitive. | `HOLD_TIME_EXCEEDED` |

### WARN Conditions

| Condition | Warning | Action |
|-----------|---------|--------|
| Health factor < 0.5 | `[WARN] 健康因子 < 0.5，清算可能已在進行中，入場或許太遲` | Show prominently. May be too late. |
| Health factor < 1.0 | `[WARN] 健康因子 < 1.0，清算正在發生，需確認 CEX 價格是否已反映` | Check if CEX price already dropped. |
| Cascade impact 1.0-2.0% | `[WARN] 級聯影響僅 {X}%，扣除成本後利潤空間有限` | Use conservative sizing (0.3 factor). |
| Cascade impact > 5.0% | `[WARN] 級聯影響達 {X}%，可能為系統性事件，尾部風險較高` | Add systemic risk warning. |
| Multiple protocols affected | `[WARN] 多個協議同時面臨清算，系統性風險升高，倉位縮小 50%` | Halve position size. |
| Negative funding rate | `[WARN] 資金費率為負 ({rate}%)，空單需支付資金費用` | Add funding cost to total cost projection. |
| DeFiLlama API unavailable | `[WARN] DeFi 數據源不可用，僅使用 OKX 數據，信號可靠度降低` | Degrade to OKX-only analysis. |

---

## 13. Output Templates (Inlined)

### Global Header Template

Used at the top of every skill output:

```
══════════════════════════════════════════
  Liquidation Cascade Monitor
  [{MODE}] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: {TIMESTAMP}
  Data sources: {DATA_SOURCES}
══════════════════════════════════════════
```

For liquidation-cascade-monitor: Data sources = "DeFiLlama + OKX REST (perps + orderbook)"

### Formatting Rules

- Monetary: `+$1,234.56` or `-$1,234.56` (2 decimals, comma thousands)
- Percentages: `12.5%` (1 decimal), HF: 3 decimal places (e.g. `1.154`)
- Basis Points: integer only (e.g., `21 bps`)
- Risk Levels: `[SAFE]`, `[WARN]`, `[BLOCK]`
- Risk Gauge: `▓▓▓░░░░░░░ 3/10` (scale 1-10)
- Sparklines: `▁▂▃▄▅▆▇█` for trend visualization
- Signal Strength: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`
- Timestamps: `YYYY-MM-DD HH:MM UTC`
- Health Factor: 3 decimal places (e.g. `1.154`)

### Risk Gauge Template

```
── Risk Assessment ─────────────────────────

  Overall Risk:  {RISK_GAUGE}  {RISK_SCORE}/10
                 {RISK_LABEL}

  Breakdown:
  ├─ Cascade Certainty: {CC_GAUGE}  {CC_SCORE}/10
  │                     {CC_NOTE}
  ├─ Market Depth:      {MD_GAUGE}  {MD_SCORE}/10
  │                     {MD_NOTE}
  ├─ Timing Risk:       {TR_GAUGE}  {TR_SCORE}/10
  │                     {TR_NOTE}
  └─ Execution Risk:    {ER_GAUGE}  {ER_SCORE}/10
                        {ER_NOTE}
```

Gauge format: `1/10: ▓░░░░░░░░░`, `5/10: ▓▓▓▓▓░░░░░`, `10/10: ▓▓▓▓▓▓▓▓▓▓`
Risk labels: 1-2 LOW RISK, 3-4 MODERATE-LOW, 5-6 MODERATE, 7-8 ELEVATED, 9-10 HIGH RISK

### 13.1 Scan Output

```
══════════════════════════════════════════
  Liquidation Cascade Monitor -- SCAN
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: {TIMESTAMP}
  Data sources: DeFiLlama + OKX REST
══════════════════════════════════════════

── 清算風險掃描結果 ──────────────────────

  掃描: {PROTOCOLS}
  篩選: 抵押品 >= ${MIN_COLLATERAL}, HF <= {MAX_HF}

  #{RANK}  {PROTOCOL} / {CHAIN}
  ├─ 抵押品:           {COLLATERAL_TOKEN} (${COLLATERAL_USD})
  ├─ 借款:             {BORROW_TOKEN} (${BORROW_USD})
  ├─ 健康因子:         {HEALTH_FACTOR}
  ├─ 距清算:           {DISTANCE_PCT}% ({DISTANCE_NOTE})
  ├─ 清算價格:         ${LIQUIDATION_PRICE}
  ├─ 級聯影響:         {CASCADE_IMPACT}%
  ├─ 信號強度:         {SIGNAL_STRENGTH}
  ├─ OKX 永續:         {HAS_PERP} ({PERP_INSTID})
  └─ 風險:             {RISK_LEVEL}

  ...

──────────────────────────────────────────
  已掃描 {TOTAL_POSITIONS} 個倉位
  符合條件: {MATCHING} 個
  可交易 (有 OKX 永續): {TRADEABLE} 個
```

### 13.2 Evaluate Output

```
══════════════════════════════════════════
  Liquidation Cascade Monitor -- EVALUATE
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:30 UTC
  Data sources: DeFiLlama + OKX REST (perps + orderbook)
══════════════════════════════════════════

── DeFi 清算數據 ─────────────────────────

  協議:         Aave V3 (Ethereum)
  抵押品:       1,500 ETH ($5,100,000.00)
  借款:         $3,800,000.00 USDC
  健康因子:     1.154
  清算門檻:     0.860 (86% LTV)
  距清算:       13.3% ($453,960)
  清算價格:     $2,946.44

── 級聯影響分析 ──────────────────────────

  強制賣出量:    $5,100,000 (1,500 ETH)
  市場深度 (2%): $50,000,000 (ETH-USDT-SWAP bid side)
  理論影響:      5.0%
  保守估計 (60%): 3.0%
  信號強度:      HIGH

── 建議倉位 ──────────────────────────────

  方向:         SHORT (做空)
  合約:         ETH-USDT-SWAP
  入場價格:     $3,400.00
  倉位大小:     $1,250.00 (moderate confidence)
  槓桿:         3x
  所需保證金:   $416.67
  止損:         $3,468.00 (+2.0%)
  止盈:         $3,315.00 (-2.5%)
  最長持有:     4 小時
  風險/回報:    1.25x

── 成本分析 (VIP0) ──────────────────────

  開倉 (taker):     $0.63  (5.0 bps)
  平倉 (taker):     $0.63  (5.0 bps)
  資金費用 (4h):    -$0.06 (收取)
  ──────────────────────────────────────
  總成本:           $1.19  (9.5 bps)
  盈虧平衡跌幅:     0.10%

── 情境分析 ──────────────────────────────

  | 情境            | 價格變動  | 毛利     | 淨利     |
  |----------------|----------|---------|---------|
  | 完整級聯 (-5.0%) | -5.0%   | +$62.50 | +$61.25 |
  | 部分級聯 (-2.5%) | -2.5%   | +$31.25 | +$30.00 |
  | 無級聯 (0%)      | 0.0%    | $0.00   | -$1.25  |
  | 反向 (+2.0%)     | +2.0%   | -$25.00 | -$26.25 |

── Risk Assessment ─────────────────────────

  Overall Risk:  ▓▓▓░░░░░░░  3/10
                 MODERATE-LOW

  Breakdown:
  ├─ Cascade Certainty: ▓▓▓░░░░░░░  3/10
  │                     HF = 1.154, decent collateral size
  ├─ Market Depth:      ▓▓░░░░░░░░  2/10
  │                     ETH-USDT-SWAP: deep orderbook ($50M+ bid depth)
  ├─ Timing Risk:       ▓▓▓▓░░░░░░  4/10
  │                     HF slowly declining; cascade not imminent
  └─ Execution Risk:    ▓▓░░░░░░░░  2/10
                        ETH perp: tight spread, high liquidity

  Result: [ACTIONABLE] -- HIGH signal with moderate risk

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 設定監控，追蹤此倉位健康因子變化:
     liquidation-cascade-monitor monitor --tokens ETH --hf-alert 1.05
  2. 查看多協議系統性風險:
     liquidation-cascade-monitor cascade-alert --tokens ETH --scenario-drop-pct 15
  3. 如入場後查看資金費率成本:
     funding-rate-arbitrage rates --asset ETH

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Cascade timing is inherently uncertain -- health factors may recover.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
  清算級聯的時間點具有不確定性 -- 健康因子可能恢復。
══════════════════════════════════════════
```

### 13.3 Monitor Output

```
══════════════════════════════════════════
  Liquidation Cascade Monitor -- MONITOR
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:30 UTC
  Refresh: every 15 minutes
══════════════════════════════════════════

── 監控狀態 ──────────────────────────────

  Token   | Protocol    | HF      | 抵押品        | 狀態      | 趨勢
  ─────────────────────────────────────────────────────────────────
  ETH     | Aave V3     | 1.154   | $5,100,000   | WATCH    | ▇▆▅▄▃ declining
  WBTC    | Compound V3 | 1.342   | $3,200,000   | SAFE     | ▅▅▅▅▅ stable
  SOL     | Aave V3     | 1.067   | $1,800,000   | ALERT    | ▆▄▃▂▁ declining fast
  LINK    | Aave V3     | 1.891   | $800,000     | SAFE     | ▃▄▅▆▇ improving

── 警報 ──────────────────────────────────

  [ALERT] SOL / Aave V3
  ├─ 健康因子: 1.067 (低於警報門檻 1.05 僅差 1.7%)
  ├─ 抵押品: $1,800,000 SOL
  ├─ 趨勢: 快速下降 (過去 1h 跌 3.2%)
  └─ 建議: 執行深入評估
     liquidation-cascade-monitor evaluate --token SOL --protocol aave-v3 --capital 10000

── 下次刷新: 2026-03-09 14:45 UTC ──────
```

### 13.4 Cascade-Alert Output

```
══════════════════════════════════════════
  Liquidation Cascade Monitor -- CASCADE ALERT
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:30 UTC
  Scenario: ETH drops 10%
══════════════════════════════════════════

── 壓力測試: ETH -10% ───────────────────

  協議              | 可清算量        | 佔 TVL    | 觸發倉位數
  ───────────────────────────────────────────────────────────
  Aave V3           | $45,000,000    | 0.8%     | 127
  Compound V3       | $12,000,000    | 0.5%     | 34
  Maker / Sky       | $28,000,000    | 1.2%     | 18
  Morpho            | $5,000,000     | 0.3%     | 8
  ───────────────────────────────────────────────────────────
  合計              | $90,000,000    |          | 187

── 級聯影響總計 ──────────────────────────

  總強制賣出:        $90,000,000
  OKX ETH 深度 (2%): $50,000,000
  理論級聯影響:      90.0%   [不切實際 -- 實際會分散]
  實際估計 (分散):   8.0-12.0%  [考慮多 DEX 路由 + 時間分散]

── 關聯資產 ──────────────────────────────

  stETH (相關性: 0.98)
  ├─ 額外可清算量: $15,000,000
  └─ 合併級聯影響: +1.5%

  wstETH (相關性: 0.97)
  ├─ 額外可清算量: $8,000,000
  └─ 合併級聯影響: +0.8%

── 系統性風險評估 ────────────────────────

  系統性風險等級:   HIGH
  骨牌效應:        YES (Aave 清算 -> Maker 被動清算 -> 二次價格衝擊)
  綜合級聯影響:    10.3-14.3%

  Risk Gauge:  ▓▓▓▓▓▓▓░░░  7/10
               ELEVATED

── 建議 ──────────────────────────────────

  行動: PREPARE_SHORT
  原因: ETH -10% 情境下，$90M+ 的強制賣出將造成顯著價格影響。
  目前 HF 尚未達到觸發水平，但多個協議同時面臨風險。
  建議: 準備好空單參數，等待 HF 進一步惡化時快速入場。

  建議指令:
    liquidation-cascade-monitor evaluate --token ETH --capital 30000 --leverage 3 --confidence moderate

══════════════════════════════════════════
  Disclaimer
══════════════════════════════════════════

  This is analysis only. No trades are executed automatically.
  Stress test scenarios are hypothetical -- actual outcomes may differ significantly.
  以上僅為分析建議，不會自動執行任何交易。
  壓力測試情境為假設性質 -- 實際結果可能大幅不同。
══════════════════════════════════════════
```

---

## 14. Error Codes & Recovery

| Code | Condition | User Message (ZH) | User Message (EN) | Recovery |
|------|-----------|-------------------|-------------------|----------|
| `MCP_NOT_CONNECTED` | okx-trade-mcp unreachable | MCP 伺服器無法連線。請確認 okx-trade-mcp 是否正在運行。 | MCP server unreachable. Check if okx-trade-mcp is running. | Verify config, restart server. |
| `AUTH_FAILED` | API key invalid or expired | API 認證失敗，請檢查 OKX API 金鑰設定 | API authentication failed. Check OKX API key configuration. | Update `~/.okx/config.toml` |
| `NO_PERP_AVAILABLE` | No SWAP instrument for target token | 找不到 {token} 的永續合約。無法做空。 | No perpetual swap found for {token}. Cannot short. | Suggest alternative tokens with OKX perps. List available: BTC, ETH, SOL, etc. |
| `CASCADE_TOO_SMALL` | Cascade impact < 1.0% | 級聯影響僅 {impact}%，低於 1.0% 門檻，不建議入場 | Cascade impact only {impact}%, below 1.0% threshold. Not recommended. | Show data anyway. Suggest monitoring for larger positions. |
| `CASCADE_IN_PROGRESS` | Health factor < 0.5 | 健康因子 < 0.5，清算可能已在進行中，入場風險極高 | Health factor < 0.5, liquidation likely in progress. Very high entry risk. | Check if CEX price already dropped. May be too late. |
| `LEVERAGE_EXCEEDED` | Leverage > 5x requested | 級聯交易最高允許 5x 槓桿（時機不確定性高） | Cascade trades allow max 5x leverage (timing uncertainty). | Inform user this is a hard rule. |
| `TRADE_SIZE_EXCEEDED` | Position size > $100,000 | 倉位大小 ${amount} 超過上限 $100,000 | Position size ${amount} exceeds limit $100,000 | Cap at limit. |
| `HOLD_TIME_EXCEEDED` | Hold time > 24 hours | 持有時間超過 24 小時上限，級聯為時效性事件 | Hold time exceeds 24-hour limit. Cascades are time-sensitive. | Cap at 24h. |
| `DATA_STALE` | Market data too old | 市場數據已過期，正在重新獲取... | Market data stale. Refetching... | Auto-retry once, then error. |
| `DEFI_DATA_UNAVAILABLE` | DeFiLlama API unreachable | DeFi 數據源不可用，降級為僅 OKX 數據模式 | DeFi data source unavailable. Degrading to OKX-only mode. | Continue with OKX data only. Confidence reduced. |
| `RATE_LIMITED` | API rate limit hit | API 請求頻率超限，{wait}秒後重試 | API rate limit reached. Retrying in {wait}s. | Wait 1s, retry up to 3x. |
| `SYSTEMIC_RISK` | Multiple protocols cascading simultaneously | 多個協議同時清算，系統性風險極高。倉位自動縮小 50%。 | Multiple protocols cascading simultaneously. Systemic risk very high. Position auto-reduced 50%. | Halve position size. Add prominent warning. |
| `INSTRUMENT_NOT_FOUND` | No SWAP instruments at all | 找不到任何永續合約，請檢查 API 連線 | No swap instruments found. Check API connectivity. | Verify MCP server is running. |

---

## 15. Conversation Examples

### Example 1: Scan for Liquidation Risks

**User:**
> 有冇大倉接近清算？

**Intent Recognition:**
- Command: `scan`
- Protocols: all (default)
- Min collateral: $500,000 (default)
- Max HF: 1.10 (default)

**Tool Calls:**

```
1. system_get_capabilities -> { authenticated: true, mode: "demo" }
2. DeFiLlama: GET /protocols -> filter lending protocols
3. DeFiLlama: GET /v2/protocols/aave-v3/liquidations
   -> Positions with HF < 1.10:
      [{ collateral: "ETH", collateral_usd: 5100000, borrow_usd: 3800000, hf: 1.154 },
       { collateral: "SOL", collateral_usd: 1800000, borrow_usd: 1500000, hf: 1.067 }]
4. DeFiLlama: GET /v2/protocols/compound-v3/liquidations
   -> [{ collateral: "WBTC", collateral_usd: 3200000, borrow_usd: 2100000, hf: 1.342 }]
5. market_get_instruments(instType: "SWAP")
   -> Verify ETH-USDT-SWAP, SOL-USDT-SWAP, BTC-USDT-SWAP exist
6. market_get_orderbook(instId: "ETH-USDT-SWAP", sz: "400")
   -> Sum bid depth within 2%: $50,000,000
7. market_get_orderbook(instId: "SOL-USDT-SWAP", sz: "400")
   -> Sum bid depth within 2%: $8,000,000
```

**Computation:**

```
ETH (Aave V3):
  cascade_impact = 5,100,000 / (50,000,000 * 2) * 100 = 5.1%
  signal_strength = "CRITICAL" (>5%)
  distance_to_liq = (1 - 1/1.154) * 100 = 13.3%

SOL (Aave V3):
  cascade_impact = 1,800,000 / (8,000,000 * 2) * 100 = 11.25%
  signal_strength = "CRITICAL" (>5%)
  distance_to_liq = (1 - 1/1.067) * 100 = 6.3%
```

**Output:** (See Section 13.1 template, filled with above data)

---

### Example 2: Evaluate a Specific Cascade

**User:**
> 幫我分析 ETH 清算級聯影響，$50,000 資金，3 倍槓桿，VIP1

**Intent Recognition:**
- Command: `evaluate`
- Token: ETH
- Capital: $50,000
- Leverage: 3x
- VIP tier: VIP1
- Confidence: moderate (default)

**Tool Calls:**

```
1. system_get_capabilities -> { authenticated: true, mode: "demo" }
2. DeFiLlama: Aave V3 ETH liquidation data
   -> collateral_at_risk: $5,100,000, HF: 1.154
3. market_get_ticker(instId: "ETH-USDT") -> { last: "3400.00" }
4. market_get_ticker(instId: "ETH-USDT-SWAP") -> { last: "3400.50" }
5. market_get_orderbook(instId: "ETH-USDT-SWAP", sz: "400")
   -> bid depth within 2%: $50,000,000
6. market_get_funding_rate(instId: "ETH-USDT-SWAP")
   -> { fundingRate: "0.0001" }
```

**Computation:**

```
cascade_impact = 5,100,000 / (50,000,000 * 2) * 100 = 5.1%
signal_strength = "CRITICAL"

position_size = min(100000, 0.051 * 50000 * 0.5)
             = min(100000, 1275)
             = $1,275.00

margin_required = 1275 / 3 = $425.00

stop_loss = 3400.50 * 1.02 = $3,468.51  (+2.0%)
take_profit = 3400.50 * (1 - 0.051 * 0.5) = 3400.50 * 0.9745 = $3,313.79  (-2.55%)

Cost (VIP1):
  entry = 1275 * 0.00045 = $0.57
  exit  = 1275 * 0.00045 = $0.57
  funding (4h) = 1275 * 0.0001 * 0.5 = $0.064 (received)
  total = $1.08

Scenarios:
  Full cascade (-5.1%):   gross = +$65.03, net = +$63.95
  Partial (-2.55%):        gross = +$32.51, net = +$31.44
  No cascade (0%):         gross = $0,      net = -$1.08
  Adverse (+2.0%):         gross = -$25.50, net = -$26.58

Risk score:
  Cascade Certainty: 3 * 0.30 = 0.90
  Market Depth: 2 * 0.25 = 0.50
  Timing Risk: 4 * 0.25 = 1.00
  Execution Risk: 2 * 0.20 = 0.40
  Total: 2.80 -> 3 (MODERATE-LOW)
```

**Output:** (See Section 13.2 template, filled with above data)

---

### Example 3: Cascade Alert / Stress Test

**User:**
> 如果 ETH 跌 10%，會唔會連環清算？

**Intent Recognition:**
- Command: `cascade-alert`
- Token: ETH
- Scenario drop: 10%
- Include correlated: true (default)

**Tool Calls:**

```
1. system_get_capabilities -> { authenticated: true, mode: "demo" }
2. For each protocol (Aave V3, Compound V3, Maker, Morpho):
   DeFiLlama liquidation data for ETH collateral positions
3. Apply -10% scenario:
   For each position:
     new_hf = hf * 0.90
     If new_hf < 1.0: add to liquidatable pool
4. Sum liquidatable amounts per protocol
5. Repeat for stETH, wstETH (correlated assets)
6. market_get_orderbook(instId: "ETH-USDT-SWAP", sz: "400")
```

**Computation:**

```
Aave V3:   127 positions would breach HF < 1.0 -> $45M liquidatable
Compound:  34 positions -> $12M
Maker:     18 positions -> $28M
Morpho:    8 positions  -> $5M
Total:     $90M

Correlated (stETH): +$15M, (wstETH): +$8M
Grand total: $113M

OKX depth: $50M
Theoretical impact: 113M / (50M * 2) * 100 = 113% (unrealistic)
Distributed estimate: 8-12% (liquidations spread across DEXes over minutes)
Combined with correlated: 10.3-14.3%

Systemic risk: HIGH
Domino effect: YES (Aave cascade -> price drop -> Maker positions breach)
```

**Output:** (See Section 13.4 template, filled with above data)

---

## 16. Implementation Notes

### DeFi Data Source Hierarchy

```
Priority 1: DeFiLlama /v2/protocols/{protocol}/liquidations (aggregated)
Priority 2: Aave V3 subgraph -> getUserAccountData() per user
Priority 3: Compound V3 subgraph -> getAccountLiquidity()
Priority 4: Maker/Sky -> Vat.urns() for individual CDPs

Fallback: If DeFiLlama is unavailable, use aggregate TVL data and
estimate liquidation exposure based on typical LTV distributions.
```

### Token-to-OKX-Perp Mapping

Common DeFi collateral tokens and their OKX perp availability:

| DeFi Collateral Token | OKX Perp instId | Available? |
|----------------------|-----------------|------------|
| ETH / WETH | ETH-USDT-SWAP | Yes |
| WBTC | BTC-USDT-SWAP | Yes (proxy via BTC) |
| SOL | SOL-USDT-SWAP | Yes |
| LINK | LINK-USDT-SWAP | Yes |
| UNI | UNI-USDT-SWAP | Yes |
| AAVE | AAVE-USDT-SWAP | Yes |
| stETH / wstETH | ETH-USDT-SWAP | Yes (proxy via ETH, ~0.98 correlation) |
| USDC / USDT | N/A | No -- stablecoin, no need to short |
| DAI | N/A | No perp available |
| CRV | CRV-USDT-SWAP | Yes |
| MKR | MKR-USDT-SWAP | Yes |

### OKX String-to-Number Convention

All OKX API values are returned as **strings**. Always parse to `float` before arithmetic:

```
WRONG:  "3400.00" - "3315.00"  -> NaN or string concat
RIGHT:  parseFloat("3400.00") - parseFloat("3315.00") -> 85.0
```

### Rate Limit Awareness

| Tool | Rate Limit | Strategy |
|------|-----------|----------|
| `market_get_ticker` | 20 req/s | Safe to batch. One call per token. |
| `market_get_orderbook` | 20 req/s | Use `sz: "400"` for depth. One call per token. |
| `market_get_instruments` | 20 req/s | Single call, cache for session. |
| `market_get_funding_rate` | 10 req/s | One call per token. |
| DeFiLlama API | ~30 req/min (estimated) | Cache results for 5 minutes. |

### Orderbook Depth Calculation

To compute bid-side USD liquidity within X% of mid price:

```
1. Fetch orderbook: market_get_orderbook(instId, sz: "400")
2. Compute mid_price = (parseFloat(bids[0][0]) + parseFloat(asks[0][0])) / 2
3. threshold_price = mid_price * (1 - X / 100)   // X = 2 for 2%
4. depth_usd = 0
5. For each [price, size, _, _] in bids:
     p = parseFloat(price)
     s = parseFloat(size)
     if p >= threshold_price:
       depth_usd += p * s * ctVal  // ctVal from instrument spec
     else:
       break   // bids are sorted best-to-worst
6. Return depth_usd

Worked Example (ETH-USDT-SWAP):
  mid_price = 3400.25
  threshold = 3400.25 * 0.98 = 3332.25
  Sum all bid levels from 3400.25 down to 3332.25
  Result: $50,000,000 (hypothetical)
```

### Health Factor Trend Detection

To assess whether a health factor is improving or declining:

```
Monitor HF at intervals (e.g., every 15 minutes).
Store last N readings (N = 4 for 1-hour window).

trend = "STABLE" by default
If HF[latest] < HF[earliest] * 0.99:  trend = "DECLINING"
If HF[latest] < HF[earliest] * 0.95:  trend = "DECLINING_FAST"
If HF[latest] > HF[earliest] * 1.01:  trend = "IMPROVING"

Sparkline mapping:
  Sort readings chronologically.
  Map each to sparkline character based on relative position within the range.
  Range: [min(readings), max(readings)]
  Characters: ▁▂▃▄▅▆▇█ (8 levels)
```

### Cascade Timing Uncertainty

This is the primary risk of this strategy. Key considerations:

```
1. Health factors can IMPROVE if:
   - Borrower adds collateral
   - Borrower repays some debt
   - Collateral price recovers

2. Cascades may NOT produce expected impact if:
   - Liquidations are split across many small positions (less impact each)
   - Keepers use DEX aggregators efficiently (distributed impact)
   - CEX market makers arbitrage quickly (absorb DEX impact)

3. Timing windows:
   - HF 1.10 -> 1.00: Could take hours, days, or never happen
   - HF 1.05 -> 1.00: Higher probability, tighter window
   - HF 1.02 -> 1.00: Very likely within hours if trend is declining

4. This is why:
   - Position sizes are intentionally small (confidence_factor)
   - Stop losses are tight (2%)
   - Max hold time is capped (4h default, 24h hard cap)
   - We target only 50% of theoretical impact
```

### Protocol-Specific Liquidation Thresholds (Quick Reference)

| Protocol | Asset | Liquidation Threshold (LTV) | Liquidation Penalty | Close Factor |
|----------|-------|-----------------------------|---------------------|-------------|
| Aave V3 | ETH | 0.860 (86%) | 5.0% | 50% |
| Aave V3 | WBTC | 0.780 (78%) | 6.5% | 50% |
| Aave V3 | wstETH | 0.795 (79.5%) | 7.0% | 50% |
| Aave V3 | LINK | 0.650 (65%) | 7.5% | 50% |
| Aave V3 | UNI | 0.580 (58%) | 10.0% | 50% |
| Compound V3 | ETH | 0.830 (83%) | 5.0% | 100% |
| Compound V3 | WBTC | 0.700 (70%) | 5.0% | 100% |
| Maker ETH-A | ETH | 0.667 (150% CR) | 13.0% | 100% |
| Maker ETH-B | ETH | 0.769 (130% CR) | 13.0% | 100% |
| Maker ETH-C | ETH | 0.588 (170% CR) | 13.0% | 100% |

Notes:
- Close Factor = maximum % of debt that can be liquidated in one tx
- Aave V3: 50% close factor means only half the debt can be repaid per liquidation tx
- Compound V3 + Maker: 100% close factor means full position can be liquidated at once
- Higher liquidation penalty = more aggressive forced selling = larger cascade impact
