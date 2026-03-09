# OnchainOS CLI Tool Reference

OnchainOS is the OKX onchain toolkit CLI. All commands are invoked via the `onchainos` binary.

> **Contract addresses must be lowercase for EVM chains.**
> **Amounts are in minimal units** (wei for EVM, lamports for Solana), not UI/display units.

---

## Table of Contents

1. [Pre-flight Check](#pre-flight-check)
2. [dex-market price](#1-dex-market-price)
3. [dex-market prices](#2-dex-market-prices)
4. [dex-market kline](#3-dex-market-kline)
5. [dex-market signal-list](#4-dex-market-signal-list)
6. [dex-market signal-chains](#5-dex-market-signal-chains)
7. [dex-token search](#6-dex-token-search)
8. [dex-token info](#7-dex-token-info)
9. [dex-token price-info](#8-dex-token-price-info)
10. [dex-token holders](#9-dex-token-holders)
11. [dex-swap quote](#10-dex-swap-quote)
12. [dex-swap chains](#11-dex-swap-chains)
13. [wallet-portfolio all-balances](#12-wallet-portfolio-all-balances)
14. [onchain-gateway gas](#13-onchain-gateway-gas)
15. [onchain-gateway gas-limit](#14-onchain-gateway-gas-limit)
16. [onchain-gateway simulate](#15-onchain-gateway-simulate)
17. [Supported Chains](#supported-chains)
18. [Critical Notes](#critical-notes)

---

## Pre-flight Check

Before using any OnchainOS command, verify the CLI is installed and functional:

```bash
# Step 1: Check installation
which onchainos
# If not found: npx skills add okx/onchainos-skills

# Step 2: Verify connectivity with a simple price check
onchainos dex-market price \
  --chain ethereum \
  --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
# Should return current ETH price in USD
```

If either step fails, the OnchainOS CLI is not properly installed or configured. See `config/mcp-setup-guide.md` for full setup instructions.

---

## 1. dex-market price

**Purpose:** Get the current price of a single token on a specific chain.

### Usage

```bash
onchainos dex-market price --chain <chain> --token <address>
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| --chain | Yes | — | string | Chain name (e.g. `ethereum`, `solana`, `base`) |
| --token | Yes | — | string | Token contract address. Use native address for chain native tokens (see chain-reference.md). |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| price | string | Token price in native chain currency |
| priceUsd | string | Token price in USD |
| volume24h | string | 24-hour trading volume in USD |

### Example

```bash
onchainos dex-market price \
  --chain ethereum \
  --token 0xdac17f958d2ee523a2206206994597c13d831ec7
```

```json
{
  "price": "0.000288",
  "priceUsd": "1.0002",
  "volume24h": "52340000.00"
}
```

---

## 2. dex-market prices

**Purpose:** Get current prices for multiple tokens in a single call. More efficient than individual `price` calls.

### Usage

```bash
onchainos dex-market prices --chain <chain> --tokens <addr1,addr2,...>
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| --chain | Yes | — | string | Chain name |
| --tokens | Yes | — | string | Comma-separated list of token contract addresses |

### Return Fields

Returns an array of price objects, each containing:

| Field | Type | Description |
|-------|------|-------------|
| token | string | Token address |
| price | string | Price in native currency |
| priceUsd | string | Price in USD |
| volume24h | string | 24-hour volume in USD |

### Example

```bash
onchainos dex-market prices \
  --chain ethereum \
  --tokens 0xdac17f958d2ee523a2206206994597c13d831ec7,0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48
```

```json
[
  {
    "token": "0xdac17f958d2ee523a2206206994597c13d831ec7",
    "priceUsd": "1.0002",
    "volume24h": "52340000.00"
  },
  {
    "token": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
    "priceUsd": "0.9998",
    "volume24h": "48120000.00"
  }
]
```

---

## 3. dex-market kline

**Purpose:** Retrieve K-line (candlestick) chart data for a DEX-traded token.

### Usage

```bash
onchainos dex-market kline <address> --chain <chain> --bar <interval>
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| address | Yes | — | string | Token contract address (positional argument) |
| --chain | Yes | — | string | Chain name |
| --bar | No | `1m` | string | Candlestick interval. Enum: `1s`, `1m`, `5m`, `15m`, `30m`, `1H`, `4H`, `1D`, `1W` |

### Return Fields

Array of candles, each containing:

| Field | Type | Description |
|-------|------|-------------|
| ts | string | Candle open timestamp |
| open | string | Open price in USD |
| high | string | High price in USD |
| low | string | Low price in USD |
| close | string | Close price in USD |
| volume | string | Volume in USD |

### Example

```bash
onchainos dex-market kline \
  0xdac17f958d2ee523a2206206994597c13d831ec7 \
  --chain ethereum \
  --bar 1H
```

---

## 4. dex-market signal-list

**Purpose:** Retrieve smart money trading signals — tracks wallets classified as smart money, KOLs, or whales.

### Usage

```bash
onchainos dex-market signal-list --chain <chain> [--wallet-type 1,2,3]
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| --chain | Yes | — | string | Chain name |
| --wallet-type | No | all | string | Comma-separated wallet type filter: `1`=smart money, `2`=KOL, `3`=whale |

### Return Fields

Array of signal objects:

| Field | Type | Description |
|-------|------|-------------|
| walletAddress | string | Address of the tracked wallet |
| walletType | number | `1`=smart money, `2`=KOL, `3`=whale |
| tokenAddress | string | Token being traded |
| tokenSymbol | string | Token symbol |
| action | string | `"buy"` or `"sell"` |
| amountUsd | string | Trade value in USD |
| timestamp | string | Signal timestamp |
| txHash | string | On-chain transaction hash |

### Example

```bash
onchainos dex-market signal-list \
  --chain solana \
  --wallet-type 1,3
```

---

## 5. dex-market signal-chains

**Purpose:** List all chains that have smart money signal support enabled.

### Usage

```bash
onchainos dex-market signal-chains
```

### Parameters

_None._

### Return Fields

Array of chain names with signal support (e.g. `["ethereum", "solana", "base", "arbitrum", "bsc"]`).

---

## 6. dex-token search

**Purpose:** Search for tokens by name or symbol across one or more chains.

### Usage

```bash
onchainos dex-token search <query> [--chains <chains>]
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| query | Yes | — | string | Search term — token name or symbol (positional argument) |
| --chains | No | all | string | Comma-separated chain filter (e.g. `ethereum,solana`) |

### Return Fields

Array of matching tokens:

| Field | Type | Description |
|-------|------|-------------|
| address | string | Token contract address |
| symbol | string | Token symbol |
| name | string | Full token name |
| chain | string | Chain the token is on |
| decimals | number | Token decimal places |

### Example

```bash
onchainos dex-token search "PEPE" --chains ethereum,base
```

---

## 7. dex-token info

**Purpose:** Get detailed metadata for a specific token.

### Usage

```bash
onchainos dex-token info <address> --chain <chain>
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| address | Yes | — | string | Token contract address (positional argument) |
| --chain | Yes | — | string | Chain name |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| symbol | string | Token ticker symbol |
| name | string | Full token name |
| decimals | number | Token decimal places |
| totalSupply | string | Total token supply |
| holderCount | string | Number of unique holders |
| isRiskToken | boolean | `true` if flagged as potentially risky |
| createTime | string | Contract deployment timestamp |
| website | string | Project website URL (if available) |
| socialLinks | object | Links to Twitter, Telegram, Discord, etc. |

### Example

```bash
onchainos dex-token info \
  0x6982508145454ce325ddbe47a25d4ec3d2311933 \
  --chain ethereum
```

```json
{
  "symbol": "PEPE",
  "name": "Pepe",
  "decimals": 18,
  "totalSupply": "420690000000000",
  "holderCount": "250000",
  "isRiskToken": false
}
```

---

## 8. dex-token price-info

**Purpose:** Get price analytics including market cap, liquidity, and price change data for a DEX token.

### Usage

```bash
onchainos dex-token price-info <address> --chain <chain>
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| address | Yes | — | string | Token contract address (positional argument) |
| --chain | Yes | — | string | Chain name |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| marketCapUsd | string | Market capitalization in USD |
| liquidityUsd | string | Total DEX liquidity in USD |
| priceUsd | string | Current price in USD |
| priceChange24h | string | 24-hour price change percentage (e.g. `"5.2"` = +5.2%) |
| priceChange7d | string | 7-day price change percentage |
| volume24h | string | 24-hour volume in USD |
| volume7d | string | 7-day volume in USD |

### Example

```bash
onchainos dex-token price-info \
  0x6982508145454ce325ddbe47a25d4ec3d2311933 \
  --chain ethereum
```

```json
{
  "marketCapUsd": "4200000000",
  "liquidityUsd": "85000000",
  "priceUsd": "0.0000098",
  "priceChange24h": "5.2",
  "volume24h": "120000000",
  "volume7d": "750000000"
}
```

---

## 9. dex-token holders

**Purpose:** Get top holder distribution for a token — useful for concentration risk assessment.

### Usage

```bash
onchainos dex-token holders <address> --chain <chain>
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| address | Yes | — | string | Token contract address (positional argument) |
| --chain | Yes | — | string | Chain name |

### Return Fields

Array of top holders:

| Field | Type | Description |
|-------|------|-------------|
| address | string | Holder wallet address |
| balance | string | Token balance held |
| percent | string | Percentage of total supply (e.g. `"5.2"`) |
| isContract | boolean | `true` if the address is a contract (DEX pool, multisig, etc.) |

### Example

```bash
onchainos dex-token holders \
  0x6982508145454ce325ddbe47a25d4ec3d2311933 \
  --chain ethereum
```

---

## 10. dex-swap quote

**Purpose:** Get a swap quote without executing — returns expected output, price impact, and token safety flags. **Read-only, does not execute a trade.**

### Usage

```bash
onchainos dex-swap quote \
  --from <from_address> \
  --to <to_address> \
  --amount <amount_minimal_units> \
  --chain <chain>
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| --from | Yes | — | string | Source token contract address |
| --to | Yes | — | string | Destination token contract address |
| --amount | Yes | — | string | Amount to swap in **minimal units** (wei for EVM, lamports for Solana) |
| --chain | Yes | — | string | Chain name |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| toTokenAmount | string | Expected output amount in minimal units |
| priceImpactPercent | string | Estimated price impact percentage |
| fromToken.isHoneyPot | boolean | Whether the source token is flagged as a honeypot |
| fromToken.taxRate | string | Detected tax rate on the source token |
| toToken.isHoneyPot | boolean | Whether the destination token is flagged as a honeypot |
| toToken.taxRate | string | Detected tax rate on the destination token |
| estimatedGas | string | Estimated gas units for the swap |
| routerAddress | string | DEX router contract used |

### Example

```bash
# Quote: swap 1 ETH worth of wei to USDT on Ethereum
onchainos dex-swap quote \
  --from 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee \
  --to 0xdac17f958d2ee523a2206206994597c13d831ec7 \
  --amount 1000000000000000000 \
  --chain ethereum
```

```json
{
  "toTokenAmount": "3456780000",
  "priceImpactPercent": "0.02",
  "fromToken": {
    "isHoneyPot": false,
    "taxRate": "0"
  },
  "toToken": {
    "isHoneyPot": false,
    "taxRate": "0"
  },
  "estimatedGas": "250000"
}
```

---

## 11. dex-swap chains

**Purpose:** List all chains supported by the DEX swap aggregator.

### Usage

```bash
onchainos dex-swap chains
```

### Parameters

_None._

### Return Fields

Array of supported chain objects with name and chain ID.

---

## 12. wallet-portfolio all-balances

**Purpose:** Retrieve all token balances for a wallet address across one or more chains.

### Usage

```bash
onchainos wallet-portfolio all-balances <address> [--chains <chains>]
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| address | Yes | — | string | Wallet address to query (positional argument) |
| --chains | No | all | string | Comma-separated chain filter |

### Return Fields

Array of balance objects:

| Field | Type | Description |
|-------|------|-------------|
| chain | string | Chain name |
| tokenAddress | string | Token contract address |
| symbol | string | Token symbol |
| balance | string | Token balance in minimal units |
| balanceUsd | string | Balance value in USD |

### Example

```bash
onchainos wallet-portfolio all-balances \
  0x1234...abcd \
  --chains ethereum,arbitrum
```

---

## 13. onchain-gateway gas

**Purpose:** Get the current gas price for a specific chain.

### Usage

```bash
onchainos onchain-gateway gas --chain <chain>
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| --chain | Yes | — | string | Chain name |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| gasPrice | string | Current gas price in native minimal units (wei for EVM) |
| gasPriceGwei | string | Gas price in Gwei (EVM chains only) |
| baseFee | string | Base fee (EIP-1559 chains) |
| priorityFee | string | Suggested priority fee (EIP-1559 chains) |

### Example

```bash
onchainos onchain-gateway gas --chain ethereum
```

```json
{
  "gasPrice": "25000000000",
  "gasPriceGwei": "25",
  "baseFee": "22000000000",
  "priorityFee": "3000000000"
}
```

---

## 14. onchain-gateway gas-limit

**Purpose:** Estimate the gas limit (units) required for a specific transaction.

### Usage

```bash
onchainos onchain-gateway gas-limit \
  --chain <chain> \
  --from <sender> \
  --to <contract> \
  --data <calldata>
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| --chain | Yes | — | string | Chain name |
| --from | Yes | — | string | Sender address |
| --to | Yes | — | string | Destination contract address |
| --data | Yes | — | string | Hex-encoded transaction calldata |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| gasLimit | string | Estimated gas units required |

---

## 15. onchain-gateway simulate

**Purpose:** Simulate a transaction without broadcasting — useful for checking if a transaction would succeed and estimating gas.

### Usage

```bash
onchainos onchain-gateway simulate \
  --chain <chain> \
  --from <sender> \
  --to <contract> \
  --data <calldata> \
  --value <value>
```

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| --chain | Yes | — | string | Chain name |
| --from | Yes | — | string | Sender address |
| --to | Yes | — | string | Destination contract address |
| --data | Yes | — | string | Hex-encoded transaction calldata |
| --value | No | `"0"` | string | ETH/native value to send in minimal units |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| success | boolean | Whether the simulation succeeded |
| gasUsed | string | Gas units consumed in simulation |
| returnData | string | Hex-encoded return data from the call |
| error | string | Error message if simulation failed |

---

## Supported Chains

Primary chains used by the skills:

| Chain Name | Type | Notes |
|-----------|------|-------|
| ethereum | EVM | Mainnet (chain ID 1) |
| solana | Non-EVM | SPL tokens |
| base | EVM | Coinbase L2 (chain ID 8453) |
| arbitrum | EVM | Arbitrum One (chain ID 42161) |
| bsc | EVM | BNB Smart Chain (chain ID 56) |
| polygon | EVM | Polygon PoS (chain ID 137) |
| xlayer | EVM | OKX L2 |
| avalanche | EVM | C-Chain (chain ID 43114) |
| optimism | EVM | OP Mainnet (chain ID 10) |

Additional supported chains: `fantom`, `cronos`, `gnosis`, `celo`, `moonbeam`, `linea`, `scroll`, `zksync`, `mantle`, `blast`, `mode`, and more. Use `dex-swap chains` for the full list.

---

## Critical Notes

1. **Native token addresses** — Each chain uses a specific address for its native token. See `references/chain-reference.md` for the full list. EVM chains typically use `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee`.

2. **Lowercase addresses** — EVM contract addresses **must** be lowercase. Checksummed (mixed-case) addresses may cause lookup failures.

3. **Minimal units** — All `--amount` values are in the token's smallest unit:
   - EVM (18 decimals): 1 ETH = `1000000000000000000` wei
   - USDT (6 decimals): 1 USDT = `1000000` minimal units
   - Solana: 1 SOL = `1000000000` lamports

4. **Read-only** — `dex-swap quote` does **not** execute a trade. It returns a quote for analysis only. The skills never auto-execute onchain transactions.

5. **Rate limits** — OnchainOS CLI respects OKX DEX API rate limits internally. If you encounter throttling, add a 1-second delay between calls.

6. **Error handling** — On failure, commands return a non-zero exit code and a JSON error object with `errorCode` and `errorMessage` fields. Always check exit codes in automated workflows.
