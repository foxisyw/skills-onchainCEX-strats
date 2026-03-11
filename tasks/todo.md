# Arb-Only CEX-DEX Skill Bundle Refactor

## Plan
- [x] Review the existing skill bundle structure, current CEX/DEX dependencies, and retail-user gaps.
- [x] Remove non-arbitrage skills and update the shipped runtime surface to arb-only dependencies.
- [x] Add task-safe manifests and retail-first setup/read-only guidance.
- [x] Rewrite `cex-dex-arbitrage` around `doctor -> scan -> evaluate`.
- [x] Add `non-atomic-execution-risk-model` and `mev-protected-dex-execution-plan`.
- [x] Reposition `price-feed-aggregator` and `profitability-calculator` as arb helpers with updated contracts.
- [x] Rewrite README, CLAUDE, AGENTS, setup guide, risk limits, and output templates to match the new bundle.
- [x] Replace inconsistent examples with threshold-safe, retail-readable outputs.
- [x] Verify repo consistency and acceptance conditions.

## Review
- [x] Remove non-arbitrage skills and update the shipped runtime surface to arb-only dependencies.
- [x] Add task-safe manifests and retail-first setup/read-only guidance.
- [x] Rewrite `cex-dex-arbitrage` around `doctor -> scan -> evaluate`.
- [x] Add `non-atomic-execution-risk-model` and `mev-protected-dex-execution-plan`.
- [x] Reposition `price-feed-aggregator` and `profitability-calculator` as arb helpers with updated contracts.
- [x] Rewrite README, CLAUDE, AGENTS, setup guide, risk limits, and output templates to match the new bundle.
- [x] Replace inconsistent examples with threshold-safe, retail-readable outputs.
- [x] Verify repo consistency and acceptance conditions.

Verification notes:
- `.mcp.json` parses as valid JSON.
- `config/risk-limits.example.yaml` parses as valid YAML.
- `find skills -maxdepth 2 -name SKILL.md | wc -l` returns `5`.
- Removed strategy IDs no longer appear in shipped docs or manifests.
