# Oraclum Labs — Technical Architecture

This document describes the full technical stack powering Oraclum Labs, including on-chain programs, backend services, data pipeline, and frontend application.

---

## 1. System Overview

Oraclum is a three-tier system:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   FRONTEND              BACKEND                  BLOCKCHAIN     │
│   ┌──────────┐         ┌─────────────┐          ┌────────────┐  │
│   │ Next.js  │────────▶│ Hono API    │─────────▶│ Solana     │  │
│   │ Tailwind │         │ Node.js     │          │ Programs   │  │
│   │ App      │         │             │          │            │  │
│   └──────────┘         │ • Auth      │          │ Distributor│  │
│        │               │ • Scoring   │          │ Treasury   │  │
│        │               │ • Indexer   │          │ Governance │  │
│        │               │ • Webhooks  │          │ Registry   │  │
│        │               │ • Rebalance │          └─────┬──────┘  │
│        │               └──────┬──────┘                │         │
│        │                      ▼                       │         │
│        │               ┌──────────────┐               │         │
│        │               │ Helius RPC   │───────────────┘         │
│        │               │ + Webhooks   │                         │
│        │               └──────┬───────┘                         │
│        │                      ▼                                 │
│        │               ┌──────────────┐                         │
│        └──────────────▶│ Supabase     │                         │
│                        │ (Postgres)   │                         │
│                        └──────────────┘                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

- **Frontend** renders user-facing dashboards, wallet flows, leaderboard, and governance UI.
- **Backend** runs the scoring engine, weekly rebalance job, indexer, and REST API.
- **Blockchain** holds the authoritative state: treasury balance, governance votes, agent registry, distribution Merkle roots.

---

## 2. Solana Programs (Anchor)

All four programs are written in Rust using the Anchor framework. They are deployed to Solana mainnet under a multisig upgrade authority (3/5 V1, DAO V2).

### 2.1 `ORACLUM_DISTRIBUTOR`

Responsible for weekly USDC dividend distribution.

**Key accounts:**

- `DistributorState` — global PDA storing total treasury balance, last distribution slot, merkle root of the most recent snapshot.
- `ClaimRecord` — per-holder PDA tracking claim status (prevents double-claims if a wallet is included in overlapping trees).

**Key instructions:**

- `initialize_distribution(root: [u8; 32], total_usdc: u64)` — called by the rebalance job after snapshot.
- `claim(proof: Vec<[u8; 32]>, amount: u64)` — holder claims their proportional share. Most holders receive airdrops rather than self-claim.
- `airdrop_batch(recipients: Vec<AirdropEntry>)` — batched admin airdrop to reduce holder gas burden.

The Merkle tree is height-capped at 20 (~1M holders), keeping proof sizes under 640 bytes.

### 2.2 `ORACLUM_TREASURY`

Holds all incoming USDC from agent revenue streams and executes buybacks.

**Key accounts:**

- `TreasuryState` — multisig threshold (3/5 V1), authority pubkey set, buyback config.
- `BuybackOrder` — ephemeral PDA representing an open buyback intent.

**Key instructions:**

- `deposit_revenue(amount: u64)` — agents or backend push USDC into the treasury.
- `execute_buyback(amount_usdc: u64, min_orac_out: u64)` — swaps USDC → `$ORAC` via Jupiter route, forwards to burn address.
- `transfer_to_distributor(amount: u64)` — weekly transfer triggered by rebalance job.

Multisig is implemented via the standard SPL-multisig account pattern. V2 replaces the multisig authority with a DAO-controlled PDA whose signer is the governance program.

### 2.3 `ORACLUM_GOVERNANCE`

Token-weighted voting with time-locked execution.

**Key accounts:**

- `Proposal` — title, description hash (IPFS CID), created_at, voting_ends_at, executed_at, for/against totals, proposal_type enum.
- `VoteRecord` — per (wallet, proposal) record preventing double-voting.

**Key instructions:**

- `create_proposal(title, description_cid, proposal_type)` — gated to wallets holding >= 20M `$ORAC` (Silver tier).
- `cast_vote(proposal_id, support: bool)` — any `$ORAC` holder may vote; weight = current balance snapshot.
- `execute(proposal_id)` — callable after 48h time-lock. Validates majority, applies parameter changes via CPI.

Vote weights snapshot at proposal-creation slot to prevent flash-loan vote buying.

### 2.4 `ORACLUM_REGISTRY`

On-chain source-of-truth for agent metadata.

**Key accounts:**

- `AgentRecord` — wallet, submitted_at, accepted_at, last_p_score, status enum (Active/Waitlist/Removed), audit_log_cid.

