# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Low ceiling; trusted-operator boundary hardening  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# `reportBatch()` forgives activation-wallet shortfall silently

## Description

A successful batch report resets expectedWalletBalance to the activation wallet’s actual current balance. Existing shortfall is thereby forgiven without a dedicated event field giving the old baseline or forgiven delta.

Prerequisites and limits: A populated reportable batch and reporting authority are required. Moving funds from the external activation wallet alone does not grant the ability to call reportBatch. The reset is within the trusted operator/custody model; no keyless theft or complete destruction of historical on-chain evidence is established. Prior state and events can reconstruct the discrepancy.

### Mechanism step by step

#### Step 1 — An authorized actor reports a populated batch

The function is operator/admin-only, and its admission path checks batch state rather than requiring wallet shortfall to be zero.

Source: `contracts/contracts/core/[project]BatchReporting.sol:20-29`.

```solidity
    function reportBatch(uint32 vaultId, uint128 xphar, bool confirmZero, string calldata memo)
        external
        onlyRoleOrAdmin(OPERATOR_ROLE)
        nonReentrant
    {
        (Vault storage v, uint32 batchId) = _checkBatchReport(vaultId, xphar, confirmZero);
        CoreStorage storage $ = _getCoreStorage();
        Batch storage batch = $.batches[batchId];

        (address[] memory contributors, uint256[] memory shares) = _contributorShares(batchId, xphar);
```

#### Step 2 — The successful report reanchors the expected balance

After booking and opening a new batch, expectedWalletBalance becomes actual balance; the event includes the new value but not the prior baseline.

Source: `contracts/contracts/core/[project]BatchReporting.sol:52-67`.

```solidity
        uint32 newBatchId = ++$.batchCount;
        v.openBatchId = newBatchId;
        $.batches[newBatchId] = Batch({
            vaultId: vaultId,
            status: uint8(BatchStatus.Open),
            openedAt: SafeCast.toUint40(block.timestamp),
            reportedAt: 0,
            totalAvax: 0,
            xpharReported: 0
        });

        uint128 walletBalanceAfter = SafeCast.toUint128(v.activationWallet.balance);
        v.expectedWalletBalance = walletBalanceAfter;

        emit BatchReported(vaultId, batchId, batchTotalAvax, xphar, walletBalanceAfter, memo);
    }
```

#### Step 3 — Other reporting paths use the reset baseline

Compound reporting’s _requireNotShort illustrates the gate whose reference value has changed.

Source: `contracts/contracts/core/[project]CompoundReporting.sol:34-46`.

```solidity
    function _checkReport(uint32 vaultId, uint32 roundId, uint128 xphar, bool confirmZero)
        internal
        view
        returns (uint256 toConvert)
    {
        _requireFlagClear(PAUSE_REPORTING);

        Vault storage v = _vault(vaultId);

        _requireNotShort(v);

        if (roundId == 0 || roundId > v.roundCount) revert UnknownRound(vaultId, roundId);

```

## Proof of concept

Source establishes that a successful authorized batch report reanchors the expected wallet balance. The required custody/reporting privileges and nonempty batch remain prerequisites; no executable shortfall-forgiveness sequence is supplied.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
