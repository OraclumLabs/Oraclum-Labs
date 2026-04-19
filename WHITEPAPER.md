# Oraclum Labs — Whitepaper

**Version 1.0 — April 2026**
**Chain: Solana · Token: $ORAC · Domain: oraclumlabs.xyz**

---

## Abstract

The AI agent economy on Solana has surpassed 1,383 active agents, yet retail users have no clean vehicle to participate in its upside. Buying a single agent token is a coin flip; buying fifty is a portfolio-management nightmare. Oraclum Labs introduces a decentralized, on-chain index fund that aggregates the top 50 AI trading agents into a single ERC-style token, `$ORAC`, and distributes the basket's aggregate yield to holders every Sunday in USDC. This paper defines the protocol's scoring methodology, smart-contract architecture, economic design, anti-gaming defenses, and the path from a trusted-oracle MVP to a trustless, DAO-governed, ZK-verified V2.

---

## 1. Problem Statement

AI trading agents have proliferated across Solana. Platforms such as Hyre, fetchr.guru, and ClawGo host thousands of strategy-driven bots that execute autonomous trades, derive on-chain yield, and occasionally disappear along with their deployer.

A retail participant faces four unattractive paths:

1. **Pick a single agent.** Concentration risk is high; agents rug, become inactive, or underperform without warning.
2. **Assemble a basket manually.** Requires active monitoring of dozens of wallets, manual rebalancing, and gas inefficiency.
3. **Hire a performance agent directly.** Up-front capital commitment with uncertain ROI.
4. **Build an agent.** Requires deep technical skill most holders do not possess.

The result: most retail participants watch the AI agent narrative from the sidelines. The capital that would naturally flow into this ecosystem is trapped behind a usability wall.

Oraclum removes that wall.

---

## 2. Solution Overview

Oraclum is an **AI Agent Index Fund** on Solana. A single token — `$ORAC` — represents fractional exposure to the aggregate performance of the top 50 on-chain AI trading agents. The protocol operates on three continuous loops:

1. **Aggregation loop.** Agent-generated yield flows into a shared treasury.
2. **Distribution loop.** Every Sunday 00:00 UTC, treasury value is snapshotted, fees deducted, and USDC dividends streamed pro-rata to `$ORAC` holders.
3. **Rebalance loop.** The bottom five agents (by P-Score) rotate out; the top five from the waitlist rotate in. Weights are recomputed from rolling 30-day performance.

Holders receive REIT-style passive yield with zero active management. Agent developers receive an auto-growing capital pool and 30% of their own contribution to the dividend pot. B2B platforms consume the scoring API to display Oraclum-Verified credibility badges on agent profiles.

---

## 3. The P-Score Methodology

At the heart of Oraclum is the **Performance Score (P-Score)** — a 0 to 100 numeric rating computed weekly per agent.

```
P-Score = (0.40 × Performance) + (0.30 × Consistency) + (0.30 × Risk-Adjusted Return)
```

### 3.1 Performance Component (40%)

Derived from three sub-signals:

- **30-day net PnL** (denominated in USD)
- **7-day rolling yield** (recency weighting)
- **Win rate × average win size** (signal strength)

Each sub-signal is z-scored against the active agent cohort and clamped to [0, 100].

### 3.2 Consistency Component (30%)

Measures whether an agent's returns are repeatable or a single lucky strike:

- Days active in the last 30 (minimum 20 to qualify)
- Standard deviation of daily returns (lower is better)
- Max consecutive loss days (penalty)
- Trade frequency stability

### 3.3 Risk-Adjusted Return Component (30%)

Captures prudence of execution:

- Annualized Sharpe ratio
- Max drawdown percent (inverse-scored)
- Liquidation event count (exponential penalty)
- Average position size relative to treasury (over-sized positions penalized)

### 3.4 Score Thresholds

| Threshold | Requirement |
|-----------|-------------|
| Waitlist entry | P-Score >= 60 |
| Active basket retention | P-Score >= 50 (3-week grace period) |
| Fast-track review (Gold nominations) | P-Score >= 75 |

---

## 4. Anti-Gaming Design

A naive scoring system invites exploitation. Oraclum ships with six layered defenses from day one:

1. **90-day history floor.** No agent is scored with less than 90 days of on-chain trade data. Eliminates pump-and-dump fakes.
2. **Wash-trade detection.** An ML classifier flags self-routing and circular trade patterns. Two strikes = permanent ban.
3. **$10k weekly volume floor.** Micro-position fakery with $50 trades is filtered out by construction.
4. **Community flagging.** Silver-tier holders (20M+ `$ORAC`) can flag suspicious agents. Five flags triggers manual review.
5. **14-day earnings lockup.** After receiving a dividend, the agent developer's wallet is lockout from selling earnings for 14 days. Aligns incentives against dump-and-vanish.
6. **On-chain audit log.** Every P-Score change is committed on-chain along with the input hash. Scoring is auditable and disputable.

In combination, these defenses make gaming economically irrational. Full mechanics are documented in [SECURITY.md](./SECURITY.md).

---

## 5. Smart Contract Architecture

Oraclum runs on four Anchor-framework programs deployed on Solana mainnet:

- **`ORACLUM_DISTRIBUTOR`** — holds the aggregated USDC treasury, snapshots `$ORAC` holders weekly via Merkle tree, and streams pro-rata dividends.
- **`ORACLUM_TREASURY`** — multisig-guarded (3/5) in V1, DAO-controlled in V2. Receives agent revenue inflows, executes buyback-and-burn operations.
- **`ORACLUM_GOVERNANCE`** — token-weighted voting with a 48-hour time-locked execution window. Silver-tier (20M `$ORAC`) and above may submit proposals.
- **`ORACLUM_REGISTRY`** — permissionless-read, gated-write storage of agent metadata, P-Score commitment hashes, and status (active/waitlist/removed).

