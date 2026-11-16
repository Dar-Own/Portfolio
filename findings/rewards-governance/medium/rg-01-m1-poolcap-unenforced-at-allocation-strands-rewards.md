# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: Medium (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Original local test source; mechanism assertions shown below
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# poolCap unenforced at allocation strands rewards

## Description

Authorized pool reserves rewards through updateUnclaimedRewards, checking only the global supply bound → claimedRewards + unclaimedRewards exceeds poolCap → mint merely moves amounts from unclaimed to claimed, preserving the excessive sum → mint’s pool-cap check reverts every claim atomically.

Reward reservations stranded; contributor allocation path has its own cap and is excluded.

Resolved at commit 1c2c2fd in [project]MintContract.updateUnclaimedRewards. The allocation path now requires claimedRewards + unclaimedRewards + newTotal <= poolCap at contracts/[project]MintContract.sol:197-203 before increasing the reservation. The separate global dynamic-cap check remains at lines 204-210. A pool can no longer reserve rewards beyond its lifetime issuance budget.

### Mechanism step by step

#### Step 1 — allocation checks only global supplyCap

no `pool.poolCap` term; unclaimed can be raised above poolCap.

Source: `contracts/[project]MintContract.sol:160-167` (commit 95fc21c).

```solidity
        require(
            totalSupply() + totalUnclaimedRewards + newTotal <= supplyCap,
            "Allocation exceeds dynamic cap"
        );

        pool.unclaimedRewards += newTotal;
        totalUnclaimedRewards += newTotal;
        emit UnclaimedRewardsUpdated(pool.unclaimedRewards);
```

#### Step 2 — mint's poolCap check is on a mint-invariant sum

amount moves unclaimed->claimed, so claimed+unclaimed is unchanged by a mint; the check only ever fails AFTER over-allocation, bricking every mint.

Source: `contracts/[project]MintContract.sol:186-194` (commit 95fc21c).

```solidity
        pool.unclaimedRewards -= amount;
        pool.claimedRewards += amount;
        totalUnclaimedRewards -= amount;

        // ✅ Enforce combined cap
        require(
            pool.claimedRewards + pool.unclaimedRewards <= pool.poolCap,
            "Mint cap exceeded"
        );
```

## Proof of concept

A pool with a 100-token cap reserves 200 tokens. The allocation succeeds, but minting one token reverts with Mint cap exceeded. The test demonstrates stranded claims rather than theft; a separate contributor allocation path has its own cap.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/cap-accounting/M1_poolCap_unenforced_at_allocation.test.ts`
- `PoCs/cap-accounting/M1_poolCap_unenforced_at_allocation.test.ts:15-26`
- `PoCs/helpers/deploy.ts`

## Relevant test excerpts

Source: `PoCs/cap-accounting/M1_poolCap_unenforced_at_allocation.test.ts:15-26`.

```typescript
  it("accepts above-cap allocation then reverts mint", async () => {
    await core.[project].connect(core.attacker).updateUnclaimedRewards(pool, toWei(200));
    const data = await core.[project].rewardContracts(pool);

    // Allocation ignores poolCap; mint later enforces claimedRewards against it.
    expect(data.unclaimedRewards).to.equal(toWei(200));
    expect(data.unclaimedRewards).to.be.gt(data.poolCap);
    await expect(
      core.[project].connect(core.attacker).mint(pool, pool, toWei(1))
    ).to.be.revertedWith("Mint cap exceeded");
  });
});
```
