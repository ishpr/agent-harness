# DunkBot Agent Identity Master Plan

**Version:** 1.0

**Date:** 2026-10-31

**Status:** Draft for agent handoff — deferred backlog, implementation not started

**Repo target:** ishpr/agent-harness (`docs/agent-identity-master-plan.md`)

**Author:** Eve (Grok) — synthesized from research session 2026-10-31


> Supplied design draft. External research claims, protocol details, deployment assumptions and statistics below have not been independently verified. ADR-001 through ADR-004 are proposed identifiers within this draft, not allocated repository ADRs.

Before implementation:

- [ ] Verify referenced standards, chain registries, signing protocols, statistics and sequence-governance support against primary sources.
- [ ] Resolve stable agent identity versus key rotation/revocation, including delegated-token and verification-cache invalidation.
- [ ] Finalize the illustrative DDR schemas, chain-qualified identity and policy-export semantics. The empty `protocols` and `scope` arrays below preserve unspecified slots in the supplied draft.
- [ ] Map the AWS-backed DunkBot components in this proposal to this repository's currently documented Cloudflare harness and assign implementation boundaries.

---

## 1. Context and Problem Statement

### 1.1 Why this matters

DunkBot (app.dunkbot.ai) is a multi-tenant AI agent platform built on AWS: Bedrock AgentCore, AppSync, Terraform, EventBridge, DynamoDB, S3, Cedar policies. Bots have persistent cloud computers (browser automation + code interpreter + S3 workspace), multi-agent chat, security via STS session tags and exact-action approvals, transactional outbox, and inbound webhooks/SES/SMS.

The platform is now moving into **cross-tenant agent commerce**: agents communicating outside their tenant, agents paying other agents, an intent network where a masked request from one tenant's bot can be fulfilled and paid by another. In that world, agent identity stops being an internal bookkeeping concern and becomes the foundation of trust, authorization, and settlement.

Human employees get Okta: a durable, revocable, auditable identity bound to a principal across every system they touch. Agents currently get STS session tags — scoped per attempt, great for a single run, but ephemeral. When money moves between agents, "who did this" must survive across interactions, across tenants, and across time.

### 1.2 Research findings (2026-10-31)

- **ERC-8004** — on-chain agent registry on Base, built on ERC-721 NFTs. Agent ID resolves to a registration file: name, description, service endpoints, supported protocols, public key. Authors from MetaMask, Ethereum Foundation, Google, Coinbase.
- **ERC-8128** — runtime verification: agent signs every request with its private key; receiving service looks up the public key from the registry and checks the signature.
- **SIWA (Sign In With Agent)** — uses the registration to prove a request came from the registered agent, not an impersonator.
- **Basenames** — human-readable names (myagent.base.eth) resolving to the wallet.
- **Reputation caveat** — BaseAgent.com notes ERC-8004 reputation is easy to fake (self-review rings). Countermeasure: weight reputation by settled USDC volume, not raw review count; detect review rings and sibling agents sharing reviewer sets.
- **Coinbase Agentic Wallets (Feb 2026)** — private keys in AWS Nitro Enclaves, never in the LLM context. Programmable session caps, per-transaction limits, built-in KYT screening. Gasless trading on Base.
- **x402** — HTTP 402 payment protocol; 165M+ transactions, $50M settled by April 2026. Members: Google, Visa, AWS, Circle, Anthropic, Vercel. 90%+ of on-chain agent transactions settle on Base, 99% in USDC.
- **AWS Dogwood (Aug 2026)** — open-source extension of Cedar governing sequences of agent tool calls, not just single actions. Official AWS sample: Cedar enforcing least-privilege across multi-agent delegation chains.
- **ENISA / EU Digital Identity Wallet** — formal standards-body track grounding AI agent identity in W3C Verifiable Credentials and DIDs. More compliance-heavy; the ERC route is the pragmatic one for DunkBot since payments already run on Base.

### 1.3 Design goals

