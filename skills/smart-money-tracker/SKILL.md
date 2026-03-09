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

- Auto-execute copy trades — **ALWAYS requires human approval**, no exceptions.
  This constraint cannot be overridden by configuration or user request.
- Provide financial advice — outputs are analysis only.
- Guarantee profits — past smart money performance does not predict future results.

---

## Pre-flight

Before executing any command, verify the following MCP servers are available.

| Server | Required | Purpose |
|--------|----------|---------|
| OnchainOS CLI | **Critical** | Primary data source — smart money signals, token data, wallet balances |
| GoPlus MCP | **Critical** | Token security check on EVERY tracked token; wallet address screening |
| DexScreener MCP | Optional | Additional market context, token pair data |
| Rug Munch MCP | Optional | Deployer history analysis, serial rugger detection |
| Nansen MCP | Optional | Premium wallet labels and entity classification |

Run the OnchainOS pre-flight check:
```bash
onchainos dex-market signal-chains
```
This returns the list of chains with signal support. If it fails, OnchainOS
is not properly configured — see `config/mcp-setup-guide.md`.

---

## Command Index

| Command | Purpose | Read/Write |
|---------|---------|------------|
| `scan` | Discover recent smart money activity across chains | Read |
| `track-wallet` | Monitor a specific wallet for position changes | Read |
| `evaluate` | Deep analysis of whether a specific signal is worth following | Read |
| `leaderboard` | Top performing wallets by realized PnL over a period | Read |
| `copy-plan` | Generate a trade plan based on a signal — **REQUIRES HUMAN APPROVAL** | Read |

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
| `--address` | string | Yes | — | Wallet address | Valid address format |
| `--chain` | string | No | auto-detect | Supported chain | OnchainOS-supported |
| `--lookback-hours` | integer | No | `72` | 1 - 720 | Positive integer |

**Pre-checks before tracking:**
1. GoPlus `check_address_security` — BLOCK if malicious, phishing, or blacklisted.
2. OnchainOS `wallet-portfolio all-balances` — retrieve current holdings.

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
| `--signal-token` | string | Yes | — | Token address | Valid address |
| `--signal-wallet` | string | Yes | — | Wallet address | Valid address |
| `--chain` | string | Yes | — | Chain name | OnchainOS-supported |
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
| `--signal-token` | string | Yes | — | Token address | Valid address |
| `--chain` | string | Yes | — | Chain name | OnchainOS-supported |
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
| **STALE** | 120+ min | `░░░░░░░░░░` | Do NOT recommend — alpha gone |

### Decay Function

```
strength_score = max(0, 1 - (age_min / 120))
```

Reference: `references/formulas.md` Section 7 (Signal Decay Function).
The smart money signal type has a decay rate of 0.02 and half-life of ~35 min.

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
   → Get list of chains with signal support
   → Filter by user-requested `--chains`

2. For each chain:
   ```bash
   onchainos dex-market signal-list --chain {chain} --wallet-type {type_ids}
   ```
   → Returns smart money signals with wallet addresses and token interactions
   → Map OnchainOS wallet types: `1`=smart money → fund, `2`=KOL → kol, `3`=whale → whale

3. For each signal token:
   ```bash
   onchainos dex-token info {token_address} --chain {chain}
   ```
   → Token metadata: symbol, name, isRiskToken, holderCount, createTime

4. For each signal token (liquidity check):
   ```bash
   onchainos dex-token price-info {token_address} --chain {chain}
   ```
   → liquidityUsd, volume24h, priceChange24h

**For `track-wallet`:**

1. GoPlus `check_address_security` on the target wallet address
   → BLOCK if `is_malicious_address === "1"` or any flag set

2. `onchainos wallet-portfolio all-balances {address} --chains {chain}`
   → Current holdings

3. Cross-reference holdings with recent `signal-list` data for the wallet

### Step 3: Safety Checks (CRITICAL)

This is the most important step. Every signal passes through ALL checks
before being surfaced.

**For EVERY token in signals:**

