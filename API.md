# Oraclum Labs — REST API Reference

Base URL: `https://oraclum.fun/api`

All endpoints return JSON. Authenticated endpoints require a Bearer API key in the `Authorization` header. Rate limits are enforced per API key (or per IP for unauthenticated calls).

---

## Tiers

| Tier | Requests/day | Monthly Cost | Features |
|------|--------------|--------------|----------|
| Free | 100 | $0 | Public endpoints, basic P-Score |
| Pro | Unlimited | $499 | Full breakdown, historical data, webhooks |
| Enterprise | Unlimited | $2,499 | White-label, SLA, custom badges |

Request an API key at `oraclum.fun/api`.

---

## Authentication

```http
GET /api/v1/basket
Authorization: Bearer sk_live_xxxxxxxxxxxxxxxxxxxx
```

For public endpoints, authentication is optional. Unauthenticated requests are subject to per-IP rate limits (60 req/minute).

---

## 1. Public Endpoints

### 1.1 GET `/api/v1/basket`

Returns the current 50 active agents in the basket.

**Response:**

```json
{
  "as_of": "2026-04-14T00:00:00Z",
  "agent_count": 50,
  "total_aum_usd": 1284500.00,
  "weekly_yield_pct": 3.42,
  "agents": [
    {
      "id": "uuid",
      "name": "AlphaScout",
      "wallet": "7x...",
      "p_score": 94.0,
      "weight_pct": 4.8,
      "yield_30d_pct": 12.4,
      "status": "active"
    }
  ]
}
```

### 1.2 GET `/api/v1/agents/{id}`

Agent detail with trade history and score breakdown.

**Response:**

```json
{
  "id": "uuid",
  "name": "AlphaScout",
  "wallet": "7x...",
  "p_score": 94.0,
  "score_breakdown": {
    "performance": 96.2,
    "consistency": 91.8,
    "risk_adjusted": 92.5
  },
  "stats": {
    "trade_count_30d": 412,
    "win_rate_pct": 67.3,
    "max_drawdown_pct": 8.1,
    "sharpe_ratio": 2.41,
    "rolling_30d_yield_pct": 12.4
  },
  "status": "active",
  "submitted_at": "2026-01-10T00:00:00Z",
  "accepted_at": "2026-01-17T00:00:00Z"
}
```

### 1.3 GET `/api/v1/score`

Score lookup for any wallet. Useful for B2B integrations embedding badges.

**Query params:**

- `wallet` (required) — base58 Solana pubkey

**Response:**

```json
{
  "wallet": "7x...",
  "p_score": 94.0,
  "tier_stars": 3,
  "last_updated": "2026-04-14T00:00:00Z",
  "in_basket": true,
  "badge_url": "https://oraclum.fun/badge/7x..."
}
```

Returns `404` if the wallet is not registered.

### 1.4 GET `/api/v1/dividends/latest`

Last weekly distribution summary.

**Response:**

```json
{
  "week_id": 17,
  "distributed_at": "2026-04-13T00:00:00Z",
  "total_pool_usd": 43921.50,
  "per_token_usd": 0.0000439,
  "holder_count": 4821,
  "merkle_root": "0x...",
  "distribution_tx": "5h..."
}
```

### 1.5 GET `/api/v1/dividends/history`

Paginated dividend history.

**Query params:**

- `limit` (default 20, max 100)
- `offset` (default 0)

### 1.6 GET `/api/v1/stats`

Top-line protocol stats.

**Response:**

```json
{
  "aum_usd": 1284500.00,
  "holder_count": 4821,
  "active_agent_count": 50,
  "waitlist_count": 127,
  "total_distributed_usd": 326100.00,
  "circulating_supply": 972000000,
  "burned_supply": 28000000
}
```

---

## 2. Authenticated Endpoints

### 2.1 POST `/api/v1/agents/submit`

Submit an agent for inclusion. Requires a one-time 0.5 SOL submission fee paid before the call (transaction signature passed in the body).

**Request:**

```json
{
  "agent_name": "MyAlphaBot",
  "agent_wallet": "7x...",
  "owner_wallet": "3k...",
  "description": "Momentum-driven memecoin trader",
  "metadata": { "strategy": "momentum", "timeframe": "15m" },
  "submission_fee_tx": "5h..."
}
```

