# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: Medium (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Original local test source; mechanism assertions shown below
Remediation: Remediated — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Reverting Trustline engine bricks claims

## Description

Claimant invokes either pool claim → configured Trustline engine reverts → mandatory external check lacks recovery replacement in original deployment → issuance reverts before state changes.

Affected reward issuance unavailable; original mock establishes [project] coupling, not real engine outage.

Remediated at commit 894c36f and strengthened at a79e61b. Both reward pools expose governance-controlled replaceValidationEngine functions at contracts/NMRP1.sol:309-331 and contracts/[project]ContributorAllocations.sol:157-177. The live deployment script now computes the IValidationEngine ERC-165 interface identifier and aborts unless the selected Trustline proxy advertises it at scripts/deploy.ts:1060-1087. The supplied Sepolia attestation records erc165Compatible=true; equivalent production evidence and a governed recovery exercise were not supplied.

### Mechanism step by step

#### Step 1 — requireTrustline gates the claim before any mint (uscy)

Claimant invokes either pool claim

Source: `contracts/mintUscyRewards.sol:192-195` (commit 95fc21c).

```solidity
        // NEW: Trustline-validate the *user* (recipient), not msg.sender
        address[] memory addrs = new address[](1);
        addrs[0] = user;
        requireTrustline(addrs);
```

#### Step 2 — same gate in the contributor pool

configured Trustline engine reverts

Source: `contracts/[project]ContributorAllocations.sol:118-121` (commit 95fc21c).

```solidity
      // NEW: Trustline-validate the *user* (recipient), not msg.sender
        address[] memory addrs = new address[](1);
        addrs[0] = user;
        requireTrustline(addrs);
```

## Proof of concept

A configurable validation mock is set to revert. Both reward claim paths revert before minting, and balances and claimed-reward accounting remain zero. This establishes dependency coupling, not a real validation-service outage.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/helpers/deploy.ts`
- `PoCs/trustline-liveness/M9_reverting_engine_bricks_claims.test.ts`
- `PoCs/trustline-liveness/M9_reverting_engine_bricks_claims.test.ts:40-76`

## Relevant test excerpts

Source: `PoCs/trustline-liveness/M9_reverting_engine_bricks_claims.test.ts:40-76`.

```typescript
  it("reverts USCY and contributor claims before mint", async () => {
    const uuid = uuidFromString("M9");
    const built = buildMerkle([{ user, accountId: 1, amount }]);
    const pool = await rewards.getAddress();

    await rewards
      .connect(core.authorizedParty)
      .submitRewardData(amount, [], built.root, uuid);

    // Engine failure happens before mint/accounting updates.
    await expect(
      rewards
        .connect(core.attacker)
        .claimRewards(
          user,
          [uuid],
          [amount],
          [built.accountIDHashes[0]],
          [built.proofs[0]],
          [pool],
          [1]
        )
    ).to.be.revertedWith("engine revert");
    expect(await core.[project].balanceOf(user)).to.equal(0n);
    expect((await core.[project].rewardContracts(pool)).claimedRewards).to.equal(0n);

    await contributor.addMember(user, amount);
    await expect(
      contributor.connect(core.attacker).claim(user, amount)
    ).to.be.revertedWith("engine revert");
    expect(await core.[project].balanceOf(user)).to.equal(0n);
    expect(
      (await core.[project].rewardContracts(await contributor.getAddress()))
        .claimedRewards
    ).to.equal(0n);
  });
});
```
