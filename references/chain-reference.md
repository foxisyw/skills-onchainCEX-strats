# Chain Reference

Comprehensive chain and token reference for all supported chains in the Onchain x CEX trading system.

---

## 1. Supported Chains

| Chain | chainIndex (OKX) | Native Token | Native Token Address (DEX) | RPC Unit | Block Time | Explorer |
|-------|------------------|-------------|---------------------------|----------|------------|----------|
| Ethereum | 1 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~12s | etherscan.io |
| BSC | 56 | BNB | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~3s | bscscan.com |
| Polygon | 137 | MATIC | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~2s | polygonscan.com |
| Arbitrum One | 42161 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~0.25s | arbiscan.io |
| Optimism | 10 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~2s | optimistic.etherscan.io |
| Base | 8453 | ETH | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~2s | basescan.org |
| Avalanche C-Chain | 43114 | AVAX | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | nAVAX | ~2s | snowscan.xyz |
| Solana | 501 | SOL | `11111111111111111111111111111111` | lamports | ~0.4s | solscan.io |
| X Layer | 196 | OKB | `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | gwei | ~3s | oklink.com/xlayer |

---

## 2. Critical Warnings

### Solana Native Token Address

```
CORRECT:   11111111111111111111111111111111
WRONG:     So11111111111111111111111111111111111111112  (this is wSOL, the wrapped version)
```

- Use `11111111111111111111111111111111` (the System Program ID) when referring to native SOL in OKX DEX API calls.
- `So11111111111111111111111111111111111111112` is Wrapped SOL (wSOL) — a different asset used within DeFi protocols.
- When the OKX DEX API returns a quote involving native SOL, it uses the system program address.

### EVM Native Token Address

All EVM-compatible chains use the same placeholder address for native tokens:

```
0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
```

- This is a convention, not a real contract address.
- Applies to: ETH (Ethereum, Arbitrum, Optimism, Base), BNB (BSC), MATIC (Polygon), AVAX (Avalanche), OKB (X Layer).

### Address Formatting

- **EVM chains**: Contract addresses **must be lowercase** (checksummed addresses also accepted by most APIs, but prefer lowercase for consistency).
- **Solana**: Addresses are **base58-encoded** and **case-sensitive**. Do not modify case.
- Always validate addresses before submitting transactions.

### Chain-Specific Gotchas

| Chain | Gotcha |
|-------|--------|
| Ethereum | High gas costs; unsuitable for small arb trades |
| Arbitrum | Gas limit values are much larger than L1 (1M-2M) but cost is low due to low gas price |
| Base | Very cheap but liquidity may be thinner for non-major pairs |
| Optimism | Similar to Base; check liquidity before large trades |
| Solana | Uses compute units + priority fees, not gas/gwei model |
| BSC | Fast blocks but more susceptible to MEV/sandwich attacks |
| Polygon | Gas spikes possible during high activity; use gas oracle |
| X Layer | OKX native chain; limited DeFi ecosystem compared to others |

---

## 3. OKX instId Formats

### Spot

Format: `{BASE}-{QUOTE}`

```
BTC-USDT
ETH-USDT
SOL-USDT
BNB-USDT
MATIC-USDT
AVAX-USDT
ARB-USDT
OP-USDT
OKB-USDT
ETH-BTC
```

### USDT-Margined Perpetual Swaps

Format: `{BASE}-USDT-SWAP`

```
BTC-USDT-SWAP
ETH-USDT-SWAP
SOL-USDT-SWAP
BNB-USDT-SWAP
MATIC-USDT-SWAP
AVAX-USDT-SWAP
ARB-USDT-SWAP
OP-USDT-SWAP
```

### Coin-Margined Perpetual Swaps

Format: `{BASE}-USD-SWAP`

```
BTC-USD-SWAP
ETH-USD-SWAP
SOL-USD-SWAP
```

### Futures (Expiry Contracts)

Format: `{BASE}-USD-{YYMMDD}` or `{BASE}-USDT-{YYMMDD}`

```
BTC-USD-260327      (quarterly, expires 2026-03-27)
BTC-USD-260627      (quarterly, expires 2026-06-27)
ETH-USD-260327
ETH-USD-260627
BTC-USDT-260327
ETH-USDT-260327
```

**Expiry Date Convention:**
- OKX uses `YYMMDD` format
- Quarterly expirations: last Friday of March, June, September, December
- Weekly/bi-weekly contracts also available for BTC and ETH
- Check active instruments via `GET /api/v5/public/instruments?instType=FUTURES`

### Options

Format: `{BASE}-USD-{YYMMDD}-{STRIKE}-{C|P}`

```
BTC-USD-260327-90000-C    (BTC call, strike 90000, expires 2026-03-27)
ETH-USD-260327-3500-P     (ETH put, strike 3500, expires 2026-03-27)
```

---

## 4. Major Token Contract Addresses

### Ethereum (chainIndex: 1)

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0xdac17f958d2ee523a2206206994597c13d831ec7` | 6 |
| USDC | `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` | 6 |
| WETH | `0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2` | 18 |
| WBTC | `0x2260fac5e5542a773aa44fbcfedf7c193bc2c599` | 8 |
| DAI | `0x6b175474e89094c44da98b954eedeac495271d0f` | 18 |
| LINK | `0x514910771af9ca656af840dff83e8264ecf986ca` | 18 |
| UNI | `0x1f9840a85d5af5bf1d1762f925bdaddc4201f984` | 18 |
| AAVE | `0x7fc66500c84a76ad7e9c93437bfc5ac33e2ddae9` | 18 |

