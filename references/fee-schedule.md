# Fee Schedule Reference

Complete fee reference for OKX CEX, DEX protocols, gas costs, and bridges.
All values are approximate and should be verified at execution time via API.

---

## 1. OKX CEX Trading Fees

### Spot Trading Fees

| Tier | 30d Volume (USD) | Maker | Taker |
|------|------------------|-------|-------|
| VIP0 | < 5M | 0.060% | 0.080% |
| VIP1 | >= 5M | 0.040% | 0.070% |
| VIP2 | >= 10M | 0.030% | 0.060% |
| VIP3 | >= 20M | 0.020% | 0.050% |
| VIP4 | >= 100M | 0.015% | 0.040% |
| VIP5 | >= 200M | 0.010% | 0.035% |

### USDT-Margined Swap (Perpetual) Fees

| Tier | 30d Volume (USD) | Maker | Taker |
|------|------------------|-------|-------|
| VIP0 | < 5M | 0.020% | 0.050% |
| VIP1 | >= 5M | 0.015% | 0.045% |
| VIP2 | >= 10M | 0.010% | 0.040% |
| VIP3 | >= 20M | 0.008% | 0.035% |
| VIP4 | >= 100M | 0.005% | 0.030% |
| VIP5 | >= 200M | 0.002% | 0.025% |

### Coin-Margined Swap / Futures Fees

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
- For **cost estimation**, assume taker fees unless the skill specifically uses limit orders
- Fees are deducted from the received asset (spot) or from margin (derivatives)
- **Negative maker fees** (rebates) may apply at VIP5+ for certain instruments — not included here as they are subject to change

### Fee Formula

```
fee_usd = notional_size_usd * fee_rate

# Round-trip cost (open + close)
roundtrip_fee_usd = notional_size_usd * (entry_fee_rate + exit_fee_rate)
roundtrip_fee_bps = (entry_fee_rate + exit_fee_rate) * 10000
```

**Quick Reference: Round-Trip Taker Fees (bps)**

| Tier | Spot RT | Swap RT |
|------|---------|---------|
| VIP0 | 16.0 bps | 10.0 bps |
| VIP1 | 14.0 bps | 9.0 bps |
| VIP2 | 12.0 bps | 8.0 bps |
| VIP3 | 10.0 bps | 7.0 bps |
| VIP4 | 8.0 bps | 6.0 bps |
| VIP5 | 7.0 bps | 5.0 bps |

---

## 2. OKX Withdrawal Fees

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

---

## 3. Gas Benchmarks per Chain

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

### Gas Cost Formula (EVM Chains)

```
gas_cost_usd = gas_price_gwei * gas_limit * native_token_price / 1e9
```

### Gas Cost Formula (Solana)

```
gas_cost_usd = (base_fee_lamports + priority_fee_lamports) * sol_price / 1e9
```

Typical Solana fees:
- Base fee: 5,000 lamports (0.000005 SOL)
- Priority fee: 10,000-500,000 lamports depending on congestion
- Compute budget: 200,000-1,400,000 compute units

### Gas Notes

- **Ethereum L1** gas is by far the most expensive; avoid for small trades.
- **L2 costs** include a small L1 data posting fee that can spike during L1 congestion.
- **Multi-hop swaps** (going through multiple pools) use significantly more gas.
- **Approval transactions** (first-time token approvals) add ~46,000 gas on EVM chains.
- Gas prices fluctuate significantly; always fetch real-time estimates.
- Consider using gas price oracles or the OKX DEX API quote which includes gas estimates.

### Cost Efficiency by Chain (for $10,000 trade)

| Chain | Gas as % of Trade | Practical Minimum Trade |
|-------|-------------------|------------------------|
| Ethereum | 0.05-0.30% | $5,000+ |
| Arbitrum | 0.001-0.005% | $100+ |
| Base | 0.0001-0.0005% | $50+ |
| Solana | 0.00001-0.0001% | $10+ |
| BSC | 0.001-0.003% | $100+ |
| Polygon | 0.0001-0.0005% | $50+ |

---

## 4. DEX Protocol Fees

Fees charged by the DEX protocol itself (separate from gas). These are typically included in the quoted swap price from aggregators.

### Uniswap (Ethereum, Arbitrum, Base, Polygon, Optimism, BSC)

| Pool Tier | Fee | Typical Use |
|-----------|-----|------------|
| 0.01% | 1 bp | Stable-stable pairs (USDC/USDT) |
| 0.05% | 5 bps | Stable pairs, high-correlation pairs |
| 0.30% | 30 bps | Standard pairs (ETH/USDC, WBTC/ETH) |
| 1.00% | 100 bps | Exotic / low-liquidity pairs |

### Curve Finance (Ethereum, Arbitrum, Polygon, others)

| Pool Type | Fee |
|-----------|-----|
| Stablecoin pools | 0.01-0.04% |
| Crypto pools (v2) | 0.04-0.40% |
| Factory pools | Variable |

