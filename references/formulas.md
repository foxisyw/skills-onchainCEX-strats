# Formula Reference

All calculation formulas used across the 7 Onchain x CEX trading skills.
Every formula includes variable definitions, a worked example, and caveats.

---

## 1. Price Spread Calculations

### CEX-DEX Spread (basis points)

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

---

### Effective Price After Slippage

```
effective_price = quoted_price * (1 + slippage_bps / 10000)   # for buys
effective_price = quoted_price * (1 - slippage_bps / 10000)   # for sells
```

| Variable | Definition |
|----------|-----------|
| `quoted_price` | Price returned by the DEX quote or CEX orderbook top-of-book |
| `slippage_bps` | Estimated execution slippage in basis points |

**Worked Example (Buy)**

```
quoted_price = 3,412.50
slippage_bps = 5

effective_price = 3412.50 * (1 + 5 / 10000)
               = 3412.50 * 1.0005
               = 3414.21
```

**Caveats**
- DEX quotes from aggregators typically already account for price impact at the quoted size. Additional slippage tolerance is for execution delay / mempool movement.
- On CEX, slippage depends on orderbook depth vs order size.

---

### Price Impact Estimation from Orderbook Depth

```
price_impact_bps = (execution_avg_price - best_price) / best_price * 10000
```

Where `execution_avg_price` is the volume-weighted average price across consumed levels:

```
execution_avg_price = sum(level_price_i * level_qty_i) / sum(level_qty_i)
    for levels consumed until order_size is filled
```

| Variable | Definition |
|----------|-----------|
| `best_price` | Best bid (sell) or best ask (buy) |
| `level_price_i` | Price at orderbook level i |
| `level_qty_i` | Quantity available at level i |
| `order_size` | Total size of the order |

**Worked Example (Selling 10 ETH)**

```
Orderbook bids:
  Level 1: 3,412.50 x 3 ETH
  Level 2: 3,412.00 x 4 ETH
  Level 3: 3,411.20 x 5 ETH

execution_avg_price = (3412.50*3 + 3412.00*4 + 3411.20*3) / (3+4+3)
                    = (10237.50 + 13648.00 + 10233.60) / 10
                    = 34119.10 / 10
                    = 3,411.91

price_impact_bps = (3412.50 - 3411.91) / 3412.50 * 10000
                 = 0.59 / 3412.50 * 10000
                 = 1.7 bps
```

**Caveats**
- Orderbook depth is a snapshot; it can change by the time the order executes.
- Hidden/iceberg orders are not visible.
- For large sizes, use OKX `books5` or `books` channel for deeper levels.

---

## 2. Cost Calculations

### CEX Trading Fee

```
cex_fee = size_usd * fee_rate
```

| Variable | Definition |
|----------|-----------|
| `size_usd` | Notional trade size in USD |
| `fee_rate` | Maker or taker rate per VIP tier (see fee-schedule.md) |

**OKX VIP Tier Quick Reference**

| Tier | Maker | Taker |
|------|-------|-------|
| VIP0 | 0.060% | 0.080% |
| VIP1 | 0.040% | 0.070% |
| VIP2 | 0.030% | 0.060% |
| VIP3 | 0.020% | 0.050% |
| VIP4 | 0.015% | 0.040% |
| VIP5 | 0.010% | 0.035% |

**Worked Example (VIP1 Taker, $50,000 trade)**

```
cex_fee = 50000 * 0.0007
        = $35.00
```

**Caveats**
- Maker/taker distinction matters: limit orders that rest on the book = maker; market orders = taker.
- For a round-trip (open + close), fees are incurred twice.
- Swap/futures fees differ from spot fees — see fee-schedule.md for full table.

---

### DEX Gas Cost

```
gas_cost_usd = gas_price_gwei * gas_limit * native_token_price_usd / 1e9
```

| Variable | Definition |
|----------|-----------|
| `gas_price_gwei` | Current gas price in gwei (for EVM chains) |
| `gas_limit` | Gas units required for the transaction |
| `native_token_price_usd` | Current USD price of the chain's native token (ETH, BNB, MATIC, etc.) |

**Worked Example (Uniswap swap on Ethereum)**

