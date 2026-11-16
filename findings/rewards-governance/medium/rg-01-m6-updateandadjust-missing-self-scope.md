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

# updateAndAdjust missing self-scope

## Description

Whitelisted pool receives AUTHORIZED_PARTY_ROLE → calls updateAndAdjust(newMA) using authority shared with global cap management → supplied moving-average input adjusts protocol-wide cap thresholds/cap without separate governance authority.

Pool role changes protocol-wide issuance capacity; local test moved cap 1B to 500M.

Resolved at commit 894c36f. updateAndAdjust remains restricted to CAP_MANAGER_ROLE at contracts/[project]MintContract.sol:164-191. The deployment script passes the actual timelock to the minter constructor at scripts/deploy.ts:426-435, asserts the timelock holds the role at lines 922-925, and removes the deployer's copy at lines 731-735. The governance suite executes updateAndAdjust through a proposal and timelock while confirming unauthorized callers and reward pools lack cap-management authority.

### Mechanism step by step

#### Step 1 — Pool role crosses the global cap boundary

A governance-admitted pool holds AUTHORIZED_PARTY_ROLE. updateAndAdjust uses that same role to accept the global moving-average input.

Source: `contracts/[project]MintContract.sol:119-132` (commit 95fc21c).

```solidity
    function updateAndAdjust(
        uint256 newMA
    ) external onlyRole(AUTHORIZED_PARTY_ROLE) {
        // Clamp instead of revert so huge jumps don’t brick the adjustment
        uint256 boundedMA = newMA;
        if (boundedMA > MAX_REASONABLE_MA) {
            emit MAClamped(newMA, MAX_REASONABLE_MA);
            boundedMA = MAX_REASONABLE_MA;
        }

        uint256 oldMA = movingAvg180d;
        movingAvg180d = boundedMA;
        emit MovingAvgUpdated(oldMA, boundedMA);

```

#### Step 2 — The supplied value irreversibly advances cap thresholds

The loop records processed thresholds and lowers the common cap, affecting every pool rather than only the caller.

Source: `contracts/[project]MintContract.sol:134-144` (commit 95fc21c).

```solidity
        for (uint256 i = 1; i <= 10; i++) {
            if (
                !thresholdProcessed[i] && boundedMA >= i * 1_000_000_000 ether
            ) {
                thresholdProcessed[i] = true;
                uint256 oldCap = supplyCap;
                supplyCap = (oldCap > TARGET_HARD_CAP + DEDUCTION)
                    ? oldCap - DEDUCTION
                    : TARGET_HARD_CAP;
                emit CapAdjusted(oldCap, supplyCap, i);
            }
```

## Proof of concept

A whitelisted pool with a cap of 1 invokes updateAndAdjust with a 20-billion moving-average input. The test asserts the highest threshold is processed and the global cap falls from 1 billion to 500 million. This requires a role granted by governance.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/cap-accounting/M6_updateAndAdjust_unscoped.test.ts`
- `PoCs/cap-accounting/M6_updateAndAdjust_unscoped.test.ts:15-25`
- `PoCs/helpers/deploy.ts`

## Relevant test excerpts

Source: `PoCs/cap-accounting/M6_updateAndAdjust_unscoped.test.ts:15-25`.

```typescript
  it("lets a pool process all thresholds", async () => {
    const auth = await core.[project].AUTHORIZED_PARTY_ROLE();
    expect(await core.[project].hasRole(auth, pool)).to.equal(true);
    expect(await core.[project].supplyCap()).to.equal(toWei(1_000_000_000));

    await core.[project].connect(core.attacker).updateAndAdjust(toWei(20_000_000_000));

    expect(await core.[project].thresholdProcessed(10)).to.equal(true);
    expect(await core.[project].supplyCap()).to.equal(toWei(500_000_000));
  });
});
```
