---
name: yield-optimizer
description: >
  Triggers: "yield", "收益", "APY", "利息", "哪裡收益最好", "best yield",
  "DeFi vs CEX", "lending rate", "借貸利率", "staking", "質押", "earn",
  "理財", "定期", "活期"
allowed-tools:
  - okx-trade-mcp (market tools)
  - DeFiLlama MCP
  - CoinGecko MCP
  - GoPlus MCP
---

# Yield Optimizer

Compare and optimize allocation between CEX lending, DeFi yields, staking,
and LP farming. Extends the Interest Information Aggregator project by
unifying CEX earn products and DeFi yield sources into a single risk-adjusted
comparison framework.

## Role

- Scan and aggregate yield opportunities across CEX (OKX Earn) and DeFi
  (lending, staking, LP, vaults) for user-specified assets.
- Compute risk-adjusted yields, switching costs, and break-even periods.
- Recommend optimal allocation per risk tolerance.

### Does NOT

- Execute trades, deposits, or withdrawals.
- Provide financial advice -- outputs are analysis only.
- Handle arbitrage -- use `cex-dex-arbitrage`, `funding-rate-arbitrage`, or
  `basis-trading` instead.

---

## Pre-flight

Before executing any command, verify the following MCP servers are available.

| Server | Required | Purpose |
|--------|----------|---------|
| `okx-trade-mcp` | Yes | CEX market data, OKX Earn rate reference |
| DeFiLlama MCP | **Critical** | Protocol TVL, pool yield data |
| GoPlus MCP | **Critical** | Token/contract security checks for DeFi opportunities |
| CoinGecko MCP | Optional | Staking yield data, market price backup |

Run `system_get_capabilities` on `okx-trade-mcp` to confirm connectivity
and mode (`demo` / `live`).

### Known Limitations

- **OKX Earn rates** -- `okx-trade-mcp` may not directly expose OKX Earn
  (savings/staking) product rates. When unavailable, document this gap in
  output and advise the user to check OKX Earn manually. Use publicly
  available snapshots or CoinGecko lending rate data as a fallback.
- **DeFiLlama pool data** -- Pool-level yields change frequently. Data should
  be treated as a point-in-time snapshot, not a guarantee.

---

## Command Index

| Command | Purpose | Read/Write |
|---------|---------|------------|
| `scan` | Discover yield opportunities for given assets | Read |
| `evaluate` | Deep-dive a single opportunity with full risk analysis | Read |
| `compare` | Side-by-side CEX vs DeFi comparison for one asset | Read |
| `rebalance` | Suggest portfolio reallocation to maximize risk-adjusted yield | Read |
| `rates-snapshot` | Quick snapshot of current CEX and DeFi rates for an asset | Read |

---

## Parameter Reference

### scan

Discover yield opportunities across multiple assets and categories.

```bash
yield-optimizer scan --assets USDT,ETH,SOL --categories lending,staking,lp,earn
```

| Parameter | Type | Required | Default | Enum | Validation |
|-----------|------|----------|---------|------|------------|
| `--assets` | string[] | Yes | -- | Any token symbol | Uppercase, comma-separated |
| `--categories` | string[] | No | all | `lending`, `staking`, `lp`, `earn`, `vault` | Valid category name |
| `--risk-tolerance` | string | No | `moderate` | `conservative`, `moderate`, `aggressive` | Exact match |
| `--min-apy-pct` | number | No | `1.0` | 0 - 1000 | Positive number |
| `--chains` | string[] | No | all | Any supported chain | Supported by DeFiLlama / OnchainOS |
| `--top-n` | integer | No | `10` | 1 - 50 | Positive integer |

**Return Schema:**

```yaml
YieldScanResult:
  timestamp: integer          # Unix ms
  asset: string               # e.g. "ETH"
  opportunities:
    - source: string          # "OKX Earn" | "Aave v3" | "Lido" | "Curve" etc.
      category: string        # "lending" | "staking" | "lp" | "earn" | "vault"
      chain: string           # "cex" | "ethereum" | "solana" | "arbitrum" etc.
      raw_apy_pct: number     # Advertised APY
      risk_score: integer     # 1-10 (see Risk Score Components)
      risk_adjusted_apy_pct: number  # raw_apy * (1 - risk_score/10)
      tvl_usd: number         # Protocol TVL from DeFiLlama
      lock_period: string     # "none" | "7d" | "30d" | "90d"
      min_amount: number      # Minimum deposit in asset units
      security_status: string # "verified" | "unverified" | "flagged"
      notes: string[]         # Additional context
```

### evaluate

Deep-dive into a single yield opportunity.

```bash
yield-optimizer evaluate --source "Aave v3" --asset ETH --chain ethereum --size-usd 10000
```

| Parameter | Type | Required | Default | Enum | Validation |
|-----------|------|----------|---------|------|------------|
| `--source` | string | Yes | -- | Protocol name | Non-empty |
| `--asset` | string | Yes | -- | Token symbol | Uppercase |
| `--chain` | string | Yes | -- | Chain name | Supported chain |
| `--size-usd` | number | No | `10000` | 100 - 10,000,000 | Positive |

