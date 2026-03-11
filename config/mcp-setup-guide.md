# Arb-Only Setup Guide

This bundle has one supported runtime path:

1. configure OKX
2. install OnchainOS
3. optionally add GoPlus
4. run `cex-dex-arbitrage doctor`

## 1. Configure OKX

Install the MCP:

```bash
npm install -g okx-trade-mcp
```

Create `~/.okx/config.toml`:

```toml
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
```

Security notes:

- set file permissions to `600`
- do not commit the file
- use demo first

Verify:

```bash
okx market ticker BTC-USDT
okx-trade-mcp --help
```

The shipped `.mcp.json` already uses `--read-only`.

## 2. Install OnchainOS

```bash
npx skills add okx/onchainos-skills
```

Verify:

```bash
which onchainos
onchainos dex-market price --chain ethereum --token 0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
```

If `onchainos` is missing, `cex-dex-arbitrage doctor` must return `SETUP_BLOCKED`.

## 3. Optional: Add GoPlus

Set an environment variable before launching OpenClaw:

```bash
export GOPLUS_API_KEY="your-goplus-api-key"
```

GoPlus is optional in this bundle:

- native tokens do not need it
- non-native tokens use it when available
- if unavailable, non-native outputs are labeled `UNCHECKED`

## 4. Run Doctor First

Recommended first prompt:

```text
Run cex-dex-arbitrage doctor and tell me if my setup is ready.
```

`doctor` should report:

- OKX connectivity
- current mode (`demo` or `live`)
- OnchainOS health
- GoPlus status
- risk-limit source
- optional balance readiness
- next recommended prompt

## Shipped Runtime Surface

| Component | Status |
|-----------|--------|
| `okx-DEMO-simulated-trading` | required |
| `okx-LIVE-real-money` | optional |
| `goplus-security` | optional |
| OnchainOS CLI | required |

No extra market-data backup MCPs are shipped in this arb-only bundle.
