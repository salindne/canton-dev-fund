## Development Fund Proposal

**Author:** ChainSafe Systems
**Status:** Submitted
**Created:** 2026-06-02
**Label:** wallet-apps

**[Champion](https://github.com/canton-foundation/canton-dev-fund/blob/main/sig-directory.md):** Viv Dikawar (Canton Foundation)

---

## Abstract

This proposal funds the continued development, security review, and long-term maintenance of a MetaMask-compatible middleware for the Canton Network. The middleware exposes an Ethereum JSON-RPC facade over CIP-56 token contracts, a distributed indexer for CIP-56 contract state, and an Ethereum–Canton bridge relayer. The result: any MetaMask user, any EVM dapp, indexer, or block explorer can transact against Canton-native tokens through the same RPC surface they already use on Ethereum, while Canton's privacy-preserving ledger model is preserved.

The work is delivered as a sequence of quarter-sized milestones — a CIP-56-compliant middleware, indexer, and bridge, then a non-custodial MetaMask Snap and an institutional custody path — followed by two years of maintenance covering CIP-0112 (Token Standard V2) migration, CVE response, and standards-tracking compatibility updates. A separate ecosystem-adoption plan, not tied to milestone payments, describes the path to real usage.

---

## Specification

### 1. Objective

Deliver an EVM-tooling-compatible interface to CIP-56 tokens on Canton so that any MetaMask user or EVM-native dapp, indexer, or block explorer can transact against Canton-native tokens through the JSON-RPC surface they already use on Ethereum, without bespoke client integration.

### 2. Implementation Mechanics

The middleware sits between EVM tooling and the Canton ledger. End users register as Canton **external parties** through an EIP-191-signed payload. The middleware's Ethereum JSON-RPC server (`pkg/ethrpc/`, `cmd/api-server/`) implements the `eth_*`, `net_*`, and `web3_*` namespaces required for MetaMask and the broader EVM tooling ecosystem (`eth_chainId`, `eth_blockNumber`, `eth_gasPrice`, `eth_estimateGas`, `eth_getBalance`, `eth_getTransactionCount`, `eth_getCode`, `eth_call`, `eth_sendRawTransaction`, `eth_getTransactionReceipt`, `eth_getTransactionByHash`, `eth_getLogs`, `eth_getBlockByNumber`, `eth_getBlockByHash`). ERC-20 operations (`transfer`, `transferFrom`, `approve`, `balanceOf`, `allowance`, `totalSupply`) are reachable through standard contract-call encoding via `eth_call` / `eth_sendRawTransaction`, exactly as on Ethereum.

The Transaction Orchestration Engine maps EVM calldata to Canton Ledger API commands against CIP-56 contracts, executes them through the **Interactive Submission API** (`PrepareSubmission` → sign → `ExecuteSubmission`), and returns an EVM-shaped receipt. The signer is a pluggable interface (`pkg/cantonsdk/token/types.go`: `SignDER(message []byte) ([]byte, error)`, `Fingerprint() (string, error)`), wired through a `KeyResolver` callback at API-server initialisation. The middleware first ships one concrete implementation backed by custodial secp256k1 keys with AES-256-GCM at-rest encryption (Milestone 2). Milestones 5–6 add Snap-backed and institutional-custody-backed signers behind the same interface, selectable per deployment, per tenant, or per user.

A Go-based distributed indexer (`pkg/indexer/`, `cmd/indexer/`) subscribes to the Canton Ledger API, processes CIP-56 token and bridge lifecycle events (Holding creations/archivals, TransferFactory choices, Offer events), and maintains deterministic UTXO-aggregated balance, holdings, and allowance views in PostgreSQL. The indexer is deployable per node so each operator runs its own indexer scoped to its visibility.

The bridge relayer (`pkg/relayer/`, `cmd/relayer/`) uses a generic `Processor` with `Source`/`Destination` adapters for both Canton and Ethereum, enabling bidirectional token movement. The Canton side is modelled in the `bridge-core` Daml package; the Ethereum side in Solidity. PROMPT (ERC-20 → CIP-56 holding) is the reference bridged asset.

Middleware deployment supports two operational modes:

- **Full Visibility Mode**: for tokens where Super Validators have full ledger visibility (e.g., Canton Coin), the middleware can be operated by validator nodes for globally accurate ERC-20 responses.
- **Scoped Visibility Mode**: for stablecoins or tokenised RWAs, the middleware is operated by entities with global visibility into the token (issuers) or by end users for personal visibility. `balanceOf` returns only addresses the operator is authorised to see, preserving Canton's privacy model.

Daml package layout: `cip56-token` (Token, TransferFactory, Events, Config, Compliance — implementing Splice HoldingV1 and TransferFactory with DNS-prefixed metadata keys, e.g., `splice.chainsafe.io/symbol`); `bridge-core` (lock/unlock and mint/burn state machine); `common/FingerprintAuth.daml` for external-party authorization; integration test packages validated against Canton mainnet.

### 3. Architectural Alignment

This work targets Q2 ecosystem priorities directly:

- **App Building & Developer Experience**: every existing EVM tool — MetaMask, Hardhat, Foundry, Ethers, Viem, Etherscan-style explorers — works against CIP-56 tokens without modification. The cost of bringing an EVM-native dapp to Canton becomes "point your RPC at a different endpoint."
- **Token & Asset Standards**: the implementation is built against the ratified **CIP-0056 Splice Token Standard** (HoldingV1, TransferFactory) and is structured for forward migration to **CIP-0112 (Token Standard V2)** via parallel packages (see Rationale § CIP-112 Migration Plan).
- **Stability & Maintainability**: the pluggable signer architecture decouples custody choice from protocol code, so swapping custodial → Snap → KMS → custody-partner is a deployment-time configuration change, not a fork.
- **Security & Resilience**: a third-party security audit covers every component (API server, relayer, indexer, Daml contracts, Snap) before the corresponding milestone is accepted, with re-audit triggers for V2 dual-interface delivery.

The middleware uses Canton's **Interactive Submission API** as the signer-pluggability seam, which is exactly the seam the API was designed to expose. It does not introduce new ledger semantics, does not modify Canton consensus, and does not alter the trust model for any party beyond the operator running the middleware itself.

### 4. Backward Compatibility

No backward compatibility impact on existing Canton flows. CIP-56 packages are deployed alongside any existing token packages on a participant. When CIP-0112 is ratified, V2 packages will be deployed in parallel with V1 per CIP-0112 §5.2 — V1 implementations continue to operate untouched, and the middleware's orchestrator selects the V1 vs V2 command builder per token at runtime based on the token's advertised compatibility.

---

## Milestones and Deliverables

The engineering work is broken into quarter-sized milestones (M1–M6), followed by recurring maintenance milestones applying quarter-on-quarter after initial delivery (M7–M10). Payment is released milestone-by-milestone on committee acceptance. Quarters are approximate; exact dates are fixed at each acceptance. Ecosystem adoption is addressed as a plan (see § Ecosystem Adoption Plan) rather than a funded milestone.

### Milestone 1: Architecture & Daml / Bridge Contracts

- **Estimated Delivery:** Q4 2026 (≈1 quarter from grant acceptance)
- **Focus:** Establish the on-ledger foundation — the system design and the Daml contracts every other component builds on.
- **Deliverables:**
  - **Architecture & Design Documentation**: system architecture diagrams (deployment topology + data flow), API interface schemas, security and privacy model covering both visibility modes.
  - **Daml CIP-56 + Bridge contracts**: `cip56-token` (Token, TransferFactory, Events, Config, Compliance), `bridge-core` (lock/unlock/mint/burn state machine), `common/FingerprintAuth.daml`, plus unit and integration test suites validated against Canton mainnet.
- **Acceptance Criteria:** architecture and security/privacy design documents published and open to committee review; `cip56-token` and `bridge-core` packages pass their test suites against Canton mainnet; a CIP-56 token transfer and allocation is demonstrable on DevNet.
- **Amount:** 3,000,000 CC upon committee acceptance.

### Milestone 2: Middleware JSON-RPC Service

- **Estimated Delivery:** Q4 2026 (≈1 quarter; overlaps the tail of M1)
- **Focus:** Ship the MetaMask-compatible Ethereum JSON-RPC server and the orchestration/identity layer that turns EVM calls into Canton submissions.
- **Deliverables:**
  - **Middleware Service**: Ethereum JSON-RPC API server, Transaction Orchestration Engine (EVM calldata → Canton Interactive Submission), identity/auth/registration (EIP-191 + external-party allocation + JWT session management), pluggable signer interface with the custodial implementation, Splice Registry client, Contract State Resolver, indexer integration adapter, and Docker/Kubernetes deployment artifacts.
- **Acceptance Criteria:** a stock MetaMask wallet connects to the middleware and completes a transfer against a CIP-56 token on DevNet through the `eth_*` surface; end-user registration (EIP-191 → external-party allocation → JWT session) works end-to-end; deployment artifacts published.
- **Amount:** 4,500,000 CC upon committee acceptance.

### Milestone 3: Indexer Service

- **Estimated Delivery:** Q1 2027 (≈1 quarter)
- **Focus:** Ship the distributed indexer that provides ERC-20-shaped balance, holdings, and allowance views over CIP-56 state.
- **Deliverables:**
  - **Indexer Backend Service**: Go-based indexer subscribing to the Canton Ledger API, PostgreSQL storage with deterministic UTXO aggregation, HTTP query API, per-node distributed deployment, and Docker/Kubernetes artifacts.
- **Acceptance Criteria:** indexer-reported balances, allowances, and total supply reconcile with the Canton ledger for a live CIP-56 token; the indexer runs per-node scoped to operator visibility; an EVM dapp or explorer successfully queries token state through indexer-backed responses.
- **Amount:** 2,500,000 CC upon committee acceptance.

### Milestone 4: Bridge, Relayer & Integration

- **Estimated Delivery:** Q1 2027 (≈1 quarter)
- **Focus:** Ship the Ethereum↔Canton bridge and relayer, wire the components together end-to-end, and release developer-facing documentation and deployment images.
- **Deliverables:**
  - **EVM ↔ Canton Bridge and Relayer**: relayer service with generic Source/Destination adapters, Solidity bridge contracts, bridge state store, PROMPT reference bridged asset, and local-bootstrap end-to-end test harness (Canton + Anvil + middleware + relayer).
  - **Integration Testing & Demo**: end-to-end functional test suite, Daml scenario tests, lightweight CLI/web demo application.
  - **Documentation & User Guide**: developer integration guide, full API reference, deployment guide.
- **Acceptance Criteria:** a bridged-asset round-trip (Ethereum → Canton → Ethereum) is demonstrated end-to-end; the end-to-end functional suite passes on the local-bootstrap harness; developer integration guide, API reference, and public Docker images / Kubernetes manifests are released.
- **Amount:** 4,000,000 CC upon committee acceptance.

_Milestones 1–4 constitute the core CIP-56 middleware, indexer, and bridge (formerly "Phase 1")._

### Milestone 5: Non-Custodial MetaMask Snap

- **Estimated Delivery:** Q2 2027 (≈1 quarter)
- **Focus:** Deliver retail non-custodial Canton signing inside MetaMask via a published Snap.
- **Deliverables:**
  - **Architecture & Design Refresh**: updated diagrams for each signer implementation; deterministic Canton-party derivation from MetaMask seed phrase with new-device recovery; per-key authorization policies for the institutional path; threat models for each signing surface; custody-partner evaluation framework with a go/no-go decision gate at week 4 of the custody workstream.
  - **MetaMask Snap for Canton Signing**: Ed25519 signing inside MetaMask's isolated origin, key material derived deterministically from the user's existing MetaMask seed phrase (so seed-phrase recovery also recovers the Canton party), no key material on the server or on disk outside MetaMask. Browser onboarding (install Snap → allocate external party → register with API server → recover on new device via standard MetaMask flow). Snap published to the MetaMask Snap registry with versioned signed updates.
- **Acceptance Criteria:** the Snap is published to the MetaMask Snap registry and installable by any user; a non-custodial transfer signed inside the Snap against a CIP-56 token is demonstrable; new-device recovery from the MetaMask seed phrase restores the Canton party.
- **Amount:** 3,000,000 CC upon committee acceptance.

### Milestone 6: Institutional Custody + Security Audit + Docs

- **Estimated Delivery:** Q2 2027 (≈1 quarter)
- **Focus:** Deliver the institutional custody path behind the same signer interface, complete the independent third-party audit of the new signing surfaces, and publish integrator documentation.
- **Deliverables:**
  - **Institutional Custody Integration (Track A or Track B)**: Track A (preferred) — partnership integration against an established custody provider (evaluation set: Fireblocks, BitGo, Anchorage Digital, Copper, plus any further providers identified). Track B (fallback if no partnership reached on commercially or technically acceptable terms within the first four weeks of the workstream) — in-house KMS-backed signer against AWS KMS as primary target, abstracted so a second provider can be added without rework. Both tracks terminate at the same signer interface.
  - **Integration Testing, Security Review, and Demo**: end-to-end tests across custodial / Snap / institutional / mixed-mode deployments; **independent third-party security audit** of the Snap (BIP-44 derivation correctness, DER signing flow, dialog prompts, supply-chain posture, manifest permission scope) and of the institutional integration; findings remediated before acceptance; audit reports published under `docs/audits/`.
  - **Documentation & Integrator Guides**: a reference dapp per signer mode (custodial, Snap, institutional), Snap user guide, Snap integration guide for dapp developers, institutional custody deployment guide, updated API reference, per-path threat-model summaries suitable for risk-conscious integrators.
- **Acceptance Criteria:** at least 1 issuer live on the institutional custody path; independent third-party audit report covering the Snap and institutional integration published under `docs/audits/` with all critical/high findings remediated; integrator guides and per-path threat-model summaries published.
- **Amount:** 3,000,000 CC upon committee acceptance.

_Milestones 5–6 constitute the non-custodial Snap and institutional custody path (formerly "Phase 2")._

### Milestone 7: Maintenance Year 1 — CIP-0112 (Token Standard V2) Migration

- **Estimated Delivery:** Q2 2028
- **Focus:** Ship CIP-0112 (Token Standard V2) dual-interface support alongside V1.
- **Deliverables:**
  - **CIP-0112 V2 dual-interface delivery**: V1 and V2 packages running in parallel via Daml module-prefixes (per CIP-0112 §5.2); V2-aware indexer decoder; V2 command builder in the middleware orchestrator selected per token at runtime; bridge updates to use V2 non-holder Account destinations for mint/burn. Timeline is conditional on CIP-0112 ratification — see contingency in Rationale § CIP-112 Migration Plan.
- **Acceptance Criteria:** V1 and V2 dual-interface support live across the middleware, indexer, and bridge; existing V1 issuers continue operating untouched; a V2 token transfer is demonstrable once CIP-0112 is ratified (subject to the ratification-timing contingency).
- **Amount:** 5,000,000 CC upon committee acceptance.

### Milestone 8: Maintenance Year 1 — Sustainment

- **Estimated Delivery:** Q4 2028
- **Focus:** Ongoing security and compatibility upkeep for the first maintenance year. Recurring maintenance milestone applying quarter-on-quarter.
- **Deliverables:**
  - CVE response and security patches on a triage cadence aligned with severity.
  - Compatibility tracking: timely updates as CIP-56 / CIP-112 standards and Canton mainnet evolve.
- **Acceptance Criteria:** zero unresolved P0/P1 CVEs at end of period; at least 2 issuers sustained in production; compatibility maintained across at least one Canton mainnet upgrade during the period.
- **Amount:** 5,000,000 CC upon committee acceptance.

### Milestone 9: Maintenance Year 2 — Sustainment (First Half)

- **Estimated Delivery:** Q2 2029
- **Focus:** Continued upkeep, standards tracking, and security patching. Recurring maintenance milestone applying quarter-on-quarter.
- **Deliverables:**
  - Ongoing security patches and CVE response.
  - Continued tracking of CIP-56 / CIP-112 standards evolution with timely compatibility updates.
  - Critical bug fixes affecting correctness of token operations, bridge flows, or signing paths.
- **Acceptance Criteria:** zero unresolved P0/P1 CVEs at end of period; sustained production use maintained; public Docker images and the MetaMask Snap remain available throughout.
- **Amount:** 5,000,000 CC upon committee acceptance.

### Milestone 10: Maintenance Year 2 — Sustainment (Second Half)

- **Estimated Delivery:** Q4 2029
- **Focus:** Final maintenance period; continued upkeep, standards tracking, and security patching.
- **Deliverables:**
  - Ongoing security patches and CVE response.
  - Continued standards-evolution tracking and compatibility updates.
  - Critical bug fixes affecting correctness of token operations, bridge flows, or signing paths.
- **Acceptance Criteria:** sustained production use by at least 3 issuers; zero unresolved P0/P1 CVEs at end of period; published artifacts (Docker images, Snap registry listing, documentation) remain publicly available.
- **Amount:** 5,000,000 CC upon final release and acceptance.

---

## Acceptance Criteria

Each milestone above states its own acceptance criteria. Whenever a milestone is claimed, the Tech & Ops Committee assesses the claim against that milestone's stated criteria and votes on continuation before the next milestone's funding is budgeted. Across all milestones, evaluation is based on:

- Deliverables completed as specified for the milestone
- Demonstrated functionality and operational readiness through working reference deployments
- Documentation and knowledge transfer provided
- Alignment with the milestone's stated ecosystem-value criteria

Acceptance is based on value delivered to the ecosystem, not on artifact delivery in isolation. Audit findings (where applicable) must be remediated before the corresponding milestone is accepted.

---

## Funding

**Total Funding Request:** 40,000,000 CC (40M Canton Coin)

### Payment Breakdown by Milestone

| Milestone | Amount |
| :---- | ----: |
| M1 — Architecture & Daml / Bridge Contracts | 3,000,000 CC |
| M2 — Middleware JSON-RPC Service | 4,500,000 CC |
| M3 — Indexer Service | 2,500,000 CC |
| M4 — Bridge, Relayer & Integration | 4,000,000 CC |
| M5 — Non-Custodial MetaMask Snap | 3,000,000 CC |
| M6 — Institutional Custody + Security Audit + Docs | 3,000,000 CC |
| M7 — Maintenance Year 1: CIP-0112 (V2) Migration | 5,000,000 CC |
| M8 — Maintenance Year 1: Sustainment | 5,000,000 CC |
| M9 — Maintenance Year 2: Sustainment (First Half) | 5,000,000 CC |
| M10 — Maintenance Year 2: Sustainment (Second Half) | 5,000,000 CC |
| **Total** | **40,000,000 CC** |

All milestone payments are made upon committee acceptance of the milestone (M10 upon final release and acceptance). Build milestones (M1–M6) total 20,000,000 CC; maintenance milestones (M7–M10) total 20,000,000 CC. Ecosystem adoption is presented as a plan (see § Ecosystem Adoption Plan) and carries no funding; should the committee request an explicit adoption weighting, it would be funded by reallocating from the maintenance milestones.

### Volatility Stipulation

The project duration is greater than 6 months. The grant is denominated in fixed Canton Coin and will require a re-evaluation at the 6-month mark.

---

## Co-Marketing

Upon release, ChainSafe will collaborate with the Foundation on:

- **Announcement coordination**: joint launch communications for each milestone, particularly the MetaMask Snap registry listing in Milestone 5.
- **Case study and technical blog**: published walkthroughs of the EVM-tooling-against-Canton experience, the Snap UX, and integration patterns for issuers.
- **Developer and ecosystem promotion**: integration guides, reference dapps per signer mode, and developer outreach through ChainSafe's existing ecosystem channels.

---

## Ecosystem Adoption Plan

Adoption is presented here as a plan of intent, not a funded milestone — no token reward or acceptance gate is tied to these targets. The aim is a credible path to real ecosystem usage of the middleware once the engineering milestones (M1–M6) land:

- **Reference deployments per signer mode** — publicly accessible reference dapps for the custodial, Snap, and institutional signing modes, so integrators can evaluate the middleware against a live endpoint.
- **Issuer onboarding** — work with CIP-56 issuers (stablecoins, tokenised RWAs, Canton-native assets) to route real token activity through the middleware on MainNet.
- **Wallet / dapp integration** — enable external wallets and EVM-native dapps to reach Canton tokens through the JSON-RPC surface and the Splice Registry, beyond ChainSafe's own reference dapps.
- **Snap distribution** — drive installs of the MetaMask Snap from the registry as the non-custodial on-ramp for retail users.
- **Reporting** — track and publish adoption signals (issuers live, integrations shipped, bridged-asset activity, Snap installs) as evidence of ecosystem value.

Should the committee wish to formalise an adoption weighting, ChainSafe is open to converting these targets into a funded, attestable milestone, reallocated from the maintenance budget.

---

## Motivation

**Ecosystem impact.** MetaMask has the dominant retail wallet share on EVM networks. Every CIP-56 issuer that wants retail reach today must either build a bespoke wallet integration or accept that their token is unreachable from the wallet most users already have installed. This middleware closes that gap for every CIP-56 token on Canton — not by changing Canton, but by speaking the protocol every existing EVM tool already speaks.

**Portion of ecosystem benefiting.** The work is directly applicable to:

- **Every CIP-56 token issuer** seeking EVM-tool reachability — stablecoins (e.g., USDCx), tokenised RWAs, future Canton-native assets. Universal benefit across the CIP-56 issuer population.
- **Every EVM-native dapp, indexer, and block explorer** that today does not support Canton — the JSON-RPC facade means they don't need to. Hardhat, Foundry, Ethers, Viem, Etherscan-style explorers, EVM-native analytics platforms all become Canton-aware through configuration alone.
- **Retail users** for whom MetaMask is the only wallet they will install. The non-custodial Snap (Milestone 5) converts this from a custodial bridge to a fully non-custodial Canton signing experience inside MetaMask, derived from the user's existing seed phrase.
- **Institutional issuers and counterparties** requiring audited external custody. The institutional path (Milestone 6) is the same signer interface against either a custody partner or a cloud KMS, selectable per deployment.

**Strategic importance.** Canton's privacy model and EVM-tool reachability are typically presented as a trade-off. This middleware demonstrates they aren't: the same RPC surface MetaMask uses can be served against Canton without compromising the privacy-scoped visibility model — the operator simply returns the answers it is authorised to give, exactly as Canton's trust model requires.

---

## Rationale

### Why a JSON-RPC facade rather than a custom Canton SDK

The EVM tool ecosystem (wallets, indexers, explorers, dapp frameworks) is enormous and converged on the Ethereum JSON-RPC surface. A facade lets every one of those tools work against Canton at the cost of one server-side translation layer. A custom SDK requires every tool to be re-integrated; the cost is paid by every dapp and tool author indefinitely. The facade pays the cost once.

### Why the Canton Interactive Submission API for signer pluggability

Interactive Submission separates command preparation from signing. That separation is the natural seam for swapping signer implementations: the orchestrator prepares a Canton submission, hands the prepared payload off to a signer (custodial / Snap / partner / KMS) for a DER signature, and submits the signed result. The protocol does not need to change when the signer changes. This is why the middleware ships a custodial signer first (Milestone 2) and adds Snap + institutional signers behind the same interface (Milestones 5–6).

### Why a dual-track institutional path

Custody is the single biggest determinant of whether an institutional issuer will adopt a system. A partner integration is lower-risk (established compliance posture, faster path to enterprise adoption) but is contingent on commercial and technical alignment. Beginning the custody workstream with a structured 4-week partner evaluation, with an in-house KMS-backed signer as the documented fallback against the same signer interface, guarantees an institutional path is delivered regardless of partnership outcome (Milestone 6). The budget envelope is identical for either track.

### CIP-112 (Token Standard V2) migration

CIP-0112 is a backwards-compatible evolution of CIP-0056 introducing a new EventLog interface, committed allocations and iterated settlement, non-holder Account destinations, privacy-preserving batch settlement, and a pause-status metadata flag. V2 ships as new major-version `splice-api-token-*` packages alongside V1.

Per-layer reuse estimates for V2 delivery:

| Layer | V1 → V2 reuse | Coupling | Notes |
| :---- | :---- | :---- | :---- |
| Daml `cip56-token` packages | ~40% | Tight | Field names and interface instantiations are V1-bound; contract logic patterns survive. |
| Indexer (`pkg/indexer/engine/`) | ~70% | Medium-loose | Decoder pattern is structural; event field renames are the main lift. |
| Middleware orchestration (`pkg/ethrpc/`, `pkg/transfer/`) | ~60% | Medium | Module/entity/choice names are V1-specific; the prepare/execute flow is V-agnostic. |
| Bridge contracts (`bridge-core`) | ~30% | Tight | Mint/burn calls are V1-coupled; flow patterns survive. |

V1 and V2 packages run in parallel via Daml module-prefixes. The indexer gains a V2-aware decoder reusing the existing decoder pattern with V2 field names. The middleware orchestrator gains a V2 command builder alongside the V1 builder, selected per token at runtime based on the token's advertised compatibility (per CIP-0112 §5.2). Core dual-interface effort: 4–6 weeks of engineering with the parallel-packages approach.

**Contingency:** if CIP-0112 is ratified before the Snap + custody milestones (M5–M6) complete, V2 dual-interface delivery is pulled forward into that window in coordination with Foundation timing, funded out of the M5–M6 budget envelope. If ratification slips, V2 work is delivered in Milestone 7 (Maintenance Year 1 — CIP-0112 Migration) as scoped.

### Audit Policy and Cadence

The middleware includes security-critical components: an API server brokering signing and Canton submission, a relayer holding and moving bridge funds, an indexer that serves as the source of truth for balance queries, on-chain Daml contracts custodying value, and (in Milestone 5) a MetaMask Snap loaded into the user's wallet. The proposal commits to a comprehensive audit posture across the entire stack.

**Pre-release audit for the MetaMask Snap** (before any release to the MetaMask Snap registry) covers: key derivation against BIP-44 `m/44'/60'/1'/0/0`; DER signing flow and absence of key/signature leakage to the host page; dialog-prompt UX (no signing without explicit confirmation, accurate transaction information); supply-chain posture (`package.json`, lockfile, pinned cryptographic dependencies, build reproducibility); `snap.manifest.json` permission scope (`snap_getEntropy`, `snap_dialog`, `snap_manageState`).

**Full middleware stack audit** (parallel with Snap audit) covers: API server (JSON-RPC surface, Interactive Submission orchestration, session/auth, EIP-191 registration, custodial key handling, AES-256-GCM at-rest storage); relayer (bidirectional engine, bridge state machine, nonce tracking, chain-reorg handling); indexer (event ingestion correctness, deterministic aggregation, visibility-scoped query semantics); Daml contracts (reviewed by a Daml-fluent auditor — Digital Asset, Sygnum, or equivalent); cross-component integration as a single attack surface.

Findings remediated before the corresponding milestone is accepted. Full audit reports published under `docs/audits/`, linked from package READMEs and the Snap registry listing. Reproducible-build instructions published alongside each release.

### Licensing and Open-Source Posture

All grant-funded deliverables are released under **Apache License 2.0** with full source on public GitHub repositories. No component is proprietary, source-available-only, or otherwise restricted.

License coverage per subtree:

- Go middleware (`cmd/`, `pkg/`): Apache 2.0
- Daml contracts (`contracts/canton-erc20/daml/`, including `cip56-token`, `bridge-core`, `bridge-wayfinder`, `common`): Apache 2.0
- Solidity bridge contracts (`contracts/ethereum-wayfinder/`, `contracts/canton-erc20/ethereum/`): Apache 2.0
- MetaMask Snap (`canton-snap` repository): Apache 2.0
- Documentation, deployment templates, and reference apps: Apache 2.0

**Out of scope (operational infrastructure, not grant deliverables):** ChainSafe-operated hosted services such as Canton DevNet endpoints, Auth0 tenants, OAuth client secrets, per-tenant deployment credentials, and any third-party RPC accounts. Docker images, Kubernetes manifests, and configuration templates that describe deployments are in scope as source artifacts; the running deployments are not.

**Third-party dependency disclosure:** a transitive dev/test dependency (`halmos-cheatcodes` under `openzeppelin-contracts` test tooling) is licensed under AGPL-3.0. The dependency is build-time only, used in Solidity test suites, and is not bundled into any distributed binary or contract artifact. The dependency will be replaced or vendor-isolated before Milestone 4 acceptance to keep the project's effective license footprint Apache-2.0-compatible end-to-end.

All released packages ship `SPDX-License-Identifier` headers, a root `LICENSE` file containing the Apache 2.0 text, and a `NOTICE` file enumerating third-party licenses.

### Long-Term Sustainment (Post-Year 2)

Beyond Maintenance Year 2, ChainSafe commits to ongoing best-effort upkeep as open infrastructure: security patches against disclosed CVEs, critical correctness fixes affecting token operations / bridge flows / signing paths, and tracking of CIPs affecting CIP-56 / CIP-112 semantics. This baseline is funded by ChainSafe out of pocket and continues indefinitely. GitHub repositories remain public and Apache 2.0-licensed; Docker images and the MetaMask Snap registry listing remain available; documentation remains accessible. ChainSafe will not lock out downstream operators, relicense, or remove published artifacts.

Two options remain open for the Foundation and ecosystem to consider closer to end of Year 2 — Foundation-funded maintainer rotation, or a ChainSafe paid-SLA tier for issuers requiring guaranteed response times — neither of which is a precondition for the baseline sustainment commitment.
