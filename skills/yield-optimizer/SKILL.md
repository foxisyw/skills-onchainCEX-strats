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
- Provide financial advice — outputs are analysis only.
- Handle arbitrage — use `cex-dex-arbitrage`, `funding-rate-arbitrage`, or
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

- **OKX Earn rates** — `okx-trade-mcp` may not directly expose OKX Earn
  (savings/staking) product rates. When unavailable, document this gap in
  output and advise the user to check OKX Earn manually. Use publicly
  available snapshots or CoinGecko lending rate data as a fallback.
- **DeFiLlama pool data** — Pool-level yields change frequently. Data should
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
| `--assets` | string[] | Yes | — | Any token symbol | Uppercase, comma-separated |
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
| `--source` | string | Yes | — | Protocol name | Non-empty |
| `--asset` | string | Yes | — | Token symbol | Uppercase |
| `--chain` | string | Yes | — | Chain name | Supported chain |
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
| `--asset` | string | Yes | — | Token symbol | Uppercase |
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
| `--assets` | string[] | Yes | — | Token symbols | Uppercase, comma-separated |
| `--total-usd` | number | Yes | — | 100 - 10,000,000 | Positive |
| `--risk-tolerance` | string | No | `moderate` | `conservative`, `moderate`, `aggressive` | Exact match |
| `--max-protocols` | integer | No | `5` | 1 - 20 | Positive integer |
| `--current-allocation` | string | No | — | JSON or "none" | Valid JSON or omit |

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
| `--asset` | string | Yes | — | Token symbol | Uppercase |
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
1. `okx-trade-mcp` → `market_get_ticker` for asset spot price (freshness reference)
2. OKX Earn rates — attempt to retrieve via API; if unavailable, note the
   limitation and use publicly referenced rates or CoinGecko lending data

**DeFi Lending Yields:**
3. DeFiLlama MCP → `get_latest_pool_data` → filter by asset symbol
   - Extract: pool name, chain, APY, TVL
   - Focus on: Aave v3, Compound, Morpho, Spark, Venus, Benqi

**DeFi Staking Yields:**
4. CoinGecko MCP → `coins_markets` → staking APR data for PoS assets
   - Lido (stETH), Rocket Pool (rETH), Jito (jitoSOL), Marinade (mSOL)

**LP Yields:**
5. DeFiLlama MCP → pool data for LP positions
   - Curve, Uniswap, Raydium, Orca, Aerodrome
   - Include impermanent loss estimate for volatile pairs

**For Each DeFi Opportunity:**
6. GoPlus → `check_token_security` on protocol governance/reward token
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
```
Reference: `references/formulas.md` Section 5.

**Switching Costs:**
```
switching_cost = exit_gas + exit_slippage + bridge_fee + entry_gas + entry_slippage
```
For CEX → DeFi: add withdrawal fee from `references/fee-schedule.md`.
For DeFi → DeFi: add gas on both chains + bridge fee if cross-chain.
For DeFi → CEX: add gas + deposit confirmation time (no fee).

**Break-Even Period:**
```
break_even_days = switching_cost / (new_daily_yield - old_daily_yield)
daily_yield = position_size * apy / 365
```
Reference: `references/formulas.md` Section 5.

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

## Safety Checks

All checks run through the Pre-Trade Safety Checklist defined in
`references/safety-checks.md`. Additional yield-specific checks:

| # | Check | Source | Threshold | Action |
|---|-------|--------|-----------|--------|
| 1 | Protocol TVL | DeFiLlama MCP | < $10M | **BLOCK** — `INSUFFICIENT_LIQUIDITY` |
| 2 | GoPlus token security | GoPlus `check_token_security` | Any flag on protocol token | **BLOCK** — `SECURITY_BLOCKED` |
| 3 | Unaudited contract | Config `reject_unaudited_contracts` | `true` + no audit | **BLOCK** — `SECURITY_BLOCKED` |
| 4 | Single protocol allocation | Config `max_protocol_allocation_pct` | > 25% (default) | **BLOCK** — reduce allocation |
| 5 | Lock period vs horizon | User's stated horizon | Lock > horizon | **WARN** — inform user |
| 6 | Gas cost vs yield | `references/fee-schedule.md` | Gas > 5% of first-year yield | **WARN** — `GAS_TOO_HIGH` |
| 7 | Switching benefit | Config `min_switching_benefit_pct` | < 0.5% APY improvement | **WARN** — may not be worth switching |
| 8 | APY sustainability | 7-day APY trend | Declining > 30% in 7d | **WARN** — yield may not persist |

**Important:** Include switching costs in ALL recommendations. A higher-APY
opportunity with high switching costs may be worse than the current position.

---

## Risk Limits

From `config/risk-limits.example.yaml` — yield-optimizer section:

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| `max_protocol_allocation_pct` | 25% | 50% | Max allocation to any single DeFi protocol |
| `min_protocol_tvl_usd` | $10,000,000 | $1,000,000 | Minimum protocol TVL to consider |
| `reject_unaudited_contracts` | `true` | — | Block protocols without public audit |
| `min_switching_benefit_pct` | 0.5% | — | Min APY improvement to recommend switch |
| `max_gas_cost_pct` | 5% | — | Gas as % of first-year yield must be below this |

---

## Output Templates

### Yield Comparison Table (Primary Output for `scan` and `compare`)

Uses the Yield Comparison Template from `references/output-templates.md` Section 7.

```
══════════════════════════════════════════
  Yield Optimizer
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
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

  [  SAFE  ] Lido — GoPlus verified, TVL $14.2B
  [  SAFE  ] Curve — GoPlus verified, TVL $2.1B
  [  WARN  ] Uniswap V3 LP — Impermanent loss risk: -0.41% at ±20% price move
  [  SAFE  ] Aave v3 — GoPlus verified, TVL $8.5B

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
  Yield Optimizer — Rebalance
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
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
3. Calculate switching costs (CEX withdrawal → staking contract)
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

## Impermanent Loss Reference

For LP opportunities involving volatile pairs, always calculate and display
the IL estimate. Reference: `references/formulas.md` Section 6.

```
IL = 2 * sqrt(price_ratio) / (1 + price_ratio) - 1
```

Display the IL Quick Reference Table when evaluating LP positions:

| Price Change | IL |
|-------------|-----|
| ±10% | -0.14% |
| ±25% | -0.60% |
| ±50% | -2.02% |
| ±100% | -5.72% |

For stable pairs (ETH/stETH, USDC/USDT), note that IL is minimal as long as
the peg holds, but include de-peg risk in the risk score.

---

## Error Handling

| Error Code | Trigger | User Message |
|------------|---------|-------------|
| `MCP_NOT_CONNECTED` | DeFiLlama MCP unreachable | DeFi 收益數據服務無法連線，僅顯示 CEX 收益 |
| `SECURITY_BLOCKED` | GoPlus flags protocol token | 安全檢查未通過：{reason}。已從推薦中移除。 |
| `INSUFFICIENT_LIQUIDITY` | Protocol TVL < $10M | 協議 TVL 不足 (${ tvl })，不符合最低 $10M 要求 |
| `DATA_STALE` | DeFiLlama data > 60s old | 收益數據已過期，正在重新獲取... |
| `NOT_PROFITABLE` | Switching cost > yield benefit | 切換成本高於收益差額，不建議轉移 |

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
