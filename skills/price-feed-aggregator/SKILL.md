---
name: price-feed-aggregator
description: >
  Provides unified, synchronized price snapshots across OKX CEX and onchain DEX venues.
  Activates when user asks about price comparisons, spread monitoring, or cross-venue pricing.
  Trigger phrases include: "price comparison", "價格比較", "spread", "價差", "DEX price",
  "CEX vs DEX", "cross-venue", "monitor prices", "監控價格", "price snapshot", "快照".
  Do NOT use for: trade execution, token security checks, portfolio management.
  Requires: okx-trade-mcp + OnchainOS CLI + CoinGecko MCP (optional).
allowed-tools: >
  okx-DEMO-simulated-trading:market_get_ticker,
  okx-DEMO-simulated-trading:market_get_candles,
  okx-DEMO-simulated-trading:market_get_orderbook,
  okx-DEMO-simulated-trading:system_get_capabilities,
  okx-LIVE-real-money:market_get_ticker,
  okx-LIVE-real-money:market_get_candles,
  okx-LIVE-real-money:market_get_orderbook,
  okx-LIVE-real-money:system_get_capabilities
---

# price-feed-aggregator

Foundation skill for the Onchain x CEX Strats system. Collects, normalizes, and compares price data from CEX (OKX via okx-trade-mcp) and DEX (OKX DEX via OnchainOS CLI) venues simultaneously. Detects cross-venue spreads and provides structured price snapshots consumed by all downstream strategy skills.

---

## 1. Role

**Foundation data layer** that all strategy skills depend on.

This skill is responsible for:
- Fetching real-time prices from multiple venues (OKX CEX, OKX DEX, CoinGecko)
- Normalizing all prices to USD
- Calculating cross-venue spread in basis points
- Enforcing data staleness thresholds
- Providing structured `PriceSnapshot` objects for downstream consumption

This skill does **NOT**:
- Execute any trades (read-only)
- Perform token security checks (delegate to GoPlus MCP directly)
- Calculate profitability after costs (delegate to `profitability-calculator`)
- Make buy/sell recommendations
- Manage positions or portfolio state

---

## 2. Language

Match the user's language. Default: Traditional Chinese (繁體中文).

Metric labels may use English abbreviations regardless of language:
- `bps`, `spread`, `bid`, `ask`, `snapshot`, `staleness`
- Timestamps always displayed in UTC

---

## 3. Account Safety

| Rule | Detail |
|------|--------|
| Default mode | Demo (`okx-DEMO-simulated-trading`) |
| Mode display | Every output header shows `[DEMO]` or `[LIVE]` |
| Read-only | This skill performs **zero** write operations — no trades, no transfers, no approvals |
| Recommendation header | Always show `[RECOMMENDATION ONLY — 不會自動執行]` |
| Live switch | Requires explicit user confirmation per protocol in `references/safety-checks.md` |

Even in `[LIVE]` mode, this skill only reads market data. There is no risk of accidental execution.

---

## 4. Pre-flight (Machine-Executable Checklist)

Run these checks **in order** before any command. BLOCK at any step halts execution.

| # | Check | Command / Tool | Success Criteria | Failure Action |
|---|-------|---------------|-----------------|----------------|
| 1 | okx-trade-mcp connected | `system_get_capabilities` (DEMO or LIVE server) | `authenticated: true`, `modules` includes `"market"` | BLOCK — output `MCP_NOT_CONNECTED`. Tell user to verify `~/.okx/config.toml` and restart MCP server. |
| 2 | okx-trade-mcp mode | `system_get_capabilities` → `mode` field | Returns `"demo"` or `"live"` matching expected mode | WARN — display actual mode in header. If user requested live but got demo, surface mismatch. |
| 3 | OnchainOS CLI installed | `which onchainos` (Bash) | Exit code 0, returns a valid path | WARN — DEX venue unavailable. Proceed with CEX-only data. Tell user: `npx skills add okx/onchainos-skills` |
| 4 | OnchainOS CLI functional | `onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee` | Returns valid JSON with `priceUsd` field | WARN — DEX venue unavailable. Proceed with CEX-only data. Suggest checking network connectivity. |
| 5 | CoinGecko MCP (optional) | `simple_price(ids: "bitcoin", vs_currencies: "usd")` | Returns valid price object | INFO — CoinGecko venue unavailable. Proceed without it. This is a backup source. |

### Pre-flight Decision Tree