**Return Schema:**

```yaml
YieldEvaluation:
  source: string
  asset: string
  chain: string
  raw_apy_pct: number
  risk_score: integer
  risk_adjusted_apy_pct: number
  risk_breakdown:
    audit_status: integer     # 0-2
    tvl_tier: integer         # 0-2
    token_concentration: integer  # 0-2
    contract_age: integer     # 0-2
    chain_risk: integer       # 0-2
  tvl_usd: number
  apy_trend_7d: number[]      # Last 7 daily snapshots
  apy_sparkline: string       # e.g. "▃▄▅▅▆▆▇▇"
  lock_period: string
  withdrawal_conditions: string
  security_check: object      # GoPlus result summary
  switching_cost_usd: number
  switching_cost_bps: number
  break_even_days: number
  projected_30d_income_usd: number
  projected_365d_income_usd: number
  notes: string[]
```

### compare

Side-by-side comparison between CEX and DeFi options for a single asset.

```bash
yield-optimizer compare --asset ETH --size-usd 10000
```

| Parameter | Type | Required | Default | Enum | Validation |
|-----------|------|----------|---------|------|------------|
| `--asset` | string | Yes | -- | Token symbol | Uppercase |
| `--size-usd` | number | No | `10000` | 100 - 10,000,000 | Positive |
| `--include-locked` | boolean | No | `true` | `true`, `false` | Boolean |
| `--risk-tolerance` | string | No | `moderate` | `conservative`, `moderate`, `aggressive` | Exact match |

Returns a `YieldComparisonResult` containing rows for each opportunity with
switching costs and break-even calculations relative to the user's current
position (or a baseline of "holding on CEX spot").

### rebalance

Suggest how to redistribute capital across yield sources for maximum
risk-adjusted return.

```bash
yield-optimizer rebalance --assets USDT,ETH,SOL --total-usd 50000 --risk-tolerance conservative
```

| Parameter | Type | Required | Default | Enum | Validation |
|-----------|------|----------|---------|------|------------|
| `--assets` | string[] | Yes | -- | Token symbols | Uppercase, comma-separated |
| `--total-usd` | number | Yes | -- | 100 - 10,000,000 | Positive |
| `--risk-tolerance` | string | No | `moderate` | `conservative`, `moderate`, `aggressive` | Exact match |
| `--max-protocols` | integer | No | `5` | 1 - 20 | Positive integer |
| `--current-allocation` | string | No | -- | JSON or "none" | Valid JSON or omit |

**Return Schema:**

```yaml
RebalancePlan:
  timestamp: integer
  total_usd: number
  risk_tolerance: string
  current_weighted_apy: number    # If current allocation provided
  proposed_weighted_apy: number
  proposed_risk_adjusted_apy: number
  switching_cost_total_usd: number
  break_even_days: number
  allocations:
    - source: string
      asset: string
      chain: string
      allocation_pct: number      # % of total capital
      allocation_usd: number
      raw_apy_pct: number
      risk_score: integer
      risk_adjusted_apy_pct: number
      switching_cost_usd: number
```

### rates-snapshot

Quick reference of current rates without deep analysis.

```bash
yield-optimizer rates-snapshot --asset USDT
```

| Parameter | Type | Required | Default | Enum | Validation |
|-----------|------|----------|---------|------|------------|
| `--asset` | string | Yes | -- | Token symbol | Uppercase |
| `--categories` | string[] | No | all | `lending`, `staking`, `lp`, `earn`, `vault` | Valid category |

Returns a lightweight table of source/rate pairs without switching cost
calculations.

---

## Operation Flow

### Step 1: Parse Intent

Determine the target command from user input. Match trigger words to command:

| Trigger Pattern | Command |
|----------------|---------|
| "收益最高", "best yield", "邊度收益" | `scan` |
| "詳細分析", "evaluate", "深入" | `evaluate` |
| "比較", "compare", "DeFi vs CEX", "邊個好" | `compare` |
| "重新分配", "rebalance", "最大化", "optimize" | `rebalance` |
| "利率", "rates", "現在幾多" | `rates-snapshot` |

### Step 2: Fetch Yield Data from Multiple Sources

Execute the following data fetches in parallel where possible:

**CEX (OKX) Yields:**
1. `okx-trade-mcp` -> `market_get_ticker` for asset spot price (freshness reference)
2. OKX Earn rates -- attempt to retrieve via API; if unavailable, note the
   limitation and use publicly referenced rates or CoinGecko lending data

**DeFi Lending Yields:**
3. DeFiLlama MCP -> `get_latest_pool_data` -> filter by asset symbol
   - Extract: pool name, chain, APY, TVL
   - Focus on: Aave v3, Compound, Morpho, Spark, Venus, Benqi

**DeFi Staking Yields:**
4. CoinGecko MCP -> `coins_markets` -> staking APR data for PoS assets
   - Lido (stETH), Rocket Pool (rETH), Jito (jitoSOL), Marinade (mSOL)