1. Every DunkBot bot has a **durable, immutable root identity** that survives across runs, tenants, and commerce interactions.
2. Short-lived delegated credentials derive from that root and are independently verifiable by any party — not just the DunkBot gateway.
3. Authorization policies are **portable**: internal Cedar enforcement plus a machine-readable export external agents can evaluate.
4. Every action — especially payments — is attributed to the root identity in a tamper-evident audit log.
5. Keys never appear in prompts, conversation history, or LLM context (prompt-injection resistance).
6. Reputation is payment-weighted and ring-resistant, treated as a separate signal from identity.

---

## 2. The Four-Layer Identity Model

### Layer 1 — Root Identity (immutable)

- An **ERC-8004 registration** on Base: agent ID (ERC-721 token), registration file (name, description, endpoints, protocols, public key), optional Basenames.
- Key pair generated inside a **TEE** (AWS Nitro Enclave or equivalent). The private key is created and used only inside the enclave; it is never exported to the agent's runtime, prompt, or storage outside the enclave boundary.
- The root identity is **minted once** at bot creation. It is not rotated away — rotation means a new identity, which breaks trust chains. If compromise is suspected, the path is **revocation** (burning or flagging the registration), not silent key swap.
- Registration happens on **Base mainnet** for production bots. Sepolia is for development and architecture validation only.

### Layer 2 — Delegated Credentials (ephemeral, derived)

- Short-lived tokens (existing run tokens, extended) whose claims bind: tenant ID, bot/agent ID (root), conversation ID, turn ID, scope (allowed actions), expiry, and a nonce.
- Tokens are **signed by the root key inside the TEE** so any external verifier can check them against the on-chain public key without calling DunkBot's gateway.
- Session caps and per-transaction limits enforced at the wallet layer (Agentic Wallets pattern), independent of the token.

### Layer 3 — Authorization Policy (portable Cedar)

- Internal: existing Cedar policies gating exact actions (send, pay, etc.) evaluated by interceptors.
- External: a **machine-readable policy export** (JSON/Cedar text) published alongside the ERC-8004 registration so a responding agent can evaluate "is this requester authorized for this action?" before fulfilling.
- Dogwood-style sequence governance for multi-step delegation chains (agent A delegates to agent B who calls agent C).

### Layer 4 — Audit and Reputation (tamper-evident, weighted)

- Every cross-tenant action and payment emits an **audit event** attributed to the root agent ID, written to an append-only log (DynamoDB with conditional writes + periodic anchoring, or on-chain event log).
- Reputation is a **separate, queryable signal**: settled USDC volume, successful completion rate, review count weighted by counterparty's own reputation. Review-ring detection: flag clusters of agents sharing reviewer sets or reviewing each other.
- Reputation informs trust decisions but never substitutes for cryptographic verification.

### Immutability rules

| Asset | Immutable? | Notes |
| --- | --- | --- |
| Root key pair / agent ID | Yes | Minted once; revocation only |
| Registration file core fields (ID, pubkey) | Yes | Metadata fields (description, endpoints) mutable via signed update |
| Delegated tokens | No | Short-lived by design |
| Cedar policies | No | Versioned; changes audited |
| Audit log | Append-only | Tamper-evident |
| Reputation scores | No | Derived, recomputed |

---

## 3. Epics, Features, and User Stories

### Epic 1 — On-Chain Root Identity

**Feature 1.1: ERC-8004 registration service**

- As a DunkBot platform operator, I want each bot to receive an ERC-8004 agent registration on Base when created, so the bot has a durable on-chain identity.
- AC: On bot creation, a Lambda (or AgentCore action) calls the ERC-8004 registry contract; the returned agent ID is persisted on the bot's DynamoDB record; registration file includes name, description, service endpoint (DunkBot gateway URL), supported protocols (x402, ERC-8128), and the TEE-generated public key.
- AC: Registration is idempotent — retrying creation does not mint a second agent.
- AC: Dev/staging bots register on Sepolia; production bots on mainnet, controlled by an env flag.