**Key instructions:**

- `register_agent(wallet: Pubkey, metadata_cid: String)` — callable by backend after off-chain verification.
- `update_score(wallet: Pubkey, score: u8, proof_hash: [u8; 32])` — writes weekly score with commitment hash for audit trail.
- `set_status(wallet: Pubkey, status: u8)` — transitions agent through state machine.

Registry is public-read via standard Anchor account fetch; only the oracle-authority key can write.

---

## 3. Backend Services

The backend is a Hono-based Node.js application deployed on a managed container platform. It exposes a REST API and runs background jobs.

### 3.1 Services

```
backend/
├── api/              # Hono HTTP routes (/api/v1/*)
├── indexer/          # Helius webhook consumer, trade hydration
├── scoring/          # P-Score computation engine
├── rebalance/        # Weekly cron job (Sundays 00:00 UTC)
├── distributor/      # Merkle tree builder + on-chain push
├── governance/       # Proposal lifecycle manager
└── shared/           # Solana client, Supabase client, config
```

### 3.2 Indexer

The indexer subscribes to agent wallets via Helius webhooks. On each trade event:

1. Parse transaction, extract token-in/token-out, slippage, PnL.
2. Validate the trade is not a wash pattern (circular routes, self-transfers).
3. Insert into `trades` table in Supabase.
4. Trigger incremental score update if trade is within scoring window.

Fallback polling runs every 5 minutes against Helius RPC `getSignaturesForAddress` to catch missed webhooks.

### 3.3 Scoring Engine

Implements the P-Score formula described in [WHITEPAPER.md](./WHITEPAPER.md):

```
P-Score = 0.40 × Performance + 0.30 × Consistency + 0.30 × Risk-Adjusted
```

Each agent's score is recomputed nightly. Inputs are pulled from `trades` over rolling 30-day windows; outputs are written to `scores` with a `snapshot_at` timestamp so history is preserved.

A commitment hash (`keccak256(serialized_inputs)`) is posted on-chain via `update_score` so any third party can reproduce and dispute the score.

### 3.4 Weekly Rebalance Job

Runs Sundays 00:00 UTC:

1. Freeze scoring (24h grace window closed).
2. Rank all agents by latest P-Score.
3. Identify bottom-5 in active basket → mark `Removed`.
4. Identify top-5 in waitlist → mark `Active`.
5. Recompute weights: `weight_i = p_score_i / sum(p_scores)` (normalized to 100%).
6. Aggregate treasury USDC → subtract 2% management fee + 20% performance fee.
7. Call `initialize_distribution` with the Merkle root of holder snapshots.
8. Execute `airdrop_batch` transactions to spray USDC to holder wallets.
9. Publish dividend record to `dividends` table for UI display.

The job is idempotent; if it halts mid-run, re-execution skips already-completed steps.

---

## 4. Data Layer (Supabase / Postgres)

```sql
-- Agents registered with Oraclum
CREATE TABLE agents (
  id              UUID PRIMARY KEY,
  name            TEXT NOT NULL,
  wallet          TEXT UNIQUE NOT NULL,
  owner_pubkey    TEXT NOT NULL,
  p_score         NUMERIC(5,2) DEFAULT 0,
  weight_pct      NUMERIC(5,2) DEFAULT 0,
  status          TEXT CHECK (status IN ('waitlist','active','removed')),
  submitted_at    TIMESTAMPTZ NOT NULL,
  accepted_at     TIMESTAMPTZ,
  last_active_at  TIMESTAMPTZ,
  metadata        JSONB
);

-- Trade history powering P-Score
CREATE TABLE trades (
  id              UUID PRIMARY KEY,
  agent_id        UUID REFERENCES agents(id),
  tx_signature    TEXT UNIQUE NOT NULL,
  timestamp       TIMESTAMPTZ NOT NULL,
  token_in        TEXT,
  amount_in       NUMERIC,
  token_out       TEXT,
  amount_out      NUMERIC,
  pnl_usd         NUMERIC,
  was_profitable  BOOLEAN,
  verified_at     TIMESTAMPTZ DEFAULT NOW()
);

-- Historical P-Score snapshots
CREATE TABLE scores (
  id                  UUID PRIMARY KEY,
  agent_id            UUID REFERENCES agents(id),
  snapshot_at         TIMESTAMPTZ NOT NULL,
  p_score             NUMERIC(5,2),
  performance         NUMERIC(5,2),
  consistency         NUMERIC(5,2),
  risk_adjusted       NUMERIC(5,2),
  rolling_30d_yield   NUMERIC,
  drawdown_pct        NUMERIC,
  trade_count_30d     INT
);

-- Current holder state
CREATE TABLE holders (
  wallet_address       TEXT PRIMARY KEY,
  orac_balance         NUMERIC NOT NULL DEFAULT 0,
  tier                 TEXT CHECK (tier IN ('retail','bronze','silver','gold')),
  lifetime_dividends   NUMERIC DEFAULT 0,
  last_updated         TIMESTAMPTZ DEFAULT NOW()
);

-- Weekly dividend records
CREATE TABLE dividends (
  id                UUID PRIMARY KEY,
  week_id           INT UNIQUE NOT NULL,
  snapshot_at       TIMESTAMPTZ NOT NULL,
  total_pool_usd    NUMERIC NOT NULL,
  per_token_usd     NUMERIC NOT NULL,
  merkle_root       TEXT,
  distribution_tx   TEXT,
  status            TEXT CHECK (status IN ('pending','distributed'))
);

-- Governance
CREATE TABLE governance_proposals (
  id              UUID PRIMARY KEY,
  proposer_wallet TEXT NOT NULL,
  title           TEXT NOT NULL,
  description_cid TEXT NOT NULL,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  voting_ends     TIMESTAMPTZ NOT NULL,
  executed_at     TIMESTAMPTZ,
  for_votes       NUMERIC DEFAULT 0,
  against_votes   NUMERIC DEFAULT 0,
  status          TEXT CHECK (status IN ('active','passed','rejected','executed')),
  proposal_type   TEXT
);
```

