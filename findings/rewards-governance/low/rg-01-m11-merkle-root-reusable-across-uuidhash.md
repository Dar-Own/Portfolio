# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: Low (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Original local test source; mechanism assertions shown below
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Merkle root reusable across uuidHash

## Description

Authorized backend resubmits same Merkle root under fresh uuidHash → no global used-root registry → fresh per-UUID claimed namespace → same distribution becomes claimable again.

Duplicate issuance for same leaf/batch; requires backend action, not ordinary user root submission.

Resolved at commit 1c2c2fd in current NMRP1. A contract-wide usedMerkleRoots registry is declared at contracts/NMRP1.sol:54-55. submitRewardData rejects a root used under any UUID and marks it consumed before reserving rewards at lines 159-195. The same Merkle distribution can no longer be registered as a second independently claimable batch. The contract cannot detect two different roots containing overlapping economic entitlements; off-chain root correctness remains an Authorized Party/backend control outside the same-root replay finding.

### Mechanism step by step

#### Step 1 — set-once is per uuidHash, not per root value

only merkleRoots[uuidHash]==0 is checked; the root value has no global uniqueness guard.

Source: `contracts/mintUscyRewards.sol:147-151` (commit 95fc21c).

```solidity
        // Allow setting a root only once (or re-setting before any claim)
        require(
            merkleRoots[uuidHash] == bytes32(0),
            "Set-once policy (cannot reset even before claims)"
        );
```

Source: `contracts/mintUscyRewards.sol:157-157` (commit 95fc21c).

```solidity
        merkleRoots[uuidHash] = newRoot;
```

#### Step 2 — dedup key is namespaced by uuidHash

no global used-root registry

Source: `contracts/mintUscyRewards.sol:216-216` (commit 95fc21c).

```solidity
            require(!hasClaimed[uuidHash][user][eid], "Already claimed");
```

## Proof of concept

The authorized backend submits the same root under two UUIDs. The same leaf then mints twice in separate claim namespaces. This requires trusted-backend action; ordinary users cannot register roots.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/helpers/deploy.ts`
- `PoCs/merkle-replay/M11_root_reuse_uuid.test.ts`
- `PoCs/merkle-replay/M11_root_reuse_uuid.test.ts:28-58`

## Relevant test excerpts

Source: `PoCs/merkle-replay/M11_root_reuse_uuid.test.ts:28-58`.

```typescript
  it("claims the same leaf under both uuids", async () => {
    const pool = await rewards.getAddress();

    await rewards
      .connect(core.attacker)
      .claimRewards(
        user,
        [uuid1],
        [amount],
        [built.accountIDHashes[0]],
        [built.proofs[0]],
        [pool],
        [1]
      );

    // Same root and leaf are accepted again under a fresh uuid namespace.
    await rewards
      .connect(core.attacker)
      .claimRewards(
        user,
        [uuid2],
        [amount],
        [built.accountIDHashes[0]],
        [built.proofs[0]],
        [pool],
        [1]
      );

    expect(await core.[project].balanceOf(user)).to.equal(amount * 2n);
  });
});
```