**Feature 1.2: TEE-isolated key generation**

- As a security engineer, I want bot key pairs generated inside an AWS Nitro Enclave so private keys never exist outside the enclave.
- AC: Keygen runs as an enclave image; the private key is used only for signing inside the enclave and is zeroized after use per operation where feasible (or held ephemerally in enclave memory only).
- AC: No code path logs, stores, or returns the private key to the agent runtime, S3 workspace, or conversation history.
- AC: Enclave attestation document is recorded at registration so verifiers can confirm keygen happened in a genuine TEE.

**Feature 1.3: Basenames and human-readable identity**

- As a bot owner, I want my bot to have a readable name (e.g., mybot.base.eth) resolving to its wallet.
- AC: Optional step during bot creation registers a Basenames; stored on the bot record; displayed in the DunkBot UI.

### Epic 2 — Runtime Verification

**Feature 2.1: ERC-8128 request signing**

- As a cross-tenant agent, I want every outbound request signed with the bot's root key so the receiving agent can verify it independently.
- AC: The gateway (or enclave signer) attaches an ERC-8128 signature to every cross-tenant request: payload hash, timestamp, nonce, agent ID.
- AC: Signature verification uses the public key from the ERC-8004 registry — no DunkBot API call required.
- AC: Replay protection via nonce + timestamp window (e.g., 5 minutes); expired or reused nonces rejected.

**Feature 2.2: SIWA flow**

- As a responding agent, I want to run a Sign-In-With-Agent challenge before fulfilling a request from an unknown bot.
- AC: SIWA challenge-response implemented: challenger issues a nonce; requester signs it with root key; challenger verifies against registry.
- AC: SIWA result cached per (requester agent ID, responder) with TTL to avoid repeated challenges within a session.

**Feature 2.3: Gateway signature validation**

- As the DunkBot gateway, I want to validate inbound cross-tenant request signatures before routing.
- AC: Inbound cross-tenant requests without a valid ERC-8128 signature are rejected with 401.
- AC: Validation is cached for the signature's validity window to reduce registry lookups.
- AC: Failed validations increment a per-agent failure counter; threshold triggers review flag.

### Epic 3 — Wallet and Payments

**Feature 3.1: Agentic Wallet integration**

- As a bot, I want a TEE-isolated wallet (Coinbase Agentic Wallets pattern or equivalent) so payments can be authorized without exposing keys.
- AC: Each production bot gets a wallet whose keys live in an enclave; the agent runtime holds only an enclave-mediated signing capability.
- AC: Wallet address is linked to the bot's ERC-8004 registration (same or associated address).

**Feature 3.2: x402 settlement**

- As a bot, I want to pay and be paid via x402 so commerce works over plain HTTP.
- AC: x402 middleware on the DunkBot gateway handles 402 challenges, payment proofs, and settlement confirmation.
- AC: Payments settle in USDC on Base; settlement tx hash recorded in the audit log attributed to both agent IDs.

**Feature 3.3: Spend caps and KYT**

- As a platform operator, I want programmable per-transaction and per-session spend limits plus Know-Your-Transaction screening.
- AC: Session cap and per-tx limit enforced at the wallet/enclave layer, not just in policy.
- AC: KYT screening (e.g., Chainalysis or similar) runs on inbound and outbound payments; flagged transactions held for review.
- AC: Cap configuration is per-bot, versioned, and audited.

### Epic 4 — Portable Cedar Policies

**Feature 4.1: Policy export**

- As an external agent, I want to download a machine-readable authorization policy for a bot so I can evaluate its permissions without calling DunkBot.
- AC: Each bot's effective Cedar policy set is exportable as JSON + Cedar text via an authenticated endpoint.
- AC: Export includes policy version and a hash so the evaluator can detect staleness.
- AC: Export is signed by the platform so tampering is detectable.

**Feature 4.2: Dogwood-style sequence governance**

