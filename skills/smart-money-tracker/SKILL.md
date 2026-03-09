---
name: smart-money-tracker
description: >
  Triggers: "smart money", "聰明錢", "whale", "鯨魚", "大戶", "KOL",
  "follow", "跟單", "wallet tracking", "錢包追蹤", "who's buying",
  "邊個買緊"
allowed-tools:
  - OnchainOS (dex-market signal tools, wallet-portfolio, dex-token)
  - GoPlus MCP
  - DexScreener MCP (optional)
  - Rug Munch MCP (optional)
---

# Smart Money Tracker

Track onchain smart money (whales, funds, KOLs, MEV bots) activity. Detect
position changes and evaluate whether to follow.

**This skill enforces the MOST RESTRICTIVE safety rules of all skills.**

## Role

- Scan smart money signals across supported chains via OnchainOS.
- Evaluate signal quality using time-decay models and security checks.
- Track specific wallets for position changes over time.
- Generate copy-trade plans that ALWAYS require human approval.

### Does NOT

- Auto-execute copy trades -- **ALWAYS requires human approval**, no exceptions.
  This constraint cannot be overridden by configuration or user request.
- Provide financial advice -- outputs are analysis only.
- Guarantee profits -- past smart money performance does not predict future results.

---

## Pre-flight

Before executing any command, verify the following MCP servers are available.

| Server | Required | Purpose |
|--------|----------|---------|
| OnchainOS CLI | **Critical** | Primary data source -- smart money signals, token data, wallet balances |
| GoPlus MCP | **Critical** | Token security check on EVERY tracked token; wallet address screening |
| DexScreener MCP | Optional | Additional market context, token pair data |
| Rug Munch MCP | Optional | Deployer history analysis, serial rugger detection |
| Nansen MCP | Optional | Premium wallet labels and entity classification |

Run the OnchainOS pre-flight check:
```bash
onchainos dex-market signal-chains
```
This returns the list of chains with signal support. If it fails, OnchainOS
is not properly configured -- see `config/mcp-setup-guide.md`.

---

## Command Index

| Command | Purpose | Read/Write |
|---------|---------|------------|
| `scan` | Discover recent smart money activity across chains | Read |
| `track-wallet` | Monitor a specific wallet for position changes | Read |
| `evaluate` | Deep analysis of whether a specific signal is worth following | Read |
| `leaderboard` | Top performing wallets by realized PnL over a period | Read |
| `copy-plan` | Generate a trade plan based on a signal -- **REQUIRES HUMAN APPROVAL** | Read |

All commands are **read-only**. No trades are executed.

---

## Parameter Reference

### scan

Discover recent smart money activity.

```bash
smart-money-tracker scan --chains ethereum,solana,base --categories whale,fund --lookback-hours 24
```

| Parameter | Type | Required | Default | Enum | Validation |
|-----------|------|----------|---------|------|------------|
| `--chains` | string[] | No | `["ethereum","solana","base"]` | All OnchainOS-supported chains | Supported by `signal-chains` |
| `--categories` | string[] | No | all | `whale`, `fund`, `kol`, `mev`, `dex_trader` | Valid category name |
| `--lookback-hours` | integer | No | `24` | 1 - 168 | Positive integer |
| `--min-amount-usd` | number | No | `10000` | 0 - infinity | Positive number |
| `--action-filter` | string | No | all | `buy`, `sell`, `add_liquidity`, `remove_liquidity` | Valid action |
| `--top-n` | integer | No | `20` | 1 - 100 | Positive integer |

**Return Schema:**

```yaml
SmartMoneySignal:
  timestamp: integer              # Unix ms
  scan_params:
    chains: string[]
    categories: string[]
    lookback_hours: integer
  signals:
    - wallet_address: string
      wallet_type: string         # "whale" | "fund" | "kol" | "mev" | "dex_trader"
      wallet_label: string        # Known label if available, else truncated address
      chain: string
      action: string              # "buy" | "sell" | "add_liquidity" | "remove_liquidity"
      token_address: string
      token_symbol: string
      amount_usd: number
      tx_hash: string
      signal_timestamp: integer   # When the onchain action occurred (Unix ms)
      signal_age_min: integer     # (now - signal_timestamp) / 60000
      signal_strength: string     # "HIGH" | "MEDIUM" | "LOW" | "STALE"
      strength_score: number      # 0.0 - 1.0
      security_check:             # GoPlus result summary
        is_honeypot: boolean
        buy_tax_pct: number
        sell_tax_pct: number
        is_open_source: boolean
        holder_concentration_pct: number
        overall: string           # "SAFE" | "WARN" | "BLOCK"
      is_actionable: boolean      # true only if: strength != STALE AND security == SAFE
```

### track-wallet

Monitor a specific wallet address for position changes.

```bash
smart-money-tracker track-wallet --address 0x1234...5678 --chain ethereum --lookback-hours 72
```

| Parameter | Type | Required | Default | Enum | Validation |
|-----------|------|----------|---------|------|------------|
| `--address` | string | Yes | -- | Wallet address | Valid address format |
| `--chain` | string | No | auto-detect | Supported chain | OnchainOS-supported |
| `--lookback-hours` | integer | No | `72` | 1 - 720 | Positive integer |

