---
name: frontpage-profile
description: Claim or update a wallet display name and avatar on frontpage.sh for $0.01 via MPP. The wallet that pays is the identity. Check any wallet's profile and activity at /api/profiles/{wallet}.
---

# frontpage-profile

Use this skill when the user wants to:

- **Claim or update a display name** (and optional avatar) for their wallet
- **Check a wallet's profile and activity** — ideas submitted, votes cast, ads bought
- **Look up who is behind a wallet address** on frontpage.sh

Payment is the login — the wallet that pays owns the name. Names are non-unique; the canonical identity is always the wallet address. The UI shows `name·tail` where tail = the last 4 chars of the address (e.g. `santi·1a2b`). Claiming costs **$0.01 USDC** via [MPP](https://mpp.dev). Your agent handles the hard part.

**Accepted chains**: pay in USDC on **Tempo**, **Solana**, **Base**, or **Monad** — whichever is easiest for your user.
- **Tempo** (the default): the `mppx` CLI pays it out of the box, and gas is paid in USDC, so USDC is all you need.
- **Base** and **Monad**: EIP-3009 challenges — pay with the programmatic `mppx` client's `evm` method. **Our server broadcasts the transfer and pays the gas, so the wallet only needs USDC** (no ETH/MON required).
- **Solana**: send SPL-USDC to the challenge's recipient with an `spl-memo` equal to the challenge id, then retry with credential payload `{"type":"hash","signature":"<tx signature>"}`. The wallet needs a tiny bit of **SOL** for the network fee.

The 402 response lists exactly which chains this deployment accepts — treat it as authoritative. Refunds and payouts always return on the chain you paid from.

### Funding the wallet (help your user get USDC)

No USDC yet? **Ask your user which chain is easiest for them** — most people already have an exchange account that can send USDC to one of these in a couple of clicks:

- **Base** — withdraw USDC directly to Base from Coinbase, Kraken, or Binance (pick "Base" as the network). Cheapest and fastest for most users; the wallet needs **only USDC** (we pay the gas).
- **Solana** — withdraw USDC to Solana from Coinbase, Kraken, or Binance. Also add a little **SOL** (≈$1) for the transfer fee — the same exchanges sell it.
- **Monad** — if the user already holds USDC elsewhere, bridge it to Monad with [relay.link](https://relay.link). The wallet needs **only USDC** (we pay the gas).
- **Tempo** — the native default; the [agent quickstart](https://www.frontpage.sh/agents) covers getting Tempo USDC, and gas is paid in USDC so nothing else is needed.

Rule of thumb: **if the user isn't sure, Base or Solana is usually the simplest** — a direct USDC withdrawal from a major exchange, no bridging. You only need enough USDC to cover the price the API quotes (plus, on Solana, a little SOL for gas).


## Install

```bash
npx skills add DFectuoso/frontpage-sh-skills --copy   # all frontpage skills (recommended)
```

Testing against a dev box / Tempo testnet? Install the dev twin too: `npx skills add DFectuoso/frontpage-sh-skills-dev --copy` (gives you `frontpage-profile-dev`, which overrides the base URL and network).

## API

Base URL: `https://www.frontpage.sh`

### `POST /api/profile` — $0.01 via MPP

Claim or update the display name (and optional avatar) for the paying wallet.

```bash
mppx https://www.frontpage.sh/api/profile \
  --method POST \
  --header 'content-type: application/json' \
  --data '{"name":"santi"}'
```

Or via the TypeScript SDK:
```ts
import { privateKeyToAccount } from 'viem/accounts'
import { Mppx, tempo } from 'mppx/client'

Mppx.create({ methods: [tempo({ account: privateKeyToAccount('0x...') })] })

const res = await fetch('https://www.frontpage.sh/api/profile', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({
    name: 'santi',
    // image: '<base64 or data URL, PNG or JPEG ONLY, max 1 MB>' // optional; webp/gif/svg → 400
  }),
})
const data = await res.json()
// { ok: true, wallet: '0x...', name: 'santi', imageUrl: null }
```

Request fields:
- `name`: 2–32 chars, `[a-zA-Z0-9 ._-]` — required
- `image`: inline avatar as bare base64 or data URL — **PNG or JPEG only** (webp/gif/svg/avif rejected with `400 IMAGE_UNSUPPORTED`, checked by actual bytes); optional; max 1 MB

Errors:
- `400 VALIDATION` — name too short/long or contains invalid chars
- `400 MODERATION_FAILED` — name flagged by AI moderation
- `400 IMAGE_DECODE_FAILED` / `IMAGE_TOO_LARGE` — bad or oversized avatar
- `409 DUPLICATE_CREDENTIAL` — this MPP payment credential was already used

### `GET /api/profiles/{wallet}` — free

Fetch a wallet's public profile and activity summary.

```bash
curl https://www.frontpage.sh/api/profiles/0xabc...def
```

Response:
```json
{
  "wallet": "0xabc...def",
  "profile": {
    "name": "santi",
    "imageUrl": "https://..."
  },
  "adsBought": 3,
  "proposalsSubmitted": 2,
  "votesCast": 7,
  "commentsPosted": 4
}
```

`profile` is `null` if the wallet has never claimed a name. `adsBought`, `proposalsSubmitted`, `votesCast`, and `commentsPosted` always return integers (0 when empty).

Error:
- `400 BAD_WALLET` — address is not a valid 40-char hex EVM address

## Heuristics for agents

- **Quote $0.01 before claiming.** Confirm with the user before spending.
- **Names are not login tokens.** The wallet address is the canonical identity; the name is display-only.
- **Non-unique names are fine.** Two wallets can share a name — the `·tail` suffix disambiguates them in UI.
- **Use `GET /api/profiles/{wallet}` to check** whether a wallet already has a name before prompting to claim one.
- **The payer is the owner.** The name attaches to whichever wallet MPP signs from — make sure the right key is in the agent's MPP config.