### PancakeSwap (BSC, Ethereum, Arbitrum, Base)

| Pool Tier | Fee |
|-----------|-----|
| V3 (stable) | 0.01% |
| V3 (standard) | 0.25% |
| V3 (exotic) | 1.00% |
| V2 | 0.25% |

### Jupiter (Solana)

- Jupiter itself charges **no protocol fee** (aggregator layer)
- Underlying pool fees vary:
  - Orca Whirlpools: 0.01%-2.00% (tier-based)
  - Raydium: 0.25% (standard), 0.01% (concentrated)
  - Meteora: 0.01%-1.00% (dynamic)
- Route optimization selects the best fee-adjusted path automatically

### DEX Fee Notes

- **Aggregator quotes already include DEX fees** in the output amount. Do not double-count.
- When comparing CEX vs DEX prices, the DEX price is already net of protocol fees.
- The OKX DEX API aggregates across multiple DEX protocols and returns the best net quote.
- Some protocols have **dynamic fees** that adjust based on volatility (e.g., Uniswap v4 hooks, Meteora).

---

## 5. Bridge Fees

Typical fees for cross-chain asset transfers. Highly variable by bridge, route, and conditions.

### Major Bridge Protocols

| Bridge | Typical Fee Range | Speed | Notes |
|--------|------------------|-------|-------|
| OKX Bridge (via DEX API) | 0-0.1% | 1-15 min | Aggregates multiple bridges |
| Across Protocol | 0.04-0.12% | 1-5 min | Fast, uses relayers |
| Stargate | 0.06% | 5-15 min | LayerZero-based |
| Hop Protocol | 0.05-0.20% | 2-10 min | Uses AMMs |
| Synapse | 0.05-0.15% | 5-15 min | Multi-chain AMM |
| Native Bridges | Free (gas only) | 7 min - 7 days | Slowest, cheapest |

### Native Bridge Wait Times

| Route | Deposit Time | Withdrawal Time |
|-------|-------------|-----------------|
| Ethereum -> Arbitrum | ~10 min | ~7 days (challenge period) |
| Ethereum -> Optimism | ~10 min | ~7 days (challenge period) |
| Ethereum -> Base | ~10 min | ~7 days (challenge period) |
| Ethereum -> Polygon | ~10 min | ~30 min (PoS bridge) |
| Ethereum -> BSC | N/A | N/A (use 3rd party) |

### Bridge Fee Notes

- **OKX withdrawal is often cheaper than bridging** for moving assets from CEX to another chain. Withdraw directly to the target chain when supported.
- Bridge fees include: protocol fee + gas on source chain + gas on destination chain.
- For large amounts (>$100K), verify bridge liquidity on the destination chain to avoid high slippage.
- Bridge exploits are a major risk vector; prefer established bridges with audited contracts.
- When possible, avoid bridging altogether by using native CEX withdrawal to the target chain.

---

## 6. Fee Comparison: CEX Withdrawal vs Bridge

When moving USDT from Ethereum to Arbitrum:

| Method | Cost | Time |
|--------|------|------|
| OKX withdraw to Arbitrum directly | 0.1 USDT | 1-5 min |
| Bridge via Across | ~$2-5 (0.04-0.1%) + gas | 1-5 min |
| Bridge via native bridge | Gas only (~$15-30) | ~10 min + 7 days |

**Recommendation:** Always prefer direct CEX withdrawal to target chain when possible. It is almost always the cheapest and fastest route.

---

## 7. Cost Summary for Common Trade Patterns

### Pattern A: CEX-DEX Arbitrage (Buy CEX, Sell DEX on Arbitrum)

```
Trade size: $50,000

CEX buy (VIP1 taker):         $35.00  (7.0 bps)
CEX withdrawal (ETH to Arb):   $0.34  (0.0001 ETH)
DEX sell gas (Arbitrum):        $0.30
DEX slippage:                  $10.00  (est. 2 bps)
                               ------
Total one-way cost:            $45.64  (9.1 bps)
```

### Pattern B: Funding Harvest (Spot long + Perp short)

```
Position size: $100,000

Spot buy (VIP1 taker):        $70.00  (7.0 bps)
Perp short (VIP1 taker):      $45.00  (4.5 bps)
                               ------
Entry cost:                   $115.00  (11.5 bps)
Exit cost (same):             $115.00  (11.5 bps)
Round-trip:                   $230.00  (23.0 bps)
```

### Pattern C: Basis Trade (Spot long + Futures short)

```
Position size: $100,000

Spot buy (VIP1 taker):        $70.00  (7.0 bps)
Futures short (VIP1 taker):   $45.00  (4.5 bps)
                               ------
Entry cost:                   $115.00  (11.5 bps)
Exit at expiry: futures settle automatically, close spot:
  Spot sell:                   $70.00  (7.0 bps)
                               ------
Total cost:                   $185.00  (18.5 bps)
```