**Pre-checks before tracking:**
1. GoPlus `check_address_security` -- BLOCK if malicious, phishing, or blacklisted.
2. OnchainOS `wallet-portfolio all-balances` -- retrieve current holdings.

**Return Schema:**

```yaml
WalletTrackResult:
  wallet_address: string
  wallet_label: string
  address_security: object        # GoPlus check_address_security result
  chain: string
  current_holdings:
    - token_symbol: string
      token_address: string
      balance_usd: number
      pct_of_portfolio: number
  recent_activity:
    - action: string
      token_symbol: string
      amount_usd: number
      timestamp: integer
      tx_hash: string
      signal_age_min: integer
  stats:
    total_portfolio_usd: number
    num_tokens: integer
    largest_position_pct: number
    activity_count_7d: integer
```

### evaluate

Deep analysis of whether a specific signal is worth following.

```bash
smart-money-tracker evaluate --signal-token 0xabcd...ef01 --signal-wallet 0x1234...5678 --chain ethereum
```

| Parameter | Type | Required | Default | Enum | Validation |
|-----------|------|----------|---------|------|------------|
| `--signal-token` | string | Yes | -- | Token address | Valid address |
| `--signal-wallet` | string | Yes | -- | Wallet address | Valid address |
| `--chain` | string | Yes | -- | Chain name | OnchainOS-supported |
| `--size-usd` | number | No | `500` | 1 - 5,000 | Below max_copy_size |

Returns full evaluation: token security, signal decay status, wallet
historical performance, liquidity analysis, and a FOLLOW / SKIP / BLOCK
recommendation.

### leaderboard

Top performing smart money wallets ranked by realized PnL.

```bash
smart-money-tracker leaderboard --chain solana --period 30d --category whale
```

| Parameter | Type | Required | Default | Enum | Validation |
|-----------|------|----------|---------|------|------------|
| `--chain` | string | No | all | Supported chain | OnchainOS-supported |
| `--period` | string | No | `30d` | `7d`, `30d`, `90d` | Valid period |
| `--category` | string | No | all | `whale`, `fund`, `kol`, `mev`, `dex_trader` | Valid category |
| `--top-n` | integer | No | `10` | 1 - 50 | Positive integer |

Returns ranked list of wallets with: PnL, win rate, number of trades,
average holding period, and best/worst trades.

### copy-plan

Generate a trade plan based on a signal. **This command ALWAYS outputs with
the `[REQUIRES HUMAN APPROVAL]` header.** No exception. No override.

```bash
smart-money-tracker copy-plan --signal-token 0xabcd...ef01 --chain ethereum --size-usd 500
```

| Parameter | Type | Required | Default | Enum | Validation |
|-----------|------|----------|---------|------|------------|
| `--signal-token` | string | Yes | -- | Token address | Valid address |
| `--chain` | string | Yes | -- | Chain name | OnchainOS-supported |
| `--size-usd` | number | No | `500` | 1 - 1,000 (default cap) | Below max_copy_size |
| `--slippage-pct` | number | No | `1.0` | 0.1 - 5.0 | Positive |

**Pre-conditions (ALL must pass):**
1. Signal age < 30 min (configurable via `signal_max_age_min`)
2. GoPlus token security = SAFE (no BLOCK flags)
3. Token liquidity > $100K TVL
4. Size <= `max_copy_size_usd` ($1,000 default)

Returns: entry route, estimated costs, risk assessment, and the
**mandatory** `[REQUIRES HUMAN APPROVAL]` block.

---

## Signal Decay Model

Smart money signals lose alpha over time as the market absorbs the
information. This skill applies a time-decay model to classify signal
freshness.

### Strength Tiers

| Tier | Age Range | Visual | Description |
|------|-----------|--------|-------------|
| **HIGH** | 0-5 min | `████████▓░` | Fresh signal, highest alpha potential |
| **MEDIUM** | 5-30 min | `█████▓░░░░` | Some alpha remaining, evaluate quickly |
| **LOW** | 30-120 min | `██▓░░░░░░░` | Most alpha likely captured by faster actors |
| **STALE** | 120+ min | `░░░░░░░░░░` | Do NOT recommend -- alpha gone |

### Decay Function (Inlined from formulas.md Section 7)

```
signal_strength = initial_strength * exp(-decay_rate * time_elapsed_minutes)

Variables:
  initial_strength       = Original signal strength (0.0 to 1.0)
  decay_rate             = Decay constant (higher = faster decay)
  time_elapsed_minutes   = Minutes since signal was generated

Typical Decay Rates (Half-Life Table):

  | Signal Type            | Decay Rate | Half-Life   |
  |------------------------|------------|-------------|
  | CEX-DEX arb spread     | 0.15       | ~4.6 min    |
  | Funding rate anomaly   | 0.01       | ~69 min     |
  | Basis mispricing       | 0.005      | ~139 min    |
  | Smart money signal     | 0.02       | ~35 min     |
  | Yield opportunity      | 0.001      | ~693 min    |

For this skill, the smart money signal type has a decay rate of 0.02 and half-life of ~35 min.

Worked Example:
  initial_strength = 0.90  (strong smart money signal)
  decay_rate = 0.02
  time_elapsed_minutes = 20

  signal_strength = 0.90 * exp(-0.02 * 20)
                  = 0.90 * exp(-0.40)
                  = 0.90 * 0.6703
                  = 0.603

  Signal has decayed from 0.90 to 0.60 in 20 minutes -- still MEDIUM tier.

Caveats:
  - Decay rates are empirical estimates; calibrate based on observed opportunity persistence.
  - Signal strength below 0.3 generally means the opportunity has likely been arbitraged away.
  - Combine signal decay with a freshness timestamp in output displays.
```

