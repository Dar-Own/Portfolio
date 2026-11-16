# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Low ceiling; conditional can-check/write parity gap  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# `canRetireCollection()` omits the membership-list check

## Description

canRetireCollection checks lifecycle status and operational gates but not the vault membership index. If a separate registry inconsistency exists, it can succeed while the write’s _leaveVault rejects the missing membership.

Prerequisites and limits: No current unprivileged path to status/member-index desynchronization is known. A test that corrupts storage would demonstrate conditional behavior, not establish a reachable exploit. The preview lens separately checks bookkeeping consistency.

### Mechanism step by step

#### Step 1 — A caller asks the retirement can-check

The module returns _checkRetire.

Source: `contracts/contracts/[project]AdminModule.sol:24-26`.

```solidity
    function canRetireCollection(address collection) external view returns (uint32 vaultId) {
        (, vaultId) = _checkRetire(collection);
    }
```

#### Step 2 — The preliminary lifecycle guard passes

Status, open batch, settlement and round gates omit memberIndexPlusOne.

Source: `contracts/contracts/core/[project]Lifecycle.sol:21-33`.

```solidity
    function _checkRetire(address collection) internal view returns (Collection storage c, uint32 vaultId) {
        c = _collection(collection);

        _requireLiveMember(c);

        vaultId = c.vaultId;
        CoreStorage storage $ = _getCoreStorage();
        if ($.batchAvax[$.vaults[vaultId].openBatchId][collection] != 0) revert OpenBatchNotEmpty();

        _requireSettled(vaultId);

        _requireNotRoundDue(vaultId);
    }
```

#### Step 3 — The write checks the missing membership index

Leaving the vault rejects a zero index.

Source: `contracts/contracts/core/[project]Lifecycle.sol:220-235`.

```solidity

        emit XpharAdjusted(adjustmentId, collection, vaultId, reason, delta, refId, newXphar, "");
    }

    function _leaveVault(CoreStorage storage $, Vault storage v, uint32 vaultId, address collection) private {
        uint32 indexPlusOne = $.memberIndexPlusOne[vaultId][collection];
        if (indexPlusOne == 0) revert NotVaultMember(collection, vaultId);

        address[] storage members = $.vaultMembers[vaultId];
        uint256 index = uint256(indexPlusOne) - 1;
        uint256 lastIndex = members.length - 1;
        if (index != lastIndex) {
            address moved = members[lastIndex];
            members[index] = moved;
            $.memberIndexPlusOne[vaultId][moved] = uint32(index + 1);
        }
```

## Proof of concept

The can-check and write guards differ in source. Reaching a state where lifecycle status and membership index disagree requires a separate, unproved registry failure. No executable composition test is supplied.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