```
Check 1 FAIL → BLOCK (cannot proceed without CEX data)
Check 1 PASS → Check 2
  Check 2 mismatch → WARN + continue
  Check 2 PASS → Check 3
    Check 3 FAIL → WARN: DEX unavailable, CEX-only mode
    Check 3 PASS → Check 4
      Check 4 FAIL → WARN: DEX unavailable, CEX-only mode
      Check 4 PASS → Check 5
        Check 5 FAIL → INFO: CoinGecko unavailable (optional)
        Check 5 PASS → ALL SYSTEMS GO
```

If DEX venue is unavailable and user requested cross-venue spread: output error explaining DEX is offline and suggest retrying after `npx skills add okx/onchainos-skills`.

---

## 5. Skill Routing Matrix

| User Need | Use THIS Skill? | Delegate To |
|-----------|----------------|-------------|
| "BTC 而家幾錢" / "what's the price of BTC" | Yes — `snapshot` | — |
| "CEX 同 DEX 價差幾多" / "price comparison" | Yes — `spread` | — |
| "幫我監控 ETH 價差" / "monitor ETH spread" | Yes — `monitor` | — |
| "過去幾個鐘嘅價差變化" / "spread history" | Yes — `history` | — |
| "呢個幣安唔安全" / "is this token safe" | No | GoPlus MCP directly |
| "有冇套利機會" / "is this arb profitable" | No | `cex-dex-arbitrage` (calls this skill internally for prices) |
| "計埋成本淨利潤幾多" / "net profitability" | No | `profitability-calculator` |
| "幫我買" / "execute trade" | No | Refuse — no skill executes trades |
| "我嘅帳戶餘額" / "my balance" | No | okx-trade-mcp `account_*` directly |
| "資金費率" / "funding rate" | No | `funding-rate-arbitrage` |

---

## 6. Command Index

| Command | Function | Read/Write | Description |
|---------|----------|-----------|-------------|
| `snapshot` | Multi-venue price snapshot | Read | Fetch current prices from 2+ venues for 1+ assets, calculate spread |
| `spread` | Directional spread analysis | Read | Deep spread analysis between two specific venues for one asset, including orderbook-based slippage |
| `monitor` | Continuous spread monitoring | Read | Poll prices at intervals and alert when spread exceeds threshold |
| `history` | Historical spread time series | Read | Retrieve historical spread data using candle endpoints |

---

## 7. Parameter Reference

### 7.1 Command: `snapshot`

Fetch a synchronized price snapshot across multiple venues for one or more assets.

```bash
price-feed-aggregator snapshot --assets BTC,ETH,SOL --venues okx-cex,okx-dex --chains ethereum,solana
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--assets` | string[] | Yes | — | Any valid symbol | Uppercase, comma-separated, max 20 items |
| `--venues` | string[] | No | `["okx-cex","okx-dex"]` | `okx-cex`, `okx-dex`, `coingecko` | Must have >= 2 venues for spread calc; 1 venue allowed for price-only |
| `--chains` | string[] | No | `["ethereum"]` | `ethereum`, `solana`, `base`, `arbitrum`, `bsc`, `polygon`, `xlayer` | Must be a supported chain (see `references/chain-reference.md`) |
| `--mode` | string | No | `"analysis"` | `analysis`, `arb` | `"arb"` enforces 5s staleness threshold instead of 60s |

#### Return Schema

```yaml
PriceSnapshot:
  timestamp: integer          # Unix ms — when the snapshot was assembled
  asset: string               # e.g. "BTC"
  venues:
    - venue: string           # e.g. "okx-cex"
      price: string           # Last price in USD — parse to float before arithmetic
      bid: string             # Best bid price (CEX only, null for DEX)
      ask: string             # Best ask price (CEX only, null for DEX)
      volume_24h: string      # 24h volume in base currency
      source_tool: string     # e.g. "market_get_ticker" or "dex-market price"
      chain: string           # Chain name (DEX only, null for CEX)
      data_age_ms: integer    # Milliseconds since venue's own timestamp
  max_spread_bps: number      # Largest spread across all venue pairs
  max_spread_venues:          # The two venues with the largest spread
    - string                  # e.g. "okx-cex"
    - string                  # e.g. "okx-dex:ethereum"
  staleness_ok: boolean       # true if all venue data_age_ms < threshold
```