### Simplified Linear Decay (for tier classification)

```
strength_score = max(0, 1 - (age_min / 120))
```

### Signal Classification Logic

```
if age_min <= 5:
    strength = "HIGH"
    is_actionable = true (if security passes)
elif age_min <= 30:
    strength = "MEDIUM"
    is_actionable = true (if security passes)
elif age_min <= 120:
    strength = "LOW"
    is_actionable = false  # Do NOT recommend
else:
    strength = "STALE"
    is_actionable = false  # REJECT entirely
```

**Important:** Signals with `strength = LOW` are displayed for informational
purposes but are NOT included in recommendations. Signals with
`strength = STALE` are omitted from output entirely unless the user
explicitly requests historical data.

---

## Operation Flow

### Step 1: Parse Intent

Determine the target command from user input. Match trigger words:

| Trigger Pattern | Command |
|----------------|---------|
| "聰明錢買緊乜", "smart money", "whale activity" | `scan` |
| "追蹤呢個錢包", "track wallet", "monitor address" | `track-wallet` |
| "呢個信號值唔值得跟", "evaluate signal", "should I follow" | `evaluate` |
| "邊個錢包最叻", "leaderboard", "top wallets" | `leaderboard` |
| "幫我出跟單計劃", "copy plan", "follow trade" | `copy-plan` |

### Step 2: Fetch Smart Money Data

**For `scan`:**

1. `onchainos dex-market signal-chains`
   -> Get list of chains with signal support
   -> Filter by user-requested `--chains`

2. For each chain:
   ```bash
   onchainos dex-market signal-list --chain {chain} --wallet-type {type_ids}
   ```
   -> Returns smart money signals with wallet addresses and token interactions
   -> Map OnchainOS wallet types: `1`=smart money -> fund, `2`=KOL -> kol, `3`=whale -> whale

3. For each signal token:
   ```bash
   onchainos dex-token info {token_address} --chain {chain}
   ```
   -> Token metadata: symbol, name, isRiskToken, holderCount, createTime

4. For each signal token (liquidity check):
   ```bash
   onchainos dex-token price-info {token_address} --chain {chain}
   ```
   -> liquidityUsd, volume24h, priceChange24h

**For `track-wallet`:**

1. GoPlus `check_address_security` on the target wallet address
   -> BLOCK if `is_malicious_address === "1"` or any flag set

2. `onchainos wallet-portfolio all-balances {address} --chains {chain}`
   -> Current holdings

3. Cross-reference holdings with recent `signal-list` data for the wallet

### Step 3: Safety Checks (CRITICAL)

This is the most important step. Every signal passes through ALL checks
before being surfaced.

**For EVERY token in signals:**

1. **GoPlus `check_token_security`:**
   ```
   GoPlus check_token_security(chain_id, token_address)
   ```
   Apply the full decision matrix (see GoPlus Decision Matrix below).

2. **OnchainOS `isRiskToken` flag:**
   ```
   if dex-token info returns isRiskToken === true -> BLOCK
   ```

3. **Signal age calculation:**
   ```
   signal_age_min = (Date.now() - signal_timestamp) / 60000
   ```
   Apply decay model (see Signal Decay Model above).
   - age > 120 min -> REJECT (omit from output)
   - age > 30 min -> Mark as LOW, do NOT recommend

4. **Liquidity check:**
   ```
   if dex-token price-info liquidityUsd < 100000 -> BLOCK
   ```

5. **Optional -- Rug Munch deployer analysis:**
   If Rug Munch MCP is available, check deployer address:
   - Serial rugger history -> WARN (display to user)
   - Known rug pattern -> BLOCK

6. **Optional -- GoPlus `check_address_security` on signal wallet:**
   For `track-wallet` and `evaluate`, screen the wallet itself:
   - Flagged as malicious/phishing -> BLOCK

### Step 4: Output

Format using Smart Money Signal Template (see Output Templates below).

**MANDATORY:** Every output includes the disclaimer block (see Mandatory
Disclaimers below).

---

## GoPlus Decision Matrix (Inlined -- FULL)

For every token encountered in smart money signals, run GoPlus `check_token_security`.

### check_token_security

**Purpose:** Comprehensive token security audit -- detects honeypots, hidden taxes, mintable supplies, ownership risks, and holder concentration. **This is the primary safety gate for all onchain token recommendations.**

**Parameters:**

| Param | Required | Type | Description |
|-------|----------|------|-------------|
| chain_id | Yes | string | Blockchain identifier (see Chain ID Reference below) |
| contract_address | Yes | string | Token contract address to audit |