**LP Yields:**
5. DeFiLlama MCP -> pool data for LP positions
   - Curve, Uniswap, Raydium, Orca, Aerodrome
   - Include impermanent loss estimate for volatile pairs

**For Each DeFi Opportunity:**
6. GoPlus -> `check_token_security` on protocol governance/reward token
   - Required for all non-CEX opportunities
   - Cache results for 5 minutes within session

### Step 3: Compute

**Normalize to Annualized APY:**
All rates must be converted to annualized APY for comparison:
```
annualized_apy = (1 + period_rate) ^ (365 / period_days) - 1
```

**Risk Score (1-10):**

| Component | Weight | Scoring |
|-----------|--------|---------|
| Audit status | 0-2 | 0 = multiple audits, 1 = single audit, 2 = unaudited |
| TVL tier | 0-2 | 0 = >$1B, 1 = $100M-$1B, 2 = <$100M |
| Token concentration | 0-2 | 0 = decentralized, 1 = moderate, 2 = concentrated |
| Contract age | 0-2 | 0 = >2 years, 1 = 6mo-2yr, 2 = <6 months |
| Chain risk | 0-2 | 0 = Ethereum/established L2, 1 = newer L2, 2 = new chain |

CEX (OKX) receives a baseline risk_score of 1 (regulated, custodial risk only).

**Risk-Adjusted APY:**

```
risk_adjusted_apy = raw_apy * (1 - risk_score / 10)

Variables:
  raw_apy    = Advertised or observed annual percentage yield
  risk_score = Risk assessment score on a 0-10 scale (0 = lowest risk, 10 = highest risk)

Risk Score Components:
  - Protocol audit status (0-2)
  - TVL tier (0-2)
  - Token concentration risk (0-2)
  - Smart contract age (0-2)
  - Chain risk (0-2)

Worked Example:
  raw_apy = 12.5%
  risk_score = 3  (audited, high TVL, moderate token risk)

  risk_adjusted_apy = 12.5% * (1 - 3/10)
                    = 12.5% * 0.70
                    = 8.75%

Caveats:
  - Risk score is subjective; use consistently across opportunities for relative comparison.
  - A risk_score of 10 yields 0% adjusted APY (effectively "do not enter").
  - Raw APY may include token emissions that are volatile in value.
```

**Switching Cost Calculation:**

```
switching_cost = exit_gas + exit_slippage + bridge_fee + entry_gas + entry_slippage

For yield switching in USD:
  switching_cost_bps = switching_cost / position_size * 10000

For CEX -> DeFi: add withdrawal fee (see OKX Withdrawal Fees below).
For DeFi -> DeFi: add gas on both chains + bridge fee if cross-chain.
For DeFi -> CEX: add gas + deposit confirmation time (no fee).

Worked Example:
  exit_gas (Ethereum)     = $12.00
  exit_slippage           =  $5.00
  bridge_fee (ETH -> Arb) =  $2.00
  entry_gas (Arbitrum)    =  $0.30
  entry_slippage          =  $3.00

  switching_cost = 12.00 + 5.00 + 2.00 + 0.30 + 3.00 = $22.30

  position_size = $50,000
  switching_cost_bps = 22.30 / 50000 * 10000 = 4.5 bps
```

**Break-Even Period for Rebalancing:**

```
break_even_days = switching_cost / (new_daily_yield - old_daily_yield)

Where:
  daily_yield = position_size * apy / 365

Worked Example:
  old_apy = 5.00%
  new_apy = 8.50%
  position_size = $50,000
  switching_cost = $22.30

  old_daily_yield = 50000 * 0.05 / 365 = $6.85
  new_daily_yield = 50000 * 0.085 / 365 = $11.64

  break_even_days = 22.30 / (11.64 - 6.85)
                  = 22.30 / 4.79
                  = 4.66 days

Caveats:
  - Only switch if you expect the yield differential to persist longer than break-even.
  - Factor in risk-adjusted APY, not raw APY, for a fairer comparison.
  - Yield rates change constantly; a 3.5% edge today may vanish tomorrow.
```

### Step 4: Output

Produce the comparison table ranked by risk-adjusted APY (descending).
Include switching costs and break-even in ALL recommendations.
Apply risk-tolerance filter:

| Risk Tolerance | Max Risk Score | Included Categories |
|---------------|---------------|---------------------|
| `conservative` | 3 | lending, earn, staking (liquid) |
| `moderate` | 6 | All categories |
| `aggressive` | 10 | All categories, including unaudited |

---

## OKX Withdrawal Fees (Inlined)

Withdrawal fees are flat (not percentage-based) and vary by asset and network.

### BTC Withdrawals

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Bitcoin (BTC) | 0.0001 BTC | 0.001 BTC |
| Lightning Network | 0.000001 BTC | 0.000001 BTC |

### ETH Withdrawals

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Ethereum (ERC-20) | 0.00035 ETH | 0.001 ETH |
| Arbitrum One | 0.0001 ETH | 0.0001 ETH |
| Optimism | 0.00004 ETH | 0.0001 ETH |
| Base | 0.00004 ETH | 0.0001 ETH |
| zkSync Era | 0.000065 ETH | 0.0001 ETH |
| Linea | 0.00015 ETH | 0.0001 ETH |