#### Return Fields Detail

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | integer | Unix epoch milliseconds when snapshot was assembled |
| `asset` | string | Asset symbol in uppercase (e.g., `"BTC"`, `"ETH"`) |
| `venues[]` | array | Array of venue price objects, one per venue queried |
| `venues[].venue` | string | Venue identifier: `"okx-cex"`, `"okx-dex:ethereum"`, `"coingecko"` |
| `venues[].price` | string | Last traded / quoted price in USD. **String — parse to float.** |
| `venues[].bid` | string | Best bid price. CEX only. `null` for DEX venues. |
| `venues[].ask` | string | Best ask price. CEX only. `null` for DEX venues. |
| `venues[].volume_24h` | string | 24-hour volume in base currency |
| `venues[].source_tool` | string | MCP tool or CLI command that produced this data point |
| `venues[].chain` | string | Chain name for DEX venues (e.g., `"ethereum"`). `null` for CEX. |
| `venues[].data_age_ms` | integer | Age of data in milliseconds: `now() - venue_timestamp` |
| `max_spread_bps` | number | Maximum pairwise spread in basis points across all venues |
| `max_spread_venues` | [string, string] | The two venue identifiers forming the widest spread |
| `staleness_ok` | boolean | `true` if all venues have `data_age_ms` below the threshold (60s default, 5s in arb mode) |

---

### 7.2 Command: `spread`

Deep spread analysis between exactly two venues for a single asset. Includes orderbook-based slippage estimation for a given trade size.

```bash
price-feed-aggregator spread --asset ETH --venue-a okx-cex --venue-b okx-dex --chain base --size-usd 50000
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | — | Any valid symbol | Uppercase, single asset only |
| `--venue-a` | string | No | `"okx-cex"` | `okx-cex`, `okx-dex`, `coingecko` | Must differ from venue-b |
| `--venue-b` | string | No | `"okx-dex"` | `okx-cex`, `okx-dex`, `coingecko` | Must differ from venue-a |
| `--chain` | string | No | `"ethereum"` | `ethereum`, `solana`, `base`, `arbitrum`, `bsc`, `polygon`, `xlayer` | Required when venue-b is `okx-dex` |
| `--size-usd` | number | No | `10000` | — | Min: 100, Max: 1,000,000. Used for slippage estimation. |

#### Return Schema

```yaml
SpreadResult:
  timestamp: integer
  asset: string
  venue_a:
    venue: string
    price: string
    bid: string
    ask: string
  venue_b:
    venue: string
    price: string
    chain: string
  gross_spread_bps: number        # abs(price_a - price_b) / min(price_a, price_b) * 10000
  buy_venue: string               # Venue where price is lower (buy here)
  sell_venue: string              # Venue where price is higher (sell here)
  direction: string               # e.g. "Buy CEX / Sell DEX" or "Buy DEX / Sell CEX"
  estimated_slippage:
    venue_a_slippage_bps: number  # Based on orderbook depth (CEX) or quote impact (DEX)
    venue_b_slippage_bps: number
  effective_spread_bps: number    # gross_spread_bps - total_slippage_bps
  data_age:
    venue_a_ms: integer
    venue_b_ms: integer
  staleness_ok: boolean
```

#### Return Fields Detail

| Field | Type | Description |
|-------|------|-------------|
| `gross_spread_bps` | number | Unsigned spread before slippage, in basis points |
| `buy_venue` | string | The venue with the lower price — buy side |
| `sell_venue` | string | The venue with the higher price — sell side |
| `direction` | string | Human-readable direction string |
| `estimated_slippage.venue_a_slippage_bps` | number | Estimated execution slippage at `--size-usd` on venue A |
| `estimated_slippage.venue_b_slippage_bps` | number | Estimated execution slippage at `--size-usd` on venue B |
| `effective_spread_bps` | number | Gross spread minus total estimated slippage. Negative = unprofitable. |
| `staleness_ok` | boolean | Whether both venues have fresh data |

---

### 7.3 Command: `monitor`

Continuously poll prices across venues and alert when the spread exceeds a user-defined threshold.

```bash
price-feed-aggregator monitor --assets ETH,SOL --threshold-bps 20 --duration-min 30 --check-interval-sec 15
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--assets` | string[] | Yes | — | Any valid symbol | Uppercase, comma-separated, max 10 |
| `--threshold-bps` | number | No | `20` | — | Min: 1, Max: 500 |
| `--duration-min` | number | No | `30` | — | Min: 1, Max: 1440 (24h) |
| `--check-interval-sec` | number | No | `15` | — | Min: 5, Max: 300 |
| `--venues` | string[] | No | `["okx-cex","okx-dex"]` | `okx-cex`, `okx-dex`, `coingecko` | Must have >= 2 |
| `--chains` | string[] | No | `["ethereum"]` | See chain reference | Must be supported |

#### Return Schema (per alert)

```yaml
MonitorAlert:
  alert_type: "SPREAD_THRESHOLD_CROSSED"
  timestamp: integer
  asset: string
  current_spread_bps: number
  threshold_bps: number
  direction: string               # "Buy CEX / Sell DEX" etc.
  venue_prices:
    - venue: string
      price: string
  previous_spread_bps: number     # Spread at last check
  spread_change_bps: number       # current - previous
  checks_completed: integer       # How many polling cycles so far
  checks_remaining: integer       # How many cycles left
  suggested_action: string        # e.g. "Evaluate with cex-dex-arbitrage.scan"