```
gas_price_gwei = 30
gas_limit = 150,000
native_token_price_usd = 3,400

gas_cost_usd = 30 * 150000 * 3400 / 1e9
             = 15,300,000,000 / 1,000,000,000
             = $15.30
```

**Worked Example (Jupiter swap on Solana)**

Solana uses lamports instead of gwei:

```
tx_fee_lamports = 5,000  (base fee)
priority_fee_lamports = 50,000  (typical priority fee)
sol_price_usd = 145

gas_cost_usd = (5000 + 50000) * 145 / 1e9
             = 7,975,000 / 1,000,000,000
             = $0.008
```

**Caveats**
- Gas prices are highly volatile, especially on Ethereum mainnet.
- Complex swaps (multi-hop, multi-pool) consume more gas.
- L2s (Arbitrum, Base) also pay an L1 data posting fee that fluctuates.
- Always fetch real-time gas estimates; benchmarks are rough guides only.

---

### Slippage Cost

```
slippage_cost_usd = size_usd * estimated_slippage_bps / 10000
```

| Variable | Definition |
|----------|-----------|
| `size_usd` | Notional trade size in USD |
| `estimated_slippage_bps` | Expected slippage in basis points based on liquidity analysis |

**Worked Example**

```
size_usd = 50,000
estimated_slippage_bps = 8

slippage_cost_usd = 50000 * 8 / 10000
                  = $40.00
```

**Caveats**
- Slippage is an estimate; actual execution may differ.
- For DEX, the quote price already includes price impact — slippage here refers to additional movement between quote and execution.
- For CEX, slippage depends on orderbook depth and execution method.

---

### Total Cost Aggregation

```
total_cost = cex_fee + dex_gas_cost + dex_protocol_fee + slippage_cost + bridge_fee + withdrawal_fee
```

For a round-trip trade:

```
total_cost_roundtrip = (cex_fee_entry + cex_fee_exit)
                     + (dex_gas_entry + dex_gas_exit)
                     + (dex_protocol_fee_entry + dex_protocol_fee_exit)
                     + (slippage_entry + slippage_exit)
                     + bridge_fee  (if cross-chain)
                     + withdrawal_fee  (if moving between CEX/DEX)
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

---

### Net Profit

```
net_profit = gross_spread_usd - total_cost
```

```
gross_spread_usd = size_usd * spread_bps / 10000
```

**Worked Example**

```
size_usd = 50,000
spread_bps = 21.4

gross_spread_usd = 50000 * 21.4 / 10000 = $107.00
total_cost = $51.30

net_profit = 107.00 - 51.30 = $55.70
```

**Caveats**
- Profit is only realized if both legs execute successfully.
- Execution risk (partial fills, failed txns) is not captured in this formula.
- Always compute profit-to-cost ratio (see Risk Metrics) to assess attractiveness.

---

## 3. Funding Rate Formulas

### Annualized Funding Rate

```
annualized_funding = funding_rate_per_8h * 3 * 365
```

| Variable | Definition |
|----------|-----------|
| `funding_rate_per_8h` | Single funding payment rate (e.g., 0.01% = 0.0001) |
| `3` | Funding payments per day (every 8 hours) |
| `365` | Days per year |

**Worked Example**

```
funding_rate_per_8h = 0.0150%  (= 0.000150)

annualized_funding = 0.000150 * 3 * 365
                   = 0.16425
                   = 16.43%
```

**Caveats**
- Funding rates are variable and reset every 8h; annualization assumes a constant rate, which is unrealistic.
- Use a rolling average (e.g., 7-day) for more stable projections.
- Negative funding means shorts pay longs; positive means longs pay shorts.

---

### Net Carry (Funding Harvest)

```
net_carry = annualized_funding - borrow_rate - annualized_trading_fees
```

| Variable | Definition |
|----------|-----------|
| `annualized_funding` | Expected annualized funding income (from above formula) |
| `borrow_rate` | Annualized cost of borrowing margin (if applicable) |
| `annualized_trading_fees` | Entry + exit fee annualized, or per-trade fees if not holding long-term |

**Worked Example**

```
annualized_funding = 16.43%
borrow_rate = 5.00%  (USDT borrow on OKX)
annualized_trading_fees = 0.14%  (VIP1 taker round-trip: 0.07% * 2)