**Response:**

```json
{
  "submission_id": "uuid",
  "status": "pending_verification",
  "estimated_review_hours": 72,
  "min_history_satisfied": true
}
```

Errors:

- `400` — insufficient trade history (less than 90 days)
- `402` — submission fee not confirmed on-chain
- `409` — wallet already submitted

### 2.2 POST `/api/v1/governance/propose`

Submit a governance proposal. Requires Silver tier (>= 20M `$ORAC`).

**Request:**

```json
{
  "title": "Lower management fee to 1.5%",
  "description_cid": "Qm...",
  "proposal_type": "fee_change",
  "params": { "new_management_fee_bps": 150 }
}
```

**Response:**

```json
{
  "proposal_id": 7,
  "voting_ends_at": "2026-04-21T00:00:00Z",
  "on_chain_tx": "5h..."
}
```

### 2.3 POST `/api/v1/governance/vote`

Cast a vote. Any holder may call.

**Request:**

```json
{
  "proposal_id": 7,
  "support": true
}
```

**Response:**

```json
{
  "recorded": true,
  "vote_weight": 142500,
  "current_for_pct": 67.2,
  "current_against_pct": 32.8
}
```

### 2.4 GET `/api/v1/portfolio`

Authenticated holder's position summary.

**Query params:**

- `wallet` (required) — base58 Solana pubkey

**Response:**

```json
{
  "wallet": "3k...",
  "orac_balance": 142500,
  "tier": "retail",
  "portfolio_value_usd": 4275.00,
  "lifetime_dividends_usd": 1247.32,
  "next_dividend_estimate_usd": 89.40,
  "next_dividend_at": "2026-04-20T00:00:00Z"
}
```

---

## 3. Webhooks

Pro and Enterprise tiers can subscribe to webhook events pushed to a URL of their choice.

### 3.1 Registration

```http
POST /api/v1/webhooks
Authorization: Bearer sk_live_xxx

{
  "url": "https://example.com/oraclum-webhook",
  "events": ["dividend.distributed", "agent.added", "agent.removed", "proposal.created"],
  "secret": "your-shared-secret"
}
```

### 3.2 Event Payloads

All events include `event`, `id`, `created_at`, and `data`. Example:

```json
{
  "event": "dividend.distributed",
  "id": "evt_01H...",
  "created_at": "2026-04-13T00:00:00Z",
  "data": {
    "week_id": 17,
    "total_pool_usd": 43921.50,
    "per_token_usd": 0.0000439
  }
}
```

### 3.3 Signature Verification

Each webhook request includes an `X-Oraclum-Signature` header:

```
sha256=<hmac_sha256(body, your_secret)>
```

Verify server-side to authenticate the event source.

---

## 4. Error Response Format

All errors follow a consistent shape:

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "You have exceeded 100 requests/day on the Free tier.",
    "docs_url": "https://oraclum.fun/docs/api#rate-limits"
  }
}
```

Common error codes:

| Code | HTTP | Description |
|------|------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid API key |
| `FORBIDDEN` | 403 | Tier does not permit this action |
| `NOT_FOUND` | 404 | Resource does not exist |
| `VALIDATION_ERROR` | 400 | Malformed request body |
| `INSUFFICIENT_HISTORY` | 400 | Less than 90 days of trade data |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## 5. SDK

A TypeScript SDK is available for integrators:

```bash
pnpm add @oraclumlabs/sdk
```

```ts
import { OraclumClient } from "@oraclumlabs/sdk";

const client = new OraclumClient({ apiKey: process.env.ORACLUM_API_KEY });

const basket = await client.getBasket();
const score = await client.getScore({ wallet: "7x..." });
```

See [github.com/OraclumLabs/sdk](https://github.com/OraclumLabs/sdk) for full SDK docs.

---

## 6. Changelog

API versioning follows semver. Breaking changes bump the `v{N}` prefix (e.g., `/api/v2`). Non-breaking additions are announced in [CHANGELOG.md](./CHANGELOG.md).

Current stable version: `v1`.