```

#### Return Fields Detail

| Field | Type | Description |
|-------|------|-------------|
| `alert_type` | string | Always `"SPREAD_THRESHOLD_CROSSED"` for this command |
| `current_spread_bps` | number | Spread at the moment of alert |
| `threshold_bps` | number | User-defined threshold that was crossed |
| `previous_spread_bps` | number | Spread at the previous check cycle |
| `spread_change_bps` | number | Change from previous cycle |
| `suggested_action` | string | Recommended next step for the user |

---

### 7.4 Command: `history`

Retrieve historical price data from both venues and compute a spread time series.

```bash
price-feed-aggregator history --asset BTC --lookback-hours 24 --granularity 15m --venues okx-cex,okx-dex --chain ethereum
```

#### Parameters

| Parameter | Type | Required | Default | Enum Values | Validation Rule |
|-----------|------|----------|---------|-------------|-----------------|
| `--asset` | string | Yes | — | Any valid symbol | Uppercase, single asset |
| `--lookback-hours` | number | No | `24` | — | Min: 1, Max: 2160 (90 days — OKX data retention limit) |
| `--granularity` | string | No | `"15m"` | `1m`, `5m`, `15m`, `1H` | Must match candle intervals supported by both OKX CEX and OnchainOS |
| `--venues` | string[] | No | `["okx-cex","okx-dex"]` | `okx-cex`, `okx-dex` | Must have exactly 2 for history comparison |
| `--chain` | string | No | `"ethereum"` | See chain reference | Required when one venue is `okx-dex` |

#### Return Schema

```yaml
SpreadHistory:
  asset: string
  venue_a: string
  venue_b: string
  granularity: string
  data_points:
    - timestamp: integer          # Candle open time, Unix ms
      venue_a_close: string       # Close price at this candle
      venue_b_close: string       # Close price at this candle
      spread_bps: number          # Spread at this data point
  summary:
    avg_spread_bps: number
    max_spread_bps: number
    min_spread_bps: number
    std_dev_bps: number
    trend: string                 # "widening", "narrowing", "stable", "volatile"
    sparkline: string             # e.g. "▁▂▃▅▇▅▃▂"
