# GoPlus MCP Tool Reference

GoPlus Security provides on-chain security detection via MCP. Used for token safety checks, approval auditing, and address/phishing screening before any onchain recommendation.

> **API:** Free tier available. Get an API key at https://gopluslabs.io/security-api
> Configure via the `GOPLUS_API_KEY` environment variable.

---

## Table of Contents

1. [check_token_security](#1-check_token_security)
2. [check_approval_security](#2-check_approval_security)
3. [check_address_security](#3-check_address_security)
4. [check_phishing_site](#4-check_phishing_site)
5. [Chain ID Reference](#chain-id-reference)
6. [Decision Matrix](#decision-matrix)
7. [Integration Notes](#integration-notes)

---

## 1. check_token_security

**Purpose:** Comprehensive token security audit — detects honeypots, hidden taxes, mintable supplies, ownership risks, and holder concentration. **This is the primary safety gate for all onchain token recommendations.**

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| chain_id | Yes | — | string | Blockchain identifier (see [Chain ID Reference](#chain-id-reference)) |
| contract_address | Yes | — | string | Token contract address to audit |

### Return Fields (Key Fields)

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
| total_supply | string | Total token supply |
| holders | array | Top holders list (see below) |
| is_in_dex | string | `"0"` = no DEX liquidity, `"1"` = traded on DEX |
| is_blacklisted | string | `"0"` = no blacklist, `"1"` = contract has blacklist function |
| is_whitelisted | string | `"0"` = no whitelist, `"1"` = contract has whitelist function |
| slippage_modifiable | string | `"0"` = fixed, `"1"` = slippage/tax can be changed by owner |
| transfer_pausable | string | `"0"` = always transferable, `"1"` = transfers can be paused |
| trading_cooldown | string | `"0"` = no cooldown, `"1"` = cooldown between trades enforced |
| personal_slippage_modifiable | string | `"0"` = no, `"1"` = can set different tax per address |
| cannot_sell_all | string | `"0"` = can sell all, `"1"` = cannot sell entire balance |
| owner_address | string | Current contract owner address |
| creator_address | string | Contract deployer address |

### Holders Array

Each element in the `holders` array:

| Field | Type | Description |
|-------|------|-------------|
| address | string | Holder wallet address |
| balance | string | Token balance held |
| percent | string | Percentage of total supply (e.g. `"0.152"` = 15.2%) |
| is_contract | number | `0` = EOA wallet, `1` = smart contract (DEX pool, multisig, etc.) |

### Example

**Call:**
```json
{
  "tool": "check_token_security",
  "params": {
    "chain_id": "1",
    "contract_address": "0x6982508145454ce325ddbe47a25d4ec3d2311933"
  }
}
```

**Response (abbreviated):**
```json
{
  "is_honeypot": "0",
  "buy_tax": "0",
  "sell_tax": "0",
  "is_mintable": "0",
  "can_take_back_ownership": "0",
  "owner_change_balance": "0",
  "is_open_source": "1",
  "is_proxy": "0",
  "holder_count": "250000",
  "total_supply": "420690000000000000000000000000000",
  "holders": [
    {
      "address": "0xdead000000000000000000000000000000000000",
      "balance": "184920000000000000000000000000000",
      "percent": "0.439",
      "is_contract": 0
    },
    {
      "address": "0x1234...5678",
      "balance": "8400000000000000000000000000000",
      "percent": "0.020",
      "is_contract": 1
    }
  ]
}
```

---

## 2. check_approval_security

**Purpose:** Analyze token approval risks — checks if a contract has risky approval patterns that could drain user funds.

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| chain_id | Yes | — | string | Blockchain identifier |
| contract_address | Yes | — | string | Token or contract address to check |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| is_open_source | string | Whether the approved contract is open source |
| is_proxy | string | Whether the approved contract is upgradeable |
| malicious_behavior | array | List of detected malicious behaviors |
| tag | string | Risk classification tag |
| is_contract | string | Whether the address is a contract |
| doubt_list | array | Addresses flagged with doubts |

### Example

**Call:**
```json
{
  "tool": "check_approval_security",
  "params": {
    "chain_id": "1",
    "contract_address": "0x68b3465833fb72a70ecdf485e0e4c7bd8665fc45"
  }
}
```

---

## 3. check_address_security

**Purpose:** Check whether a wallet or contract address is associated with malicious activity, phishing, or blacklists.

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| address | Yes | — | string | Wallet or contract address to check |

### Return Fields

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
| data_source | string | Source of the security data |

### Example

**Call:**
```json
{
  "tool": "check_address_security",
  "params": {
    "address": "0x1234567890abcdef1234567890abcdef12345678"
  }
}
```

**Response:**
```json
{
  "is_malicious_address": "0",
  "phishing_activities": "0",
  "blacklist_doubt": "0",
  "honeypot_related_address": "0",
  "blackmail_activities": "0",
  "stealing_attack": "0",
  "fake_kyc": "0",
  "contract_address": "1"
}
```

---

## 4. check_phishing_site

**Purpose:** Check whether a URL is a known phishing site. Use before recommending any external links to users.

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| url | Yes | — | string | Full URL to check (e.g. `"https://example.com"`) |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| phishing_site | string | `"0"` = safe, `"1"` = confirmed phishing site |
| website_contract_security | array | Any flagged contracts associated with the site |

### Example

**Call:**
```json
{
  "tool": "check_phishing_site",
  "params": {
    "url": "https://app.uniswap.org"
  }
}
```

**Response:**
```json
{
  "phishing_site": "0",
  "website_contract_security": []
}
```

---

## Chain ID Reference

| Chain | GoPlus chain_id | Notes |
|-------|----------------|-------|
| Ethereum | `"1"` | Mainnet |
| BSC | `"56"` | BNB Smart Chain |
| Polygon | `"137"` | Polygon PoS |
| Arbitrum | `"42161"` | Arbitrum One |
| Base | `"8453"` | Coinbase L2 |
| Avalanche | `"43114"` | C-Chain |
| Optimism | `"10"` | OP Mainnet |
| Fantom | `"250"` | Fantom Opera |
| Cronos | `"25"` | Cronos Mainnet |
| Gnosis | `"100"` | xDAI |
| Linea | `"59144"` | Linea Mainnet |
| Scroll | `"534352"` | Scroll Mainnet |
| zkSync Era | `"324"` | zkSync |
| Solana | `"solana"` | Use string `"solana"`, not a numeric ID |

---

## Decision Matrix

Use this matrix to determine BLOCK vs WARN outcomes from `check_token_security` results.

| GoPlus Field | BLOCK if | WARN if |
|-------------|----------|---------|
| is_honeypot | `=== "1"` | — |
| buy_tax | `> "0.05"` (5%) | `> "0.01"` (1%) |
| sell_tax | `> "0.10"` (10%) | `> "0.02"` (2%) |
| is_mintable | `=== "1"` AND contract < 7 days old | `=== "1"` |
| can_take_back_ownership | `=== "1"` | — |
| owner_change_balance | `=== "1"` | — |
| is_open_source | `=== "0"` | — |
| is_proxy | — | `=== "1"` |
| top 10 holders % | > 80% combined | > 50% combined |
| slippage_modifiable | `=== "1"` | — |
| transfer_pausable | — | `=== "1"` |
| cannot_sell_all | `=== "1"` | — |
| personal_slippage_modifiable | `=== "1"` | — |

### How to Calculate Top 10 Holder Concentration

```
1. Sort holders[] by percent descending
2. Filter out known contract addresses (is_contract === 1) that are
   DEX pools or burn addresses (these are NOT concentration risk)
3. Sum the top 10 non-contract holder percentages
4. Compare against 80% (BLOCK) and 50% (WARN) thresholds
```

### Decision Logic (Pseudocode)

```
security = check_token_security(chain_id, address)

// Hard blocks — any single trigger stops the recommendation
if security.is_honeypot === "1":           return BLOCK("Honeypot detected")
if security.buy_tax > 0.05:               return BLOCK("Buy tax > 5%")
if security.sell_tax > 0.10:              return BLOCK("Sell tax > 10%")
if security.can_take_back_ownership === "1": return BLOCK("Ownership reclaimable")
if security.owner_change_balance === "1":  return BLOCK("Owner can change balances")
if security.is_open_source === "0":        return BLOCK("Unverified contract")
if security.slippage_modifiable === "1":   return BLOCK("Tax modifiable by owner")
if security.cannot_sell_all === "1":       return BLOCK("Cannot sell all tokens")
if top10HolderPct > 0.80:                 return BLOCK("Top 10 holders > 80%")

// Soft warnings — flag but allow with disclosure
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

---

## Integration Notes

1. **Always call before recommending** — Every skill that involves an onchain token MUST call `check_token_security` before outputting a recommendation. No exceptions.

2. **Cache results** — Security data is relatively stable. Cache results for **5 minutes** within a session to avoid redundant API calls.

3. **Solana tokens** — Use `chain_id: "solana"` (string, not numeric). Field availability may differ slightly from EVM tokens.

4. **Multiple tokens** — Call `check_token_security` once per token. There is no batch endpoint. For multi-token scans, make parallel calls.

5. **GoPlus API response wrapper** — The actual security data is nested inside `result.<contract_address>`. Always extract using the contract address as key:
   ```
   response.result["0x6982508145454ce325ddbe47a25d4ec3d2311933"]
   ```

6. **Free tier limits** — The free tier allows approximately 100 requests/day. For production use, upgrade to a paid plan or implement caching aggressively.

7. **Fallback** — If GoPlus is unreachable, surface a `[WARN]` to the user: "Security check unavailable — proceed with extreme caution." Never silently skip the check.
