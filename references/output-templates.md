# Output Templates Reference

Reusable output format templates for all 7 skills. Each template includes placeholders
marked with `{FIELD_NAME}` that skills replace with real values at runtime.

---

## Formatting Rules

### Monetary Values
- Currency: USD prefix with `$` sign
- Positive/negative: `+$1,234.56` or `-$1,234.56`
- Precision: 2 decimal places
- Thousands separator: comma
- Examples: `$12,345.67`, `+$1,234.56`, `-$89.10`

### Percentages
- Precision: 1 decimal place (e.g., `12.5%`)
- Prefix with `+` or `-` for changes (e.g., `+3.2%`, `-1.8%`)
- APY/APR: 2 decimal places (e.g., `8.75%`)

### Basis Points
- Integer only (e.g., `21 bps`, `145 bps`)
- Prefix with `+` or `-` for changes

### Risk Levels
- `[SAFE]` — green light, proceed
- `[WARN]` — caution, review before proceeding
- `[BLOCK]` — do not proceed, hard stop

### Risk Gauge (Visual)
- Scale: 1-10
- Low risk (1-3): `▓▓▓░░░░░░░ 3/10`
- Medium risk (4-6): `▓▓▓▓▓░░░░░ 5/10`
- High risk (7-10): `▓▓▓▓▓▓▓▓░░ 8/10`

### Sparklines (Trend Visualization)
- Characters: `▁▂▃▄▅▆▇█`
- Use for 8-24 data points showing trend
- Example (rising): `▁▂▃▃▄▅▆▇`
- Example (falling): `▇▆▅▄▃▂▂▁`
- Example (volatile): `▃▇▂▆▁▅▃▇`

### Section Separators
- Major sections: double line
  ```
  ══════════════════════════════════════════
  ```
- Sub-sections: labeled divider
  ```
  ── Section Name ──────────────────────────
  ```
- Minor divisions: single line
  ```
  ──────────────────────────────────────────
  ```

### Timestamps
- Format: `YYYY-MM-DD HH:MM UTC`
- Always show UTC

### Bilingual Safety Notice
- All safety headers include both English and Traditional Chinese

---

## 1. Global Header Template

Used at the top of every skill output.

```
══════════════════════════════════════════
  {SKILL_ICON} {SKILL_NAME}
  [{MODE}] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: {TIMESTAMP}
  Data sources: {DATA_SOURCES}
══════════════════════════════════════════
```

**Field Definitions:**

| Field | Description | Example |
|-------|-------------|---------|
| `{SKILL_ICON}` | Emoji icon for the skill | (see skill list below) |
| `{SKILL_NAME}` | Full skill name | `CEX-DEX Arbitrage Scanner` |
| `{MODE}` | Current mode (SCAN / EVALUATE / MONITOR) | `SCAN` |
| `{TIMESTAMP}` | Current UTC timestamp | `2026-03-09 14:30 UTC` |
| `{DATA_SOURCES}` | APIs used for this output | `OKX REST + Jupiter + Arbiscan` |

**Skill Icons:**
- Skill 1 (CEX-DEX Arb): Crossed swords
- Skill 2 (Funding): Clock
- Skill 3 (Basis): Calendar
- Skill 4 (Order Routing): Router
- Skill 5 (Yield): Seedling
- Skill 6 (LP Hedge): Shield
- Skill 7 (Smart Money): Eye

---

## 2. Opportunity Table Template

Used by scan commands to display discovered opportunities.

```
── Opportunities Found: {COUNT} ────────────────────

  #{RANK}  {PAIR} ({CHAIN})
  ├─ Spread:      {SPREAD_BPS} bps ({SPREAD_PCT}%)
  ├─ Direction:    {DIRECTION}
  ├─ Est. Profit:  {NET_PROFIT}  (after costs)
  ├─ Costs:        {TOTAL_COST}
  ├─ Profit/Cost:  {PROFIT_TO_COST}x
  ├─ Signal Age:   {SIGNAL_AGE}
  ├─ Confidence:   {CONFIDENCE}
  └─ Risk:         {RISK_LEVEL}

  #{RANK}  {PAIR} ({CHAIN})
  ├─ Spread:      {SPREAD_BPS} bps ({SPREAD_PCT}%)
  ...

──────────────────────────────────────────
  Showing top {COUNT} of {TOTAL_SCANNED} pairs scanned
```

