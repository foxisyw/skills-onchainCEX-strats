---
name: mev-protected-dex-execution-plan
description: >
  Compares public mempool and private relay submission paths for EVM-chain DEX execution
  in CEX-DEX arbitrage. Use to describe sandwich risk, expected route cost, and submission
  guidance. Returns not_applicable on chains where the private-relay framing does not fit,
  including Solana. Analysis only.
allowed-tools: >
  okx-DEMO-simulated-trading:system_get_capabilities,
  okx-LIVE-real-money:system_get_capabilities
---

# mev-protected-dex-execution-plan

## Role

This skill answers:

> If I execute the onchain leg now, should I use the public mempool or a private relay path?

## Input Contract

```yaml
MevRouteInput:
  chain: string
  asset: string
  size_usd: number
  dex_price_impact_pct: number
  framework: string
```

## Output Contract

```yaml
MevRoutePlan:
  route: string                   # "public" | "private_relay" | "not_applicable"
  sandwich_risk: string           # "low" | "medium" | "high" | "not_applicable"
  expected_success_rate: number
  expected_extra_cost_usd: number
  submission_guidance: string
```

## Commands

### `plan`

Use for the default recommendation.

```bash
mev-protected-dex-execution-plan plan --chain base --asset ETH --size-usd 1000
```

Rules:

- EVM chains: prefer `private_relay` when sandwich risk is medium or high.
- Solana: return `not_applicable`.
- If price impact is negligible and the route is liquid, `public` may be acceptable on EVM.

### `compare`

Use when the caller wants both options side by side.

```bash
mev-protected-dex-execution-plan compare --chain arbitrum --asset ETH --size-usd 1000
```

Output:

```yaml
MevCompareResult:
  public:
    route: string
    sandwich_risk: string
    expected_success_rate: number
    expected_extra_cost_usd: number
  private_relay:
    route: string
    sandwich_risk: string
    expected_success_rate: number
    expected_extra_cost_usd: number
```

## Calling Rules

- `cex-dex-arbitrage.evaluate` should show a MEV block for every EVM scenario.
- Solana scenarios must say `not_applicable`, not pretend Flashbots-style relay logic exists.