```

#### Return Fields Detail

| Field | Type | Description |
|-------|------|-------------|
| `data_points[].timestamp` | integer | Candle open time in Unix milliseconds |
| `data_points[].venue_a_close` | string | Closing price for venue A at this interval |
| `data_points[].venue_b_close` | string | Closing price for venue B at this interval |
| `data_points[].spread_bps` | number | Calculated spread at this data point |
| `summary.avg_spread_bps` | number | Mean spread over the lookback period |
| `summary.max_spread_bps` | number | Maximum spread observed |
| `summary.min_spread_bps` | number | Minimum spread observed |
| `summary.std_dev_bps` | number | Standard deviation of spread — indicates volatility |
| `summary.trend` | string | Qualitative trend assessment |
| `summary.sparkline` | string | Visual representation of spread over time using `▁▂▃▄▅▆▇█` |

---

## 8. Operation Flow

### Step 1: Intent Recognition

Parse user message to extract:

| Element | Extraction Logic | Fallback |
|---------|-----------------|----------|
| Command | Map to `snapshot` / `spread` / `monitor` / `history` based on keywords | Default: `snapshot` |
| Assets | Extract token symbols (BTC, ETH, SOL...) | **Ask user** — assets are required |
| Venues | Look for "CEX", "DEX", "OKX", "鏈上", "交易所" | Default: `["okx-cex", "okx-dex"]` |
| Chains | Look for chain names or "Ethereum", "Solana", "Base", "Arbitrum", "以太坊" | Default: `["ethereum"]` |
| Mode | Look for "arb", "套利", "arbitrage" → set `arb` mode | Default: `"analysis"` |

**Keyword-to-command mapping:**

| Keywords | Command |
|----------|---------|
| "價格", "price", "幾錢", "快照", "snapshot", "而家" | `snapshot` |
| "價差", "spread", "差價", "bid ask" | `spread` |
| "監控", "monitor", "watch", "alert", "通知" | `monitor` |
| "歷史", "history", "trend", "趨勢", "過去" | `history` |

### Step 2: Data Collection

For each venue requested, execute the appropriate tool call:

#### okx-cex

```
Tool: market_get_ticker
Params: { instId: "{ASSET}-USDT" }
Extract: last, bidPx, askPx, vol24h, ts
```

**Important:** OKX returns all values as strings. Parse to float before any calculation.

For `spread` command with `--size-usd`, also fetch orderbook:

```
Tool: market_get_orderbook
Params: { instId: "{ASSET}-USDT", sz: "20" }
Extract: bids, asks — walk the book to estimate slippage at trade size
```

#### okx-dex

Step A — Resolve token address (if not cached):

```bash
onchainos dex-token search {ASSET} --chains {chain}
# Extract: address, symbol, chain, decimals
# Pick the result matching the target chain with highest liquidity
```

Step B — Fetch price:

```bash
onchainos dex-market price --chain {chain} --token {address}
# Extract: priceUsd, volume24h
```

**Note:** For native tokens (ETH, SOL, BNB, etc.), use the native address from `references/chain-reference.md` directly — skip the search step.

Native address quick reference:
- EVM chains: `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee`
- Solana: `11111111111111111111111111111111`

#### coingecko (optional backup)

```
Tool: simple_price
Params: { ids: "{asset_id}", vs_currencies: "usd" }
Extract: usd price
```

CoinGecko asset IDs: `bitcoin`, `ethereum`, `solana`, `binancecoin`, etc. Map from symbol.

### Step 3: Normalize & Compare

1. **Normalize to USD** — All prices are already in USD from the extraction step.

2. **Calculate pairwise spread** for every venue combination:

   ```
   spread_bps = abs(price_a - price_b) / min(price_a, price_b) * 10000
   ```

   Reference: `references/formulas.md` Section 1 — CEX-DEX Spread.

3. **Check staleness** — For each venue, compute `data_age_ms = Date.now() - venue_timestamp`:
   - `analysis` mode: WARN if > 30s, BLOCK if > 60s
   - `arb` mode: WARN if > 3s, BLOCK if > 5s

4. **Anomaly detection** — If spread between CEX and DEX exceeds 5000 bps (50%), flag as `PRICE_ANOMALY`:
   - Likely cause: wrong token address resolved, stale data, or low-liquidity DEX pool
   - Action: BLOCK the snapshot for that asset, output warning

5. **Identify max spread** — Find the venue pair with the widest spread; populate `max_spread_bps` and `max_spread_venues`.

### Step 4: Format Output & Suggest Next Steps

Use the output template from `references/output-templates.md`:

- **Header:** Global Header Template with skill icon and mode
- **Body:** Price table with venue comparison
- **Footer:** Next Steps Template

Suggested follow-up actions (vary by spread magnitude):

| Spread Range | Suggested Actions |
|-------------|-------------------|
| < 10 bps | "價差偏小，暫無套利機會。可設定監控等待價差擴大。" |
| 10-30 bps | "掃描套利機會 → `cex-dex-arbitrage.scan`"; "計算盈利性 → `profitability-calculator.estimate`" |
| > 30 bps | "值得評估！建議立即執行 → `cex-dex-arbitrage.evaluate`"; "檢查代幣安全 → GoPlus" |

---

## 9. Safety Checks

| # | Check | Threshold | Action |
|---|-------|-----------|--------|
| 1 | Data staleness (analysis) | > 60s | BLOCK — refetch once, then error `DATA_STALE` |
| 2 | Data staleness (arb mode) | > 5s | BLOCK — refetch once, then error `DATA_STALE` |
| 3 | Venue availability | < 2 venues responding | BLOCK if spread requested; WARN if snapshot (proceed with single venue) |
| 4 | Price anomaly (CEX vs DEX) | > 50% difference (5000 bps) | BLOCK for that asset — output `PRICE_ANOMALY`. Likely wrong token address or stale DEX pool. |
| 5 | Token address resolution | `dex-token search` returns 0 results | BLOCK for that asset on that chain — output `TOKEN_NOT_FOUND` |
| 6 | Rate limiting | OKX API returns 429 or rate limit error | Wait 1s, retry up to 3 times. If still failing, output `RATE_LIMITED`. |
| 7 | CEX instrument validity | `market_get_ticker` returns error for instId | BLOCK for that asset — output `INSTRUMENT_NOT_FOUND`. Suggest checking the symbol. |

---

## 10. Error Codes & Recovery

| Code | Condition | User Message | Recovery |
|------|-----------|-------------|----------|
| `MCP_NOT_CONNECTED` | okx-trade-mcp server unreachable | MCP 伺服器無法連線。請確認 okx-trade-mcp 是否正在運行，並檢查 `~/.okx/config.toml` 設定。 | Verify MCP config, restart server, re-run pre-flight. |
| `VENUE_UNAVAILABLE` | A requested venue (CEX/DEX/CoinGecko) cannot be reached | {venue} 數據源暫時無法使用。已從可用數據源獲取價格。 | Proceed with available venues. If < 2 venues, cannot compute spread — inform user. |
| `DATA_STALE` | Price data exceeds staleness threshold | 市場數據已過期（{venue} 數據延遲 {age}ms，上限 {threshold}ms）。正在重新獲取... | Auto-retry fetch once. If still stale after retry, output error and stop. |
| `TOKEN_NOT_FOUND` | `dex-token search` returns no results for the given asset on the given chain | 在 {chain} 上找不到 {asset} 的代幣合約地址。請確認代幣名稱及鏈名是否正確。 | Suggest user verify the token symbol. Offer to search on alternative chains. |
| `PRICE_ANOMALY` | CEX vs DEX price differs by > 50% | {asset} 的 CEX 與 DEX 價格差異異常大（{spread}%），可能是代幣地址解析錯誤或 DEX 流動性極低。已排除此資產。 | Verify token address is correct. Check DEX liquidity via `dex-token price-info`. |
| `INSTRUMENT_NOT_FOUND` | OKX CEX does not list this instrument | OKX 上找不到 {instId} 交易對。請確認資產代號是否正確。 | Suggest similar instruments. Check if the asset is listed on OKX. |
| `RATE_LIMITED` | API rate limit hit after 3 retries | API 請求頻率超限。請稍候 {wait} 秒後重試。 | Exponential backoff: 1s, 2s, 4s. After 3 failures, surface error. |

---

## 11. Cross-Skill Integration Contracts

### Data Handoff to `cex-dex-arbitrage`

The `PriceSnapshot` object is the primary input for arbitrage scanning.

| Output Field | Consumer Skill | Consumer Parameter | Type |
|-------------|---------------|-------------------|------|
| `PriceSnapshot[]` (full array) | `cex-dex-arbitrage.scan` | `--prices` (internal) | PriceSnapshot[] |
| `PriceSnapshot.max_spread_bps` | `cex-dex-arbitrage.scan` | Filter: skip if < `min_spread_bps` | number |
| `PriceSnapshot.staleness_ok` | `cex-dex-arbitrage.scan` | Gate: BLOCK if `false` in arb mode | boolean |
| `PriceSnapshot.max_spread_venues` | `cex-dex-arbitrage.evaluate` | Identify buy/sell venues | [string, string] |

### Data Handoff to `profitability-calculator`

The `profitability-calculator` skill constructs `TradeLeg` objects from venue prices.

| Output Field | Consumer Skill | Consumer Parameter | Type |
|-------------|---------------|-------------------|------|
| `PriceSnapshot.venues[].price` | `profitability-calculator.estimate` | Leg entry/exit price | string (parse to float) |
| `PriceSnapshot.venues[].bid` | `profitability-calculator.estimate` | CEX sell price (use bid) | string |
| `PriceSnapshot.venues[].ask` | `profitability-calculator.estimate` | CEX buy price (use ask) | string |
| `PriceSnapshot.venues[].chain` | `profitability-calculator.estimate` | Gas cost chain lookup | string |
| `SpreadResult.effective_spread_bps` | `profitability-calculator.estimate` | Gross spread input | number |
| `SpreadResult.estimated_slippage` | `profitability-calculator.estimate` | Slippage cost input | object |

### Data Handoff to `funding-rate-arbitrage`

| Output Field | Consumer Skill | Consumer Parameter | Type |
|-------------|---------------|-------------------|------|
| `PriceSnapshot.venues[].price` (CEX spot) | `funding-rate-arbitrage.evaluate` | Spot entry price | string |

### Data Handoff to `basis-trading`

| Output Field | Consumer Skill | Consumer Parameter | Type |
|-------------|---------------|-------------------|------|
| `PriceSnapshot.venues[].price` (CEX spot) | `basis-trading.evaluate` | Spot price for basis calc | string |

---

## 12. Conversation Examples

### Example 1: BTC Price Comparison (snapshot)

**User:**
> BTC 而家 CEX 同 DEX 價差幾多？

**Intent Recognition:**
- Command: `snapshot` (price comparison)
- Asset: BTC
- Venues: okx-cex, okx-dex (default)
- Chain: ethereum (default)

**Tool Calls:**

```
1. system_get_capabilities → { authenticated: true, mode: "demo" }
2. market_get_ticker({ instId: "BTC-USDT" })
   → { last: "87234.5", bidPx: "87234.1", askPx: "87234.9", ts: "1741521600000" }
3. onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
   → { priceUsd: "87252.3", volume24h: "..." }
```

**Formatted Output:**

```
══════════════════════════════════════════
  Price Feed Aggregator — SNAPSHOT
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:30 UTC
  Data sources: OKX CEX + OKX DEX (Ethereum)
══════════════════════════════════════════

── BTC 跨場所價格快照 ────────────────────

  場所              價格              Bid / Ask
  ──────────────────────────────────────────────
  OKX CEX          $87,234.50       $87,234.10 / $87,234.90
  OKX DEX (ETH)    $87,252.30       — / —

  價差: 2.0 bps (0.020%)
  方向: CEX 較便宜 → Buy CEX / Sell DEX
  數據新鮮度: [SAFE] 全部 < 5 秒

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 價差偏小 (2.0 bps)，暫無明顯套利機會
  2. 設定監控等待價差擴大:
     price-feed-aggregator monitor --assets BTC --threshold-bps 15
  3. 查看歷史價差趨勢:
     price-feed-aggregator history --asset BTC --lookback-hours 24

  ── Disclaimer ────────────────────────
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

---

### Example 2: ETH Spread on Base (spread)

**User:**
> ETH 喺 Base 上面同 OKX 嘅 spread 幾大？我想用 $50,000 做

**Intent Recognition:**
- Command: `spread` (detailed spread with size)
- Asset: ETH
- Venue A: okx-cex
- Venue B: okx-dex
- Chain: base
- Size: $50,000

