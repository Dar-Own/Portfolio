# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Low — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Source/configuration analysis
Remediation: Remediated (removal; rotation not established) — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Hardcoded RPC Authentication Tokens in Source

## Description

Repository reader obtains embedded RPC authentication URLs from three CLI files and uses provider access; credentials are not separated from source or rotated by source removal.

Potential provider rate-limit exhaustion/billing and operational RPC denial; no signing-key or on-chain authority exposure established. Secret values omitted.

No executable abuse PoC or provider-billing receipt is established. Evidence is source inspection and the final report, which also names cli/src/test_close_position.ts:7. The final report marks removal Remediated and separately calls for rotation; rotation was not established. Signing keys and on-chain authority are not exposed by an RPC credential alone.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Read the embedded provider credential

A repository reader can obtain an RPC authentication URL from source. The endpoint and token are intentionally redacted in this excerpt; no provider request is needed to establish their presence.

Source: `cli/src/test_close_short.ts:7-7`

```typescript
const RPC = "[redacted RPC endpoint and credential]";
```

#### Step 2 — Fall back silently to the embedded endpoint

When RPC_URL is unset, this CLI chooses the credential-bearing fallback. Source removal does not revoke copies already present in history.

Source: `cli/src/find_instant_sellers.ts:27-27`

```typescript
        rpc: process.env.RPC_URL || "[redacted RPC endpoint and credential]",
```

## Proof of concept

No executable exploit test is available for this case. The evidence is source/configuration inspection and the historical assessment.

No executable abuse PoC or provider-billing receipt is established. Evidence is source inspection and the final report, which also names cli/src/test_close_position.ts:7. The final report marks removal Remediated and separately calls for rotation; rotation was not established. Signing keys and on-chain authority are not exposed by an RPC credential alone.

## Relevant test excerpts

No finding-specific executable test excerpt is available. The source excerpts above are the supporting evidence.