- As a platform, I want Cedar to govern multi-step delegation chains (A delegates to B who calls C) across agent boundaries.
- AC: Dogwood (or equivalent sequence-aware Cedar extension) integrated into the interceptor chain.
- AC: Delegation depth and scope limits enforced; a delegated agent cannot exceed the delegator's grants.
- AC: Cross-tenant delegation requires the delegator's root-key signature on the delegation grant.

### Epic 5 — Reputation Layer

**Feature 5.1: Payment-weighted reputation**

- As an agent choosing a counterparty, I want reputation scores weighted by settled volume rather than raw review count.
- AC: Reputation service aggregates: settled USDC volume, completion rate, on-time rate; score is a function of these, not review count alone.
- AC: Scores are queryable via API and included in ERC-8004-adjacent metadata where supported.

**Feature 5.2: Review-ring detection**

- As a reputation system, I want to flag coordinated fake reviews.
- AC: Detection flags: agents sharing reviewer sets, mutual-review clusters, sibling agents (same owner/wallet family) reviewing each other.
- AC: Flagged reviews excluded from score computation; flag reason logged.

### Epic 6 — Migration and Governance

**Feature 6.1: Sepolia-to-mainnet promotion checklist**

- As a platform operator, I want a documented, testable checklist for promoting a bot from Sepolia to mainnet.
- AC: Checklist covers: enclave keygen verified, registration on mainnet, wallet funded with test USDC then real, spend caps set, KYT enabled, audit log verified, SIWA tested end-to-end.
- AC: Promotion is a gated workflow (approval + automated verification), not a manual contract call.

**Feature 6.2: Revocation flow**

- As a security operator, I want to revoke a compromised bot's root identity.
- AC: Revocation burns or flags the ERC-8004 registration; all delegated tokens for that agent ID are immediately invalidated.
- AC: Revocation is itself an audited, root-or-operator-signed action.
- AC: Post-revocation, the bot cannot register a new identity without operator review (prevents instant re-entry).

**Feature 6.3: Key rotation policy**

- As a platform, I want a clear policy: root keys are not rotated; compromise means revocation. Delegated credential signing keys inside the enclave may be rotated with a signed handoff.
- AC: Policy documented in ADR-002; enclave rotation procedure tested.

---

## 4. Design Records

### 4.1 PDR — Product Design Record: Agent Identity for DunkBot

**Problem:** Cross-tenant agent commerce requires durable, verifiable, revocable identity. Current STS-tag-based identity is per-attempt and internal-only.

**Solution:** Four-layer model (root identity on ERC-8004/Base, delegated credentials signed in TEE, portable Cedar policies, payment-weighted audit/reputation). Pragmatic crypto-native path over W3C DID/VC track for now; revisit ENISA alignment if EU compliance becomes a requirement.

**Sequencing:** Epic 1 (root identity) is the foundation. Epic 2 (verification) depends on 1. Epic 3 (wallets/payments) depends on 1, can parallel 2. Epic 4 (portable policies) depends on existing Cedar (done) and benefits from 2. Epic 5 (reputation) is independent, can start immediately. Epic 6 (migration/governance) depends on 1-3 being functional.

**Non-goals (v1):** W3C DID/VC issuance, on-chain reputation contracts (off-chain service with on-chain anchoring later), live trading authorization (paper/preview only until separate enablement).

**Risks:** Mainnet real-USDC exposure before TEE isolation is proven; reputation gaming; cross-tenant data leakage via registration files (mitigate: minimal public metadata, private endpoints via signed discovery).

### 4.2 ADRs

**ADR-001: ERC-8004 over W3C DIDs for root identity**

- Context: Two credible paths exist. W3C DIDs + VCs are the standards-body (ENISA/EU) route with institutional credibility but heavier infrastructure. ERC-8004 is crypto-native, live on Base, integrated with x402/Agentic Wallets/Basenames, and DunkBot already settles on Base.
- Decision: ERC-8004 for v1. Monitor ENISA/DID developments; design the registration file schema to be mappable to a DID document later if needed.
- Consequences: Tied to Base/Ethereum ecosystem; non-EVM agents need a bridge or separate identity.

