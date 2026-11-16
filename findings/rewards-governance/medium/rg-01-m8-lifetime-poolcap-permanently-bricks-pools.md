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

# Lifetime poolCap permanently bricks pools

## Description

Normal issuance eventually consumes the pool’s lifetime poolCap → user burns/spending reduce circulating supply but never claimedRewards → further minting fails because no cap increase/reset or same-address re-whitelisting restores capacity; migration to a new pool address is the documented recovery.

Pool lifecycle exhaustion despite zero circulating balance; source identifies design decision, no attacker prerequisite.

Acknowledged at commit 1c2c2fd. [project]MintContract.RewardContract defines poolCap and claimedRewards as lifetime issuance values. The migration runbook documents governance deactivation, release of unclaimed reservations, and replacement at a new reviewed address. NMRP1.voidBatch releases unused batch reservations at contracts/NMRP1.sol:333-357. Burns intentionally do not restore pool capacity.

### Mechanism step by step

#### Step 1 — claimedRewards only ever increments; checked against poolCap

Normal issuance eventually consumes the pool’s lifetime poolCap

Source: `contracts/[project]MintContract.sol:187-187` (commit 95fc21c).

```solidity
        pool.claimedRewards += amount;
```

Source: `contracts/[project]MintContract.sol:191-194` (commit 95fc21c).

```solidity
        require(
            pool.claimedRewards + pool.unclaimedRewards <= pool.poolCap,
            "Mint cap exceeded"
        );
```

#### Step 2 — burn reduces supply but NOT claimedRewards

user burns/spending reduce circulating supply but never claimedRewards

Source: `contracts/[project]MintContract.sol:201-204` (commit 95fc21c).

```solidity
    function burn(uint256 amount) public override {
        super._burn(_msgSender(), amount);
        emit TokensBurned(_msgSender(), amount);
    }
```

#### Step 3 — spend-to-receiver burns, also without touching claimedRewards

further minting fails because no cap increase/reset or same-address re-whitelisting restores capacity; migration to a new pool address is the documented recovery.

Source: `contracts/[project]MintContract.sol:106-111` (commit 95fc21c).

```solidity
        if (from != address(0) && to == spendReceiver) {
            super._update(from, address(0), value);
            emit TokensBurned(from, value);
            emit PointsSpent(from, value);
            return;
        }
```

## Proof of concept

The pool mints its full 100-token lifetime cap and burns all circulating tokens through the spending route. Assertions show zero circulating supply but an exhausted lifetime counter, followed by a failed new mint. No malicious actor is needed.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/cap-accounting/M8_lifetime_poolCap_brick.test.ts`
- `PoCs/cap-accounting/M8_lifetime_poolCap_brick.test.ts:15-35`
- `PoCs/helpers/deploy.ts`

## Relevant test excerpts

Source: `PoCs/cap-accounting/M8_lifetime_poolCap_brick.test.ts:15-35`.

```typescript
  it("keeps claimedRewards at poolCap after supply is burned", async () => {
    await core.[project]
      .connect(core.attacker)
      .updateUnclaimedRewards(pool, toWei(100));
    await core.[project].connect(core.attacker).mint(pool, pool, toWei(100));
    expect((await core.[project].rewardContracts(pool)).claimedRewards).to.equal(
      toWei(100)
    );

    await core.[project]
      .connect(core.attacker)
      .transfer(await core.spendReceiver.getAddress(), toWei(100));
    expect(await core.[project].totalSupply()).to.equal(0n);

    // claimedRewards is lifetime accounting, so burned supply does not reopen cap.
    await core.[project].connect(core.attacker).updateUnclaimedRewards(pool, toWei(1));
    await expect(
      core.[project].connect(core.attacker).mint(pool, pool, toWei(1))
    ).to.be.revertedWith("Mint cap exceeded");
  });
});
```