net_carry = 16.43% - 5.00% - 0.14%
          = 11.29%
```

**Caveats**
- Borrow rate may be zero if using own capital with no margin borrowing.
- For delta-neutral funding harvest (spot long + perp short), no directional risk but still exposed to funding rate flipping negative.

---

### Daily Income Projection

```
daily_income = position_size * funding_rate_per_8h * 3
```

**Worked Example**

```
position_size = $100,000
funding_rate_per_8h = 0.0150%

daily_income = 100000 * 0.000150 * 3
             = $45.00
```

---

### Break-Even Holding Period

```
break_even_days = (entry_cost + exit_cost) / daily_income
```

| Variable | Definition |
|----------|-----------|
| `entry_cost` | Total cost to enter the position (CEX fees, DEX gas, slippage, etc.) |
| `exit_cost` | Total cost to exit the position |
| `daily_income` | Projected daily income from funding (from above) |

**Worked Example**

```
entry_cost = $80.00  (spot buy fee + perp short fee + slippage)
exit_cost  = $80.00  (closing both legs)
daily_income = $45.00

break_even_days = (80 + 80) / 45
               = 3.56 days
```

**Caveats**
- If funding rate drops or flips, break-even extends.
- Monitor funding rate trend — a declining trend suggests exiting before break-even may be optimal.

---

## 4. Basis Trading Formulas

### Basis Percentage

```
basis_pct = (futures_price - spot_price) / spot_price
```

| Variable | Definition |
|----------|-----------|
| `futures_price` | Current price of the futures contract (e.g., BTC-USD-260327) |
| `spot_price` | Current spot price |

**Worked Example**

```
futures_price = 88,250.00  (BTC-USD-260627)
spot_price    = 87,500.00

basis_pct = (88250 - 87500) / 87500
          = 750 / 87500
          = 0.00857
          = 0.857%
```

**Caveats**
- Positive basis (contango) = futures > spot. Negative basis (backwardation) = futures < spot.
- Basis captures both interest rate expectations and market sentiment.

---

### Annualized Basis Yield

```
annualized_yield = basis_pct * (360 / days_to_expiry)
```

| Variable | Definition |
|----------|-----------|
| `basis_pct` | Current basis percentage (from above) |
| `days_to_expiry` | Calendar days until futures settlement |

**Worked Example**

```
basis_pct = 0.857%
days_to_expiry = 110

annualized_yield = 0.00857 * (360 / 110)
                 = 0.00857 * 3.2727
                 = 0.02804
                 = 2.80%
```

**Caveats**
- Uses 360-day convention (money market standard). Some sources use 365.
- Annualization assumes you can roll at the same basis, which is not guaranteed.

---

### Net Yield After Fees and Margin Cost

```
net_yield = annualized_yield - annualized_entry_exit_fees - margin_cost
```

```
annualized_entry_exit_fees = (entry_fee_bps + exit_fee_bps) / 10000 * (360 / days_to_expiry)
margin_cost = borrow_rate * margin_utilization
```

| Variable | Definition |
|----------|-----------|
| `entry_fee_bps` | Total entry fee in bps (spot + futures leg) |
| `exit_fee_bps` | Total exit fee in bps (spot + futures leg) |
| `borrow_rate` | Annualized cost of margin borrowing |
| `margin_utilization` | Fraction of position that is borrowed (e.g., 0.5 for 2x) |

**Worked Example**

```
annualized_yield = 2.80%
entry_fee_bps = 14  (7 bps spot taker + 7 bps futures taker, VIP1)
exit_fee_bps  = 14
days_to_expiry = 110
borrow_rate = 5.00%
margin_utilization = 0.0  (fully funded, no borrowing)

annualized_entry_exit_fees = (14 + 14) / 10000 * (360 / 110)
                           = 0.0028 * 3.2727
                           = 0.917%

net_yield = 2.80% - 0.917% - 0.00%
          = 1.88%
