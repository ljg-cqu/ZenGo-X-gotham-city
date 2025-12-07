# Remediation Roadmap & Compliance

## Priority-Based Action Plan

| Priority | Action | Finding(s) | Category | Effort | Timeline |
|----------|--------|-----------|----------|--------|----------|
| **Immediate** | Treat this report as a **partial baseline**; schedule a fuller audit with expanded coverage (incl. MPC engine crates and gotham-client/demo-wallet) | — | Process / Coverage | 1–2 days | This quarter |
| **Short-term** | Replace the stubbed `granted` authorization hook with concrete policy logic and tests | F-002 | Business Logic / AuthZ | 1–3 days | Before any production use |
| **Short-term** | Define environment-specific Rocket configurations and deployment guidance (local/test vs production bindings, TLS/fronting) | F-001 | Configuration / Infrastructure | 1–2 days | Before any production use |
| **Short-term** | Design and implement an at-rest protection and retention strategy for RocksDB-backed MPC state, or migrate to a hardened datastore | F-003 | Data Security | 3–5 days | Before handling real-value assets |
| **Medium-term** | Integrate automated dependency and RustSec scanning into CI for all workspace crates | — | Dependencies / Supply Chain | 2–4 days | Next 1–2 sprints |
| **Medium-term** | Expand tests to cover authorization decisions, failure paths, and MPC flows end-to-end | F-002, F-003 (future) | Testing / Robustness | 1–2 weeks | Next 1–2 sprints |
| **Long-term** | Perform separate, in-depth review (or obtain third-party audits) of `two-party-ecdsa` and `gotham-engine` and map any issues back into gotham-city | — | Crypto / Architecture | Project-level | Next major cycle |

## Remediation Notes

### F-001 – Rocket Configuration
- Separate dev/test and production Rocket configs.
- For production, avoid exposing debug profiles directly to the internet; expect a reverse proxy or ingress with TLS and auth in front of Rocket.

### F-002 – Authorization Hook
- Treat `granted` as a sample only; implement real transaction-approval policies and tests before any real-world deployment.

### F-003 – RocksDB Storage
- Decide whether protocol state is considered sensitive; if yes, layer encryption-at-rest and access controls or move to a hardened external datastore.

---

## High-Level Compliance Mapping (Indicative Only)

This partial audit does **not** provide a full compliance mapping. The table below is illustrative and should be expanded in a future, more complete audit.

### OWASP Top 10 2021 (Indicative)

| Rank | Category | Findings | Status |
|------|----------|----------|--------|
| A05 | Security Misconfiguration | F-001 | Open |
| A02 | Cryptographic Failures | F-003 (design gap around storage hardening; engine not reviewed) | Open |

### CWE (Indicative)

| CWE | Title | Findings | Status |
|-----|-------|----------|--------|
| CWE-16 | Configuration | F-001 | Open |
| CWE-200 (context-dependent) | Exposure of Sensitive Information | F-003 (potential, depending on stored content) | Open (design gap) |

> A complete audit should refine these mappings, add dependency-related F-IDs if any, and cover engine-level issues once those crates are reviewed.