**Field Definitions:**

| Field | Description | Example |
|-------|-------------|---------|
| `{COUNT}` | Number of opportunities shown | `3` |
| `{RANK}` | Rank by profitability | `1` |
| `{PAIR}` | Trading pair | `ETH-USDT` |
| `{CHAIN}` | Chain where DEX leg executes | `Arbitrum` |
| `{SPREAD_BPS}` | Gross spread in bps | `21` |
| `{SPREAD_PCT}` | Gross spread as percentage | `0.21` |
| `{DIRECTION}` | Trade direction | `Buy CEX / Sell DEX` |
| `{NET_PROFIT}` | Estimated net profit in USD | `+$55.70` |
| `{TOTAL_COST}` | Total estimated costs | `$51.30` |
| `{PROFIT_TO_COST}` | Profit-to-cost ratio | `1.09` |
| `{SIGNAL_AGE}` | How old the signal is | `< 1 min` |
| `{CONFIDENCE}` | Signal confidence | `HIGH` / `MEDIUM` / `LOW` |
| `{RISK_LEVEL}` | Risk assessment | `[SAFE]` / `[WARN]` / `[BLOCK]` |
| `{TOTAL_SCANNED}` | Total pairs checked | `48` |

---

## 3. Cost Breakdown Template

Used by evaluate commands to show detailed cost analysis.

```
── Cost Breakdown ──────────────────────────

  Trade Size:    {TRADE_SIZE}
  VIP Tier:      {VIP_TIER}

  Leg 1 — {LEG1_VENUE} ({LEG1_SIDE})
  ├─ Trading Fee:    {LEG1_FEE}      ({LEG1_FEE_BPS} bps)
  ├─ Slippage Est:   {LEG1_SLIPPAGE} ({LEG1_SLIP_BPS} bps)
  └─ Subtotal:       {LEG1_TOTAL}

  Leg 2 — {LEG2_VENUE} ({LEG2_SIDE})
  ├─ Trading Fee:    {LEG2_FEE}      ({LEG2_FEE_BPS} bps)
  ├─ Gas Cost:       {LEG2_GAS}
  ├─ Slippage Est:   {LEG2_SLIPPAGE} ({LEG2_SLIP_BPS} bps)
  └─ Subtotal:       {LEG2_TOTAL}

  Transfer Costs
  ├─ Withdrawal Fee: {WITHDRAWAL_FEE}
  └─ Bridge Fee:     {BRIDGE_FEE}

  ──────────────────────────────────────
  Gross Spread:      {GROSS_SPREAD}   ({GROSS_BPS} bps)
  Total Costs:      -{TOTAL_COST}     ({COST_BPS} bps)
  ══════════════════════════════════════
  Net Profit:        {NET_PROFIT}     ({NET_BPS} bps)
  Profit/Cost:       {PROFIT_TO_COST}x
  ══════════════════════════════════════
```

**Field Definitions:**