**Tool Calls:**

```
1. system_get_capabilities → { authenticated: true, mode: "demo" }
2. market_get_ticker({ instId: "ETH-USDT" })
   → { last: "3412.50", bidPx: "3412.10", askPx: "3412.90", ts: "..." }
3. market_get_orderbook({ instId: "ETH-USDT", sz: "20" })
   → { bids: [...], asks: [...], ts: "..." }
4. onchainos dex-market price --chain base --token 0x4200000000000000000000000000000000000006
   → { priceUsd: "3419.80", volume24h: "..." }
5. onchainos dex-swap quote --chain base \
     --from 0x4200000000000000000000000000000000000006 \
     --to 0x833589fcd6edb6e08f4c7c32d4f71b54bda02913 \
     --amount 14640000000000000000
   → { toTokenAmount: "...", priceImpactPercent: "0.08" }
```

**Formatted Output:**

```
══════════════════════════════════════════
  Price Feed Aggregator — SPREAD
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:32 UTC
  Data sources: OKX CEX + OKX DEX (Base)
══════════════════════════════════════════

── ETH 跨場所價差分析 ────────────────────

  Trade Size: $50,000.00

  OKX CEX
  ├─ Last:     $3,412.50
  ├─ Bid:      $3,412.10
  ├─ Ask:      $3,412.90
  └─ Slippage: 1.7 bps (at $50K, from orderbook)

  OKX DEX (Base)
  ├─ Price:    $3,419.80
  ├─ Impact:   0.08% (from swap quote)
  └─ Slippage: 0.8 bps (at $50K)

  ──────────────────────────────────────
  Gross Spread:      21.4 bps (0.214%)
  Direction:         Buy CEX / Sell DEX
  Est. Slippage:    -2.5 bps (total)
  ══════════════════════════════════════
  Effective Spread:  18.9 bps (0.189%)
  ══════════════════════════════════════

  數據新鮮度: [SAFE] CEX: 2s | DEX: 3s

══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. 有效價差 18.9 bps，值得進一步評估
  2. 掃描套利機會:
     cex-dex-arbitrage evaluate ETH base size=50000
  3. 計算扣除所有成本後淨利潤:
     profitability-calculator estimate

  ── Disclaimer ────────────────────────
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

---

### Example 3: Monitor SOL Prices (monitor)

**User:**
> 幫我監控 SOL 嘅 CEX/DEX 價差，超過 25 bps 就通知我，監控 1 小時

**Intent Recognition:**
- Command: `monitor`
- Asset: SOL
- Threshold: 25 bps
- Duration: 60 min
- Check interval: 15s (default)
- Venues: okx-cex, okx-dex (default)
- Chain: solana (inferred from SOL)

**Tool Calls (initial + per cycle):**

```
1. system_get_capabilities → { authenticated: true, mode: "demo" }
2. (verify) onchainos dex-market price --chain solana --token 11111111111111111111111111111111
   → { priceUsd: "145.20" }

Per cycle (every 15s):
  a. market_get_ticker({ instId: "SOL-USDT" }) → { last: "145.50", ... }
  b. onchainos dex-market price --chain solana --token 11111111111111111111111111111111
     → { priceUsd: "...", ... }
  c. Calculate spread_bps
  d. If spread_bps > 25 → emit alert