### USDT Withdrawals

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
| TON | 0.5 USDT | 0.1 USDT |

### USDC Withdrawals

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Ethereum (ERC-20) | 3.0 USDC | 10.0 USDC |
| Polygon | 0.8 USDC | 0.1 USDC |
| Arbitrum One | 0.1 USDC | 0.1 USDC |
| Optimism | 0.1 USDC | 0.1 USDC |
| Base | 0.1 USDC | 0.1 USDC |
| Solana | 1.0 USDC | 1.0 USDC |
| BSC (BEP-20) | 0.3 USDC | 10.0 USDC |
| Avalanche C-Chain | 1.0 USDC | 1.0 USDC |

### SOL Withdrawals

| Network | Fee | Min Withdrawal |
|---------|-----|---------------|
| Solana | 0.008 SOL | 0.1 SOL |

### Other Major Assets

| Asset | Network | Fee | Min Withdrawal |
|-------|---------|-----|---------------|
| BNB | BSC (BEP-20) | 0.0005 BNB | 0.01 BNB |
| MATIC | Polygon | 0.1 MATIC | 0.1 MATIC |
| AVAX | Avalanche C-Chain | 0.01 AVAX | 0.1 AVAX |
| ARB | Arbitrum One | 0.1 ARB | 1.0 ARB |
| OP | Optimism | 0.1 OP | 1.0 OP |
| OKB | X Layer | 0.001 OKB | 0.01 OKB |

### Withdrawal Fee Notes

- Withdrawal fees are **flat amounts**, not percentages.
- Fees are updated periodically based on network conditions.
- Always verify current fees via OKX API before calculating costs.
- Internal transfers between OKX accounts (sub-accounts) are **free**.
- Deposits to OKX are **free** (user only pays network gas).
- **OKX withdrawal is often cheaper than bridging** for moving assets from CEX to another chain. Withdraw directly to the target chain when supported.

---

## Gas Benchmarks per Chain (Inlined)

Approximate gas costs for a typical DEX swap transaction.

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

### Gas Cost Formulas

**EVM Chains:**
```
gas_cost_usd = gas_price_gwei * gas_limit * native_token_price / 1e9
```

**Solana:**
```
gas_cost_usd = (base_fee_lamports + priority_fee_lamports) * sol_price / 1e9
```

### Cost Efficiency by Chain (for $10,000 trade)

| Chain | Gas as % of Trade | Practical Minimum Trade |
|-------|-------------------|------------------------|
| Ethereum | 0.05-0.30% | $5,000+ |
| Arbitrum | 0.001-0.005% | $100+ |
| Base | 0.0001-0.0005% | $50+ |
| Solana | 0.00001-0.0001% | $10+ |
| BSC | 0.001-0.003% | $100+ |
| Polygon | 0.0001-0.0005% | $50+ |

### Gas Notes

- **Ethereum L1** gas is by far the most expensive; avoid for small trades.
- **L2 costs** include a small L1 data posting fee that can spike during L1 congestion.
- **Multi-hop swaps** use significantly more gas.
- **Approval transactions** (first-time token approvals) add ~46,000 gas on EVM chains.
- Gas prices fluctuate significantly; always fetch real-time estimates.

---

## GoPlus Decision Matrix for DeFi Protocol Token Checks (Inlined)

For every DeFi opportunity, run GoPlus `check_token_security` on the protocol governance/reward token.

### check_token_security Parameters

| Param | Required | Type | Description |
|-------|----------|------|-------------|
| chain_id | Yes | string | Blockchain identifier (e.g. `"1"` for Ethereum, `"42161"` for Arbitrum, `"solana"` for Solana) |
| contract_address | Yes | string | Token contract address to audit |

### Chain ID Reference

| Chain | GoPlus chain_id |
|-------|----------------|
| Ethereum | `"1"` |
| BSC | `"56"` |
| Polygon | `"137"` |
| Arbitrum | `"42161"` |
| Base | `"8453"` |
| Avalanche | `"43114"` |
| Optimism | `"10"` |
| Solana | `"solana"` |

### Decision Matrix

| GoPlus Field | BLOCK if | WARN if |
|-------------|----------|---------|
| is_honeypot | `=== "1"` | -- |
| buy_tax | `> "0.05"` (5%) | `> "0.01"` (1%) |
| sell_tax | `> "0.10"` (10%) | `> "0.02"` (2%) |
| is_mintable | `=== "1"` AND contract < 7 days old | `=== "1"` |
| can_take_back_ownership | `=== "1"` | -- |
| owner_change_balance | `=== "1"` | -- |
| is_open_source | `=== "0"` | -- |
| is_proxy | -- | `=== "1"` |
| top 10 holders % | > 80% combined | > 50% combined |
| slippage_modifiable | `=== "1"` | -- |
| transfer_pausable | -- | `=== "1"` |
| cannot_sell_all | `=== "1"` | -- |
| personal_slippage_modifiable | `=== "1"` | -- |