```

**Caveats**
- For short-duration contracts, fees eat a larger % of basis.
- Margin cost only applies if using borrowed funds or cross-margin with utilization.

---

### Roll Yield

```
roll_yield = (near_basis_pct - far_basis_pct) * (360 / days_between_expiries)
```

Alternatively, when rolling from expiring contract to next:

```
roll_yield = (new_futures_price - old_futures_price) / spot_price - (near_basis already captured)
```

| Variable | Definition |
|----------|-----------|
| `near_basis_pct` | Basis of the expiring (near) contract |
| `far_basis_pct` | Basis of the next (far) contract |
| `days_between_expiries` | Days between the two expiry dates |

**Worked Example**

```
near_basis_pct = 0.30%   (BTC-USD-260327, 18 days to expiry)
far_basis_pct  = 0.857%  (BTC-USD-260627, 110 days to expiry)
days_between_expiries = 92

roll_yield = (0.30% - 0.857%) is negative, meaning rolling costs money in contango
           = the new position locks in far_basis over a longer period

In practice, you evaluate:
  near_annualized = 0.30% * (360/18) = 6.00%
  far_annualized  = 0.857% * (360/110) = 2.80%

  If near_annualized > far_annualized, the near contract is richer.
```

**Caveats**
- Roll involves closing one position and opening another — double the trading fees.
- If far contract has better annualized yield, rolling may be worthwhile despite fees.
- Always compare net annualized yields (after fees) for roll decisions.

---

## 5. Yield Optimization Formulas

### Risk-Adjusted APY

```
risk_adjusted_apy = raw_apy * (1 - risk_score / 10)
```

| Variable | Definition |
|----------|-----------|
| `raw_apy` | Advertised or observed annual percentage yield |
| `risk_score` | Risk assessment score on a 0-10 scale (0 = lowest risk, 10 = highest risk) |

**Risk Score Components:**
- Protocol audit status (0-2)
- TVL tier (0-2)
- Token concentration risk (0-2)
- Smart contract age (0-2)
- Chain risk (0-2)

**Worked Example**

```
raw_apy = 12.5%
risk_score = 3  (audited, high TVL, moderate token risk)

risk_adjusted_apy = 12.5% * (1 - 3/10)
                  = 12.5% * 0.70
                  = 8.75%
```

**Caveats**
- Risk score is subjective; use consistently across opportunities for relative comparison.
- A risk_score of 10 yields 0% adjusted APY (effectively "do not enter").
- Raw APY may include token emissions that are volatile in value.

---

### Switching Cost Calculation

```
switching_cost = exit_gas + exit_slippage + bridge_fee + entry_gas + entry_slippage
```

For yield switching in USD:

```
switching_cost_bps = switching_cost / position_size * 10000
```

**Worked Example**

```
exit_gas (Ethereum)     = $12.00
exit_slippage           =  $5.00
bridge_fee (ETH -> Arb) =  $2.00
entry_gas (Arbitrum)    =  $0.30
entry_slippage          =  $3.00

switching_cost = 12.00 + 5.00 + 2.00 + 0.30 + 3.00 = $22.30

position_size = $50,000
switching_cost_bps = 22.30 / 50000 * 10000 = 4.5 bps
```

---

### Break-Even Period for Rebalancing

```
break_even_days = switching_cost / (new_daily_yield - old_daily_yield)
```

Where:

```
daily_yield = position_size * apy / 365
```

**Worked Example**

```
old_apy = 5.00%
new_apy = 8.50%
position_size = $50,000
switching_cost = $22.30

old_daily_yield = 50000 * 0.05 / 365 = $6.85
new_daily_yield = 50000 * 0.085 / 365 = $11.64

break_even_days = 22.30 / (11.64 - 6.85)
               = 22.30 / 4.79
               = 4.66 days
```

**Caveats**
- Only switch if you expect the yield differential to persist longer than break-even.
- Factor in risk-adjusted APY, not raw APY, for a fairer comparison.
- Yield rates change constantly; a 3.5% edge today may vanish tomorrow.

---

## 6. Impermanent Loss (LP Hedge Reference)

### Impermanent Loss Formula

```
IL = 2 * sqrt(price_ratio) / (1 + price_ratio) - 1
```

| Variable | Definition |
|----------|-----------|
| `price_ratio` | New price / original price (e.g., if ETH went from $3,000 to $3,600, ratio = 1.20) |
| `IL` | Fractional loss compared to simply holding (always negative or zero) |

**Worked Example**

```
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