**Key Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| is_honeypot | string | `"0"` = safe, `"1"` = honeypot (cannot sell). **BLOCK if "1".** |
| buy_tax | string | Buy tax rate as decimal (e.g. `"0.05"` = 5%) |
| sell_tax | string | Sell tax rate as decimal (e.g. `"0.10"` = 10%) |
| is_mintable | string | `"0"` = fixed supply, `"1"` = owner can mint new tokens |
| can_take_back_ownership | string | `"0"` = safe, `"1"` = renounced ownership can be reclaimed |
| owner_change_balance | string | `"0"` = safe, `"1"` = owner can modify token balances |
| is_open_source | string | `"0"` = unverified source, `"1"` = verified/open source |
| is_proxy | string | `"0"` = not upgradeable, `"1"` = proxy/upgradeable contract |
| holder_count | string | Total number of unique token holders |
| holders | array | Top holders list (each: address, balance, percent, is_contract) |
| slippage_modifiable | string | `"0"` = fixed, `"1"` = slippage/tax can be changed by owner |
| transfer_pausable | string | `"0"` = always transferable, `"1"` = transfers can be paused |
| cannot_sell_all | string | `"0"` = can sell all, `"1"` = cannot sell entire balance |
| personal_slippage_modifiable | string | `"0"` = no, `"1"` = can set different tax per address |

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
| top 10 holders % | > 80% combined (non-contract) | > 50% combined (non-contract) |
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

### check_address_security

**Purpose:** Check whether a wallet or contract address is associated with malicious activity, phishing, or blacklists.

**Parameters:**

| Param | Required | Type | Description |
|-------|----------|------|-------------|
| address | Yes | string | Wallet or contract address to check |

**Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| is_malicious_address | string | `"0"` = clean, `"1"` = flagged as malicious |
| phishing_activities | string | `"0"` = no phishing detected, `"1"` = phishing activity found |
| blacklist_doubt | string | `"0"` = not on any doubt lists, `"1"` = on doubt list |
| honeypot_related_address | string | `"0"` = clean, `"1"` = linked to honeypot contracts |
| blackmail_activities | string | `"0"` = none, `"1"` = blackmail activity detected |
| stealing_attack | string | `"0"` = none, `"1"` = associated with theft |
| fake_kyc | string | `"0"` = none, `"1"` = fake KYC detected |
| contract_address | string | `"0"` = EOA, `"1"` = is a contract |

**BLOCK if ANY of these fields returns `"1"`: is_malicious_address, phishing_activities, honeypot_related_address, blackmail_activities, stealing_attack.**

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
| Fantom | `"250"` |
| Linea | `"59144"` |
| Scroll | `"534352"` |
| zkSync Era | `"324"` |
| Solana | `"solana"` |

### GoPlus Integration Notes

- **Always call before recommending** -- every token MUST have a `check_token_security` call before being output. No exceptions.
- **Cache results** for 5 minutes within a session to avoid redundant API calls.
- **Solana tokens** -- use `chain_id: "solana"` (string, not numeric).
- **GoPlus API response wrapper** -- security data is nested inside `result.<contract_address>`. Always extract using the contract address as key.
- **If GoPlus is unreachable**, surface a `[WARN]`: "Security check unavailable -- proceed with extreme caution." Never silently skip.

---

## OnchainOS Tool Specifications (Inlined)

### dex-market signal-list

**Purpose:** Retrieve smart money trading signals -- tracks wallets classified as smart money, KOLs, or whales.

**Usage:**
```bash
onchainos dex-market signal-list --chain <chain> [--wallet-type 1,2,3]
```

**Parameters:**

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| --chain | Yes | -- | string | Chain name |
| --wallet-type | No | all | string | Comma-separated wallet type filter: `1`=smart money, `2`=KOL, `3`=whale |

**Return Fields:**

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

### dex-market signal-chains

**Purpose:** List all chains that have smart money signal support enabled.

**Usage:**
```bash
onchainos dex-market signal-chains
```

**Parameters:** None.

**Return:** Array of chain names with signal support (e.g. `["ethereum", "solana", "base", "arbitrum", "bsc"]`).

### dex-token info

**Purpose:** Get detailed metadata for a specific token.

**Usage:**
```bash
onchainos dex-token info <address> --chain <chain>
```

**Parameters:**

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| address | Yes | -- | string | Token contract address (positional argument) |
| --chain | Yes | -- | string | Chain name |

**Return Fields:**

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

### dex-token price-info

**Purpose:** Get price analytics including market cap, liquidity, and price change data for a DEX token.

**Usage:**
```bash
onchainos dex-token price-info <address> --chain <chain>
```

**Parameters:**

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| address | Yes | -- | string | Token contract address (positional argument) |
| --chain | Yes | -- | string | Chain name |

**Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| marketCapUsd | string | Market capitalization in USD |
| liquidityUsd | string | Total DEX liquidity in USD |
| priceUsd | string | Current price in USD |
| priceChange24h | string | 24-hour price change percentage (e.g. `"5.2"` = +5.2%) |
| priceChange7d | string | 7-day price change percentage |
| volume24h | string | 24-hour volume in USD |
| volume7d | string | 7-day volume in USD |

### wallet-portfolio all-balances

**Purpose:** Retrieve all token balances for a wallet address across one or more chains.

**Usage:**
```bash
onchainos wallet-portfolio all-balances <address> [--chains <chains>]
```

**Parameters:**

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| address | Yes | -- | string | Wallet address to query (positional argument) |
| --chains | No | all | string | Comma-separated chain filter |

**Return Fields:**

