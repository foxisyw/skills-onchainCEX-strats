# Onchain x CEX Synergy Trading Skills

AI agent skills for identifying and evaluating trading opportunities that arise from the synergies between centralized exchanges (CEX) and decentralized exchanges (DEX).

**Analysis and recommendation only** — these skills do not execute trades automatically.

## What This Is

A modular set of Claude Code / OpenClaw skills that systematically scan for:

- **CEX-DEX Arbitrage** — price differentials between OKX and onchain DEXes
- **Funding Rate Arbitrage** — carry trades exploiting funding rate differentials
- **Basis Trading** — spot-futures premium capture
- **Yield Optimization** — CEX lending vs DeFi yield comparison
- **Smart Money Tracking** — follow onchain whale/fund activity with safety checks

Each skill follows the [OKX OnchainOS](https://github.com/okx/onchainos-skills) SKILL.md standard with full parameter schemas, error codes, cross-skill data contracts, and safety checks.

## Skills

| Skill | Commands | Purpose |
|-------|----------|---------|
| **price-feed-aggregator** | `snapshot` `spread` `monitor` `history` | Unified price data across CEX + DEX venues |
| **profitability-calculator** | `estimate` `breakdown` `min-spread` `sensitivity` | Net P&L after ALL costs (fees, gas, slippage, withdrawal) |
| **cex-dex-arbitrage** | `scan` `evaluate` `monitor` `backtest` | CEX vs DEX price differential detection |
| **funding-rate-arbitrage** | `scan` `evaluate` `rates` `carry-pnl` `unwind-check` | Funding rate carry trade analysis |
| **basis-trading** | `scan` `evaluate` `curve` `track` `roll` | Spot-futures basis capture |
| **yield-optimizer** | `scan` `evaluate` `compare` `rebalance` `rates-snapshot` | CEX vs DeFi yield comparison |
| **smart-money-tracker** | `scan` `track-wallet` `evaluate` `leaderboard` `copy-plan` | Onchain whale/fund activity tracking |
| **liquidation-cascade-monitor** | `scan` `evaluate` `monitor` `cascade-alert` | DeFi liquidation cascade → CEX perp positioning |
| **stablecoin-depeg-arbitrage** | `scan` `evaluate` `monitor` `curve-ratio` | Stablecoin peg deviation CEX↔DEX arbitrage |
| **lp-hedge** | `scan` `evaluate` `hedge-calc` `rebalance-check` | LP yield farming + CEX perp delta-neutral hedge |
| **cross-chain-arbitrage** | `scan` `evaluate` `bridge-compare` `transit-hedge` | Multi-chain price differential with CEX bridge hedge |

## Architecture

```
price-feed-aggregator ─────────────────────────────┐
  │                                                 │
  ├──> cex-dex-arbitrage ──> profitability-calculator│
  ├──> funding-rate-arbitrage ──> profitability-calculator
  ├──> basis-trading ──> profitability-calculator    │
  ├──> yield-optimizer                              │
  ├──> smart-money-tracker                          │
  │                                                 │
  │  V2 Skills (CEX + Onchain Synergy)              │
  ├──> liquidation-cascade-monitor ──> profitability-calculator
  ├──> stablecoin-depeg-arbitrage ──> profitability-calculator
  ├──> lp-hedge ──> funding-rate-arbitrage           │
  └──> cross-chain-arbitrage ──> profitability-calculator
                                                    │
  Output: Recommendations only (no execution)       │
  Data: okx-trade-mcp (CEX) + OnchainOS CLI (DEX)  │
  Security: GoPlus MCP (token safety)  ◄────────────┘
```

## Quick Start

### 1. Install Required Tools

```bash
# OKX CEX trading data
npm install -g okx-trade-mcp

# OKX OnchainOS (onchain DEX data)
npx skills add okx/onchainos-skills
```

### 2. Configure API Credentials

```bash
mkdir -p ~/.okx
cat > ~/.okx/config.toml << 'EOF'
default_profile = "demo"

[profiles.demo]
api_key = "your-demo-api-key"
secret_key = "your-demo-secret-key"
passphrase = "your-demo-passphrase"
demo = true

[profiles.live]
api_key = "your-live-api-key"
secret_key = "your-live-secret-key"
passphrase = "your-live-passphrase"
EOF
```

Get API keys at [OKX API Management](https://www.okx.com/account/my-api).

### 3. Configure MCP Servers

The `.mcp.json` file declares all required MCP servers. You need API keys for:

| Server | Get API Key | Required |
|--------|------------|----------|
| **okx-trade-mcp** | [OKX API](https://www.okx.com/account/my-api) | Yes |
| **OnchainOS CLI** | Same OKX API key | Yes |
| **GoPlus MCP** | [GoPlus Security API](https://gopluslabs.io/security-api) (free) | Yes |
| **CoinGecko MCP** | [CoinGecko API](https://www.coingecko.com/en/api) (free tier) | Optional |
| **DeFiLlama MCP** | No key needed | Optional |

See [`config/mcp-setup-guide.md`](config/mcp-setup-guide.md) for detailed setup instructions.

### 4. Verify

```bash
# Test CEX connection
okx market ticker BTC-USDT

# Test OnchainOS
onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
```

## Safety Architecture

All recommendations pass through 5 safety layers:

| Layer | What It Does | Tool |
|-------|-------------|------|
| **1. Per-Skill Guardrails** | Liquidity, slippage, gas, data staleness thresholds | Built-in |
| **2. Token Security** | Honeypot, rug pull, tax rate, holder concentration | GoPlus MCP |
| **3. Position Limits** | Max size, max exposure, max leverage, daily loss cap | `risk-limits.yaml` |
| **4. Account Safety** | Demo mode by default, explicit opt-in for live | okx-trade-mcp |
| **5. Recommendation Only** | No auto-execution, human approval required | All skills |

Default risk limits (customizable in [`config/risk-limits.example.yaml`](config/risk-limits.example.yaml)):

```yaml
max_position_size_usd: 500
max_total_exposure_usd: 2000
max_leverage: 3
max_loss_per_day_usd: 100
default_mode: "demo"
```

## Security — Not Reinventing the Wheel

Instead of building custom security checks, we delegate to established tools:

| Check | Tool | Cost |
|-------|------|------|
| Honeypot / rug pull / tax rate | [GoPlus MCP](https://github.com/GoPlusSecurity/goplus-mcp) | Free |
| Address risk (AML/KYT) | [MistTrack MCP](https://github.com/slowmist/MistTrackMCP) | Paid |
| Deployer analysis / KOL shill | [Rug Munch MCP](https://github.com/CryptoRugMunch/rug-munch-mcp) | Free tier |
| Sanctions screening | [Chainalysis API](https://www.chainalysis.com/free-cryptocurrency-sanctions-screening-tools/) | Free |
| Contract risk flag | OKX OnchainOS `dex-token info` | Free |

## Project Structure

```
.
├── CLAUDE.md                     # Plugin manifest
├── AGENTS.md                     # Cross-skill orchestration
├── .mcp.json                     # MCP server declarations
│
├── skills/
│   ├── price-feed-aggregator/    # Unified CEX+DEX price data
│   ├── profitability-calculator/ # Net P&L engine
│   ├── cex-dex-arbitrage/        # CEX vs DEX spread detection
│   ├── funding-rate-arbitrage/   # Funding rate carry trades
│   ├── basis-trading/            # Spot-futures basis capture
│   ├── yield-optimizer/          # CEX vs DeFi yield comparison
│   ├── smart-money-tracker/      # Whale/fund activity tracking
│   ├── liquidation-cascade-monitor/ # DeFi cascade → CEX perp positioning
│   ├── stablecoin-depeg-arbitrage/  # Stablecoin depeg CEX↔DEX arb
│   ├── lp-hedge/                    # LP + CEX perp delta-neutral hedge
│   └── cross-chain-arbitrage/       # Cross-chain price diff + bridge hedge
│
├── references/
│   ├── formulas.md               # All calculation formulas
│   ├── fee-schedule.md           # OKX fees + gas benchmarks
│   ├── mcp-tools.md              # okx-trade-mcp reference
│   ├── onchainos-tools.md        # OnchainOS CLI reference
│   ├── goplus-tools.md           # GoPlus security tools
│   ├── safety-checks.md          # Security check procedures
│   ├── chain-reference.md        # Chain IDs, native addresses
│   └── output-templates.md       # Output format templates
│
└── config/
    ├── mcp-setup-guide.md        # MCP server setup guide
    └── risk-limits.example.yaml  # Risk limit configuration
```

## Supported Chains

| Chain | CEX (OKX) | DEX (OnchainOS) | GoPlus |
|-------|-----------|-----------------|--------|
| Ethereum | Yes | Yes | Yes |
| Solana | Yes | Yes | Yes |
| Base | Yes | Yes | Yes |
| Arbitrum | Yes | Yes | Yes |
| BSC | Yes | Yes | Yes |
| Polygon | Yes | Yes | Yes |
| XLayer | Yes | Yes | - |

## Example Output

```
══════════════════════════════════════════
[DEMO] [RECOMMENDATION ONLY — 不會自動執行]
══════════════════════════════════════════

## CEX-DEX 套利機會掃描結果

| # | Asset | CEX Price  | DEX Price  | Spread   | Net Profit | Safety |
|---|-------|-----------|-----------|----------|------------|--------|
| 1 | ETH   | $3,450.12 | $3,467.89 | 51 bps   | +$8.23     | [SAFE] |
| 2 | SOL   | $142.30   | $143.15   | 60 bps   | -$1.20     | [WARN] |

── Cost Breakdown (ETH) ────────────────
| Item               | Amount  | Share  |
|--------------------|---------|--------|
| Gross Spread       | +$17.77 | 100%   |
| CEX Taker Fee      | -$2.76  | 15.5%  |
| DEX Gas            | -$3.50  | 19.7%  |
| Slippage (est.)    | -$1.73  | 9.7%   |
| Withdrawal Fee     | -$1.55  | 8.7%   |
| **Net Profit**     | **+$8.23** | **46.3%** |
```

## Roadmap

### V1
- [x] 7 skills (2 infrastructure + 5 strategy)
- [x] Analysis and recommendation only
- [x] GoPlus security integration
- [x] Multi-chain support
- [x] Comprehensive reference documentation

### V2 (Current)
- [x] Liquidation cascade monitor — DeFi health factor tracking → CEX perp positioning
- [x] Stablecoin depeg arbitrage — peg deviation exploitation with solvency safety checks
- [x] LP hedge — delta-neutral LP yield farming with CEX perp hedge
- [x] Cross-chain arbitrage — multi-chain price differentials with bridge transit hedge

### V3 (Planned)
- [ ] Execution orchestrator skill (trade execution with safety gates)
- [ ] Deposit/withdrawal arbitrage skill
- [ ] Strategy dashboard skill (portfolio-level monitoring)
- [ ] Canvas/A2UI visual dashboard output

## License

Apache-2.0

## Acknowledgments

- [OKX OnchainOS](https://github.com/okx/onchainos-skills) — skill format standard and onchain tools
- [GoPlus Security](https://github.com/GoPlusSecurity/goplus-mcp) — token security checks
- [SlowMist MistTrack](https://github.com/slowmist/MistTrackMCP) — address risk intelligence
- [DeFiLlama](https://defillama.com/) — TVL and yield data
- [CoinGecko](https://www.coingecko.com/) — market data