### Arbitrum One (chainIndex: 42161)

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0xfd086bc7cd5c481dcc9c85ebe478a1c0b69fcbb9` | 6 |
| USDC | `0xaf88d065e77c8cc2239327c5edb3a432268e5831` | 6 |
| USDC.e (bridged) | `0xff970a61a04b1ca14834a43f5de4533ebddb5cc8` | 6 |
| WETH | `0x82af49447d8a07e3bd95bd0d56f35241523fbab1` | 18 |
| WBTC | `0x2f2a2543b76a4166549f7aab2e75bef0aefc5b0f` | 8 |
| ARB | `0x912ce59144191c1204e64559fe8253a0e49e6548` | 18 |
| GMX | `0xfc5a1a6eb076a2c7ad06ed22c90d7e710e35ad0a` | 18 |

### Base (chainIndex: 8453)

| Token | Address | Decimals |
|-------|---------|----------|
| USDC | `0x833589fcd6edb6e08f4c7c32d4f71b54bda02913` | 6 |
| USDbC (bridged) | `0xd9aaec86b65d86f6a7b5b1b0c42ffa531710b6ca` | 6 |
| WETH | `0x4200000000000000000000000000000000000006` | 18 |
| cbETH | `0x2ae3f1ec7f1f5012cfeab0185bfc7aa3cf0dec22` | 18 |
| DAI | `0x50c5725949a6f0c72e6c4a641f24049a917db0cb` | 18 |

### Optimism (chainIndex: 10)

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0x94b008aa00579c1307b0ef2c499ad98a8ce58e58` | 6 |
| USDC | `0x0b2c639c533813f4aa9d7837caf62653d097ff85` | 6 |
| USDC.e (bridged) | `0x7f5c764cbc14f9669b88837ca1490cca17c31607` | 6 |
| WETH | `0x4200000000000000000000000000000000000006` | 18 |
| OP | `0x4200000000000000000000000000000000000042` | 18 |

