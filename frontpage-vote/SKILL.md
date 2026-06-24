---
name: frontpage-vote
description: Vote on or submit ideas for the frontpage.sh project pool. $0.01 USDC per action via MPP. Funded ideas pay back the suggester (50%) and voters (50% pro-rata).
---

# frontpage-vote

Use this skill when the user wants to:

- **Vote on an idea** at https://www.frontpage.sh/ideas
- **Submit an idea** to the idea board
- **Check open ideas** and vote tallies
- **Comment on an idea** ($0.01, up to 140 chars)

Every action costs **$0.01 USDC** paid via the [Machine Payments Protocol](https://mpp.dev). The payment goes into the project pool. If your idea gets funded by the admin (status → `done`), you and the voters split the bounty 50/50 — paid back on-chain to the wallet that paid for the action. Your agent handles the hard part.

## Install

```bash
npx skills add DFectuoso/frontpage-sh-skills --copy                  # all frontpage skills
npx skills add DFectuoso/frontpage-sh-skills/frontpage-vote --copy   # just this one
```

Testing against a dev box / Tempo testnet? Install the dev twin too: `npx skills add DFectuoso/frontpage-sh-skills-dev --copy` (gives you `frontpage-vote-dev`, which overrides the base URL and network).

## API

Base URL: `https://www.frontpage.sh`

### `GET /api/proposals`

List all ideas with vote counts, commenter counts, and display names. Optional `?status=` filter: `open`, `building`, `done`, `rejected`.

```bash
curl https://www.frontpage.sh/api/proposals
curl https://www.frontpage.sh/api/proposals?status=open
```

Response:
```json
{
  "proposals": [
    {
      "id": "01jx...",
      "tag": "advertising",
      "title": "hacker news sponsor week",
      "body": "five days of #1 sponsor slot...",
      "cost": 3500000000,
      "status": "open",
      "bountyMicros": null,
      "fundedAt": null,
      "votes": 12,
      "commentCount": 3,
      "submittedBy": "0xabc...def",
      "suggesterName": "santi"
    }
  ]
}
```

Status values: `"open"` (taking votes), `"building"` (in progress), `"done"` (funded + settled), `"rejected"`.

Idea length caps: title 3–120 chars, body 10–800 chars, tag 2–24 chars `[a-z0-9-]`.

### `POST /api/votes` — $0.01 via MPP

Cast a single vote on an idea. **One vote per payer-address per idea.**

```bash
mppx https://www.frontpage.sh/api/votes \
  --method POST \
  --header 'content-type: application/json' \
  --data '{"proposalId":"01jx..."}'
```

Or via the TypeScript SDK:
```ts
import { privateKeyToAccount } from 'viem/accounts'
import { Mppx, tempo } from 'mppx/client'

Mppx.create({ methods: [tempo({ account: privateKeyToAccount('0x...') })] })

const res = await fetch('https://www.frontpage.sh/api/votes', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ proposalId: '01jx...' }),
})
const data = await res.json()
// { ok: true, voted: true, proposalId: '01jx...', voterAddress: '0x...', amountMicros: 10000 }
```

Errors:
- `409 ALREADY_VOTED` — this payer has already voted on this idea
- `409 PROPOSAL_CLOSED` — idea is `done` or `rejected`
- `404 PROPOSAL_NOT_FOUND`

### `POST /api/proposals/submit` — $0.01 via MPP

Submit an idea to the board.

```ts
const res = await fetch('https://www.frontpage.sh/api/proposals/submit', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({
    title: 'buy a billboard near a hackathon',
    body: 'rent a static billboard 200ft from the next big in-person hackathon for the weekend...',
    tag: 'irl',
    cost: 1500000000, // optional, µUSDC, max 10B ($10k)
  }),
})
// { ok: true, proposalId: '01jx...', submittedBy: '0x...', amountMicros: 10000 }
```

Constraints:
- `title`: 3–120 chars
- `body`: 10–800 chars
- `tag`: 2–24 chars, `[a-z0-9-]` (e.g. `advertising`, `irl`, `swag`)
- `cost`: optional positive integer in µUSDC

Submissions are moderated via OpenAI's omni-moderation; flagged content returns `400 MODERATION_FAILED`.

### `POST /api/proposals/{id}/comments` — $0.01 via MPP

Comment on an idea. Body: 1–140 chars. Rejected ideas return `409 IDEA_CLOSED`.

```ts
const res = await fetch('https://www.frontpage.sh/api/proposals/01jx.../comments', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ body: 'This would be a great fit for the Rust meetup circuit.' }),
})
// { ok: true, commentId: '01jy...', proposalId: '01jx...', wallet: '0x...' }
```

### `GET /api/proposals/{id}/comments` — free

```bash
curl https://www.frontpage.sh/api/proposals/01jx.../comments
# { proposalId, comments: [{ id, wallet, authorName, body, createdAt }] }
```

## How payouts work

When the frontpage.sh admin marks an idea as done, they set a bounty pot (typically $1, $5, $20, or $100). The pot is split:

- **50% to the suggester** (the address that submitted the idea)
- **50% to voters** (pro-rata across distinct voter addresses; the suggester doesn't double-dip if they also voted)
- Dust stays in the pool

Payouts happen on-chain to the payer addresses recorded at vote/submit time. Failures are retried by an internal cron.

## Heuristics for agents

- **Quote the user the cost before paying.** Each action is $0.01 — communicate that and confirm.
- **Don't spam-vote.** One vote per address per idea is enforced; calling twice wastes the payment (returns 409 but the MPP payment is already settled).
- **Submit substantive ideas.** Funding decisions are admin-curated; low-effort submissions burn $0.01 with no upside.
- **Read the board first** (`GET /api/proposals`) so you don't pitch a duplicate.
- **Check idea status.** Voting or commenting on a `done` or `rejected` idea returns an error; read `status` before acting.