### Decision Logic (Pseudocode)

```
security = check_token_security(chain_id, address)

// Hard blocks -- any single trigger stops the recommendation
if security.is_honeypot === "1":           return BLOCK("Honeypot detected")
if security.buy_tax > 0.05:               return BLOCK("Buy tax > 5%")
if security.sell_tax > 0.10:              return BLOCK("Sell tax > 10%")
if security.can_take_back_ownership === "1": return BLOCK("Ownership reclaimable")
if security.owner_change_balance === "1":  return BLOCK("Owner can change balances")
if security.is_open_source === "0":        return BLOCK("Unverified contract")
if security.slippage_modifiable === "1":   return BLOCK("Tax modifiable by owner")
if security.cannot_sell_all === "1":       return BLOCK("Cannot sell all tokens")
if top10HolderPct > 0.80:                 return BLOCK("Top 10 holders > 80%")

// Soft warnings -- flag but allow with disclosure
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

### How to Calculate Top 10 Holder Concentration

```
1. Sort holders[] by percent descending
2. Filter out known contract addresses (is_contract === 1) that are
   DEX pools or burn addresses (these are NOT concentration risk)
3. Sum the top 10 non-contract holder percentages
4. Compare against 80% (BLOCK) and 50% (WARN) thresholds
```

### Integration Notes

- **Always call before recommending** -- every DeFi opportunity MUST have a `check_token_security` call before being output.
- **Cache results** for 5 minutes within a session to avoid redundant API calls.
- **GoPlus API response wrapper** -- security data is nested inside `result.<contract_address>`. Always extract using the contract address as key.
- **If GoPlus is unreachable**, surface a `[WARN]`: "Security check unavailable -- proceed with extreme caution." Never silently skip.

---

## Safety Checks (Inlined)

### Pre-Trade Safety Checklist

Every skill runs through these checks **in order** before producing a recommendation. A BLOCK at any step halts the pipeline immediately.

| # | Check | Tool | BLOCK Threshold | WARN Threshold | Error Code |
|---|-------|------|----------------|----------------|------------|
| 1 | MCP connectivity | `system_get_capabilities` | Server not reachable | -- | `MCP_NOT_CONNECTED` |
| 2 | Authentication | `system_get_capabilities` | `authenticated: false` | -- | `AUTH_FAILED` |
| 3 | Data freshness | Internal timestamp comparison | > 60s stale | > 30s stale | `DATA_STALE` |
| 4 | Token honeypot | GoPlus `check_token_security` | `is_honeypot === "1"` | -- | `SECURITY_BLOCKED` |
| 5 | Token tax rate | GoPlus `check_token_security` | buy > 5% OR sell > 10% | buy > 1% | `SECURITY_BLOCKED` |
| 6 | Holder concentration | GoPlus `check_token_security` | Top 10 non-contract holders > 80% | Top 10 > 50% | `SECURITY_BLOCKED` |
| 7 | Contract verified | GoPlus `check_token_security` | `is_open_source === "0"` | -- | `SECURITY_BLOCKED` |
| 8 | Net profitability | Cost calculation | `net_profit <= 0` | `profit_to_cost_ratio < 2` | `NOT_PROFITABLE` |

### Yield-Specific Safety Checks

| # | Check | Source | Threshold | Action |
|---|-------|--------|-----------|--------|
| 1 | Protocol TVL | DeFiLlama MCP | < $10M | **BLOCK** -- `INSUFFICIENT_LIQUIDITY` |
| 2 | GoPlus token security | GoPlus `check_token_security` | Any flag on protocol token | **BLOCK** -- `SECURITY_BLOCKED` |
| 3 | Unaudited contract | Config `reject_unaudited_contracts` | `true` + no audit | **BLOCK** -- `SECURITY_BLOCKED` |
| 4 | Single protocol allocation | Config `max_protocol_allocation_pct` | > 25% (default) | **BLOCK** -- reduce allocation |
| 5 | Lock period vs horizon | User's stated horizon | Lock > horizon | **WARN** -- inform user |
| 6 | Gas cost vs yield | Gas benchmarks (above) | Gas > 5% of first-year yield | **WARN** -- `GAS_TOO_HIGH` |
| 7 | Switching benefit | Config `min_switching_benefit_pct` | < 0.5% APY improvement | **WARN** -- may not be worth switching |
| 8 | APY sustainability | 7-day APY trend | Declining > 30% in 7d | **WARN** -- yield may not persist |

**Important:** Include switching costs in ALL recommendations. A higher-APY
opportunity with high switching costs may be worse than the current position.

### Yield-Optimizer Risk Limits

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| `max_protocol_allocation_pct` | 25% | 50% | Max allocation to any single DeFi protocol |
| `min_protocol_tvl_usd` | $10,000,000 | $1,000,000 | Minimum protocol TVL to consider |
| `reject_unaudited_contracts` | `true` | -- | Block protocols without public audit |
| `min_switching_benefit_pct` | 0.5% | -- | Min APY improvement to recommend switch |
| `max_gas_cost_pct` | 5% | -- | Gas as % of first-year yield must be below this |

---

## Impermanent Loss Reference (Inlined)

For LP opportunities involving volatile pairs, always calculate and display
the IL estimate.

### Impermanent Loss Formula

```
IL = 2 * sqrt(price_ratio) / (1 + price_ratio) - 1