| Field | Type | Description |
|-------|------|-------------|
| chain | string | Chain name |
| tokenAddress | string | Token contract address |
| symbol | string | Token symbol |
| balance | string | Token balance in minimal units |
| balanceUsd | string | Balance value in USD |

### OnchainOS Critical Notes

- **Contract addresses must be lowercase** for EVM chains.
- **Amounts are in minimal units** (wei for EVM, lamports for Solana), not UI/display units.
- `dex-swap quote` does **not** execute a trade. Read-only.
- Rate limits are handled internally by OnchainOS CLI. If throttled, add a 1-second delay between calls.
- On failure, commands return a non-zero exit code and a JSON error object.

---

## Gas Benchmarks for Copy-Plan Cost Estimates (Inlined)

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

### Gas Cost Formulas

**EVM Chains:**
```
gas_cost_usd = gas_price_gwei * gas_limit * native_token_price / 1e9
```

**Solana:**
```
gas_cost_usd = (base_fee_lamports + priority_fee_lamports) * sol_price / 1e9
```

Typical Solana fees:
- Base fee: 5,000 lamports (0.000005 SOL)
- Priority fee: 10,000-500,000 lamports depending on congestion

### Gas Notes

- **Ethereum L1** is the most expensive; for $500 copy trades, gas could be 1-6% of the trade.
- **L2s and Solana** are much more practical for small copy trades.
- Always include gas in the copy-plan cost estimate.
- Gas prices fluctuate significantly; always fetch real-time estimates when available.

---

## Safety Checks -- Summary Table (Inlined)

This skill applies the **most restrictive** safety checks in the project.

| # | Check | Source | Threshold | Action |
|---|-------|--------|-----------|--------|
| 1 | Human approval | Skill policy | ALWAYS for `copy-plan` | **BLOCK** until user confirms -- non-negotiable |
| 2 | Token honeypot | GoPlus `check_token_security` | `is_honeypot === "1"` | **BLOCK** -- `SECURITY_BLOCKED` |
| 3 | Token tax rate | GoPlus `check_token_security` | buy > 5% OR sell > 10% | **BLOCK** -- `SECURITY_BLOCKED` |
| 4 | Contract verification | GoPlus `check_token_security` | `is_open_source === "0"` | **BLOCK** -- `SECURITY_BLOCKED` |
| 5 | Owner can change balance | GoPlus `check_token_security` | `owner_change_balance === "1"` | **BLOCK** -- `SECURITY_BLOCKED` |
| 6 | Slippage modifiable | GoPlus `check_token_security` | `slippage_modifiable === "1"` | **BLOCK** -- `SECURITY_BLOCKED` |
| 7 | Holder concentration | GoPlus `check_token_security` | Top 10 non-contract > 80% | **BLOCK** -- `SECURITY_BLOCKED` |
| 8 | OnchainOS risk flag | OnchainOS `dex-token info` | `isRiskToken === true` | **BLOCK** -- `SECURITY_BLOCKED` |
| 9 | Signal age | Decay model | > 30 min | **BLOCK** -- do not recommend |
| 10 | Max copy size | Config `max_copy_size_usd` | > $1,000 (default) | **BLOCK** unless user overrides (hard cap $5,000) |
| 11 | Token liquidity | OnchainOS `dex-token price-info` | `liquidityUsd < $100,000` | **BLOCK** -- `INSUFFICIENT_LIQUIDITY` |
| 12 | Deployer history | Rug Munch MCP (optional) | Serial rugger pattern | **WARN** -- display to user |
| 13 | Wallet address security | GoPlus `check_address_security` | Any malicious flag | **BLOCK** -- do not track/follow |
| 14 | Holder concentration (soft) | GoPlus `check_token_security` | Top 10 non-contract > 50% | **WARN** -- display warning |
| 15 | Proxy contract | GoPlus `check_token_security` | `is_proxy === "1"` | **WARN** -- upgradeable contract |

---

## Mandatory Disclaimers

Every single output from this skill MUST include the following disclaimer
block. This is non-negotiable and cannot be suppressed.

```
[CAUTION: 跟隨聰明錢有風險，過往表現不代表未來結果。]
[所有跟單建議均需人工確認後方可執行。]
```

For `copy-plan` output, the additional header is required:

```
══════════════════════════════════════════
  [REQUIRES HUMAN APPROVAL -- 需人工確認]
══════════════════════════════════════════
```

---

## Risk Limits (Inlined)

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| `max_copy_size_usd` | $1,000 | $5,000 | Maximum USD per copied trade |
| `require_human_approval` | **ALWAYS** | **ALWAYS** | Cannot be overridden |
| `signal_max_age_min` | 30 | 60 | Maximum signal age in minutes |
| `min_wallet_pnl_usd` | $100,000 | -- | Only follow wallets with > $100K realized PnL |
| `min_wallet_win_rate` | 60% | -- | Minimum tracked wallet win rate |
| `blocked_wallet_types` | -- | -- | Wallets flagged by GoPlus `check_address_security` |

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

For smart-money-tracker: Skill icon = Eye, Data sources = "OnchainOS + GoPlus"

### Formatting Rules

- Monetary: `+$1,234.56` or `-$1,234.56` (2 decimals, comma thousands)
- Percentages: `12.5%` (1 decimal)
- Risk Levels: `[SAFE]`, `[WARN]`, `[BLOCK]`
- Risk Gauge: `▓▓▓░░░░░░░ 3/10` (scale 1-10)
  - 1-2: LOW RISK, 3-4: MODERATE-LOW, 5-6: MODERATE, 7-8: ELEVATED, 9-10: HIGH RISK
