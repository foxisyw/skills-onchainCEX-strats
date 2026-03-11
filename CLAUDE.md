# Onchain x CEX Synergy Trading Skills

Onchain 與中心化交易所 (CEX) 協同交易策略技能包。提供 CEX-DEX 套利、資金費率套利、基差交易、收益優化及聰明錢追蹤等分析與建議功能。

## Core Principles

1. **Analysis only** — 所有 skills 僅提供分析與建議，不會自動執行交易
2. **Security first** — 每筆建議均經過 GoPlus 安全檢查、流動性驗證、成本分析
3. **Demo default** — 預設使用模擬帳戶 (`okx-DEMO-simulated-trading`)，切換真實帳戶需用戶明確確認
4. **No wheel reinvention** — 安全檢查委託 GoPlus MCP / MistTrack MCP，不自建

## Skills

| Skill | Purpose | Key Commands |
|-------|---------|-------------|
| `price-feed-aggregator` | CEX + DEX 統一價格快照、價差偵測 | snapshot, spread, monitor, history |
| `profitability-calculator` | 扣除所有成本後的淨盈利計算 | estimate, breakdown, min-spread, sensitivity |
| `cex-dex-arbitrage` | CEX vs DEX 價差套利機會偵測與評估 | scan, evaluate, monitor, backtest |
| `funding-rate-arbitrage` | 資金費率差異套利 / Carry Trade | scan, evaluate, rates, carry-pnl, unwind-check |
| `basis-trading` | 現貨-期貨基差捕獲 | scan, evaluate, curve, track, roll |
| `yield-optimizer` | CEX 借貸 vs DeFi 收益比較與優化 | scan, evaluate, compare, rebalance, rates-snapshot |
| `smart-money-tracker` | 鏈上聰明錢追蹤與信號評估 | scan, track-wallet, evaluate, leaderboard, copy-plan |
| `liquidation-cascade-monitor` | DeFi 清算級聯偵測 → CEX 永續合約定位 | scan, evaluate, monitor, cascade-alert |
| `stablecoin-depeg-arbitrage` | 穩定幣脫錨套利 (CEX↔DEX) | scan, evaluate, monitor, curve-ratio |
| `lp-hedge` | LP 做市 + CEX 永續合約 Delta 中性對沖 | scan, evaluate, hedge-calc, rebalance-check |
| `cross-chain-arbitrage` | 跨鏈價差套利 + CEX 橋接對沖 | scan, evaluate, bridge-compare, transit-hedge |

## MCP Servers Required

| Server | Purpose | Config |
|--------|---------|--------|
| `okx-trade-mcp` | OKX CEX 數據 | `~/.okx/config.toml` |
| OnchainOS CLI | OKX DEX 數據 | `npx skills add okx/onchainos-skills` |
| GoPlus MCP | 代幣安全檢查 | `GOPLUS_API_KEY` env var |
| CoinGecko MCP | 市場數據備份 | `COINGECKO_API_KEY` env var (optional) |
| DeFiLlama MCP | TVL / 收益數據 | No key needed |

See `config/mcp-setup-guide.md` for step-by-step setup instructions.

## Safety Architecture

All recommendations pass through 5 safety layers:
1. **Per-skill guardrails** — liquidity, slippage, gas, staleness thresholds
2. **Token/contract security** — GoPlus honeypot, rug pull, tax rate checks
3. **Position & loss limits** — from `config/risk-limits.example.yaml`
4. **Account safety** — Demo default, `[DEMO]`/`[LIVE]` headers
5. **Recommendation safety** — `[RECOMMENDATION ONLY]` header, no auto-execution

## Output Format

All skill outputs follow standardized formatting:
- Headers: `[DEMO] [RECOMMENDATION ONLY — 不會自動執行]`
- Monetary: `±$X,XXX.XX` (2 decimals, comma thousands)
- Percentages: `XX.X%` (1 decimal)
- Risk levels: `[SAFE]` `[WARN]` `[BLOCK]`
- Risk gauge: `▓░` blocks (scale 1-10)
- Tables: Markdown pipe format
- See `references/output-templates.md` for full templates

## Language

Match user's language. Default: Traditional Chinese (繁體中文).
Metric labels may use English abbreviations (PnL, APY, bps) regardless of language.

## Reference Files

| File | Contents |
|------|----------|
| `references/formulas.md` | All calculation formulas |
| `references/fee-schedule.md` | OKX fee tiers, gas benchmarks |
| `references/mcp-tools.md` | okx-trade-mcp tool reference |
| `references/onchainos-tools.md` | OnchainOS CLI command reference |
| `references/goplus-tools.md` | GoPlus MCP tool reference |
| `references/safety-checks.md` | Security check procedures |
| `references/chain-reference.md` | Chains, IDs, native addresses |
| `references/output-templates.md` | Output format templates |
