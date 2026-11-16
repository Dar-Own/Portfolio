# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Low ceiling; conditional accounting-drift hardening, related to LHF-04  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Saturating dead counters allow cross-level drift

## Description

The movement path subtracts at most the dead liability held by the source vault but adds the collection’s full dead counter to the destination. If collection dead already exceeds source-vault dead, movement can increase the sum of vault counters without the same global increase.

Prerequisites and limits: The cross-level mismatch is a prerequisite, not a proven result of LHF-04. LHF-04’s uniform undercount does not by itself establish collectionDead>vaultDead. No executable composition proof or naturally reachable cross-level mismatch was located. Total liability/solvency loss is not established.

### Mechanism step by step

#### Step 1 — A pre-existing counter mismatch reaches movement

moveCollection reads the collection’s dead amount, removes it from the source through a capped helper, then credits it in full to the destination.

Source: `contracts/contracts/core/[project]Lifecycle.sol:157-170`.

```solidity
        _leaveVault($, from, fromVault, collection);
        _joinVault($, to, toVault, collection);

        uint40 memberSince = _nextGridCutoff(to.roundAnchor, to.roundInterval, uint40(block.timestamp));
        c.vaultId = toVault;
        c.memberSince = memberSince;
        c.pendingStatus = uint8(Status.None);
        c.statusFrom = 0;

        uint128 dead = $.collectionStats[collection].deadUnclaimedAvax;
        _removeDeadFromVault($, fromVault, dead);
        $.vaultStats[toVault].deadUnclaimedAvax += dead;

        emit CollectionMoved(collection, fromVault, toVault, released, xpharArrived, memberSince);
```

#### Step 2 — The source subtraction saturates

The helper subtracts only the amount available in the source vault.

Source: `contracts/contracts/core/[project]Lifecycle.sol:252-257`.

```solidity
    function _removeDeadFromVault(CoreStorage storage $, uint32 vaultId, uint128 amount) private {
        VaultStats storage vs = $.vaultStats[vaultId];
        uint128 held = vs.deadUnclaimedAvax;

        vs.deadUnclaimedAvax = held - (amount < held ? amount : held);
    }
```

#### Step 3 — Claims also saturate each counter independently

Per-level saturation avoids underflow but can conceal a pre-existing mismatch instead of surfacing it.

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

Source establishes asymmetric capped subtraction and full addition only after a cross-level counter mismatch. That mismatch has not been demonstrated as a consequence of LHF-04. No executable composition or solvency loss is established.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