- Sparklines: `▁▂▃▄▅▆▇█` for trend visualization
- Timestamps: `YYYY-MM-DD HH:MM UTC`

### Smart Money Signal Template

```
── Smart Money Signals ─────────────────────

  Signal #{NUM}: {SIGNAL_TYPE}
  ┌─ Wallet:    {WALLET_LABEL} ({WALLET_ADDR})
  ├─ Action:    {ACTION}
  ├─ Asset:     {ASSET} on {CHAIN}
  ├─ Size:      {SIZE}
  ├─ Tx:        {TX_HASH}
  ├─ Time:      {TX_TIME} ({AGE} ago)
  ├─ Strength:  {SIGNAL_STRENGTH}  (decay: {DECAY_NOTE})
  └─ Context:   {CONTEXT}

  ── Aggregated View ───────────────────────
  {ASSET}: {FLOW_DIRECTION}
  ├─ Net flow (24h):  {NET_FLOW}
  ├─ Whale txns:      {WHALE_COUNT} ({BUY_COUNT} buys / {SELL_COUNT} sells)
  ├─ Avg size:        {AVG_SIZE}
  └─ Trend:           {SPARKLINE} ({TREND_LABEL})
```

### Safety Check Results Template

```
── Safety Checks ───────────────────────────

  [  {CHECK1_STATUS}  ] {CHECK1_NAME}
                        {CHECK1_DETAIL}

  [  {CHECK2_STATUS}  ] {CHECK2_NAME}
                        {CHECK2_DETAIL}

  ──────────────────────────────────────
  Overall: {OVERALL_STATUS}
  {OVERALL_MESSAGE}
```

### Risk Gauge Template

```
── Risk Assessment ─────────────────────────

  Overall Risk:  {RISK_GAUGE}  {RISK_SCORE}/10
                 {RISK_LABEL}

  Breakdown:
  ├─ Execution Risk:   {EXEC_GAUGE}  {EXEC_SCORE}/10
  │                    {EXEC_NOTE}
  ├─ Market Risk:      {MKT_GAUGE}   {MKT_SCORE}/10
  │                    {MKT_NOTE}
  ├─ Smart Contract:   {SC_GAUGE}    {SC_SCORE}/10
  │                    {SC_NOTE}
  └─ Liquidity Risk:   {LIQ_GAUGE}   {LIQ_SCORE}/10
                       {LIQ_NOTE}
```

Gauge format: `1/10: ▓░░░░░░░░░`, `5/10: ▓▓▓▓▓░░░░░`, `10/10: ▓▓▓▓▓▓▓▓▓▓`

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
  Past performance does not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

### Scan Output (Full Example)

```
══════════════════════════════════════════
  Smart Money Tracker
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 12:00 UTC
  Data sources: OnchainOS + GoPlus
══════════════════════════════════════════

## 聰明錢活動掃描

掃描時間: 2026-03-09 12:00:00 UTC
鏈: Ethereum, Solana, Base | 回溯: 24h

### 信號列表

| # | 類型       | 鏈   | 動作 | 代幣    | 金額     | 信號強度              | 安全       |
|---|-----------|------|------|--------|---------|----------------------|-----------|
| 1 | Whale     | ETH  | 買入 | PEPE   | $250K   | [HIGH] ████████▓░    | [SAFE]    |
| 2 | Fund      | SOL  | 買入 | JUP    | $180K   | [MEDIUM] █████▓░░░░  | [SAFE]    |
| 3 | KOL       | Base | 買入 | UNKNOWN| $50K    | [LOW] ██▓░░░░░░░     | [BLOCK]   |

── 信號 #1 詳細 ──────────────────────────

錢包: 0x1234...5678 (Whale, 歷史勝率 62%)
代幣: PEPE (0x6982...cafe)
動作: 買入 $250,000 @ $0.00001234
時間: 3 分鐘前 -> [HIGH] 信號強度

── 安全檢查 ───────────────────────────

| 檢查項目       | 結果              | 來源      |
|---------------|-------------------|----------|
| Honeypot      | [SAFE]            | GoPlus   |
| Tax Rate      | [SAFE] 0% / 0%   | GoPlus   |
| 合約驗證       | [SAFE] Open Source| GoPlus   |
| 持有者集中度    | [WARN] Top 10 = 55%| GoPlus |
| 流動性         | [SAFE] $12.5M     | OnchainOS|
| isRiskToken   | [SAFE] false      | OnchainOS|

── 信號 #3 已攔截 ────────────────────────

代幣 UNKNOWN (0xdead...beef) 已被安全檢查攔截:
  [BLOCK] 合約未驗證 (is_open_source === "0")
  [BLOCK] 流動性不足 ($45K < $100K 最低要求)
  此信號不會出現在推薦中。

[CAUTION: 跟隨聰明錢有風險，過往表現不代表未來結果。]
[所有跟單建議均需人工確認後方可執行。]

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 詳細評估 PEPE 信號: smart-money-tracker evaluate --signal-token 0x6982...cafe --signal-wallet 0x1234...5678 --chain ethereum
  2. 查看這個錢包的歷史表現: smart-money-tracker track-wallet --address 0x1234...5678 --chain ethereum
  3. 生成跟單計劃: smart-money-tracker copy-plan --signal-token 0x6982...cafe --chain ethereum --size-usd 500

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Past performance does not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

### Copy-Plan Output

```
══════════════════════════════════════════
  Smart Money Tracker -- Copy Plan
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
══════════════════════════════════════════
  [REQUIRES HUMAN APPROVAL -- 需人工確認]