Variables:
  price_ratio = New price / original price (e.g., if ETH went from $3,000 to $3,600, ratio = 1.20)
  IL          = Fractional loss compared to simply holding (always negative or zero)

Worked Example:
  Original ETH price: $3,000
  Current ETH price:  $3,600
  price_ratio = 3600 / 3000 = 1.20

  IL = 2 * sqrt(1.20) / (1 + 1.20) - 1
     = 2 * 1.09545 / 2.20 - 1
     = 2.19089 / 2.20 - 1
     = 0.99586 - 1
     = -0.00414
     = -0.41%
```

### IL Quick Reference Table

| Price Change | Price Ratio | IL |
|-------------|------------|-----|
| -50% | 0.50 | -5.72% |
| -25% | 0.75 | -0.60% |
| -10% | 0.90 | -0.14% |
| 0% | 1.00 | 0.00% |
| +10% | 1.10 | -0.14% |
| +25% | 1.25 | -0.60% |
| +50% | 1.50 | -2.02% |
| +100% | 2.00 | -5.72% |
| +200% | 3.00 | -13.40% |
| +400% | 5.00 | -25.46% |

### Caveats

- IL is symmetric with respect to price movement magnitude, not direction.
- Concentrated liquidity (Uniswap v3) amplifies IL within the range but earns more fees.
- IL is only "realized" when you withdraw; if price returns to entry, IL = 0.
- LP fee income may offset IL; always compare net position value.
- For stable pairs (ETH/stETH, USDC/USDT), note that IL is minimal as long as the peg holds, but include de-peg risk in the risk score.

---

## Output Templates (Inlined)

### Global Header Template

```
══════════════════════════════════════════
  {SKILL_ICON} {SKILL_NAME}
  [{MODE}] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: {TIMESTAMP}
  Data sources: {DATA_SOURCES}
══════════════════════════════════════════
```

For yield-optimizer: Skill icon = Seedling, Data sources = "OKX REST + DeFiLlama + GoPlus"

### Formatting Rules

- Monetary: `+$1,234.56` or `-$1,234.56` (2 decimals, comma thousands)
- Percentages: `12.5%` (1 decimal), APY/APR: 2 decimal places
- Basis Points: integer only (e.g., `21 bps`)
- Risk Levels: `[SAFE]`, `[WARN]`, `[BLOCK]`
- Risk Gauge: `▓▓▓░░░░░░░ 3/10` (scale 1-10)
- Sparklines: `▁▂▃▄▅▆▇█` for trend visualization
- Timestamps: `YYYY-MM-DD HH:MM UTC`

### Yield Comparison Template

```
── Yield Opportunities ─────────────────────

  Current Position: {CURRENT_PROTOCOL} on {CURRENT_CHAIN}
  Current APY: {CURRENT_APY}  |  Risk-Adj: {CURRENT_RADJ}

  #  Protocol         Chain      Raw APY  Risk  Adj APY  Switch Cost  Break-Even
  ────────────────────────────────────────────────────────────────────────────────
  1  {P1_NAME}       {P1_CHAIN}  {P1_APY}  {P1_RS}   {P1_RADJ}   {P1_SC}     {P1_BE}
  2  {P2_NAME}       {P2_CHAIN}  {P2_APY}  {P2_RS}   {P2_RADJ}   {P2_SC}     {P2_BE}
  3  {P3_NAME}       {P3_CHAIN}  {P3_APY}  {P3_RS}   {P3_RADJ}   {P3_SC}     {P3_BE}
  4  {P4_NAME}       {P4_CHAIN}  {P4_APY}  {P4_RS}   {P4_RADJ}   {P4_SC}     {P4_BE}
  5  {P5_NAME}       {P5_CHAIN}  {P5_APY}  {P5_RS}   {P5_RADJ}   {P5_SC}     {P5_BE}

  ── Recommendation ────────────────────────
  {RECOMMENDATION_TEXT}
```

**Example (filled):**

```
── Yield Opportunities ─────────────────────

  Current Position: Aave V3 USDC on Ethereum
  Current APY: 4.20%  |  Risk-Adj: 3.78%

  #  Protocol          Chain       Raw APY  Risk   Adj APY  Switch Cost  Break-Even
  ──────────────────────────────────────────────────────────────────────────────────
  1  Morpho USDC       Base         8.50%   3/10    5.95%     $22.30       4.7d
  2  Aave V3 USDC      Arbitrum     6.10%   2/10    4.88%     $15.80       5.2d
  3  Compound USDC     Ethereum     5.80%   2/10    4.64%      $8.50       9.8d
  4  Spark DAI         Ethereum     5.20%   3/10    3.64%     $12.00      n/a
  5  Venus USDT        BSC          9.20%   5/10    4.60%     $18.50       7.1d

  ── Recommendation ────────────────────────
  Switch to Morpho USDC on Base: +2.17% risk-adjusted edge,
  break-even in 4.7 days. Low gas on Base keeps switching cost minimal.