**ADR-002: TEE-isolated keys over agent-held keys**

- Context: If the agent's private key exists in its prompt/context/workspace, prompt injection can exfiltrate it and drain the wallet. Coinbase Agentic Wallets (Feb 2026) established the enclave pattern as the reference.
- Decision: All root and wallet keys generated and used exclusively inside AWS Nitro Enclaves. Agent runtime never sees key material.
- Consequences: Enclave ops complexity (image builds, attestation verification); signing latency; need for enclave-mediated API for the agent to request signatures.

**ADR-003: Cedar over OPA for authorization**

- Context: OPA is the broader ecosystem choice; Cedar is purpose-built for AWS, already integrated in DunkBot, and Dogwood extends it to agent tool-call sequences.
- Decision: Stay on Cedar. Adopt Dogwood for sequence governance. Do not introduce OPA.
- Consequences: Less portable outside AWS; mitigated by the portable policy export (Feature 4.1).

**ADR-004: Mainnet for production, Sepolia for dev**

- Context: Testnet gives architecture for free but no real immutability or economic skin in the game. Mainnet gives the security properties but real USDC at risk.
- Decision: Sepolia for all development and CI. Mainnet promotion is a gated workflow (Feature 6.1) that requires enclave verification, spend caps, and KYT to be green.
- Consequences: Dual-registry ops; promotion checklist must be automated to avoid human error.

### 4.3 DDR — Data Design Record

The following are illustrative draft records, not finalized schemas. Empty arrays indicate fields left unspecified in the supplied draft.

**DDR-001: ERC-8004 registration file schema**

```json
{
  "agentId": "uint256 (ERC-721 token ID)",
  "name": "string",
  "description": "string",
  "serviceEndpoint": "https://gateway.dunkbot.ai/agents/{agentId}",
  "protocols": [],
  "publicKey": "hex (TEE-attested)",
  "attestation": "COSE_Sign1 (Nitro enclave doc)",
  "walletAddress": "0x...",
  "basename": "optional mybot.base.eth",
  "policyExportUrl": "https://gateway.dunkbot.ai/agents/{agentId}/policy",
  "policyVersion": "semver",
  "createdAt": "ISO-8601",
  "revokedAt": "null | ISO-8601"
}
```

**DDR-002: Run token claims (extended)**

```json
{
  "tenantId": "string",
  "agentId": "uint256 (root identity)",
  "conversationId": "string",
  "turnId": "string",
  "scope": [],
  "exp": "unix",
  "nbf": "unix",
  "nonce": "string",
  "signature": "hex (root key, enclave-signed)"
}
```

**DDR-003: Cedar policy export format**

```json
{
  "agentId": "uint256",
  "version": "semver",
  "hash": "sha256 of canonical policy text",
  "cedar": "permit(principal, action, resource) when { ... };",
  "json": { "...equivalent structured form...": "TBD" },
  "signedBy": "platform key",
  "exportedAt": "ISO-8601"
}
```

**DDR-004: Audit event schema**

```json
{
  "eventId": "uuid",
  "agentId": "uint256",
  "tenantId": "string",
  "action": "string (e.g., cross_tenant_pay)",
  "counterpartyAgentId": "uint256 | null",
  "amountUsdc": "string | null",
  "txHash": "string | null",
  "timestamp": "ISO-8601",
  "signature": "hex (enclave-signed)",
  "prevEventHash": "sha256 of previous event (hash chain)"
}
```

**DDR-005: Reputation event schema**

```json
{
  "eventId": "uuid",
  "subjectAgentId": "uint256",
  "reviewerAgentId": "uint256",
  "settledVolumeUsdc": "string",
  "completionRate": "0..1",
  "onTimeRate": "0..1",
  "ringFlag": "boolean",
  "ringReason": "string | null",
  "timestamp": "ISO-8601"
}
```

## 5. Build Order

### Sequential (hard dependencies)

