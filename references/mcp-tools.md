# okx-trade-mcp Tool Reference

MCP server for OKX CEX market data, account info, and trade operations.
All tools are invoked via the MCP protocol with `server: okx-trade-mcp`.

> **Important:** All OKX API values are returned as **strings** — always parse to numbers before computation.
> Data retention: **3 months max** for most endpoints.
> Demo mode: all tools work identically but return simulated data.

---

## Table of Contents

1. [system_get_capabilities](#1-system_get_capabilities)
2. [market_get_ticker](#2-market_get_ticker)
3. [market_get_orderbook](#3-market_get_orderbook)
4. [market_get_candles](#4-market_get_candles)
5. [market_get_funding_rate](#5-market_get_funding_rate)
6. [market_get_instruments](#6-market_get_instruments)
7. [account_get_positions_history](#7-account_get_positions_history)
8. [account_get_bills](#8-account_get_bills)
9. [Pagination Pattern](#pagination-pattern)
10. [Rate Limits](#rate-limits)

---

## 1. system_get_capabilities

**Purpose:** Health check — verifies MCP server connectivity, authentication status, and available modules.

### Parameters

_None._

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| authenticated | boolean | Whether OKX API keys are valid and connected |
| mode | string | `"demo"` or `"live"` — current trading environment |
| modules | string[] | List of available modules (e.g. `["market", "account", "trade"]`) |
| serverVersion | string | MCP server version string |
| timestamp | string | Server time in ISO-8601 format |

### Rate Limit

No explicit rate limit. Use sparingly (once per session or on error recovery).

### Example

**Call:**
```json
{
  "tool": "system_get_capabilities",
  "params": {}
}
```

**Response:**
```json
{
  "authenticated": true,
  "mode": "demo",
  "modules": ["market", "account", "trade"],
  "serverVersion": "1.2.0",
  "timestamp": "2026-03-09T12:00:00.000Z"
}
```

---

## 2. market_get_ticker

**Purpose:** Retrieve real-time ticker data for a single instrument — last price, bid/ask, 24h volume and price range.

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | — | string | Instrument ID (e.g. `"BTC-USDT"`, `"ETH-USDT-SWAP"`) |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| last | string | Last traded price |
| bidPx | string | Best bid price |
| askPx | string | Best ask price |
| open24h | string | Opening price 24 hours ago |
| high24h | string | 24-hour high |
| low24h | string | 24-hour low |
| vol24h | string | 24-hour volume in base currency |
| ts | string | Data timestamp in milliseconds since epoch |

### Rate Limit

20 requests/second.

### Example

**Call:**
```json
{
  "tool": "market_get_ticker",
  "params": {
    "instId": "BTC-USDT"
  }
}
```

**Response:**
```json
{
  "last": "87234.5",
  "bidPx": "87234.1",
  "askPx": "87234.9",
  "open24h": "86100.0",
  "high24h": "88000.0",
  "low24h": "85900.0",
  "vol24h": "12345.6789",
  "ts": "1741521600000"
}
```

---

## 3. market_get_orderbook

**Purpose:** Retrieve order book depth (bids and asks) for a given instrument.

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | — | string | Instrument ID (e.g. `"BTC-USDT"`) |
| sz | No | `5` | string | Book depth — number of price levels per side. Max `400`. |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| asks | array | Array of ask levels, each `[price, size, num_orders]` (strings) |
| bids | array | Array of bid levels, each `[price, size, num_orders]` (strings) |
| ts | string | Snapshot timestamp in milliseconds since epoch |

Each level element:

| Index | Type | Description |
|-------|------|-------------|
| [0] | string | Price |
| [1] | string | Size (quantity at that price) |
| [2] | string | Number of orders at that price level |

### Rate Limit

20 requests/second.

### Example

**Call:**
```json
{
  "tool": "market_get_orderbook",
  "params": {
    "instId": "ETH-USDT",
    "sz": "3"
  }
}
```

**Response:**
```json
{
  "asks": [
    ["3456.78", "12.5", "3"],
    ["3456.90", "8.2", "2"],
    ["3457.10", "25.0", "5"]
  ],
  "bids": [
    ["3456.50", "10.0", "4"],
    ["3456.30", "15.3", "6"],
    ["3456.10", "7.8", "2"]
  ],
  "ts": "1741521600000"
}
```

---

## 4. market_get_candles

**Purpose:** Retrieve OHLCV candlestick data for technical analysis and historical price tracking.

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | — | string | Instrument ID (e.g. `"BTC-USDT"`) |
| bar | No | `"1m"` | string | Candlestick interval. Enum: `1m`, `5m`, `15m`, `30m`, `1H`, `4H`, `1D`, `1W` |
| after | No | — | string | Pagination cursor — timestamp in ms. Returns data **before** this time. |
| before | No | — | string | Pagination cursor — timestamp in ms. Returns data **after** this time. |
| limit | No | `100` | string | Number of candles to return. Max `100`. |

### Return Fields

Each candle is an array with the following positional elements:

| Index | Field | Type | Description |
|-------|-------|------|-------------|
| [0] | ts | string | Candle open timestamp in ms since epoch |
| [1] | open | string | Open price |
| [2] | high | string | High price |
| [3] | low | string | Low price |
| [4] | close | string | Close price |
| [5] | vol | string | Volume in contracts (for derivatives) or base currency (for spot) |
| [6] | volCcy | string | Volume in currency |
| [7] | volCcyQuote | string | Volume in quote currency |
| [8] | confirm | string | `"0"` = incomplete candle, `"1"` = confirmed/closed |

### Rate Limit

20 requests/second.

### Example

**Call:**
```json
{
  "tool": "market_get_candles",
  "params": {
    "instId": "BTC-USDT",
    "bar": "1H",
    "limit": "3"
  }
}
```

**Response:**
```json
[
  ["1741510800000", "87000.0", "87500.0", "86800.0", "87234.5", "1234.56", "1234.56", "107654321.00", "1"],
  ["1741507200000", "86500.0", "87100.0", "86400.0", "87000.0", "1100.23", "1100.23", "95612345.00", "1"],
  ["1741503600000", "86800.0", "86900.0", "86200.0", "86500.0", "980.45", "980.45", "84812345.00", "1"]
]
```

---

## 5. market_get_funding_rate

**Purpose:** Retrieve the current and predicted funding rate for perpetual swap instruments.

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instId | Yes | — | string | Perpetual swap instrument ID. **Must** end in `-SWAP` (e.g. `"BTC-USDT-SWAP"`). |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| fundingRate | string | Current period funding rate (e.g. `"0.0001"` = 0.01%) |
| realizedRate | string | Last settled (realized) funding rate |
| fundingTime | string | Next funding settlement time in ms since epoch |
| nextFundingRate | string | Predicted next funding rate (may be empty) |
| instId | string | Instrument ID echo |

### Rate Limit

10 requests/second.

### Example

**Call:**
```json
{
  "tool": "market_get_funding_rate",
  "params": {
    "instId": "BTC-USDT-SWAP"
  }
}
```

**Response:**
```json
{
  "fundingRate": "0.00015",
  "realizedRate": "0.00012",
  "fundingTime": "1741536000000",
  "nextFundingRate": "0.00018",
  "instId": "BTC-USDT-SWAP"
}
```

---

## 6. market_get_instruments

**Purpose:** List available trading instruments with contract specifications and trading rules.

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instType | Yes | — | string | Instrument type. Enum: `SPOT`, `SWAP`, `FUTURES`, `OPTION` |
| instId | No | — | string | Filter by specific instrument ID |
| uly | No | — | string | Filter by underlying (e.g. `"BTC-USDT"`) — applies to derivatives only |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| instId | string | Instrument ID (e.g. `"BTC-USDT"`, `"BTC-USDT-SWAP"`) |
| instType | string | Instrument type |
| uly | string | Underlying index (derivatives only) |
| settleCcy | string | Settlement currency |
| ctVal | string | Contract value (face value of one contract) |
| ctMult | string | Contract multiplier |
| listTime | string | Listing time in ms since epoch |
| expTime | string | Expiry time in ms since epoch (futures/options only; `""` for perpetuals) |
| lever | string | Maximum leverage available |
| tickSz | string | Tick size (minimum price increment) |
| lotSz | string | Lot size (minimum order quantity increment) |
| minSz | string | Minimum order size |

### Rate Limit

20 requests/second.

### Example

**Call:**
```json
{
  "tool": "market_get_instruments",
  "params": {
    "instType": "SWAP",
    "uly": "BTC-USDT"
  }
}
```

**Response:**
```json
[
  {
    "instId": "BTC-USDT-SWAP",
    "instType": "SWAP",
    "uly": "BTC-USDT",
    "settleCcy": "USDT",
    "ctVal": "0.01",
    "ctMult": "1",
    "listTime": "1611100800000",
    "expTime": "",
    "lever": "125",
    "tickSz": "0.1",
    "lotSz": "1",
    "minSz": "1"
  }
]
```

---

## 7. account_get_positions_history

**Purpose:** Retrieve closed position history with realized P&L, fees, and funding costs.

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instType | No | — | string | Filter by instrument type: `SPOT`, `SWAP`, `FUTURES`, `OPTION` |
| instId | No | — | string | Filter by instrument ID |
| mgnMode | No | — | string | Margin mode: `cross` or `isolated` |
| type | No | — | string | Close type filter: `1`=close position, `2`=partially closed, `3`=liquidation, `4`=partial liquidation, `5`=ADL, `6`=tp/sl |
| posId | No | — | string | Filter by position ID |
| after | No | — | string | Pagination: return records after this position ID |
| before | No | — | string | Pagination: return records before this position ID |
| limit | No | `100` | string | Number of results. Max `100`. |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| posId | string | Position ID |
| instId | string | Instrument ID |
| direction | string | `"long"` or `"short"` |
| lever | string | Leverage used |
| pnl | string | Total P&L (realized + unrealized at close) |
| realizedPnl | string | Realized P&L component |
| pnlRatio | string | P&L ratio (e.g. `"0.05"` = 5%) |
| fee | string | Trading fees paid (negative value) |
| fundingFee | string | Cumulative funding fees (positive = received, negative = paid) |
| liqPenalty | string | Liquidation penalty (if applicable) |
| openAvgPx | string | Average entry price |
| closeAvgPx | string | Average exit price |
| cTime | string | Position creation time in ms since epoch |
| uTime | string | Position last update time in ms since epoch |

### Rate Limit

1 request/second. **Use pagination wisely — this endpoint is heavily rate-limited.**

### Example

**Call:**
```json
{
  "tool": "account_get_positions_history",
  "params": {
    "instType": "SWAP",
    "limit": "5"
  }
}
```

**Response:**
```json
[
  {
    "posId": "1234567890",
    "instId": "BTC-USDT-SWAP",
    "direction": "long",
    "lever": "5",
    "pnl": "125.50",
    "realizedPnl": "125.50",
    "pnlRatio": "0.025",
    "fee": "-12.30",
    "fundingFee": "-3.20",
    "liqPenalty": "0",
    "openAvgPx": "86500.0",
    "closeAvgPx": "87000.0",
    "cTime": "1741435200000",
    "uTime": "1741521600000"
  }
]
```

---

## 8. account_get_bills

**Purpose:** Retrieve the financial ledger — detailed transaction history including trades, funding, liquidations, and transfers.

### Parameters

| Param | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| instType | No | — | string | Filter by instrument type: `SPOT`, `SWAP`, `FUTURES`, `OPTION` |
| ccy | No | — | string | Filter by currency (e.g. `"USDT"`, `"BTC"`) |
| mgnMode | No | — | string | Margin mode: `cross` or `isolated` |
| type | No | — | string | Bill type: `2`=trade, `5`=liquidation, `8`=funding fee, `1`=transfer |
| after | No | — | string | Pagination: return records after this bill ID |
| before | No | — | string | Pagination: return records before this bill ID |
| begin | No | — | string | Start time filter in ms since epoch |
| end | No | — | string | End time filter in ms since epoch |
| limit | No | `100` | string | Number of results. Max `100`. |

### Return Fields

| Field | Type | Description |
|-------|------|-------------|
| billId | string | Unique bill/ledger entry ID |
| type | string | Bill type code (see `type` param above) |
| subType | string | Sub-type for finer categorization |
| balChg | string | Balance change from this entry |
| bal | string | Balance after this entry |
| sz | string | Size/quantity involved |
| px | string | Price at time of transaction |
| pnl | string | Realized P&L from this entry |
| fee | string | Fee charged (negative value) |
| ts | string | Timestamp in ms since epoch |

### Rate Limit

5 requests/second.

### Example

**Call:**
```json
{
  "tool": "account_get_bills",
  "params": {
    "type": "8",
    "instType": "SWAP",
    "limit": "3"
  }
}
```

**Response:**
```json
[
  {
    "billId": "9876543210",
    "type": "8",
    "subType": "173",
    "balChg": "-0.85",
    "bal": "10234.50",
    "sz": "10",
    "px": "87000.0",
    "pnl": "0",
    "fee": "0",
    "ts": "1741521600000"
  }
]
```

---

## Pagination Pattern

Most list endpoints support cursor-based pagination using the `after` parameter.

```
Call 1: tool(limit: 100)
  → Process results
  → Note last item's ID (posId, billId, or ts depending on endpoint)

Call 2: tool(limit: 100, after: <last_id>)
  → Process results
  → Note last item's ID

Call N: Continue until results.length < limit

Cap at 10 pages (1,000 items) to prevent runaway loops.
```

**Pagination fields per tool:**

| Tool | Cursor Field |
|------|-------------|
| account_get_positions_history | `posId` |
| account_get_bills | `billId` |
| market_get_candles | `ts` (timestamp of last candle) |

---

## Rate Limits

| Tool | Rate Limit | Notes |
|------|-----------|-------|
| market_get_ticker | 20 req/s | Per instrument |
| market_get_orderbook | 20 req/s | Per instrument |
| market_get_candles | 20 req/s | Per instrument + bar combo |
| market_get_funding_rate | 10 req/s | Per instrument |
| market_get_instruments | 20 req/s | Per instType |
| account_get_positions_history | 1 req/s | Heavily throttled — plan pagination carefully |
| account_get_bills | 5 req/s | — |

**Rate limit handling:**
- On HTTP 429 or rate-limit error, wait 1 second before retrying.
- Max 3 retries per call, then surface the error to the user.
- For `account_get_positions_history`, add a 1-second delay between paginated calls.

---

## Important Notes

1. **String values** — Every numeric value from OKX is a string. Always use `parseFloat()` or equivalent before arithmetic.
2. **Timestamps** — All timestamps are in **milliseconds** since Unix epoch. Convert with `new Date(parseInt(ts))`.
3. **Data retention** — Historical data is available for ~3 months. Requests beyond this window return empty results.
4. **Demo mode** — When `system_get_capabilities` returns `mode: "demo"`, all data is simulated. Tools behave identically but prices/positions are not real.
5. **Instrument ID format:**
   - Spot: `BTC-USDT`, `ETH-USDT`
   - Perpetual swap: `BTC-USDT-SWAP`
   - Futures: `BTC-USDT-250328` (expiry date suffix)
   - Options: `BTC-USDT-250328-90000-C` (expiry-strike-type)
6. **Error responses** — On failure, the MCP server returns `{ "error": { "code": "...", "message": "..." } }`. Always check for the `error` field before processing results.