```

### Safety Check Results Template

```
── Safety Checks ───────────────────────────

  [  {CHECK1_STATUS}  ] {CHECK1_NAME}
                        {CHECK1_DETAIL}

  [  {CHECK2_STATUS}  ] {CHECK2_NAME}
                        {CHECK2_DETAIL}

  [  {CHECK3_STATUS}  ] {CHECK3_NAME}
                        {CHECK3_DETAIL}

  ──────────────────────────────────────
  Overall: {OVERALL_STATUS}
  {OVERALL_MESSAGE}
```

**Standard Safety Checks:**

| Check Name | What It Verifies |
|------------|-----------------|
| Liquidity Depth | Sufficient liquidity for trade size without excessive slippage |
| Gas Conditions | Current gas prices are within acceptable range |
| Price Freshness | Price data is recent (< 30 seconds old) |
| Position Size | Trade size within configured risk limits |
| Contract Risk | Token contract is verified and not flagged |

### Next Steps Template

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
  Past yields do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

### Yield Comparison Table (Full Example)

```
══════════════════════════════════════════
  Yield Optimizer
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 12:00 UTC
  Data sources: OKX REST + DeFiLlama + GoPlus
══════════════════════════════════════════

ETH 收益比較 (規模: $10,000)

類型          | 平台/協議       | 當前 APY | 風險評分  | 鎖定期 | 調整後 APY
──────────────┼─────────────────┼─────────┼──────────┼───────┼──────────
CEX 活期      | OKX Earn        | 2.10%   | ▓░░░░ 1  | 無     | 2.10%
CEX 定期      | OKX 30天        | 3.50%   | ▓░░░░ 1  | 30天   | 3.50%
Staking       | Lido (stETH)    | 3.80%   | ▓▓▓░░ 3  | 無*    | 3.42%
LP (穩定對)   | Curve ETH/stETH | 5.20%   | ▓▓▓▓░ 4  | 無     | 4.16%
LP (波動對)   | Uniswap V3      | 12.10%  | ▓▓▓▓▓▓ 6 | 無     | 7.26%
Lending       | Aave v3         | 2.80%   | ▓▓▓░░ 3  | 無     | 2.52%

* Lido stETH 無鎖定期，但退出需等待提款隊列 (通常 1-5 天)

── 切換成本分析 ──────────────────────────

平台/協議       | 切換成本  | 切換成本(bps) | 損益平衡期
────────────────┼──────────┼──────────────┼──────────
OKX Earn        | $0.00    | 0 bps        | N/A
OKX 30天        | $0.00    | 0 bps        | N/A
Lido (stETH)    | $15.30   | 15.3 bps     | 12.3 days
Curve ETH/stETH | $18.50   | 18.5 bps     | 8.9 days
Uniswap V3      | $18.50   | 18.5 bps     | 3.6 days
Aave v3         | $12.80   | 12.8 bps     | N/A (worse than CEX)

── 安全檢查 ───────────────────────────

  [  SAFE  ] Lido -- GoPlus verified, TVL $14.2B
  [  SAFE  ] Curve -- GoPlus verified, TVL $2.1B
  [  WARN  ] Uniswap V3 LP -- Impermanent loss risk: -0.41% at +/-20% price move
  [  SAFE  ] Aave v3 -- GoPlus verified, TVL $8.5B

── Recommendation ────────────────────────
風險偏好: moderate

建議：Lido stETH 提供最佳風險調整收益 (3.42%)，
無鎖定期，$15.30 切換成本可在 12.3 天內回本。
若追求更高收益且可承受 IL 風險，Curve ETH/stETH (4.16%) 為次選。

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 詳細評估 Lido: yield-optimizer evaluate --source "Lido" --asset ETH --chain ethereum
  2. 查看 LP 無常損失: yield-optimizer evaluate --source "Curve ETH/stETH" --asset ETH --chain ethereum
  3. 查看重新分配建議: yield-optimizer rebalance --assets ETH --total-usd 10000

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Past yields do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

### Rebalance Output Template

```
══════════════════════════════════════════
  Yield Optimizer -- Rebalance
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════

資產配置建議 (總額: $50,000 | 風險偏好: conservative)

#  | 平台/協議     | 資產  | 鏈        | 配置 %  | 配置金額     | 調整後 APY
───┼───────────────┼───────┼──────────┼────────┼─────────────┼──────────
1  | OKX Earn 活期 | USDT  | CEX      | 25.0%  | $12,500.00  | 3.20%
2  | Aave v3       | USDT  | Arbitrum | 25.0%  | $12,500.00  | 4.88%
3  | OKX Earn 定期 | ETH   | CEX      | 25.0%  | $12,500.00  | 3.50%
4  | Lido          | ETH   | Ethereum | 25.0%  | $12,500.00  | 3.42%

── 彙總 ──────────────────────────────────
加權調整後 APY: 3.75%
預估年收入: +$1,875.00
切換總成本: $28.10
整體損益平衡: 6.2 天

══════════════════════════════════════════
```

