# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Low ceiling; wrong-backend detection gap  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# The app ignores the indexer’s direct Core identity

## Description

The API exposes its direct Core identity in section.core, but the frontend’s BackendFacts type omits it and backendView relies on metadata deployment records. Empty metadata can classify a wrong-Core backend as silent instead of otherDeployment.

Prerequisites and limits: A backend/metadata mismatch is required. No production user was shown transacting against a wrong deployment; chain-id and available metadata checks remain.

### Mechanism step by step

#### Step 1 — The API returns its section identity

Status supplies the section Core along with readiness information.

Source: `indexer/src/api.ts:326-352`.

```typescript
      const section = await pool.query<{ schema: string; core: string | null }>(
        `select current_schema() as schema, (select '0x' || encode(core, 'hex') from indexed_core) as core`
      );

      const cursor = Math.max(0, ...state.rows.map((r) => Number(r.lastBlock)));
      const builtTo =
        projections.rows.length === 0
          ? 0
          : Math.min(...projections.rows.map((r) => Number(r.builtTo)));
      const { unknownEvents, ...metaCounts } = meta.rows[0] as Record<
        string,
        unknown
      > & { unknownEvents: number };

      return {
        cursors: state.rows,

        section: section.rows[0],
        projections: projections.rows,
        counts: counts.rows[0],
        meta: metaCounts,

        chainBlock: cursor,
        projectionBlock: builtTo,
        blocksBehind: Math.max(0, cursor - builtTo),

        ready: projections.rows.length > 0 && builtTo > 0,
```

#### Step 2 — The frontend facts omit that field

BackendFacts carries cursors and projection progress only.

Source: `app/src/admin/instance.ts:64-79`.

```typescript
export interface BackendFacts {
  readonly cursors: readonly { readonly id: string }[];
  readonly projectionBlock: number;
}

export type BackendAnswer =
  | { readonly ok: true; readonly data: BackendFacts }
  | { readonly ok: false; readonly reason: string };

export type BackendView =
  | { readonly kind: "reading" }
  | { readonly kind: "unreachable"; readonly reason: string }
  | { readonly kind: "otherChain"; readonly theirChain: number; readonly ourChain: number }
  | { readonly kind: "otherDeployment"; readonly theirs: readonly string[] }
  | { readonly kind: "silent"; readonly builtTo: number }
  | { readonly kind: "agrees"; readonly builtTo: number };
```

#### Step 3 — Empty metadata leads to a silent result

backendView compares metadata addresses and returns silent when that list is empty.

Source: `app/src/admin/instance.ts:97-118`.

```typescript
  const { health, metaStatus, metaDeployments } = input;
  if (health === undefined) return { kind: "reading" };
  if (!health.ok) return { kind: "unreachable", reason: health.reason };

  const theirChain = chainOfCursors(health.data.cursors);
  if (theirChain !== null && theirChain !== input.chainId) {
    return { kind: "otherChain", theirChain, ourChain: input.chainId };
  }

  if (metaStatus === "loading") return { kind: "reading" };
  if (metaStatus === "unreachable") {
    return {
      kind: "unreachable",
      reason: input.metaReason ?? "the descriptions could not be read",
    };
  }

  const core = input.core.toLowerCase();
  const theirs = [...new Set(metaDeployments.map((d) => d.toLowerCase()))];
  if (theirs.length === 0) return { kind: "silent", builtTo: health.data.projectionBlock };
  if (theirs.includes(core)) return { kind: "agrees", builtTo: health.data.projectionBlock };
  return { kind: "otherDeployment", theirs };
```

## Proof of concept

Source shows the frontend accepts selected backend identity fields without comparing the supplied core field. No successful attacker-controlled backend substitution is demonstrated.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