Indexes exist on `trades(agent_id, timestamp)`, `scores(agent_id, snapshot_at)`, and `holders(orac_balance)` to support performance queries.

---

## 5. Frontend (Next.js App)

The web app at `oraclum.fun` uses Next.js 15 App Router with server components where possible:

```
app/
├── page.tsx                 # Landing + live stats
├── portfolio/               # Holder dashboard
├── basket/                  # Public leaderboard
├── agents/submit/           # Agent dev submission form
├── vote/                    # Governance UI
├── docs/                    # MDX-rendered public docs
└── api/                     # Next.js route handlers (waitlist, banner)
```

### Key Dependencies

- `@solana/wallet-adapter-react` + `@solana/wallet-adapter-react-ui` — wallet connection
- `@solana/web3.js` + `@coral-xyz/anchor` — program interaction
- `@tanstack/react-query` — server state
- `tailwindcss` — styling (no Tailwind animation classes in video/Remotion contexts)
- `framer-motion` — entrance animations on web

### Rendering Strategy

- **Static generation** for docs and marketing pages.
- **Server-side rendering** for basket leaderboard (cached 60s).
- **Client-side rendering** for wallet-specific views (portfolio, vote).

### Wallet Integration

Supported wallets: Phantom, Solflare. Additional wallets can be added via `WalletAdapter` plugins.

---

## 6. Integrations

| Service | Purpose |
|---------|---------|
| Helius | Solana RPC + webhook stream for agent wallets |
| Jupiter | Swap aggregator for buyback execution |
| CoinGecko | Token price feeds for USD denomination |
| Supabase | Postgres + realtime for holder balance sync |
| PumpFun | Initial token launch and creator fees |

---

## 7. Deployment

- **Frontend:** Vercel (production), preview deploys per PR.
- **Backend:** Managed container platform, autoscaling 1–5 replicas, health checks on `/health`.
- **Cron jobs:** Dedicated worker pod running BullMQ-backed schedulers.
- **Database:** Supabase-hosted Postgres (primary + read replica).

Secrets management uses the platform's encrypted env store. Nothing is committed to Git.

---

## 8. Monitoring & Observability

- **Logs:** Pino structured logging → platform log drain.
- **Metrics:** Prometheus exporter on backend; Grafana dashboards for AUM, holder count, rebalance job duration, RPC latency.
- **Alerts:** PagerDuty on rebalance failure, distributor tx failure, or webhook lag > 5 min.
- **On-chain:** Solana Explorer + custom rebalance audit page at `oraclum.fun/audit`.

---

## 9. Upgrade Path

V1 → V2 is an additive migration:

1. Deploy ZK verifier program alongside existing Distributor/Registry.
2. Gradually migrate scoring from trusted oracle to ZK-verified proofs.
3. Transfer treasury multisig authority to governance PDA via executed proposal.
4. Deprecate backend scoring endpoints once >90% of agents are ZK-verified.

No hard fork of `$ORAC` is required. All upgrades are backward-compatible at the token layer.