```

**Monitor Start Output:**

```
══════════════════════════════════════════
  Price Feed Aggregator — MONITOR
  [DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: 2026-03-09 14:35 UTC
  Data sources: OKX CEX + OKX DEX (Solana)
══════════════════════════════════════════

── 監控設定 ──────────────────────────────

  資產:       SOL
  場所:       OKX CEX vs OKX DEX (Solana)
  閾值:       25 bps
  持續時間:   60 分鐘 (至 15:35 UTC)
  檢查間隔:   每 15 秒
  預計檢查:   240 次

  當前價差:   8.2 bps — 低於閾值，持續監控中...

══════════════════════════════════════════
```

**Alert Output (when threshold crossed):**

```
══════════════════════════════════════════
  ALERT: SPREAD OPPORTUNITY
  2026-03-09 14:52 UTC
══════════════════════════════════════════

  SOL CEX-DEX 價差超過閾值

  ├─ 當前價差:     32 bps (0.320%)
  ├─ 閾值:         25 bps
  ├─ 上次價差:     18 bps (+14 bps in 15s)
  ├─ OKX CEX:     $145.50
  ├─ OKX DEX:     $145.03 (Solana)
  ├─ 方向:         Buy DEX / Sell CEX
  └─ 已檢查:       68/240 次 (剩餘 172 次)

  ── 建議操作 ────────────────────────────
  cex-dex-arbitrage evaluate SOL solana size=10000

══════════════════════════════════════════
```

**Monitor Complete Output:**

```
══════════════════════════════════════════
  Price Feed Aggregator — MONITOR COMPLETE
  [DEMO]
══════════════════════════════════════════

── 監控摘要 ──────────────────────────────

  資產:           SOL
  監控時長:       60 分鐘
  總檢查次數:     240
  觸發警報次數:   3

  價差統計:
  ├─ 平均:   12.4 bps
  ├─ 最大:   38 bps (14:52 UTC)
  ├─ 最小:   2 bps
  └─ 趨勢:   ▁▂▃▂▃▅▇▅▃▂▁▂ (volatile)

══════════════════════════════════════════
  以上僅為分析建議，不會自動執行任何交易。
══════════════════════════════════════════
```

---

## 13. Implementation Notes

### Token Address Resolution Cache

To avoid redundant `dex-token search` calls within a session, maintain an in-memory map:

```
{ASSET}:{CHAIN} → contract_address
```

Pre-populated entries from `references/chain-reference.md`:
- `ETH:ethereum` → `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee`
- `ETH:arbitrum` → `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee`
- `ETH:base` → `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee`
- `SOL:solana` → `11111111111111111111111111111111`
- `BNB:bsc` → `0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee`
- `BTC:ethereum` → resolve WBTC address via search

For non-native tokens, resolve once per session via `dex-token search` and cache.

### OKX String-to-Number Convention

All OKX API values are returned as **strings**. Always parse to `float` before any arithmetic:

```
WRONG:  "87234.5" - "87200.0" → NaN or string concat
RIGHT:  parseFloat("87234.5") - parseFloat("87200.0") → 34.5
```

This applies to: `last`, `bidPx`, `askPx`, `ts`, `vol24h`, all orderbook levels.

### Rate Limit Awareness

| Tool | Rate Limit | Strategy |
|------|-----------|----------|
| `market_get_ticker` | 20 req/s | Safe to batch multiple assets |
| `market_get_orderbook` | 20 req/s | Only fetch when `spread` command needs slippage |
| `market_get_candles` | 20 req/s | Paginate with 100 candles per call for `history` |
| `onchainos dex-market price` | Internal OKX DEX limits | Add 1s delay between calls if throttled |

For `monitor` command with 15s interval and 10 assets: 20 calls per cycle (10 CEX + 10 DEX) — well within limits.

### Multi-Chain DEX Queries

When `--chains` contains multiple chains, query each chain separately for each asset:

```
For ETH with chains=[ethereum, arbitrum, base]:
  1. onchainos dex-market price --chain ethereum --token 0xeee...
  2. onchainos dex-market price --chain arbitrum --token 0xeee...
  3. onchainos dex-market price --chain base --token 0xeee...
```

Each chain produces a separate venue entry: `okx-dex:ethereum`, `okx-dex:arbitrum`, `okx-dex:base`.

---

## 14. Reference Files

| File | Relevance to This Skill |
|------|------------------------|
| `references/mcp-tools.md` | Full okx-trade-mcp API reference (ticker, orderbook, candles) |
| `references/onchainos-tools.md` | OnchainOS CLI commands (dex-market price, dex-token search) |
| `references/chain-reference.md` | Supported chains, native token addresses, instId formats |
| `references/formulas.md` | Spread calculation formula (Section 1) |
| `references/fee-schedule.md` | Not directly used (this skill doesn't calculate costs) |
| `references/output-templates.md` | Header, alert, and next-steps templates |
| `references/safety-checks.md` | Staleness thresholds, error code catalog |