**IL Quick Reference Table**

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

**Caveats**
- IL is symmetric with respect to price movement magnitude, not direction.
- Concentrated liquidity (Uniswap v3) amplifies IL within the range but earns more fees.
- IL is only "realized" when you withdraw; if price returns to entry, IL = 0.
- LP fee income may offset IL; always compare net position value.

---

## 7. Risk Metrics

### Position Size as % of NAV

```
position_pct = position_size / net_asset_value * 100
```

| Variable | Definition |
|----------|-----------|
| `position_size` | Notional value of the position in USD |
| `net_asset_value` | Total portfolio value in USD |

**Worked Example**

```
position_size = $50,000
net_asset_value = $500,000

position_pct = 50000 / 500000 * 100 = 10.0%
```

**Guidelines:**
- Single position: max 10-20% of NAV for moderate risk
- Correlated positions: aggregate exposure should not exceed 30-40%
- High-risk (unaudited protocols, new chains): max 5% of NAV

---

### Profit-to-Cost Ratio

```
profit_to_cost = net_profit / total_cost
```

| Variable | Definition |
|----------|-----------|
| `net_profit` | Expected net profit from the trade |
| `total_cost` | Total costs (fees, gas, slippage, etc.) |

**Worked Example**

```
net_profit = $55.70
total_cost = $51.30

profit_to_cost = 55.70 / 51.30 = 1.09x
```

**Interpretation:**
- < 1.0x: Costs exceed profit — do not trade
- 1.0x - 1.5x: Marginal — high execution risk, consider skipping
- 1.5x - 3.0x: Acceptable — proceed with caution
- 3.0x+: Attractive — high confidence trade

---

### Signal Decay Function

Used to discount stale signals (e.g., smart money movements, spread opportunities):

```
signal_strength = initial_strength * exp(-decay_rate * time_elapsed_minutes)
```

| Variable | Definition |
|----------|-----------|
| `initial_strength` | Original signal strength (0.0 to 1.0) |
| `decay_rate` | Decay constant (higher = faster decay) |
| `time_elapsed_minutes` | Minutes since signal was generated |

**Typical Decay Rates:**

| Signal Type | Decay Rate | Half-Life |
|------------|------------|-----------|
| CEX-DEX arb spread | 0.15 | ~4.6 min |
| Funding rate anomaly | 0.01 | ~69 min |
| Basis mispricing | 0.005 | ~139 min |
| Smart money signal | 0.02 | ~35 min |
| Yield opportunity | 0.001 | ~693 min |

**Worked Example**

```
initial_strength = 0.90  (strong arb signal)
decay_rate = 0.15
time_elapsed_minutes = 5

signal_strength = 0.90 * exp(-0.15 * 5)
               = 0.90 * exp(-0.75)
               = 0.90 * 0.4724
               = 0.425

Signal has decayed from 0.90 to 0.43 in 5 minutes — opportunity may be fading.
```

**Caveats**
- Decay rates are empirical estimates; calibrate based on observed opportunity persistence.
- Signal strength below 0.3 generally means the opportunity has likely been arbitraged away.
- Combine signal decay with a freshness timestamp in output displays.

---

## Cross-Reference

| Formula | Primary Skill(s) | Reference Section |
|---------|------------------|-------------------|
| CEX-DEX spread | Skill 1 (CEX-DEX Arb) | Section 1 |
| Cost aggregation | All skills | Section 2 |
| Funding rate annualization | Skill 2 (Funding Harvester) | Section 3 |
| Break-even holding | Skill 2, Skill 3 | Sections 3, 4 |
| Basis yield | Skill 3 (Basis Trading) | Section 4 |
| Risk-adjusted APY | Skill 5 (Yield Optimizer) | Section 5 |
| Impermanent loss | Skill 6 (LP Hedge) | Section 6 |
| Profit-to-cost ratio | All skills | Section 7 |
| Signal decay | Skill 7 (Smart Money) | Section 7 |