---

## Conversation Examples

### Example 1: "USDT 放邊度收益最高？"

**Intent:** Scan for USDT yield opportunities.
**Command:** `scan --assets USDT`
**Flow:**
1. Fetch OKX Earn USDT rates (flexible, fixed 7d/30d/90d)
2. Fetch DeFiLlama pools for USDT lending (Aave, Compound, Morpho, Venus)
3. GoPlus check on protocol tokens
4. Compute risk-adjusted APY, rank, output table
**Output:** Yield comparison table with CEX vs DeFi lending rates for USDT.

### Example 2: "ETH staking 同 OKX 理財邊個好？"

**Intent:** Compare ETH staking vs CEX earn products.
**Command:** `compare --asset ETH --size-usd 10000`
**Flow:**
1. Fetch OKX Earn ETH flexible/fixed rates
2. Fetch Lido, Rocket Pool, Coinbase cbETH staking rates
3. Calculate switching costs (CEX withdrawal -> staking contract)
4. Calculate break-even period
5. Side-by-side output with risk-adjusted yields
**Output:** Comparison table with switching cost analysis and recommendation.

### Example 3: "幫我重新分配收益最大化"

**Intent:** Rebalance portfolio for maximum yield.
**Command:** `rebalance --assets USDT,ETH,SOL --total-usd 50000 --risk-tolerance moderate`
**Flow:**
1. Scan all opportunities for each asset
2. Run portfolio optimization: maximize risk-adjusted APY subject to:
   - max 25% per protocol
   - respect risk-tolerance filter
   - include switching costs in optimization
3. Output allocation plan with expected returns
**Output:** Rebalance plan with allocation table and aggregate projections.

---

## Error Handling

| Error Code | Trigger | User Message (ZH) | User Message (EN) | Recovery |
|------------|---------|-------------------|-------------------|----------|
| `MCP_NOT_CONNECTED` | DeFiLlama MCP unreachable | DeFi 收益數據服務無法連線，僅顯示 CEX 收益 | DeFi yield data service unreachable. Showing CEX yields only. | Degrade gracefully to CEX-only |
| `AUTH_FAILED` | API key invalid or expired | API 認證失敗，請檢查 OKX API 金鑰設定 | API authentication failed. Check OKX API key configuration. | Update `~/.okx/config.toml` |
| `SECURITY_BLOCKED` | GoPlus flags protocol token | 安全檢查未通過：{reason}。已從推薦中移除。 | Security check failed: {reason}. Removed from recommendations. | Do not proceed. Show specific findings. |
| `INSUFFICIENT_LIQUIDITY` | Protocol TVL < $10M | 協議 TVL 不足 (${ tvl })，不符合最低 $10M 要求 | Protocol TVL insufficient (${tvl}), minimum $10M required. | Show TVL details |
| `DATA_STALE` | DeFiLlama data > 60s old | 收益數據已過期，正在重新獲取... | Yield data stale. Refetching... | Auto-retry, then error |
| `NOT_PROFITABLE` | Switching cost > yield benefit | 切換成本高於收益差額，不建議轉移 | Switching cost exceeds yield benefit. Not recommended. | Show full cost breakdown |
| `GAS_TOO_HIGH` | Gas cost exceeds % of yield | Gas 費用佔收益 {pct}%，超過上限 {limit}% | Gas cost is {pct}% of yield, exceeding {limit}% limit | Wait for lower gas or switch chains |
| `SECURITY_UNAVAILABLE` | GoPlus API unreachable | 安全檢查服務暫時無法使用，請謹慎操作 | Security check service unavailable. Proceed with extreme caution. | Retry once, then WARN and continue |

When DeFiLlama MCP is unreachable, the skill should degrade gracefully:
output CEX-only rates and clearly indicate that DeFi data is unavailable.

---

## Cross-Reference

| Reference | File | Sections Used |
|-----------|------|---------------|
| Formulas | `references/formulas.md` | Section 5 (Risk-Adjusted APY, Switching Cost, Break-Even) |
| Formulas | `references/formulas.md` | Section 6 (Impermanent Loss) |
| Fee Schedule | `references/fee-schedule.md` | Section 2 (Withdrawal Fees), Section 3 (Gas Benchmarks) |
| Safety Checks | `references/safety-checks.md` | Full checklist, yield-specific thresholds |
| Output Templates | `references/output-templates.md` | Section 7 (Yield Comparison), Section 4 (Safety Checks) |
| Risk Limits | `config/risk-limits.example.yaml` | `yield_optimizer` section |
| GoPlus Tools | `references/goplus-tools.md` | `check_token_security` for DeFi protocol tokens |
| Agent Orchestration | `AGENTS.md` | Chain D (Yield Optimization) |

> **Note:** All content from the above reference files has been inlined in this document for OpenClaw/Lark compatibility. The LLM does not need to read any external files.