| Field | Description | Example |
|-------|-------------|---------|
| `{TRADE_SIZE}` | Notional trade size | `$50,000.00` |
| `{VIP_TIER}` | OKX VIP tier used | `VIP1` |
| `{LEG1_VENUE}` | Venue for leg 1 | `OKX` |
| `{LEG1_SIDE}` | Buy or sell | `Buy` |
| `{LEG1_FEE}` | Trading fee for leg 1 | `$35.00` |
| `{LEG1_FEE_BPS}` | Fee in bps | `7.0` |
| `{LEG1_SLIPPAGE}` | Estimated slippage cost | `$15.00` |
| `{LEG1_SLIP_BPS}` | Slippage in bps | `3.0` |
| `{LEG1_TOTAL}` | Total cost for leg 1 | `$50.00` |
| `{LEG2_VENUE}` | Venue for leg 2 | `Jupiter (Solana)` |
| `{LEG2_GAS}` | Gas cost for DEX leg | `$0.01` |
| `{WITHDRAWAL_FEE}` | CEX withdrawal fee | `$1.00` |
| `{BRIDGE_FEE}` | Bridge fee (if cross-chain) | `$0.00` |
| `{GROSS_SPREAD}` | Gross spread in USD | `+$107.00` |
| `{GROSS_BPS}` | Gross spread in bps | `21` |
| `{TOTAL_COST}` | Total costs | `$51.30` |
| `{NET_PROFIT}` | Net profit | `+$55.70` |
| `{PROFIT_TO_COST}` | Profit/cost ratio | `1.09` |

---

## 4. Safety Check Results Template

Used before any trade recommendation to show pre-flight checks.

```
── Safety Checks ───────────────────────────

  [  {CHECK1_STATUS}  ] {CHECK1_NAME}
                        {CHECK1_DETAIL}

  [  {CHECK2_STATUS}  ] {CHECK2_NAME}
                        {CHECK2_DETAIL}

  [  {CHECK3_STATUS}  ] {CHECK3_NAME}
                        {CHECK3_DETAIL}

  [  {CHECK4_STATUS}  ] {CHECK4_NAME}
                        {CHECK4_DETAIL}

  [  {CHECK5_STATUS}  ] {CHECK5_NAME}
                        {CHECK5_DETAIL}

  ──────────────────────────────────────
  Overall: {OVERALL_STATUS}
  {OVERALL_MESSAGE}
```

**Standard Safety Checks:**

| Check Name | What It Verifies |
|------------|-----------------|
| Liquidity Depth | Sufficient liquidity for trade size without excessive slippage |
| Spread Stability | Spread has been consistent, not a flash anomaly |
| Gas Conditions | Current gas prices are within acceptable range |
| Price Freshness | Price data is recent (< 30 seconds old) |
| Position Size | Trade size within configured risk limits |
| Contract Risk | Token contract is verified and not flagged |
| Funding Rate Trend | Funding rate direction and stability |
| Basis Convergence | Basis is moving as expected toward settlement |

**Status Values:**
- `SAFE` — check passed
- `WARN` — check passed with caveats
- `BLOCK` — check failed, do not proceed

**Example (filled):**

```
── Safety Checks ───────────────────────────

  [  SAFE  ] Liquidity Depth
              OKX bid depth $2.3M within 10 bps; Uniswap TVL $45M

  [  SAFE  ] Spread Stability
              21 bps spread stable for last 3 minutes

  [  WARN  ] Gas Conditions
              Ethereum gas at 45 gwei (elevated); consider Arbitrum

  [  SAFE  ] Price Freshness
              Data age: 4 seconds

  [  SAFE  ] Position Size
              $50,000 = 10.0% of NAV (within 20% limit)

  ──────────────────────────────────────
  Overall: SAFE (with advisory)
  Proceed with caution on gas costs. Consider routing via Arbitrum.
```

---

## 5. Risk Gauge Template

Visual risk meter used inline in evaluations.

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

**Gauge Format by Score:**

```
1/10:  ▓░░░░░░░░░
2/10:  ▓▓░░░░░░░░
3/10:  ▓▓▓░░░░░░░
4/10:  ▓▓▓▓░░░░░░
5/10:  ▓▓▓▓▓░░░░░
6/10:  ▓▓▓▓▓▓░░░░
7/10:  ▓▓▓▓▓▓▓░░░
8/10:  ▓▓▓▓▓▓▓▓░░
9/10:  ▓▓▓▓▓▓▓▓▓░
10/10: ▓▓▓▓▓▓▓▓▓▓
```

**Risk Labels:**
- 1-2: `LOW RISK`
- 3-4: `MODERATE-LOW`
- 5-6: `MODERATE`
- 7-8: `ELEVATED`
- 9-10: `HIGH RISK`

**Example (filled):**

