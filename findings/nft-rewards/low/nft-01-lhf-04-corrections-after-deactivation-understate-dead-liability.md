# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Low ceiling; accounting-reporting drift candidate  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Corrections after deactivation understate dead liability

## Description

Deactivation books a token’s current unclaimed amount as dead liability. A subsequent correction for the same still-open cutoff can add claimable rewards and total liability without refreshing the global, collection and vault dead counters.

Prerequisites and limits: The proposed sequence requires privileged deactivation and correction reporting. Source supports a reporting inconsistency; no executable correction-after-deactivation proof is available. Total liability remains funded and correct, so insolvency is not established.

### Mechanism step by step

#### Step 1 — An administrator snapshots dead entitlement

Deactivation aggregates current unclaimed rewards and increments all three dead counters.

Source: `contracts/contracts/core/[project]Deactivation.sol:33-46`.

```solidity
        uint40 effectiveFrom = _nextGridCutoff(anchor, interval, SafeCast.toUint40(block.timestamp));

        uint32 roundCount = c.roundCount;
        uint128 unclaimedTotal = 0;
        for (uint256 i = 0; i < n; ++i) {
            unclaimedTotal += _deactivateToken($, collection, tokenIds[i], effectiveFrom, roundCount, anchor, interval, delay);
            uint32 newCount = c.deactivatedCount + 1;
            c.deactivatedCount = newCount;
            $.deactivations[collection].push(uint48(effectiveFrom), uint208(newCount));
        }

        $.collectionStats[collection].deadUnclaimedAvax += unclaimedTotal;
        if (c.status != uint8(Status.Retired)) $.vaultStats[c.vaultId].deadUnclaimedAvax += unclaimedTotal;
        $.deadLiability += unclaimedTotal;
```

#### Step 2 — A correction records additional per-token rewards

The member recorder increments cumulative reward at the correction cutoff. This path does not refresh the prior deactivation snapshot.

Source: `contracts/contracts/core/[project]Rounds.sol:289-315`.

```solidity
    function _recordMember(uint32 vaultId, uint32 vaultRoundId, RoundResult memory r, CollectionRoundResult memory m)
        private
        returns (uint64 entry)
    {
        CoreStorage storage $ = _getCoreStorage();
        address col = m.collection;
        Collection storage c = $.collections[col];
        uint32 n = c.roundCount + 1;
        c.roundCount = n;
        uint128 cum = $.collectionRounds[col][n - 1].cumRewardPerNft + m.perNft;
        int128 manualNet = c.manualNetSinceRound;
        c.manualNetSinceRound = 0;
        if (m.eligibleCount == 0) {
            int256 widened;
            unchecked {
                widened = int256(c.unattributedManualXphar) + int256(manualNet);
            }
            c.unattributedManualXphar = SafeCast.toInt128(widened);
        }
        CollectionRound storage record = $.collectionRounds[col][n];
        record.cutoff = r.cutoff;
        record.vaultId = vaultId;
        record.vaultRound = vaultRoundId;
        record.eligibleCount = m.eligibleCount;
        record.statusAtRound = m.statusAtRound;
        record.isCorrection = r.isCorrection;
        record.cumRewardPerNft = cum;
```

#### Step 3 — Total liability increases without dead-counter reconciliation

The round writer credits holder-funded liability, while claim-side reductions later cap each dead-counter decrement.

Source: `contracts/contracts/core/[project]Rounds.sol:250-257`.

```solidity
        VaultStats storage vs = $.vaultStats[vaultId];
        vs.totalRewardAvax += r.totalAvax;
        vs.totalHolderFundedAvax += r.holderFunded;
        vs.totalCompoundAvax += r.compoundAvax;
        vs.total[project]Avax += r.[project]Avax;
        vs.totalOtherAvax += r.otherAvax;
        vs.totalDivertedAvax += r.collectionDivertedAvax + r.vaultDivertedAvax;
        $.totalLiability += r.holderFunded;
```

Additional source boundary:

Source: `contracts/contracts/core/[project]Claims.sol:196-209`.

```solidity
    function _reduceDeadCounters(CoreStorage storage $, address collection, uint32 vaultId, uint128 amount) private {
        CollectionStats storage cs = $.collectionStats[collection];
        uint128 held = cs.deadUnclaimedAvax;
        cs.deadUnclaimedAvax = held - (amount < held ? amount : held);

        if ($.collections[collection].status != uint8(Status.Retired)) {
            VaultStats storage vs = $.vaultStats[vaultId];
            held = vs.deadUnclaimedAvax;
            vs.deadUnclaimedAvax = held - (amount < held ? amount : held);
        }

        held = $.deadLiability;
        $.deadLiability = held - (amount < held ? amount : held);
    }
```

## Proof of concept

The source transition increases claimable dead rewards after correction without updating the same dead counters. No executable correction-after-deactivation composition is supplied. Total liability remains funded; insolvency is not demonstrated.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
