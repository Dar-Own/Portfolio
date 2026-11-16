# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Low ceiling; operational false-green health signal  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Reconciliation failure does not make the indexer unhealthy

## Description

Indexer reconciliation can record that indexed data is incomplete while health still returns HTTP 200. Deployment checks that use that endpoint alone may admit an incomplete projection.

Prerequisites and limits: This is source-grounded readiness semantics. No actual deployment rollout with bad data or downstream financial action is demonstrated; poller and projection failures still affect health.

### Mechanism step by step

#### Step 1 — Reconciliation records a mismatch

The periodic check stores its own reconciliation result and logs failure.

Source: `indexer/src/main.ts:119-145`.

```typescript
const reconciled = {
  ok: null as boolean | null,
  checkedAt: null as string | null,
  mismatches: [] as Mismatch[],
  lastError: null as string | null,
  running: false,
};

const reconcileProvider = new JsonRpcProvider(config.rpcUrls[0]!, config.chainId, { staticNetwork: true });

async function runReconcile(): Promise<void> {
  if (reconciled.running) return;
  reconciled.running = true;
  try {
    const problems = await reconcile(pool, reconcileProvider, true);
    reconciled.ok = problems.length === 0;
    reconciled.mismatches = problems;
    reconciled.checkedAt = new Date().toISOString();
    reconciled.lastError = null;
    if (problems.length > 0) {
      log("error", { msg: "RECONCILIATION FAILED — the index is incomplete", mismatches: problems });
    }
  } catch (e) {

    reconciled.lastError = e instanceof Error ? e.message : String(e);
    log("error", { msg: "reconciliation could not run", error: reconciled.lastError });
  } finally {
```

#### Step 2 — Health ignores the reconciliation verdict

The healthy flag depends on consecutive poller/projection failures, even though the response also exposes the separate reconciled object.

Source: `indexer/src/main.ts:223-247`.

```typescript
  if (path === "/health" || path === "/") {

    const healthy = isHealthy(poller.status.consecutiveFailures, derived.consecutiveFailures);
    res.writeHead(healthy ? 200 : 503, { "content-type": "application/json" });

    res.end(
      JSON.stringify(
        {
          healthy,
          version: version(),
          chainId: config.chainId,
          indexed: { ...poller.status, lastError: classifyError(poller.status.lastError) },
          derived: { ...derived, lastError: classifyError(derived.lastError) },
          reconciled: { ...reconciled, lastError: classifyError(reconciled.lastError) },
          endpoints: rpc.status(),
          deployments: DEPLOYMENTS.map((d) => ({ id: d.id, core: d.core })),

          section: {
            schema: section.schema,
            core: section.core,
            chainId: section.chainId,
            how: section.how,
            claimedAt: section.claimedAt,
            publicHeld: section.publicHeld,
          },
```

#### Step 3 — API readiness uses progress rather than reconciliation

The status endpoint computes ready from projection existence and block progress.

Source: `indexer/src/api.ts:326-358`.

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

        headLag: headLag(),
        headLagRecent: [...runtime.headLagRecent],
        lastTickAt: runtime.lastTickAt,
        derivedAt: runtime.derivedAt,
        unknownEvents,
```

## Proof of concept

Source shows reconciliation failures are caught while the health response remains successful. No live outage or executable failure-injection test is supplied.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
