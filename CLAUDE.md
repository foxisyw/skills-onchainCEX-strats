# Arb-Only CEX-DEX Skills

OpenClaw skill bundle for retail-focused CEX-vs-DEX arbitrage analysis using OKX market data and OnchainOS liquidity data.

## Core Principles

1. **Analysis only** — no trades, transfers, approvals, or signed transactions.
2. **Retail-first clarity** — the flagship flow answers whether the user can execute now.
3. **Read-only by default** — shipped MCP config exposes only read paths.
4. **Conservative assumptions** — no dual-side inventory, no hedge, 10-minute transfer tolerance unless the user says otherwise.
5. **Trust-gated output** — non-native tokens use GoPlus when available; otherwise results are explicitly labeled `UNCHECKED`.

## Shipped Skills

| Skill | Purpose | Key Commands |
|-------|---------|-------------|
| `cex-dex-arbitrage` | Retail orchestrator for readiness, spread scanning, and single-opportunity evaluation | `doctor`, `scan`, `evaluate`, `monitor`, `backtest` |
| `price-feed-aggregator` | Spread detection and synchronized CEX/DEX snapshots | `snapshot`, `spread`, `monitor`, `history` |
| `profitability-calculator` | Cost and breakeven engine for arbitrage scenarios | `estimate`, `breakdown`, `min-spread`, `sensitivity` |
| `non-atomic-execution-risk-model` | Fill risk and worst-case loss modeling | `assess`, `stress` |
| `mev-protected-dex-execution-plan` | Mempool route comparison for EVM DEX execution | `plan`, `compare` |

## Runtime Dependencies

| Dependency | Required | Notes |
|------------|----------|-------|
| `okx-trade-mcp` | Yes | Demo/live entries are read-only |
| OnchainOS CLI | Yes | Mandatory for DEX prices, quotes, gas, liquidity |
| GoPlus MCP | Optional | Needed for non-native token contract-risk checks |

## Default User Journey

1. Run `cex-dex-arbitrage doctor`
2. Run `cex-dex-arbitrage scan`
3. Run `cex-dex-arbitrage evaluate`

## Retail Status Vocabulary

- `SETUP_BLOCKED`
- `READY_NOW`
- `NEEDS_INVENTORY`
- `NEEDS_HEDGE`
- `MARGINAL`
- `DO_NOT_TRADE`

## Output Contract

Every flagship result must show:

- `[DEMO]` or `[LIVE]`
- `[RECOMMENDATION ONLY — 不會自動執行]`
- `Executable now?`
- `Required setup`
- `Estimated completion time`
- `Worst-case loss`
- `Recommendation`

## Documentation

- [README.md](/Users/stevensze/Documents/AI%20%26%20Projects/Tool%20:%20Prototype/CEX%20SKILLS/Onchain%20x%20CEX%20Strats/README.md)
- [config/mcp-setup-guide.md](/Users/stevensze/Documents/AI%20%26%20Projects/Tool%20:%20Prototype/CEX%20SKILLS/Onchain%20x%20CEX%20Strats/config/mcp-setup-guide.md)
- [config/risk-limits.example.yaml](/Users/stevensze/Documents/AI%20%26%20Projects/Tool%20:%20Prototype/CEX%20SKILLS/Onchain%20x%20CEX%20Strats/config/risk-limits.example.yaml)
- [references/output-templates.md](/Users/stevensze/Documents/AI%20%26%20Projects/Tool%20:%20Prototype/CEX%20SKILLS/Onchain%20x%20CEX%20Strats/references/output-templates.md)
