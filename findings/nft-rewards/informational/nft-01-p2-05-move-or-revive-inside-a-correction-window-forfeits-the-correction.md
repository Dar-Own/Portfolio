# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Informational ceiling; privileged lifecycle sequencing note  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Move or revive inside a correction window forfeits the correction

## Description

Moving or reviving a collection can change the current vault identity while a prior round’s correction window is still open. Replaying that frozen snapshot then rejects the changed member identity, forfeiting that correction opportunity.

Prerequisites and limits: Requires privileged lifecycle sequencing. Operators can correct before moving/reviving or deliberately accept the forfeiture. Retirement alone does not change vaultId/roundCount and is not the same trigger. This does not establish permanent protocol-wide unavailability.

### Mechanism step by step

#### Step 1 — Lifecycle changes the collection’s current vault

Movement rewrites vaultId after leaving and joining the relevant member sets.

Source: `contracts/contracts/core/[project]Lifecycle.sol:157-164`.

```solidity
        _leaveVault($, from, fromVault, collection);
        _joinVault($, to, toVault, collection);

        uint40 memberSince = _nextGridCutoff(to.roundAnchor, to.roundInterval, uint40(block.timestamp));
        c.vaultId = toVault;
        c.memberSince = memberSince;
        c.pendingStatus = uint8(Status.None);
        c.statusFrom = 0;
```

#### Step 2 — Correction replays the frozen membership snapshot

Every member must still have the expected vaultId and collection round count; a mismatch rejects CorrectionWindowClosed.

Source: `contracts/contracts/core/[project]Rounds.sol:180-204`.

```solidity
    function _snapshotMembers(CoreStorage storage $, uint32 vaultId, uint32 correctedRoundId, RoundResult memory r)
        private
        view
    {
        uint64[] storage snapshot = $.vaultRoundMembers[vaultId][correctedRoundId];
        uint256 count = snapshot.length;
        CollectionRoundResult[] memory lines = new CollectionRoundResult[](count);
        for (uint256 i; i < count; ++i) {
            uint64 entry = snapshot[i];
            uint32 index = uint32(entry >> 32);
            uint32 collectionRoundId = uint32(entry);
            address col = $.collectionList[index];
            Collection storage c = $.collections[col];
            if (c.vaultId != vaultId || c.roundCount != collectionRoundId) revert CorrectionWindowClosed();
            CollectionRound storage stored = $.collectionRounds[col][collectionRoundId];
            CollectionRoundResult memory m = lines[i];
            m.collection = col;
            m.collectionIndex = index;
            m.eligibleCount = stored.eligibleCount;
            m.statusAtRound = stored.statusAtRound;
            m.xpharAtRound = stored.xpharAtRound;
            r.eligibleTotal += m.eligibleCount;
            r.xpharTotal += m.xpharAtRound;
        }
        r.collections = lines;
```

Additional source boundary:

Source: `contracts/contracts/core/[project]Lifecycle.sol:100-110`.

```solidity

        if (v.lastCutoff < $.collectionRounds[collection][c.roundCount].cutoff) revert CutoffMismatch();

        uint40 memberSince = _nextGridCutoff(v.roundAnchor, v.roundInterval, uint40(block.timestamp));

        c.status = uint8(Status.Active);
        c.vaultId = vaultId;
        c.memberSince = memberSince;

        _joinVault($, v, vaultId, collection);

```

## Proof of concept

Source establishes that movement/revival can invalidate a frozen correction snapshot. Privileged sequencing is required; no executable sequence or permanent protocol-wide outage is demonstrated.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
