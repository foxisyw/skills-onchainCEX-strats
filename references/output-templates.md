# Arb-Only Output Templates

Reusable templates for the retail-first CEX-DEX bundle.

## Global Header

```text
══════════════════════════════════════════
  {TITLE}
  [{MODE}] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════
  Generated: {TIMESTAMP}
  Data sources: {DATA_SOURCES}
══════════════════════════════════════════
```

## Retail Summary

Always show this block before detailed tables.

```text
── Retail Summary ────────────────────────

  Executable now?     {READINESS_STATUS}
  Required setup      {REQUIRED_SETUP}
  Est. completion     {TIME_TO_COMPLETE}
  Worst-case loss     {WORST_CASE_LOSS}
  Recommendation      {RECOMMENDATION}
```

## Doctor Summary

```text
── Doctor Result ─────────────────────────

  Setup status        {SETUP_STATUS}
  OKX                 {OKX_STATUS}
  OnchainOS           {ONCHAINOS_STATUS}
  GoPlus              {GOPLUS_STATUS}
  Risk limits         {RISK_LIMIT_SOURCE}
  Available modes     {AVAILABLE_MODES}
  Next prompt         {RECOMMENDED_NEXT_PROMPT}
```

## Opportunity Row

```text
  #{RANK}  {ASSET} ({CHAIN})
  ├─ Readiness:     {READINESS_STATUS}
  ├─ Direction:     {DIRECTION}
  ├─ Net Profit:    {NET_PROFIT}
  ├─ Worst Case:    {WORST_CASE_LOSS}
  ├─ Framework:     {RECOMMENDED_FRAMEWORK}
  ├─ MEV Route:     {MEV_ROUTE}
  └─ Trust:         {TRUST_LABEL}
```

## Evaluate Scenario Block

```text
── {SCENARIO_NAME} ───────────────────────

  Readiness          {READINESS_STATUS}
  Required capability {REQUIRED_CAPABILITIES}
  Net profit         {NET_PROFIT}
  Total costs        {TOTAL_COSTS}
  Est. completion    {TIME_TO_COMPLETE}
  Worst-case PnL     {WORST_CASE_PNL}
  Recommendation     {RECOMMENDATION}
```

## Execution Risk Block

```text
── Non-Atomic Execution Risk ─────────────

  Fill probability     {FILL_PROBABILITY}
  Partial-fill loss    {PARTIAL_FILL_LOSS}
  Worst-case PnL       {WORST_CASE_PNL}
  Latency window       {LATENCY_WINDOW}
  Risk verdict         {RISK_VERDICT}
```

## MEV Route Block

```text
── MEV Route ─────────────────────────────

  Route                {ROUTE}
  Sandwich risk        {SANDWICH_RISK}
  Success rate         {SUCCESS_RATE}
  Extra cost           {EXTRA_COST}
  Submission guidance  {SUBMISSION_GUIDANCE}
```

## Trust Labels

- `[SAFE]`
- `[WARN]`
- `[UNCHECKED]`
- `[BLOCK]`

## Readiness States

- `SETUP_BLOCKED`
- `READY_NOW`
- `NEEDS_INVENTORY`
- `NEEDS_HEDGE`
- `MARGINAL`
- `DO_NOT_TRADE`