══════════════════════════════════════════

## 跟單計劃: PEPE on Ethereum

基於信號: Whale 0x1234...5678 買入 $250K PEPE (3 分鐘前)

── 計劃詳情 ──────────────────────────────

| 項目          | 值                                  |
|--------------|-------------------------------------|
| 代幣          | PEPE (0x6982...cafe)                |
| 鏈            | Ethereum                            |
| 動作          | 買入                                 |
| 計劃金額       | $500.00                             |
| 當前價格       | $0.00001234                         |
| 預計數量       | ~40,518,638 PEPE                    |
| 滑點設定       | 1.0%                                |
| 預計 Gas      | ~$8.50 (Ethereum, 25 gwei)          |
| 預計總成本     | ~$508.50                            |

── 風險評估 ──────────────────────────────

  Overall Risk:  ▓▓▓▓▓░░░░░  5/10
                 MODERATE

  ├─ Token Security:   ▓▓▓░░░░░░░  3/10
  │                    GoPlus verified, no honeypot/tax
  ├─ Signal Strength:  ▓▓░░░░░░░░  2/10
  │                    [HIGH] -- 3 min old
  ├─ Liquidity Risk:   ▓▓░░░░░░░░  2/10
  │                    $12.5M liquidity, $500 trade = negligible impact
  ├─ Holder Risk:      ▓▓▓▓▓░░░░░  5/10
  │                    Top 10 holders = 55% (above 50% WARN threshold)
  └─ Wallet Track Record: ▓▓▓▓░░░░░░  4/10
                       62% win rate, $180K realized PnL (30d)

── 安全檢查 (全部通過) ────────────────────

  [  SAFE  ] Honeypot Check -- not a honeypot
  [  SAFE  ] Tax Rate -- 0% buy / 0% sell
  [  SAFE  ] Contract Verified -- open source
  [  WARN  ] Holder Concentration -- Top 10 = 55%
  [  SAFE  ] Liquidity -- $12.5M TVL
  [  SAFE  ] Signal Freshness -- 3 min (HIGH)

  Overall: SAFE (with advisory on holder concentration)

══════════════════════════════════════════
  此計劃需要您的明確確認才能執行。
  輸入 "確認" 或 "confirm" 繼續。
  輸入 "取消" 或 "cancel" 放棄。
══════════════════════════════════════════

[CAUTION: 跟隨聰明錢有風險，過往表現不代表未來結果。]
[所有跟單建議均需人工確認後方可執行。]
```

### Leaderboard Output

```
══════════════════════════════════════════
  Smart Money Tracker -- Leaderboard
  [DEMO] [RECOMMENDATION ONLY -- 不會自動執行]
══════════════════════════════════════════
  Period: 30d | Chain: Solana | Category: All

| #  | 錢包              | 類型   | 實現 PnL     | 勝率  | 交易數 | 平均持有期 |
|----|------------------|--------|-------------|-------|--------|----------|
| 1  | 5Kj2...mN9p      | Whale  | +$542,300   | 71%   | 45     | 4.2h     |
| 2  | 8Xp4...qR2s      | Fund   | +$318,900   | 68%   | 32     | 12.5h    |
| 3  | 3Wm7...vT5k      | KOL    | +$245,100   | 65%   | 78     | 2.1h     |
| 4  | 9Yn1...bK8j      | Whale  | +$198,400   | 58%   | 23     | 8.7h     |
| 5  | 2Hf6...cL3w      | Fund   | +$156,700   | 72%   | 18     | 24.0h    |

