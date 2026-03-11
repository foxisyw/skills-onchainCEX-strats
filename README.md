# Arb-Only CEX-DEX Skill Bundle

Retail-first OpenClaw skills for finding and evaluating CEX-vs-DEX arbitrage between OKX and onchain liquidity venues.

This bundle is **analysis only**. It never submits orders, signs transactions, or moves funds.

## What Ships

| Skill | Role |
|-------|------|
| `cex-dex-arbitrage` | Flagship retail flow: `doctor -> scan -> evaluate -> monitor/backtest` |
| `price-feed-aggregator` | Spread detector and synchronized OKX + DEX price snapshots |
| `profitability-calculator` | Net PnL engine for inventory and hedge-first scenarios |
| `non-atomic-execution-risk-model` | Partial-fill, latency, and worst-case loss analysis |
| `mev-protected-dex-execution-plan` | Public mempool vs private relay route planning |

## Product Goal

The bundle is optimized for one user question:

> Can I execute this arbitrage now, with my current setup and constraints?

That means every flagship result answers:

- Is setup ready?
- Do I already have the required inventory?
- Do I need a perp hedge?
- How long can execution take?
- What is the worst-case loss if the trade becomes non-atomic?

## Quick Start

### 1. Configure OKX

Install the OKX MCP and create `~/.okx/config.toml`.

```bash
npm install -g okx-trade-mcp
```

Use the setup guide in [config/mcp-setup-guide.md](/Users/stevensze/Documents/AI%20%26%20Projects/Tool%20:%20Prototype/CEX%20SKILLS/Onchain%20x%20CEX%20Strats/config/mcp-setup-guide.md).

### 2. Install OnchainOS

```bash
npx skills add okx/onchainos-skills
```

### 3. Optional: Add GoPlus

Set `GOPLUS_API_KEY` if you want contract-risk checks for non-native tokens.

### 4. Start with `doctor`

First prompt:

```text
Run cex-dex-arbitrage doctor and tell me if my setup is ready for retail CEX-DEX arbitrage.
```

Second prompt:

```text
Scan ETH and SOL on Arbitrum, Base, and Solana for CEX-DEX arb. Assume I do not have dual-side inventory and hedge is disabled.
```

Third prompt:

```text
Evaluate ETH on Base at $1,000 and tell me if I can execute now.
```

## Runtime Surface

Shipped `.mcp.json` includes:

| Component | Required | Purpose |
|-----------|----------|---------|
| `okx-DEMO-simulated-trading` | Yes | Read-only CEX prices, balances, perp availability |
| `okx-LIVE-real-money` | Optional | Read-only live account market/balance inspection |
| `goplus-security` | Optional | Non-native token security checks |
| OnchainOS CLI | Yes | DEX prices, token search, quote, gas, liquidity |

The OKX MCP entries are configured as `--read-only`.

## Retail-First Flow

### Step 1: `doctor`

`doctor` checks:

- OKX MCP connectivity and mode
- OnchainOS installation and health
- GoPlus availability
- Risk-limit source
- Optional CEX quote/base balance readiness
- Whether hedge-first mode is even possible

If OnchainOS is missing, the result is `SETUP_BLOCKED` immediately.

### Step 2: `scan`

`scan` ranks opportunities by:

1. readiness state
2. executable net profit

Readiness states:

- `SETUP_BLOCKED`
- `READY_NOW`
- `NEEDS_INVENTORY`
- `NEEDS_HEDGE`
- `MARGINAL`
- `DO_NOT_TRADE`

### Step 3: `evaluate`

`evaluate` always tries to show both frameworks when feasible:

- `inventory_with_rebalance`
- `hedge_first_with_unwind`

Each scenario includes:

- net profit
- total cost
- expected completion time
- worst-case PnL
- required capabilities
- recommendation state

## Execution Profile

`scan` and `evaluate` use a shared retail execution profile.

| Field | Meaning | Default |
|-------|---------|---------|
| `framework` | `auto`, `inventory`, `hedge-first` | `auto` |
| `cex_base_ready` | Already hold base asset on CEX | `false` |
| `cex_quote_ready` | Already hold quote/stablecoin on CEX | `false` |
| `dex_base_ready` | Already hold base asset on DEX wallet | `false` |
| `dex_quote_ready` | Already hold quote/stablecoin on DEX wallet | `false` |
| `hedge_enabled` | User can open temporary perp hedge | `false` |
| `max_transfer_latency_min` | Tolerated transfer/rebalance delay | `10` |

Conservative defaults mean the skill does **not** assume instant inventory or hedge access.

## Trust Rules

- Native tokens bypass contract-risk checks but still pass freshness, liquidity, price-impact, and profitability gates.
- If GoPlus is unavailable for a non-native token, results are labeled `UNCHECKED`.
- `UNCHECKED` opportunities can appear in `scan`, but cannot receive a `READY_NOW` + `PROCEED` recommendation.
- No sample output in this repo should violate `min_net_profit_usd`, `profit_to_cost_ratio`, or timing assumptions.

## Example Result Summary

Every flagship output begins with a retail summary block:

```text
Executable now?     READY_NOW
Required setup      None
Est. completion     2-4 min
Worst-case loss     -$14.20
Recommendation      PROCEED
```

Then the detailed pricing, cost, execution-risk, and MEV route sections follow.

## Repository Structure

```text
.
├── .mcp.json
├── AGENTS.md
├── CLAUDE.md
├── README.md
├── config/
│   ├── mcp-setup-guide.md
│   └── risk-limits.example.yaml
├── references/
│   └── output-templates.md
├── skills/
│   ├── cex-dex-arbitrage/
│   ├── mev-protected-dex-execution-plan/
│   ├── non-atomic-execution-risk-model/
│   ├── price-feed-aggregator/
│   └── profitability-calculator/
└── tasks/
    ├── lessons.md
    └── todo.md
```

## Acceptance Targets

- A new retail user can tell in one screen whether they are blocked by setup, inventory, hedge capability, or economics.
- A user without dual-side inventory is never told that inventory arbitrage is “instant”.
- EVM evaluations compare public mempool vs private relay.
- Solana evaluations return `not_applicable` for private relay routing.

## Non-Goals

- Auto-execution
- Cross-exchange research beyond OKX
- Additional strategy packs outside retail CEX-vs-DEX arbitrage
