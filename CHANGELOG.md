# Changelog

All notable changes to Oraclum Labs will be documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and adheres to [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

### Planned
- ZK-proof based agent verification (V2)
- DAO governance migration from 3/5 multisig
- Staking lock with +5%/+15% dividend multipliers (30d/90d)
- Cross-chain agent indexing (Base, BSC)

---

## [0.1.0] — Q2 2026 — MVP Launch

Initial mainnet deployment. Trusted-oracle model. PumpFun fair launch.

### Added

- **Smart contracts (Anchor):**
  - `ORACLUM_DISTRIBUTOR` — Merkle-tree weekly USDC distribution
  - `ORACLUM_TREASURY` — 3/5 multisig treasury with buyback execution
  - `ORACLUM_GOVERNANCE` — token-weighted voting with 48h time-lock
  - `ORACLUM_REGISTRY` — on-chain agent metadata + P-Score commitments
- **P-Score scoring engine** with 40/30/30 Performance/Consistency/Risk-Adjusted weighting
- **Six anti-gaming layers:** 90-day history floor, wash-trade detection, $10k volume floor, community flagging, 14-day earnings lockup, on-chain audit log
- **Weekly rebalance cron** executing every Sunday 00:00 UTC (Consensus Day)
- **Public REST API v1:**
  - `GET /api/v1/basket`
  - `GET /api/v1/agents/{id}`
  - `GET /api/v1/score`
  - `GET /api/v1/dividends/latest`
  - `GET /api/v1/dividends/history`
  - `GET /api/v1/stats`
- **Authenticated endpoints:**
  - `POST /api/v1/agents/submit`
  - `POST /api/v1/governance/propose` (Silver tier)
  - `POST /api/v1/governance/vote`
  - `GET /api/v1/portfolio`
- **Frontend (Next.js 15):**
  - Landing page with live stats
  - Portfolio dashboard
  - Public basket leaderboard
  - Agent submission form
  - Governance UI
- **Tier system:** Retail / Bronze (10M) / Silver (20M) / Gold (50M `$ORAC`)
- **Revenue model:** 2% management fee, 20% performance fee (0.5% hurdle), $19/mo premium analytics, B2B API tiers ($499/$2,499 Pro/Enterprise)
- **Revenue allocation:** 40% ops, 20% buyback+burn, 20% insurance fund, 10% marketing, 10% reserve
- **Token launch:** 1,000,000,000 `$ORAC` supply, 100% PumpFun fair launch, 0% team/VC allocation

### Initial Basket

- 10 pre-vetted agents at launch
- Target $50k AUM by end of Week 1
- First Consensus Day dividend payout scheduled for T+7 days

### Integrations

- Helius (RPC + webhook streams)
- Jupiter (buyback execution router)
- CoinGecko (USD price feeds)
- Supabase (data layer)
- PumpFun (launch venue + creator fee)
- Phantom / Solflare (wallet support)

### Security

- Pre-launch audit completed (report published at `oraclumlabs.xyz/security/audits`)
- Bug bounty program live ($100 – $100,000 range by severity)
- Incident response playbook published
- PGP key published for responsible disclosure

---

## [0.0.3] — Q2 2026 — Testnet Deployment

Internal testnet release for security review and partner integration testing.

### Added
- All four Anchor programs deployed to Solana devnet
- End-to-end dividend distribution flow tested with simulated holder set
- Helius webhook indexer validated against 90-day historical data
- Merkle tree distribution stress-tested up to 100k holders

### Fixed
- Race condition in concurrent score updates
- Off-by-one in weekly rebalance slot calculation
- Merkle proof verification rejecting valid proofs under specific tree shapes

---

## [0.0.2] — Q1 2026 — Alpha Program

Closed alpha with a small set of agent developers and partner platforms.

### Added
- Alpha version of scoring engine (Performance + Consistency only; Risk-Adjusted added in 0.0.3)
- Draft agent submission API
- Internal leaderboard UI

### Changed
- Scoring window extended from 14 days to 30 days after alpha feedback indicated short windows amplified noise

---

## [0.0.1] — Q1 2026 — Project Inception

Initial blueprint and architectural design.

### Added
- Full project blueprint (`ORACLUMLABS-BLUEPRINT.md`)
- Smart contract architecture specification
- P-Score methodology defined
- Tokenomics + revenue model finalized
- Brand identity locked (Oraclum Labs, `$ORAC`, oraclumlabs.xyz, @OraclumLabs)

---

## Versioning Policy

- **Major (`1.x.x`)** — breaking protocol changes, hard forks, backward-incompatible API versions
- **Minor (`0.x.0`)** — new features, new endpoints, additive contract changes
- **Patch (`0.0.x`)** — bug fixes, security patches, non-breaking improvements

API versioning is independent from the protocol version: API breaking changes bump the URL prefix (`/api/v1` → `/api/v2`).

Smart contract upgrades go through governance with a mandatory 48-hour time-lock. Non-breaking additive upgrades may be executed via multisig during the V1 phase.
