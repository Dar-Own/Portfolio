# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Low to Medium operational ceiling; wrong-deployment configuration risk  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# The indexer does not bind its configured Lens to its configured Core

## Description

The indexer validates that Core and Lens configuration strings are addresses but does not verify the Lens points to that Core. Counter reconciliation can compare one deployment’s logs with another deployment’s Lens and falsely pass when counts coincide.

Prerequisites and limits: Requires operator configuration error or control of configuration. Matching counts are a constructed false-negative possibility, not an executable wrong-deployment demonstration.

### Mechanism step by step

#### Step 1 — An operator supplies an address pair

Configuration checks presence and address syntax and constructs one Deployment record.

Source: `indexer/src/deployments.ts:17-38`.

```typescript
const core = (process.env.INDEXED_CORE ?? "").trim();
const lens = (process.env.INDEXED_LENS ?? "").trim();
const fromBlock = Number((process.env.INDEXED_FROM_BLOCK ?? "0").trim());

if ((core === "") !== (lens === "")) {
  throw new Error(
    "INDEXED_CORE and INDEXED_LENS must be set together: they describe ONE deployment, and " +
      "the reconciliation check reads the Lens of the Core it indexes.",
  );
}
if (core !== "" && !/^0x[0-9a-fA-F]{40}$/.test(core)) {
  throw new Error(`INDEXED_CORE is not an address: ${core}`);
}
if (lens !== "" && !/^0x[0-9a-fA-F]{40}$/.test(lens)) {
  throw new Error(`INDEXED_LENS is not an address: ${lens}`);
}
if (core !== "" && (!Number.isInteger(fromBlock) || fromBlock < 0)) {
  throw new Error("INDEXED_FROM_BLOCK must be a whole number when INDEXED_CORE is set");
}

export const DEPLOYMENTS: readonly Deployment[] =
  core === "" ? [] : [{ id: "primary", label: "Primary", core, lens, deployedAtBlock: fromBlock }];
```

#### Step 2 — Reconciliation counts events for configured Core

The SQL query keys logs by Core, while the contract client uses the independently supplied Lens.

Source: `indexer/src/reconcile.ts:21-35`.

```typescript
  {
    for (const deployment of DEPLOYMENTS) {
      const address = deployment.core.toLowerCase().replace(/^0x/, "");

      const { rows } = await pool.query<{ event_name: string; n: string }>(
        "select event_name, count(*)::text as n from event_log where address = decode($1,'hex') group by 1",
        [address],
      );
      const counts = new Map(rows.map((r) => [r.event_name, Number(r.n)]));
      const total = [...counts.values()].reduce((a, b) => a + b, 0);

      const lens = new Contract(deployment.lens, LENS_ABI as unknown as string[], provider);

      const onChainCollections = Number(await lens.collectionListLength!());
      const indexedCollections = counts.get("CollectionAdded") ?? 0;
```

#### Step 3 — Coincident counts can hide mismatched identity

Collection and activation comparisons inspect counts, not the Lens-to-Core binding.

Source: `indexer/src/reconcile.ts:34-57`.

```typescript
      const onChainCollections = Number(await lens.collectionListLength!());
      const indexedCollections = counts.get("CollectionAdded") ?? 0;
      if (onChainCollections !== indexedCollections) {
        problems.push({
          what: `${deployment.id}: collections`,
          onChain: String(onChainCollections),
          indexed: String(indexedCollections),
        });
      }

      let onChainActivations = 0;
      for (let i = 0; i < onChainCollections; i++) {
        const collection = (await lens.collectionListAt!(i)) as string;
        const record = await lens.getCollection!(collection);
        onChainActivations += Number(record.activatedCount);
      }
      const indexedActivations = counts.get("Activated") ?? 0;
      if (onChainActivations !== indexedActivations) {
        problems.push({
          what: `${deployment.id}: activations`,
          onChain: String(onChainActivations),
          indexed: String(indexedActivations),
        });
      }
```

## Proof of concept

Source configuration validation does not bind the Lens to its configured Core. A misleading coincident-count result is conditional; no executable wrong-deployment test is supplied.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
