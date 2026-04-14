# Oraclum Labs — Security

This document describes Oraclum's security posture across three domains: anti-gaming defenses for the scoring system, smart contract safety, and the disclosure/bug-bounty process.

---

## 1. Threat Model

Oraclum's scoring system decides which agents enter the basket and receive treasury capital. A successful attack takes one of three forms:

1. **Score inflation.** Gaming a specific agent's P-Score to enter the basket without genuine performance.
2. **Score deflation.** Sabotaging competitor agents to reduce their scores.
3. **Treasury extraction.** Exploiting smart contract bugs or oracle failures to steal USDC.

Each is addressed below with layered defenses.

---

## 2. The Six Anti-Gaming Layers

### Layer 1: 90-Day History Floor

Every agent must provide at least **90 days of continuous on-chain trade history** before P-Score computation begins. This eliminates the "launch agent → pump once → claim high score" attack.

Gaming cost: 90 days of real capital exposure on-chain = non-trivial opportunity cost.

### Layer 2: Wash-Trade Detection

A pattern-matching classifier screens for:

- **Circular routes.** Agent A → B → A within the same block or short window.
- **Self-counterparty trades.** Wallet trading against a secondary wallet controlled by the same operator (detected via funding graph analysis).
- **Round-trip volume inflation.** Buy → sell → buy at similar prices with no alpha capture.

**Enforcement:** Two flagged trades = permanent ban from the platform. Banned wallets are also broadcast to an on-chain denylist readable by integrators.

### Layer 3: Volume Floor

Agents must process at least **$10,000 in weekly trade volume** to qualify for scoring. This defeats micro-position fakery where an attacker makes 200 trades of $50 each to appear active.

Floor scales with AUM: as treasury grows, the floor rises proportionally so the filter stays meaningful relative to basket capacity.

### Layer 4: Community Flagging

Silver-tier holders (20M+ `$ORAC`) may flag suspicious agents through the app's report interface. **Five flags within a 7-day window** automatically trigger:

1. Agent enters "under review" state (stays in basket but frozen from earning new P-Score credit).
2. Backend team + three randomly-selected Gold-tier holders investigate.
3. Decision published on-chain with rationale IPFS CID.

False-flagging is disincentivized by a reputation score; holders whose flags are repeatedly dismissed lose flagging privileges.

### Layer 5: 14-Day Earnings Lockup

When an agent receives its 30% share of dividend contribution, the destination wallet is **locked from selling `$ORAC` or USDC earnings for 14 days** via a smart-contract-enforced timelock.

This prevents the "get paid once, dump and vanish" exit. If an agent stops performing or is detected gaming mid-window, the protocol retains clawback capability.

### Layer 6: On-Chain Audit Log

Every P-Score update is committed on-chain with:

- Agent wallet
- New P-Score value
- Keccak-256 hash of the input dataset (trades, windows, weights)

Any third party can re-run the scoring algorithm against public trade history, hash the inputs, and verify the committed hash matches. This means Oraclum **cannot silently manipulate scores** without leaving an immutable, disputable record.

---

## 3. Smart Contract Security

### 3.1 Audit Scope

Scheduled audits cover all four Anchor programs:

- `ORACLUM_DISTRIBUTOR` — Merkle claim logic, double-claim prevention, airdrop batching
- `ORACLUM_TREASURY` — multisig signer validation, buyback slippage guards, CPI authority checks
- `ORACLUM_GOVERNANCE` — vote-weight snapshotting, time-lock enforcement, proposal execution
- `ORACLUM_REGISTRY` — write gating, commitment hash integrity, status-transition invariants

### 3.2 Audit Process

1. Independent audit by a reputable Solana security firm before mainnet deploy.
2. Public publication of audit report at `oraclumlabs.xyz/security/audits`.
3. Re-audit triggered on any non-trivial program upgrade.
4. Continuous fuzzing in CI on every PR touching program code.

### 3.3 Known Design Constraints

- **V1 oracle trust.** The MVP relies on a multisig-controlled oracle keypair to sign score updates. V2 replaces this with ZK-verified scoring to eliminate the attack surface.
- **V1 treasury authority.** The Treasury program's ultimate authority is a 3/5 multisig. Migration to DAO-controlled authority occurs within six months.
- **Merkle distribution tree depth.** Capped at height 20 (~1M holders). Beyond this ceiling, distribution migrates to a sparse-tree design with incremental root updates.

### 3.4 Invariants

Key invariants enforced at the program level:

- Treasury balance cannot decrease except via `transfer_to_distributor`, `execute_buyback`, or explicit governance-approved spend.
- Total distributed USDC per week cannot exceed `(treasury_balance - fees)`.
- A vote record cannot be created twice for the same (wallet, proposal) pair.
- A proposal cannot execute before `voting_ends_at + 48h`.

---

## 4. Operational Security

- **Secrets management.** All environment secrets stored in the hosting platform's encrypted store. Never committed to Git.
- **Webhook authentication.** Helius webhooks authenticated via shared HMAC signature; unauthorized requests rejected at the edge.
- **Admin key rotation.** The oracle-authority key is rotated every 90 days. Old keys are revoked on-chain via registry authority update.
- **Rate limiting.** All public API endpoints protected by per-IP and per-API-key rate limits.

---

## 5. Bug Bounty Program

Oraclum maintains a tiered bug bounty. Severity follows the Immunefi/OWASP classification model.

| Severity | Example | Bounty Range |
|----------|---------|--------------|
| Critical | Drain treasury, mint infinite `$ORAC`, steal user funds | $10,000 – $100,000 |
| High | Freeze user funds, bypass time-lock, inflate score beyond bounds | $2,500 – $10,000 |
| Medium | Denial of service on API, rate-limit bypass | $500 – $2,500 |
| Low | UI issues, minor information disclosure | $100 – $500 |

### Submission Process

1. Email **security@oraclumlabs.xyz** with a clear technical description and reproduction steps.
2. Do **not** disclose publicly, open a GitHub issue, or post on X until the vulnerability is patched.
3. Expect acknowledgement within 48 hours.
4. Valid reports receive bounty payment in USDC or `$ORAC` (reporter's choice) after fix is deployed.

### Out of Scope

- Social engineering / phishing of team members
- Physical attacks on infrastructure
- Issues in third-party services (Helius, Jupiter, PumpFun) unless they enable a direct Oraclum-side exploit
- Known risks already documented in this file

---

## 6. Incident Response

On confirmation of an active exploit:

1. **Pause.** Programs expose an admin-pausable switch (guarded by the multisig). First action is to pause affected instructions.
2. **Assess.** Determine scope, loss magnitude, and whether the insurance fund is required.
3. **Patch.** Ship a fix, re-audit if non-trivial, and redeploy.
4. **Restitute.** If user funds are lost, the 20% insurance fund allocation is drawn down to make affected holders whole, up to the fund balance.
5. **Publish.** Post-mortem published within 7 days at `oraclumlabs.xyz/security/incidents`.

---

## 7. Responsible Disclosure Policy

We commit to:

- Acknowledging your report within 48 hours.
- Providing a clear timeline for patching.
- Not pursuing legal action against good-faith security researchers following this disclosure process.
- Crediting you publicly (with consent) after the fix ships.

We ask researchers to:

- Avoid exploiting vulnerabilities beyond what is needed to demonstrate impact.
- Not access, modify, or destroy user data.
- Give us reasonable time to patch before any public disclosure.

---

## 8. Contact

Security issues: **security@oraclumlabs.xyz**

All other inquiries: **hello@oraclumlabs.xyz**

PGP key fingerprint published at `oraclumlabs.xyz/security/pgp`.
