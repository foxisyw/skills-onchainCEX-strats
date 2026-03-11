# Onchain x CEX Strats — Agent Orchestration

## Skill Routing Matrix

When a user request arrives, route to the appropriate skill based on this matrix:

| User Intent | Primary Skill | Supporting Skills |
|-------------|--------------|-------------------|
| "CEX 同 DEX 有冇價差" / "arbitrage opportunities" | `cex-dex-arbitrage` | price-feed-aggregator, profitability-calculator |
| "邊個幣資金費率最高" / "funding rate scan" | `funding-rate-arbitrage` | profitability-calculator |
| "期貨基差幾多" / "basis trade" / "cash and carry" | `basis-trading` | profitability-calculator |
| "邊度收益最好" / "best yield" / "DeFi vs CEX" | `yield-optimizer` | - |
| "聰明錢買咗乜" / "smart money" / "whale activity" | `smart-money-tracker` | - |
| "呢個幣安唔安全" / "is this token safe" | Delegate to GoPlus MCP directly | - |
| "我嘅帳戶餘額" / "my balance" | Delegate to okx-trade-mcp directly | - |
| "比較兩個價格" / "price comparison" | `price-feed-aggregator` | - |
| "呢個交易賺唔賺" / "is this profitable" | `profitability-calculator` | - |
| "清算級聯" / "liquidation cascade" / "DeFi 健康因子" | `liquidation-cascade-monitor` | profitability-calculator |
| "穩定幣脫錨" / "stablecoin depeg" / "USDT 脫鈎" / "Curve 3pool 比例" | `stablecoin-depeg-arbitrage` | profitability-calculator |
| "LP 對沖" / "LP hedge" / "無常損失" / "delta neutral LP" | `lp-hedge` | profitability-calculator, funding-rate-arbitrage |
| "跨鏈套利" / "cross-chain arb" / "bridge arbitrage" / "L2 價差" | `cross-chain-arbitrage` | profitability-calculator, price-feed-aggregator |

## Cross-Skill Workflow Chains

### Chain A: CEX-DEX Arbitrage (Full Pipeline)

```
1. price-feed-aggregator.snapshot
   │  → PriceSnapshot[]
   ▼
2. cex-dex-arbitrage.scan
   │  → Calls GoPlus for token security
   │  → Calls OnchainOS for DEX liquidity
   │  → Filters by min_spread_bps
   ▼
3. profitability-calculator.estimate
   │  → Net P&L after all costs
   ▼
4. Output: Ranked opportunities with safety checks
```

### Chain B: Funding Rate Carry Trade

```
1. funding-rate-arbitrage.scan
   │  → Fetches funding rates from OKX
   │  → Compares with borrow rates
   ▼
2. profitability-calculator.estimate
   │  → Entry/exit cost analysis
   ▼
3. funding-rate-arbitrage.carry-pnl
   │  → Projected income with scenarios
   ▼
4. Output: Ranked carry trades with projections
```

### Chain C: Basis Trading

```
1. basis-trading.scan
   │  → Fetches spot + futures prices from OKX
   │  → Calculates annualized basis
   ▼
2. profitability-calculator.estimate
   │  → Trading fee impact
   ▼
3. basis-trading.curve
   │  → Term structure visualization
   ▼
4. Output: Basis opportunities with yield curves
```

### Chain D: Yield Optimization

```
1. yield-optimizer.scan
   │  → OKX Earn rates + DeFiLlama yields + OnchainOS LP APRs
   ▼
2. GoPlus check_token_security (for DeFi tokens)
   ▼
3. yield-optimizer.compare
   │  → Risk-adjusted yield comparison
   ▼
4. Output: Ranked yield opportunities with switching costs
```

### Chain E: Smart Money Following

```
1. smart-money-tracker.scan
   │  → OnchainOS dex-market signal-list
   ▼
2. GoPlus check_token_security (CRITICAL)
   ▼
3. smart-money-tracker.evaluate
   │  → Signal decay check, actionability
   ▼
4. Output: Signals with [CAUTION] + REQUIRES HUMAN APPROVAL
```

### Chain F: Liquidation Cascade Monitoring

```
1. liquidation-cascade-monitor.scan
   │  → DeFiLlama liquidation data + Aave/Compound health factors
   │  → Identifies positions with health_factor < 1.10
   ▼
2. liquidation-cascade-monitor.evaluate
   │  → Calculates cascade_impact = liquidatable_usd / market_depth
   │  → Checks OKX perp availability for affected token
   ▼
3. profitability-calculator.estimate
   │  → Entry/exit cost for CEX perp position
   ▼
4. Output: Cascade risk assessment with CEX short recommendation
```

### Chain G: Stablecoin Depeg Arbitrage

```
1. stablecoin-depeg-arbitrage.scan
   │  → CEX stablecoin prices + Curve 3pool ratios
   │  → Identifies deviations > 0.5%
   ▼
2. stablecoin-depeg-arbitrage.evaluate
   │  → Solvency vs liquidity depeg classification (CRITICAL)
   │  → GoPlus contract safety check
   ▼
3. profitability-calculator.estimate
   │  → Net P&L after CEX fees + gas + slippage
   ▼
4. Output: Depeg arb opportunity with solvency safety label
```

### Chain H: LP Hedge (Delta-Neutral Yield Farming)

```
1. lp-hedge.scan
   │  → DeFiLlama pool data with APY > 30%
   │  → Filters for OKX-hedgeable assets
   ▼
2. lp-hedge.hedge-calc
   │  → Delta calculation (V2: 0.5, V3: 0.6-0.8)
   │  → OKX funding rate for hedge cost
   ▼
3. lp-hedge.evaluate
   │  → Net yield = fees + incentives - IL - funding - rebalance
   │  → GoPlus LP token check
   ▼
4. Output: Ranked hedged LP opportunities with net yield
```

### Chain I: Cross-Chain Arbitrage

```
1. cross-chain-arbitrage.scan
   │  → OnchainOS prices on multiple chains
   │  → Identifies cross-chain spread > 100 bps
   ▼
2. cross-chain-arbitrage.bridge-compare
   │  → Bridge fee/speed comparison (Across, Stargate, Synapse)
   │  → Bridge security assessment
   ▼
3. cross-chain-arbitrage.evaluate
   │  → All-in cost: gas + bridge + slippage + hedge
   │  → CEX transit hedge sizing
   ▼
4. Output: Cross-chain arb with bridge route + hedge plan
```

## Data Handoff Conventions

When passing data between skills:

1. **Price data** → Always pass as `PriceSnapshot` schema (see price-feed-aggregator SKILL.md)
2. **Trade legs** → Always pass as `TradeLeg[]` schema (see profitability-calculator SKILL.md)
3. **Profitability results** → Always pass as `ProfitabilityResult` schema
4. **Security results** → Always pass GoPlus raw response; let consuming skill decide BLOCK/WARN

## Global Safety Rules

These rules apply to ALL skills at ALL times:

1. **Demo by default** — Use `okx-DEMO-simulated-trading` unless user explicitly says "live" or "真實帳戶"
2. **Display mode** — Always show `[DEMO]` or `[LIVE]` in output header
3. **Recommendation only** — Always show `[RECOMMENDATION ONLY — 不會自動執行]`
4. **Security check** — For ANY onchain token, call GoPlus before recommending
5. **Staleness** — Reject price data older than 60s (5s for arbitrage)
6. **Human approval** — `smart-money-tracker.copy-plan` ALWAYS requires human confirmation
7. **Language** — Match user's language. Default Traditional Chinese.
8. **Next steps** — Every output ends with 2-3 suggested follow-up actions