1. Epic 1.2 (TEE keygen) → Epic 1.1 (ERC-8004 registration) → Epic 1.3 (Basenames)
2. Epic 1.1 → Epic 2.1 (ERC-8128 signing) → Epic 2.3 (gateway validation) → Epic 2.2 (SIWA)
3. Epic 1.2 → Epic 3.1 (Agentic Wallet) → Epic 3.2 (x402) → Epic 3.3 (spend caps/KYT)
4. Epic 1.1 + Epic 4.1 existing Cedar → Epic 4.1 (policy export) → Epic 4.2 (Dogwood sequences)
5. Epic 3.2 + Epic 2.1 → Epic 6.1 (mainnet promotion checklist) → Epic 6.2 (revocation) → Epic 6.3 (rotation policy)

### Parallel (independent)

- Epic 5 (reputation) can start immediately — no dependencies on 1-4. Build the reputation service and ring detection in parallel with Epic 1.
- Epic 4.1 (policy export) can start once existing Cedar policies are inventoried — no blockchain dependency.
- DDR schemas (section 4.3) should be finalized before Epic 1.1 implementation begins so the registration file and audit event shapes are stable.

### Suggested agent split

- Agent A: Epic 1 (root identity) — Solidity/Foundry for registry interaction, enclave image for keygen.
- Agent B: Epic 2 (runtime verification) — gateway middleware, SIWA, signature validation. Can start after Agent A's registration contract interface is defined.
- Agent C: Epic 3 (wallets/payments) — Agentic Wallet integration, x402 middleware, KYT hooks. Starts after Agent A's enclave signer exists.
- Agent D: Epic 4 (portable Cedar) — policy export endpoint, Dogwood integration. Independent start.
- Agent E: Epic 5 (reputation) — reputation service, ring detection, query API. Fully independent start.
- Agent F: Epic 6 (migration/governance) — promotion workflow, revocation, rotation. Starts when A-C have working prototypes.

All agents consume the DDR schemas (section 4.3) as their interface contract. Conflicts surface at the schema layer, not the implementation layer.

## 6. Open Questions and Risks

1. Enclave cost/latency: Nitro Enclave signing adds latency per request. Quantify before committing; consider batching or caching signed tokens.
2. Registry gas costs: ERC-8004 registration on mainnet costs gas. Budget per-bot or amortize.
3. x402 maturity: Protocol is young (165M tx by Apr 2026). Monitor for breaking changes; pin versions.
4. Reputation cold start: New agents have zero reputation. Need a bootstrap mechanism (e.g., platform-vouching or minimum settled volume) so new bots aren't permanently distrusted.
5. Cross-tenant privacy: Registration files are public on-chain. Sensitive bot metadata must stay out of the registration file — only what's needed for discovery.
6. Revocation UX: Burning an ERC-721 is irreversible. Need a clear operator workflow and possibly a “revoked” flag pattern instead of burn for v1.
7. DID migration path: If ENISA/EU requirements force W3C alignment later, the registration file schema (DDR-001) should be designed to map cleanly to a DID document. Flag for review in ADR-001's revisit.

## 7. Handoff Notes for Implementing Agents

- Read this document fully before starting. Section 4.3 (DDR) is the contract between parallel agents — do not diverge from it without an ADR.
- Existing DunkBot codebase: Bedrock AgentCore, AppSync, Terraform, EventBridge, DynamoDB, S3, Cedar policies, STS session tags, testnet wallet on Base Sepolia, immutable router contract. Extend; don't rewrite.
- Sepolia contracts and wallet flow already exist — Epic 1.1 should reuse the existing router/registry interaction patterns where possible.
- All new AWS resources go through Terraform. All new Lambda/AgentCore actions follow existing patterns (TenantContext, exact-action Cedar gating).
- Tests required per AC. CI via GitHub Actions (existing).
- Commit messages and PRs reference epic/feature IDs (e.g., “feat(epic-1.1): ERC-8004 registration on bot creation”).

End of master plan. Version 1.0 — 2026-10-31.
