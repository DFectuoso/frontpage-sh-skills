---
name: frontpage-buy-ad-dev
description: Dev/testnet variant of frontpage-buy-ad — buy ad squares on a frontpage.sh dev instance with Tempo Moderato testnet USDC. Use ONLY when the user explicitly says dev, testnet, staging, or localhost.
---

# frontpage-buy-ad-dev

Dev/testnet variant of `frontpage-buy-ad`. That skill carries the full flow — endpoints, payload shape, pricing, image specs, errors — and everything there applies here, with these overrides:

## Overrides

- **Base URL**: `$FRONTPAGE_BASE_URL` if set, otherwise `http://localhost:3000`. Never `https://frontpage.sh`.
- **Network**: Tempo **Moderato testnet** (chain id `42431`). Pin mppx to it: `export MPPX_RPC_URL=https://rpc.moderato.tempo.xyz` (per-call `--network testnet` also works).
- **Money**: testnet USDC only — top up from the Tempo testnet faucet. Prices are real in shape, worthless in value.
- **Keys**: use a throwaway dev wallet. Never point a mainnet key at a dev instance.

## Worked example

```bash
export FRONTPAGE_BASE_URL=http://localhost:3000
export MPPX_RPC_URL=https://rpc.moderato.tempo.xyz

# 1) preview ($0.10 testnet)
mppx $FRONTPAGE_BASE_URL/api/preview --json-body '{
  "slot":"S1","name":"smoke test","url":"https://example.test",
  "monogram":"ST","logoColor":"#ffffff","logoBg":"#0A0A09",
  "adBg":"#1F1F1F","adHeadline":"hello\ntestnet.","ownerHandle":"smoke"
}'

# 2) settle at nextPrice
mppx $FRONTPAGE_BASE_URL/api/buy --json-body '{"previewToken":"<token from step 1>"}'
```

## When to use

Only when the user explicitly asks for dev / testnet / staging / localhost. If in doubt, use `frontpage-buy-ad` (production).