1. **GoPlus `check_token_security`:**
   ```
   GoPlus check_token_security(chain_id, token_address)
   ```
   Apply the full decision matrix from `references/goplus-tools.md`:
   - `is_honeypot === "1"` → BLOCK
   - `buy_tax > 5%` → BLOCK
   - `sell_tax > 10%` → BLOCK
   - `is_open_source === "0"` → BLOCK
   - `slippage_modifiable === "1"` → BLOCK
   - `owner_change_balance === "1"` → BLOCK
   - Top 10 holder concentration > 80% → BLOCK
   - (See full matrix in `references/goplus-tools.md`)

2. **OnchainOS `isRiskToken` flag:**
   ```
   if dex-token info returns isRiskToken === true → BLOCK
   ```

3. **Signal age calculation:**
   ```
   signal_age_min = (Date.now() - signal_timestamp) / 60000
   ```
   Apply decay model (see Signal Decay Model above).
   - age > 120 min → REJECT (omit from output)
   - age > 30 min → Mark as LOW, do NOT recommend

4. **Liquidity check:**
   ```
   if dex-token price-info liquidityUsd < 100000 → BLOCK
   ```

5. **Optional — Rug Munch deployer analysis:**
   If Rug Munch MCP is available, check deployer address:
   - Serial rugger history → WARN (display to user)
   - Known rug pattern → BLOCK

6. **Optional — GoPlus `check_address_security` on signal wallet:**
   For `track-wallet` and `evaluate`, screen the wallet itself:
   - Flagged as malicious/phishing → BLOCK

### Step 4: Output

Format using Smart Money Signal Template from
`references/output-templates.md` Section 8.

**MANDATORY:** Every output includes the disclaimer block (see Mandatory
Disclaimers below).

---

## Safety Checks — Summary Table

This skill applies the **most restrictive** safety checks in the project.

| # | Check | Source | Threshold | Action |
|---|-------|--------|-----------|--------|
| 1 | Human approval | Skill policy | ALWAYS for `copy-plan` | **BLOCK** until user confirms — non-negotiable |
| 2 | Token honeypot | GoPlus `check_token_security` | `is_honeypot === "1"` | **BLOCK** — `SECURITY_BLOCKED` |
| 3 | Token tax rate | GoPlus `check_token_security` | buy > 5% OR sell > 10% | **BLOCK** — `SECURITY_BLOCKED` |
| 4 | Contract verification | GoPlus `check_token_security` | `is_open_source === "0"` | **BLOCK** — `SECURITY_BLOCKED` |
| 5 | Owner can change balance | GoPlus `check_token_security` | `owner_change_balance === "1"` | **BLOCK** — `SECURITY_BLOCKED` |
| 6 | Slippage modifiable | GoPlus `check_token_security` | `slippage_modifiable === "1"` | **BLOCK** — `SECURITY_BLOCKED` |
| 7 | Holder concentration | GoPlus `check_token_security` | Top 10 non-contract > 80% | **BLOCK** — `SECURITY_BLOCKED` |
| 8 | OnchainOS risk flag | OnchainOS `dex-token info` | `isRiskToken === true` | **BLOCK** — `SECURITY_BLOCKED` |
| 9 | Signal age | Decay model | > 30 min | **BLOCK** — do not recommend |
| 10 | Max copy size | Config `max_copy_size_usd` | > $1,000 (default) | **BLOCK** unless user overrides (hard cap $5,000) |
| 11 | Token liquidity | OnchainOS `dex-token price-info` | `liquidityUsd < $100,000` | **BLOCK** — `INSUFFICIENT_LIQUIDITY` |
| 12 | Deployer history | Rug Munch MCP (optional) | Serial rugger pattern | **WARN** — display to user |
| 13 | Wallet address security | GoPlus `check_address_security` | Any malicious flag | **BLOCK** — do not track/follow |
| 14 | Holder concentration (soft) | GoPlus `check_token_security` | Top 10 non-contract > 50% | **WARN** — display warning |
| 15 | Proxy contract | GoPlus `check_token_security` | `is_proxy === "1"` | **WARN** — upgradeable contract |

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
  [REQUIRES HUMAN APPROVAL — 需人工確認]