### Polygon (chainIndex: 137)

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0xc2132d05d31c914a87c6611c10748aeb04b58e8f` | 6 |
| USDC | `0x3c499c542cef5e3811e1192ce70d8cc03d5c3359` | 6 |
| USDC.e (bridged) | `0x2791bca1f2de4661ed88a30c99a7a9449aa84174` | 6 |
| WETH | `0x7ceb23fd6bc0add59e62ac25578270cff1b9f619` | 18 |
| WMATIC | `0x0d500b1d8e8ef31e21c99d1db9a6444d3adf1270` | 18 |
| WBTC | `0x1bfd67037b42cf73acf2047067bd4f2c47d9bfd6` | 8 |

### BSC (chainIndex: 56)

| Token | Address | Decimals |
|-------|---------|----------|
| USDT | `0x55d398326f99059ff775485246999027b3197955` | 18 |
| USDC | `0x8ac76a51cc950d9822d68b83fe1ad97b32cd580d` | 18 |
| WBNB | `0xbb4cdb9cbd36b01bd1cbaebf2de08d9173bc095c` | 18 |
| WETH | `0x2170ed0880ac9a755fd29b2688956bd959f933f8` | 18 |
| BTCB | `0x7130d2a12b9bcbfae4f2634d864a1ee1ce3ead9c` | 18 |
| CAKE | `0x0e09fabb73bd3ade0a17ecc321fd13a19e81ce82` | 18 |

### Solana (chainIndex: 501)

| Token | Mint Address | Decimals |
|-------|-------------|----------|
| USDT | `Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB` | 6 |
| USDC | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | 6 |
| wSOL | `So11111111111111111111111111111111111111112` | 9 |
| BONK | `DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263` | 5 |
| JUP | `JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN` | 6 |
| RAY | `4k3Dyjzvzp8eMZWUXbBCjEvwSkkk59S5iCNLY3QrkX6R` | 6 |
| JTO | `jtojtomepa8beP8AuQc6eXt5FriJwfFMwQx2v2f9mCL` | 9 |
| WIF | `EKpQGSJtjMFqKZ9KQanSqYXRcF8fBopzLHYxdM65zcjm` | 6 |

---

## 5. Stablecoin Cross-Reference

Quick reference for USDT and USDC across chains.

### USDT Addresses

| Chain | Address | Decimals |
|-------|---------|----------|
| Ethereum | `0xdac17f958d2ee523a2206206994597c13d831ec7` | 6 |
| Arbitrum | `0xfd086bc7cd5c481dcc9c85ebe478a1c0b69fcbb9` | 6 |
| Optimism | `0x94b008aa00579c1307b0ef2c499ad98a8ce58e58` | 6 |
| Polygon | `0xc2132d05d31c914a87c6611c10748aeb04b58e8f` | 6 |
| BSC | `0x55d398326f99059ff775485246999027b3197955` | 18 |
| Solana | `Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB` | 6 |

### USDC Addresses (Native, not bridged)

| Chain | Address | Decimals |
|-------|---------|----------|
| Ethereum | `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` | 6 |
| Arbitrum | `0xaf88d065e77c8cc2239327c5edb3a432268e5831` | 6 |
| Optimism | `0x0b2c639c533813f4aa9d7837caf62653d097ff85` | 6 |
| Base | `0x833589fcd6edb6e08f4c7c32d4f71b54bda02913` | 6 |
| Polygon | `0x3c499c542cef5e3811e1192ce70d8cc03d5c3359` | 6 |
| BSC | `0x8ac76a51cc950d9822d68b83fe1ad97b32cd580d` | 18 |
| Solana | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | 6 |

**Important:** Many chains have both native USDC and bridged USDC.e. Always use native USDC when available — it has better liquidity and is directly redeemable.

---

## 6. Chain Selection Guide

### For CEX-DEX Arbitrage

| Priority | Chain | Reason |
|----------|-------|--------|
| 1 | Arbitrum | Low gas, deep liquidity, fast blocks |
| 2 | Base | Ultra-low gas, growing liquidity |
| 3 | Solana | Minimal gas, deep Jupiter liquidity |
| 4 | BSC | Low gas, decent liquidity |
| 5 | Polygon | Low gas, moderate liquidity |
| 6 | Ethereum | Only for very large trades ($100K+) where liquidity matters most |

### For Yield Optimization

| Priority | Chain | Reason |
|----------|-------|--------|
| 1 | Ethereum | Deepest DeFi ecosystem, highest TVL |
| 2 | Arbitrum | Strong DeFi ecosystem, low costs |
| 3 | Solana | Growing DeFi, unique protocols (Marinade, Jito) |
| 4 | Base | Emerging yield opportunities |

### For Speed-Sensitive Operations

| Chain | Finality | Best For |
|-------|----------|----------|
| Solana | ~0.4s | Fastest execution, MEV-sensitive trades |
| Arbitrum | ~0.25s (soft), ~7 min (L1) | Fast with L1 security |
| Base | ~2s (soft), ~7 min (L1) | Fast with L1 security |
| BSC | ~3s | Fast, inexpensive |
| Ethereum | ~12s (block), ~15 min (finality) | Highest security, slowest |

---

## 7. API Endpoint Reference

### OKX DEX API Chain Parameters

When calling OKX DEX API endpoints, use `chainIndex` from the table above:

```
GET /api/v5/dex/aggregator/quote
  ?chainIndex=42161          # Arbitrum
  &fromTokenAddress=0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee  # Native ETH
  &toTokenAddress=0xaf88d065e77c8cc2239327c5edb3a432268e5831    # USDC
  &amount=1000000000000000000  # 1 ETH (18 decimals)
```

### Amount Encoding

Token amounts in API calls must be in the smallest unit (no decimals):

```
1 ETH    = 1000000000000000000  (18 decimals)
1 USDT   = 1000000              (6 decimals on most chains)
1 USDT   = 1000000000000000000  (18 decimals on BSC — WATCH OUT!)
1 BTC    = 100000000            (8 decimals for WBTC)
1 SOL    = 1000000000           (9 decimals)
```

**Critical:** BSC USDT has 18 decimals, unlike most other chains where it has 6 decimals. Always check the token's `decimals` value before encoding amounts.