A full technical breakdown — including account layouts, PDA derivations, CPI diagrams, and program upgrade authority — is in [ARCHITECTURE.md](./ARCHITECTURE.md).

---

## 6. Tokenomics

### 6.1 Supply

- **Total supply:** 1,000,000,000 `$ORAC`
- **Launch venue:** PumpFun fair launch
- **Public allocation:** 100%
- **Team allocation:** 0%
- **VC allocation:** 0%

There is no presale, no insider round, no linear unlock. Founders align with holders via PumpFun creator fees and governance participation only.

### 6.2 Tier System

| Tier | Requirement | Core Benefits |
|------|-------------|---------------|
| Retail | Any amount | Weekly dividends, voting, public data |
| Bronze | 10M `$ORAC` (1%) | Premium analytics, 24h waitlist preview |
| Silver | 20M `$ORAC` (2%) | Propose governance, flag suspicious agents |
| Gold | 50M `$ORAC` (5%) | Direct team line, quarterly AMA, nominate agents |

### 6.3 Supply Reduction

- **Weekly buyback + burn.** 20% of all protocol revenue is used to buy `$ORAC` on market and burn it. Target: 50% of supply retired by Year 5.
- **Rejection burn.** 0.5 SOL submission fee from rejected agents is converted to `$ORAC` via buyback and burned.
- **Staking lock (V2).** Optional 30- or 90-day lock for boosted dividend multipliers (+5% and +15% respectively), reducing circulating float.

---

## 7. Revenue Model

Oraclum generates revenue through five streams:

1. **Management fee** — 2% annualized on AUM, pro-rated weekly.
2. **Performance fee** — 20% of net positive weekly yield, above a 0.5% hurdle rate.
3. **Premium analytics** — $19/month subscription for advanced dashboards and historical backtesting.
4. **B2B API** — Free tier, $499/month Pro, $2,499/month Enterprise.
5. **PumpFun creator fee** — 2% of every `$ORAC` market trade, permanent passive revenue.

Aggregate revenue is allocated:

- 40% operations and development
- 20% `$ORAC` buyback + burn
- 20% insurance fund (black swan reserve)
- 10% marketing
- 10% reserve

The insurance fund absorbs losses from oracle failures, smart contract exploits, or major agent rugs that slip past anti-gaming layers.

---

## 8. Distribution Mechanics

Every Sunday 00:00 UTC, dubbed **Consensus Day**, the Distributor program executes:

1. Snapshot all `$ORAC` holder balances from on-chain state.
2. Compute per-token dividend: `(treasury_usdc - fees) / circulating_supply`.
3. Generate Merkle root of (wallet, amount) leaves.
4. Publish root on-chain; emit distribution event.
5. Holders' wallets receive USDC via airdrop; no claim transaction required.

Gas costs for distribution scale with tree depth (log-N), not holder count, making dividends economically viable up to millions of holders.

---

## 9. Governance

Oraclum launches with a multisig-controlled treasury (V1) and migrates to full DAO control within six months (V2). Governance scope includes:

- Management/performance fee adjustments
- Basket size changes (50 → 75, etc.)
- Anti-gaming parameter tuning
- Treasury spend allocation
- Program upgrade authority

All proposals require a 48-hour voting window and 48-hour time-locked execution to protect against flash-loan governance attacks. Only Silver-tier (20M+ `$ORAC`) may submit; all holders may vote.

---

## 10. Roadmap

**Phase 1 — MVP (Weeks 1–6).** Smart contracts, scoring engine, dashboard, PumpFun launch, 10-agent initial basket, target $50k AUM.

**Phase 2 — Growth (Months 2–4).** 50-agent basket active, weekly dividends flowing, B2B API live, target $1M AUM and 5,000 holders.

**Phase 3 — Decentralize (Months 4–7).** ZK-proof agent verification, DAO governance migration, permissionless submissions, cross-chain indexing, target $10M AUM.

**Phase 4 — Ecosystem (Months 7–12).** Index perpetuals, tier funds (Aggressive/Balanced/Stable), sector funds, mobile apps, target $100M AUM.

---

## 11. Risk Disclosures

- **Past performance.** Historical P-Scores do not guarantee future returns. Agent strategies can decay.
- **Smart contract risk.** Despite audits, bugs may exist. The insurance fund is the first line of defense.
- **Regulatory risk.** The treatment of on-chain index products is still evolving globally. Users are responsible for their own jurisdictional compliance.
- **Dividend variability.** Weekly dividends may be zero in low-performance weeks. Oraclum is not a yield guarantee.
- **Oracle risk (V1).** The trusted-oracle MVP relies on a multisig for score integrity. V2 migrates to trustless verification to eliminate this attack surface.

---

## 12. Conclusion

Oraclum Labs turns the AI agent economy from a high-risk, high-effort speculation into a single-token, dividend-paying index. By combining a transparent P-Score methodology, a multi-layered anti-gaming regime, and a pro-holder tokenomic flywheel, Oraclum becomes the default gateway for retail participation in on-chain AI — and, over time, the industry-standard credibility benchmark for agent developers.

One oracle. Thousand agents. Infinite alpha.

---

**Legal Notice.** `$ORAC` is a utility and governance token, not a security. This document is informational only and does not constitute financial, legal, or investment advice. 18+. NFA. DYOR.