[CAUTION: 跟隨聰明錢有風險，過往表現不代表未來結果。]
[所有跟單建議均需人工確認後方可執行。]
```

---

## OnchainOS Wallet Type Mapping

OnchainOS uses numeric wallet types. This skill maps them to readable
categories:

| OnchainOS `walletType` | Skill Category | Description |
|------------------------|---------------|-------------|
| `1` | `fund` / `smart_money` | Institutional / fund wallets |
| `2` | `kol` | Known KOL / influencer wallets |
| `3` | `whale` | High-value individual wallets |

Additional categories (`mev`, `dex_trader`) may be inferred from behavior
patterns when OnchainOS data is enriched with Nansen labels.

---

## Conversation Examples

### Example 1: "聰明錢而家買緊乜？"

**Intent:** Scan for current smart money buying activity.
**Command:** `scan --action-filter buy --lookback-hours 24`
**Flow:**
1. `onchainos dex-market signal-chains` -> get supported chains
2. `onchainos dex-market signal-list --chain {chain}` for each chain
3. Filter to buy signals only
4. For each token: GoPlus `check_token_security` + OnchainOS `dex-token info`
5. Calculate signal age and apply decay model
6. Rank by signal strength, filter out STALE/BLOCK
7. Output signal table with safety checks
**Output:** Signal list with safety checks and decay indicators.

### Example 2: "追蹤呢個錢包 0x1234..."

**Intent:** Monitor a specific wallet.
**Command:** `track-wallet --address 0x1234... --chain ethereum`
**Flow:**
1. GoPlus `check_address_security` on the wallet -- BLOCK if flagged
2. `onchainos wallet-portfolio all-balances 0x1234...` -> current holdings
3. Cross-reference with `signal-list` for recent activity from this wallet
4. Compute portfolio breakdown and activity timeline
**Output:** Wallet profile with holdings, recent activity, and security status.

### Example 3: "呢個信號值唔值得跟？"

**Intent:** Evaluate a specific signal.
**Command:** `evaluate --signal-token 0xabcd... --signal-wallet 0x1234... --chain ethereum`
**Flow:**
1. Full GoPlus security check on the token
2. Signal age and decay calculation
3. Wallet historical performance lookup (leaderboard data)
4. Liquidity depth analysis via `dex-token price-info`
5. Optional: Rug Munch deployer check
6. Compile risk assessment and output FOLLOW / SKIP / BLOCK recommendation
**Output:** Detailed evaluation with risk breakdown and clear recommendation.

### Example 4: "幫我出一個跟單計劃"

**Intent:** Generate a copy-trade plan.
**Command:** `copy-plan --signal-token 0xabcd... --chain ethereum --size-usd 500`
**Flow:**
1. Verify all pre-conditions (signal age < 30 min, security SAFE, liquidity OK)
2. Get current token price and calculate expected quantity
3. Estimate gas costs for the chain
4. Compile risk assessment
5. Output plan with `[REQUIRES HUMAN APPROVAL]` header
6. **WAIT for explicit user confirmation** -- do NOT proceed without it
**Output:** Copy plan with full cost breakdown, risk assessment, and
mandatory human approval prompt.

---

## Error Handling

| Error Code | Trigger | User Message (ZH) | User Message (EN) | Recovery |
|------------|---------|-------------------|-------------------|----------|
| `MCP_NOT_CONNECTED` | OnchainOS CLI unreachable | OnchainOS 無法連線，無法獲取聰明錢數據 | OnchainOS unreachable. Cannot fetch smart money data. | Check OnchainOS installation |
| `SECURITY_BLOCKED` | GoPlus flags token | 安全檢查未通過：{reason}。此代幣存在風險，已從結果中移除。 | Security check failed: {reason}. Token flagged as risky. | Do not proceed. Show specific findings. |
| `SECURITY_UNAVAILABLE` | GoPlus API unreachable | 安全檢查服務暫時無法使用，請謹慎操作 | Security check service unavailable. Proceed with extreme caution. | Retry once, then WARN and continue |
| `SIGNAL_TOO_OLD` | Signal age > 30 min | 信號已超過 {age} 分鐘，可能已失效。不建議跟單。 | Signal is {age} minutes old. May no longer be actionable. | Show current token price vs signal price |
| `INSUFFICIENT_LIQUIDITY` | Token liquidity < $100K | 代幣流動性不足 (${liquidity})，不建議交易 | Token liquidity insufficient (${liquidity}). Not recommended. | Show liquidity details |
| `TRADE_SIZE_EXCEEDED` | Copy size > max | 跟單金額 ${amount} 超過上限 ${limit} | Copy size ${amount} exceeds limit ${limit} | Cap at limit |
| `HUMAN_APPROVAL_REQUIRED` | copy-plan needs confirmation | 此操作需要您的確認才能繼續 | This action requires your explicit approval to proceed. | Wait for user confirmation |
| `ADDRESS_FLAGGED` | GoPlus flags wallet address | 此錢包地址被標記為可疑：{reason}。不建議追蹤。 | This wallet address is flagged as suspicious: {reason}. | Do not track/follow |
| `DATA_STALE` | Market data too old | 市場數據已過期，正在重新獲取... | Market data stale. Refetching... | Auto-retry once, then error |

When OnchainOS is unreachable, this skill cannot function -- there is no
fallback data source for smart money signals. Surface the error clearly
and suggest the user check OnchainOS installation.

---

## Cross-Reference

| Reference | File | Sections Used |
|-----------|------|---------------|
| Formulas | `references/formulas.md` | Section 7 (Signal Decay Function) |
| Safety Checks | `references/safety-checks.md` | Full checklist, smart-money-specific thresholds |
| Output Templates | `references/output-templates.md` | Section 8 (Smart Money Signal), Section 4 (Safety Checks), Section 5 (Risk Gauge) |
| GoPlus Tools | `references/goplus-tools.md` | `check_token_security`, `check_address_security`, Decision Matrix |
| OnchainOS Tools | `references/onchainos-tools.md` | `dex-market signal-list`, `signal-chains`, `dex-token info`, `dex-token price-info`, `wallet-portfolio all-balances` |
| Risk Limits | `config/risk-limits.example.yaml` | `smart_money_tracker` section |
| Agent Orchestration | `AGENTS.md` | Chain E (Smart Money Following) |
| Fee Schedule | `references/fee-schedule.md` | Section 3 (Gas Benchmarks) -- for copy-plan cost estimates |

> **Note:** All content from the above reference files has been inlined in this document for OpenClaw/Lark compatibility. The LLM does not need to read any external files.