```
── Risk Assessment ─────────────────────────

  Overall Risk:  ▓▓▓░░░░░░░  3/10
                 MODERATE-LOW

  Breakdown:
  ├─ Execution Risk:   ▓▓░░░░░░░░  2/10
  │                    Both venues liquid; spread stable
  ├─ Market Risk:      ▓▓▓░░░░░░░  3/10
  │                    ETH vol moderate; 2% move unlikely in execution window
  ├─ Smart Contract:   ▓▓▓▓░░░░░░  4/10
  │                    Uniswap V3 (audited); no exotic contracts
  └─ Liquidity Risk:   ▓▓░░░░░░░░  2/10
                       Deep orderbook; $2.3M within 10 bps
```

---

## 6. Term Structure Template

Used by the Basis Trading skill to show futures term structure.

```
── Basis Term Structure ────────────────────

  {ASSET} Futures vs Spot
  Spot Price: {SPOT_PRICE}

  Contract        Price       Basis     Ann. Yield   DTE
  ─────────────────────────────────────────────────────
  {INST1_ID}     {INST1_PX}  {INST1_B}  {INST1_Y}   {INST1_DTE}
  {INST2_ID}     {INST2_PX}  {INST2_B}  {INST2_Y}   {INST2_DTE}
  {INST3_ID}     {INST3_PX}  {INST3_B}  {INST3_Y}   {INST3_DTE}
  SWAP (perp)    {PERP_PX}   {PERP_B}   N/A         perp

  Term Structure Shape: {SHAPE}
  Curve: {SPARKLINE}

  ── Net Yields (after VIP{TIER} fees) ─────
  {INST1_ID}:  {INST1_NET_Y}  (break-even: {INST1_BE} days)
  {INST2_ID}:  {INST2_NET_Y}  (break-even: {INST2_BE} days)
  {INST3_ID}:  {INST3_NET_Y}  (break-even: {INST3_BE} days)
```

**Field Definitions:**

| Field | Description | Example |
|-------|-------------|---------|
| `{ASSET}` | Underlying asset | `BTC` |
| `{SPOT_PRICE}` | Current spot price | `$87,500.00` |
| `{INST1_ID}` | Instrument ID | `BTC-USD-260327` |
| `{INST1_PX}` | Futures price | `$87,750.00` |
| `{INST1_B}` | Basis percentage | `+0.286%` |
| `{INST1_Y}` | Annualized yield | `5.72%` |
| `{INST1_DTE}` | Days to expiry | `18d` |
| `{SHAPE}` | Curve shape | `CONTANGO` / `BACKWARDATION` / `FLAT` / `INVERTED` |
| `{SPARKLINE}` | Visual of term structure | `▃▅▆▇` |
| `{INST1_NET_Y}` | Net annualized yield | `4.10%` |
| `{INST1_BE}` | Break-even holding days | `3.2` |

**Example (filled):**

```
── Basis Term Structure ────────────────────

  BTC Futures vs Spot
  Spot Price: $87,500.00

  Contract          Price        Basis     Ann. Yield   DTE
  ─────────────────────────────────────────────────────────
  BTC-USD-260327   $87,750.00   +0.286%    5.72%       18d
  BTC-USD-260627   $88,250.00   +0.857%    2.80%      110d
  BTC-USD-260925   $89,100.00   +1.829%    3.30%      200d
  SWAP (perp)      $87,520.00   +0.023%    N/A        perp

  Term Structure Shape: CONTANGO
  Curve: ▃▅▆▇

  ── Net Yields (after VIP1 fees) ──────────
  BTC-USD-260327:  4.10%  (break-even: 3.2 days)
  BTC-USD-260627:  1.88%  (break-even: 4.1 days)
  BTC-USD-260925:  2.56%  (break-even: 3.6 days)
```

---

## 7. Yield Comparison Template

Used by the Yield Optimizer skill to compare opportunities.

