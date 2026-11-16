# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: Medium (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Original local test source; mechanism assertions shown below
Remediation: Acknowledged — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# One pool reserves global headroom

## Description

Authorized pool with sufficient own cap → reserves all shared global headroom via updateUnclaimedRewards → no protected per-pool budget → other pools cannot allocate.

Cross-pool allocation denial; current design acknowledged first-come competition and operational recovery.

Acknowledged at commit 894c36f. The client confirms that reward pools intentionally compete on a first-come basis for the single global totalUnclaimedRewards headroom under supplyCap at contracts/[project]MintContract.sol:186-210; individual pool caps do not guarantee a protected share. Aggregate capacity planning, continuous reservation monitoring, and governance deactivation and reallocation are assigned to operational controls. Pool and global caps remain enforced, while releaseUnclaimedRewards at lines 214-233 and pool deactivation provide recovery after reservations consume available headroom.

### Mechanism step by step

#### Step 1 — aggregate-only check, no per-pool ceiling (same site as M1 SNIPPET 1)

Authorized pool with sufficient own cap

Source: `contracts/[project]MintContract.sol:160-166` (commit 95fc21c).

```solidity
        require(
            totalSupply() + totalUnclaimedRewards + newTotal <= supplyCap,
            "Allocation exceeds dynamic cap"
        );

        pool.unclaimedRewards += newTotal;
        totalUnclaimedRewards += newTotal;
```

## Proof of concept

Pool A is admitted with a 1-billion cap and reserves the common headroom. Pool B’s next allocation reverts. The test uses sufficient authority and capacity for pool A; a small-cap variant additionally depends on the allocation-cap defect.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/helpers/deploy.ts`
- `PoCs/pool-capture-auth/M7_global_headroom_starvation.test.ts`
- `PoCs/pool-capture-auth/M7_global_headroom_starvation.test.ts:18-28`

## Relevant test excerpts

Source: `PoCs/pool-capture-auth/M7_global_headroom_starvation.test.ts:18-28`.

```typescript
  it("blocks later allocation by another pool", async () => {
    const supplyCap = await core.[project].supplyCap();

    await core.[project].connect(core.attacker).updateUnclaimedRewards(poolA, supplyCap);
    expect(await core.[project].totalUnclaimedRewards()).to.equal(supplyCap);

    await expect(
      core.[project].connect(core.victim).updateUnclaimedRewards(poolB, 1)
    ).to.be.revertedWith("Allocation exceeds dynamic cap");
  });
});
```