══════════════════════════════════════════
```

---

## Risk Limits

From `config/risk-limits.example.yaml` — smart_money_tracker section:

| Parameter | Default | Hard Cap | Description |
|-----------|---------|----------|-------------|
| `max_copy_size_usd` | $1,000 | $5,000 | Maximum USD per copied trade |
| `require_human_approval` | **ALWAYS** | **ALWAYS** | Cannot be overridden |
| `signal_max_age_min` | 30 | 60 | Maximum signal age in minutes |
| `min_wallet_pnl_usd` | $100,000 | — | Only follow wallets with > $100K realized PnL |
| `min_wallet_win_rate` | 60% | — | Minimum tracked wallet win rate |
| `blocked_wallet_types` | — | — | Wallets flagged by GoPlus `check_address_security` |

---

## Output Templates

### Scan Output (Primary)

```
══════════════════════════════════════════
  Smart Money Tracker
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
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
時間: 3 分鐘前 → [HIGH] 信號強度

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
  Smart Money Tracker — Copy Plan
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
══════════════════════════════════════════
  [REQUIRES HUMAN APPROVAL — 需人工確認]
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
  │                    [HIGH] — 3 min old
  ├─ Liquidity Risk:   ▓▓░░░░░░░░  2/10
  │                    $12.5M liquidity, $500 trade = negligible impact
  ├─ Holder Risk:      ▓▓▓▓▓░░░░░  5/10
  │                    Top 10 holders = 55% (above 50% WARN threshold)
  └─ Wallet Track Record: ▓▓▓▓░░░░░░  4/10
                       62% win rate, $180K realized PnL (30d)

── 安全檢查 (全部通過) ────────────────────

  [  SAFE  ] Honeypot Check — not a honeypot
  [  SAFE  ] Tax Rate — 0% buy / 0% sell
  [  SAFE  ] Contract Verified — open source
  [  WARN  ] Holder Concentration — Top 10 = 55%
  [  SAFE  ] Liquidity — $12.5M TVL
  [  SAFE  ] Signal Freshness — 3 min (HIGH)

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
  Smart Money Tracker — Leaderboard
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
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
1. `onchainos dex-market signal-chains` → get supported chains
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
1. GoPlus `check_address_security` on the wallet — BLOCK if flagged
2. `onchainos wallet-portfolio all-balances 0x1234...` → current holdings
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
6. **WAIT for explicit user confirmation** — do NOT proceed without it
**Output:** Copy plan with full cost breakdown, risk assessment, and
mandatory human approval prompt.

---

## Error Handling

| Error Code | Trigger | User Message |
|------------|---------|-------------|
| `MCP_NOT_CONNECTED` | OnchainOS CLI unreachable | OnchainOS 無法連線，無法獲取聰明錢數據 |
| `SECURITY_BLOCKED` | GoPlus flags token | 安全檢查未通過：{reason}。此代幣存在風險，已從結果中移除。 |
| `SIGNAL_TOO_OLD` | Signal age > 30 min | 信號已超過 {age} 分鐘，可能已失效。不建議跟單。 |
| `INSUFFICIENT_LIQUIDITY` | Token liquidity < $100K | 代幣流動性不足 (${liquidity})，不建議交易 |
| `TRADE_SIZE_EXCEEDED` | Copy size > max | 跟單金額 ${amount} 超過上限 ${limit} |
| `HUMAN_APPROVAL_REQUIRED` | copy-plan needs confirmation | 此操作需要您的確認才能繼續 |
| `ADDRESS_FLAGGED` | GoPlus flags wallet address | 此錢包地址被標記為可疑：{reason}。不建議追蹤。 |

When OnchainOS is unreachable, this skill cannot function — there is no
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
| Fee Schedule | `references/fee-schedule.md` | Section 3 (Gas Benchmarks) — for copy-plan cost estimates |