```
── Yield Opportunities ─────────────────────

  Current Position: {CURRENT_PROTOCOL} on {CURRENT_CHAIN}
  Current APY: {CURRENT_APY}  |  Risk-Adj: {CURRENT_RADJ}

  #  Protocol         Chain      Raw APY  Risk  Adj APY  Switch Cost  Break-Even
  ────────────────────────────────────────────────────────────────────────────────
  1  {P1_NAME}       {P1_CHAIN}  {P1_APY}  {P1_RS}   {P1_RADJ}   {P1_SC}     {P1_BE}
  2  {P2_NAME}       {P2_CHAIN}  {P2_APY}  {P2_RS}   {P2_RADJ}   {P2_SC}     {P2_BE}
  3  {P3_NAME}       {P3_CHAIN}  {P3_APY}  {P3_RS}   {P3_RADJ}   {P3_SC}     {P3_BE}
  4  {P4_NAME}       {P4_CHAIN}  {P4_APY}  {P4_RS}   {P4_RADJ}   {P4_SC}     {P4_BE}
  5  {P5_NAME}       {P5_CHAIN}  {P5_APY}  {P5_RS}   {P5_RADJ}   {P5_SC}     {P5_BE}

  ── Recommendation ────────────────────────
  {RECOMMENDATION_TEXT}
```

**Field Definitions:**

| Field | Description | Example |
|-------|-------------|---------|
| `{CURRENT_PROTOCOL}` | Where funds are now | `Aave V3 USDC` |
| `{CURRENT_CHAIN}` | Current chain | `Ethereum` |
| `{CURRENT_APY}` | Current yield | `4.20%` |
| `{CURRENT_RADJ}` | Risk-adjusted current yield | `3.78%` |
| `{P1_NAME}` | Protocol name | `Morpho USDC` |
| `{P1_CHAIN}` | Chain | `Base` |
| `{P1_APY}` | Raw APY | `8.50%` |
| `{P1_RS}` | Risk score (1-10) | `3/10` |
| `{P1_RADJ}` | Risk-adjusted APY | `5.95%` |
| `{P1_SC}` | Switching cost | `$22.30` |
| `{P1_BE}` | Break-even period | `4.7d` |

**Example (filled):**

```
── Yield Opportunities ─────────────────────

  Current Position: Aave V3 USDC on Ethereum
  Current APY: 4.20%  |  Risk-Adj: 3.78%

  #  Protocol          Chain       Raw APY  Risk   Adj APY  Switch Cost  Break-Even
  ──────────────────────────────────────────────────────────────────────────────────
  1  Morpho USDC       Base         8.50%   3/10    5.95%     $22.30       4.7d
  2  Aave V3 USDC      Arbitrum     6.10%   2/10    4.88%     $15.80       5.2d
  3  Compound USDC     Ethereum     5.80%   2/10    4.64%      $8.50       9.8d
  4  Spark DAI         Ethereum     5.20%   3/10    3.64%     $12.00      n/a
  5  Venus USDT        BSC          9.20%   5/10    4.60%     $18.50       7.1d

  ── Recommendation ────────────────────────
  Switch to Morpho USDC on Base: +2.17% risk-adjusted edge,
  break-even in 4.7 days. Low gas on Base keeps switching cost minimal.
```

---

## 8. Smart Money Signal Template

Used by the Smart Money Tracker skill.

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

**Field Definitions:**

