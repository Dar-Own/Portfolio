# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: Medium (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Original local test source; partial proof with limits below
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Governance capture is cheap

## Description

Council participant consolidates transferable/delegated voting weight → reaches proposal threshold 1 and quorum max(40% of past supply, 3) → no independent-wallet constraint prevents one controller supplying the votes → proposal reaches Succeeded.

Low effective voting threshold; original PoC proves threshold only, not full executed takeover.

Resolved at commit 968adfd. [project]CouncilToken now issues one non-transferable, non-delegable voting entitlement per whitelisted council wallet: transfers revert at contracts/[project]CouncilToken.sol:133-157, and voting power is derived directly from each wallet's balance at lines 159-175. The proposal threshold is intentionally one at contracts/[project]GoverningCouncil.sol:73-80, so any council wallet may initiate governance, while the minimum quorum remains three at lines 107-117 and successful proposals remain subject to the 48-hour timelock enforced at lines 66-70. One wallet can no longer consolidate the effective quorum through token transfers or delegation.

### Mechanism step by step

#### Step 1 — settings. proposalThreshold = 1, quorum fraction 40%

__GovernorSettings_init(delay=8, period=450, threshold=1); 40% quorum fraction.

Source: `contracts/[project]GoverningCouncil.sol:73-81` (commit 95fc21c).

```solidity
        __Governor_init("[project]GoverningCouncil");
        // e.g. 8 block delay, 450 block voting period, 1 proposal threshold
        __GovernorSettings_init(8, 450, 1);
        __GovernorCountingSimple_init();
        __GovernorStorage_init();
        __GovernorVotes_init(_token);
        // 40% quorum with a minimum override of 3 votes by default
        __GovernorVotesQuorumFraction_init(40);
        __GovernorTimelockControl_init(_timelock);
```

#### Step 2 — hard quorum floor of 3

quorum() never drops below 3; with MAX_SUPPLY 9, 3 votes always meet quorum.

Source: `contracts/[project]GoverningCouncil.sol:115-117` (commit 95fc21c).

```solidity
        uint256 q = super.quorum(blockNumber);
        if (q < 3) return 3;
        return q;
```

#### Step 3 — council supply cap is 9

no independent-wallet constraint prevents one controller supplying the votes

Source: `contracts/[project]CouncilToken.sol:24-24` (commit 95fc21c).

```solidity
    uint256 public constant MAX_SUPPLY = 9;
```

The threshold test uses a three-member council and a no-op proposal. It reaches Succeeded; it does not execute an administrative takeover through the timelock.

## Proof of concept

A three-member council votes on a no-op proposal. The test asserts quorum=3 and proposal state=Succeeded. It does not queue or execute an administrative takeover, and the fixture does not establish the production distribution of voting control.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/deploy-governance/M2_three_vote_governance_capture.test.ts`
- `PoCs/deploy-governance/M2_three_vote_governance_capture.test.ts:13-43`
- `PoCs/helpers/deploy.ts`

## Relevant test excerpts

Source: `PoCs/deploy-governance/M2_three_vote_governance_capture.test.ts:13-43`.

```typescript
  it("succeeds with three for votes", async () => {
    const targets = [await deployed.token.getAddress()];
    const values = [0n];
    const calldatas = ["0x"];
    const description = "M2 three vote proposal";

    const tx = await deployed.governor
      .connect(deployed.members[0])
      .propose(targets, values, calldatas, description);
    const receipt = await tx.wait();
    const proposalId = await proposalIdFromReceipt(
      receipt,
      deployed.governor,
      targets,
      values,
      calldatas,
      description
    );

    await mineBlocks((await deployed.governor.votingDelay()) + 1n);
    const snapshot = await deployed.governor.proposalSnapshot(proposalId);
    expect(await deployed.governor.quorum(snapshot)).to.equal(3n);

    for (const member of deployed.members) {
      await deployed.governor.connect(member).castVote(proposalId, 1);
    }

    await mineBlocks((await deployed.governor.votingPeriod()) + 1n);
    expect(await deployed.governor.state(proposalId)).to.equal(4);
  });
});
```
