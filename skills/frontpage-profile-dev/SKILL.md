---
name: frontpage-profile-dev
description: Dev/testnet variant of frontpage-profile — claim wallet display names on a frontpage.sh dev instance with Tempo Moderato testnet USDC. Use ONLY when the user explicitly says dev, testnet, staging, or localhost.
---

# frontpage-profile-dev

Dev/testnet variant of `frontpage-profile`. That skill carries the full flow — endpoints, name rules, avatar limits, errors — and everything there applies here, with these overrides:

## Overrides

- **Base URL**: `$FRONTPAGE_BASE_URL` if set, otherwise `http://localhost:3000`. Never `https://frontpage.sh`.
- **Network**: Tempo **Moderato testnet** (chain id `42431`). Pin mppx to it: `export MPPX_RPC_URL=https://rpc.moderato.tempo.xyz`.
- **Money**: testnet USDC only — top up from the Tempo testnet faucet.
- **Keys**: use a throwaway dev wallet. Never point a mainnet key at a dev instance.

## Worked example

```bash
export FRONTPAGE_BASE_URL=http://localhost:3000
export MPPX_RPC_URL=https://rpc.moderato.tempo.xyz

# claim a name for the paying dev wallet ($0.01 testnet)
mppx $FRONTPAGE_BASE_URL/api/profile --json-body '{"name":"smoke tester"}'
```

## When to use

Only when the user explicitly asks for dev / testnet / staging / localhost. If in doubt, use `frontpage-profile` (production).