| Field | Description | Example |
|-------|-------------|---------|
| `{NUM}` | Signal number | `1` |
| `{SIGNAL_TYPE}` | Type of signal | `LARGE ACCUMULATION` / `WHALE DUMP` / `NEW POSITION` |
| `{WALLET_LABEL}` | Known label or shortened address | `Wintermute` |
| `{WALLET_ADDR}` | Wallet address (truncated) | `0x00...3f2a` |
| `{ACTION}` | What was done | `Bought 500 ETH via Uniswap V3` |
| `{ASSET}` | Asset involved | `ETH` |
| `{CHAIN}` | Chain | `Arbitrum` |
| `{SIZE}` | USD value | `$1,706,250` |
| `{TX_HASH}` | Transaction hash (truncated) | `0xabc...def` |
| `{TX_TIME}` | Transaction timestamp | `2026-03-09 14:22 UTC` |
| `{AGE}` | Time since transaction | `8 min` |
| `{SIGNAL_STRENGTH}` | Current strength after decay | `0.72` |
| `{DECAY_NOTE}` | Decay context | `half-life ~35 min` |
| `{CONTEXT}` | Additional context | `3rd large buy from this wallet in 24h` |
| `{NET_FLOW}` | Net buying/selling in USD | `+$4.2M (net buying)` |
| `{WHALE_COUNT}` | Number of whale transactions | `12` |
| `{SPARKLINE}` | Activity trend | `▁▂▃▄▃▅▇█` |
| `{TREND_LABEL}` | Trend description | `accelerating accumulation` |

---

## 9. Next Steps Template

Used at the end of every skill output to guide user action.

```
══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  {STEP_1}
  {STEP_2}
  {STEP_3}

  ── Related Commands ──────────────────────
  {CMD_1}
  {CMD_2}
  {CMD_3}

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Past spreads/yields do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

**Example (filled, after CEX-DEX scan):**

```
══════════════════════════════════════════
  Next Steps
══════════════════════════════════════════

  1. Review the top opportunity (ETH-USDT, Arbitrum, +$55.70)
  2. Run detailed evaluation: /cex-dex evaluate ETH-USDT arb size=50000
  3. If satisfied, manually execute both legs within 2-3 minutes

  ── Related Commands ──────────────────────
  /cex-dex evaluate {PAIR} {CHAIN} size={SIZE}
  /cex-dex monitor {PAIR} {CHAIN} threshold={BPS}
  /funding scan          (check funding rate opportunities)

  ── Disclaimer ────────────────────────────
  This is analysis only. No trades are executed automatically.
  All recommendations require manual review and execution.
  Past spreads/yields do not guarantee future results.
  以上僅為分析建議，不會自動執行任何交易。
  所有建議均需人工審核後手動操作。
══════════════════════════════════════════
```

---

## 10. Funding Rate Display Template

Used by the Funding Harvester skill.

```
── Funding Rate Overview ───────────────────

  {ASSET}-USDT-SWAP
  Current Rate:  {CURRENT_RATE} per 8h  ({ANNUALIZED} ann.)
  Next Payment:  {NEXT_TIME} ({COUNTDOWN})
  Trend (7d):    {SPARKLINE}  ({TREND_LABEL})

  ── Rate History (last 8 payments) ────────
  Time              Rate         Cumulative
  ──────────────────────────────────────────
  {T1}              {R1}         {C1}
  {T2}              {R2}         {C2}
  {T3}              {R3}         {C3}
  {T4}              {R4}         {C4}
  {T5}              {R5}         {C5}
  {T6}              {R6}         {C6}
  {T7}              {R7}         {C7}
  {T8}              {R8}         {C8}

  ── Harvest Projection ────────────────────
  Position Size:   {POSITION_SIZE}
  Daily Income:    {DAILY_INCOME}
  Entry Cost:      {ENTRY_COST}
  Break-Even:      {BREAK_EVEN_DAYS} days
  30-Day Proj:     {THIRTY_DAY_INCOME} (at current rate)
```

**Example (filled):**

```
── Funding Rate Overview ───────────────────

  ETH-USDT-SWAP
  Current Rate:  +0.0150% per 8h  (16.43% ann.)
  Next Payment:  2026-03-09 16:00 UTC (1h 28m)
  Trend (7d):    ▃▄▅▅▆▆▇▇  (rising)

  ── Rate History (last 8 payments) ────────
  Time                  Rate        Cumulative
  ──────────────────────────────────────────────
  Mar 09 08:00 UTC      +0.0150%     +0.0150%
  Mar 09 00:00 UTC      +0.0142%     +0.0292%
  Mar 08 16:00 UTC      +0.0138%     +0.0430%
  Mar 08 08:00 UTC      +0.0125%     +0.0555%
  Mar 08 00:00 UTC      +0.0118%     +0.0673%
  Mar 07 16:00 UTC      +0.0110%     +0.0783%
  Mar 07 08:00 UTC      +0.0098%     +0.0881%
  Mar 07 00:00 UTC      +0.0091%     +0.0972%

  ── Harvest Projection ────────────────────
  Position Size:   $100,000.00
  Daily Income:    +$45.00
  Entry Cost:      $115.00  (spot + perp, VIP1 taker)
  Break-Even:      2.6 days
  30-Day Proj:     +$1,350.00 (at current rate)
```

---

## 11. Monitor Alert Template

Used by continuous monitoring modes across skills.

```
══════════════════════════════════════════
  ALERT: {ALERT_TYPE}
  {TIMESTAMP}
══════════════════════════════════════════

  {ALERT_ICON} {ALERT_HEADLINE}

  ├─ Metric:     {METRIC_NAME}: {METRIC_VALUE}
  ├─ Threshold:  {THRESHOLD}
  ├─ Previous:   {PREV_VALUE} ({CHANGE})
  └─ Action:     {SUGGESTED_ACTION}

  ── Quick Evaluate ────────────────────────
  {EVALUATE_COMMAND}
══════════════════════════════════════════
```

**Alert Types:**
- `SPREAD OPPORTUNITY` — CEX-DEX spread exceeded threshold
- `FUNDING SPIKE` — Funding rate jumped above threshold
- `BASIS WIDENING` — Basis expanded beyond threshold
- `YIELD CHANGE` — Yield opportunity appeared or degraded
- `WHALE MOVEMENT` — Smart money signal detected
- `RISK WARNING` — Risk metric exceeded limit

**Example (filled):**

```
══════════════════════════════════════════
  ALERT: SPREAD OPPORTUNITY
  2026-03-09 14:35 UTC
══════════════════════════════════════════

  ETH-USDT spread on Arbitrum exceeds threshold

  ├─ Metric:     CEX-DEX Spread: 32 bps
  ├─ Threshold:  > 20 bps
  ├─ Previous:   14 bps (+18 bps in 2 min)
  └─ Action:     Evaluate for potential arb trade

  ── Quick Evaluate ────────────────────────
  /cex-dex evaluate ETH-USDT arb size=50000
══════════════════════════════════════════
```

---

## 12. Position Summary Template

Used for showing current open positions across skills.

```
── Active Positions ────────────────────────

  #{NUM} {STRATEGY_TYPE}: {PAIR}
  ├─ Entry:        {ENTRY_DATE}
  ├─ Size:         {POSITION_SIZE}
  ├─ Entry Spread: {ENTRY_SPREAD}
  ├─ Current P&L:  {CURRENT_PNL} ({PNL_PCT})
  ├─ Costs Paid:   {COSTS_PAID}
  ├─ Net P&L:      {NET_PNL}
  ├─ Risk:         {RISK_GAUGE} {RISK_SCORE}/10
  └─ Status:       {POSITION_STATUS}

  ── Portfolio Summary ─────────────────────
  Total Positions:    {TOTAL_POSITIONS}
  Total Exposure:     {TOTAL_EXPOSURE} ({EXPOSURE_PCT} of NAV)
  Aggregate P&L:      {AGG_PNL}
  Risk Utilization:   {RISK_GAUGE_TOTAL} {RISK_UTIL}/10
```

---

## Template Usage Notes

1. **All templates are recommendations** — skills should adapt formatting to fit the specific data available. Not all fields are required for every output.

2. **Consistency is key** — use the same formatting conventions (monetary, percentage, bps) across all skills for a unified user experience.

3. **Bilingual disclaimer** — always include the Chinese translation in the disclaimer section.

4. **Safety header** — the `[RECOMMENDATION ONLY]` header must appear at the top of every output, without exception.

5. **Truncation** — for very long outputs, prioritize the most actionable information and offer a "detailed" command for full data.

6. **Terminal width** — templates are designed for ~60 character width. Adjust separators and alignment for actual terminal width if needed.
