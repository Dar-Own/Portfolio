# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Low ceiling; lens/operator guidance inconsistency  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# `pendingWork()` can recommend an impossible batch report

## Description

pendingWork can recommend a batch report based only on an activation-wallet shortfall even when the newly opened batch contains no AVAX. The writer then rejects the recommended action with EmptyBatch.

Prerequisites and limits: An external wallet balance change after a prior report is required. This concerns operator guidance and wasted attempts, not loss of user funds. The dedicated batch preview can distinguish the empty-batch blocker.

### Mechanism step by step

#### Step 1 — A completed report opens an empty batch

The writer creates the next batch with totalAvax=0 and resets the wallet baseline.

Source: `contracts/contracts/core/[project]BatchReporting.sol:52-66`.

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
```

#### Step 2 — The lens recommends reporting from shortfall alone

The pending-work decision derives batchReportDue and nextAction from wallet shortfall without the batch amount guard.

Source: `contracts/contracts/lens/LensDashboards.sol:148-159`.

```solidity
            lines[i].walletShortfall = _walletShortfall(v);
            lines[i].batchReportDue = lines[i].walletShortfall != 0;

            lines[i].compoundReportDue = v.pendingCompoundRounds != 0;

            Batch memory b =
                LensDecoders.decodeBatch(_loadRange(CoreLayout.batchSlot(v.openBatchId), CoreLayout.BATCH_WORDS));
            lines[i].openBatchAvax = b.totalAvax;

            lines[i].nextAction = lines[i].batchReportDue
                ? 1
                : (lines[i].compoundReportDue ? 2 : (lines[i].roundDue ? 3 : 0));
```

#### Step 3 — The writer rejects the empty batch

The real admission check requires a nonempty open batch.

Source: `contracts/contracts/core/[project]BatchReporting.sol:117-129`.

```solidity
    function _checkBatchReport(uint32 vaultId, uint128 xphar, bool confirmZero)
        internal
        view
        returns (Vault storage v, uint32 batchId)
    {
        _requireFlagClear(PAUSE_REPORTING);

        v = _vault(vaultId);

        batchId = v.openBatchId;
        if (_getCoreStorage().batches[batchId].totalAvax == 0) revert EmptyBatch();

        if (xphar == 0 && !confirmZero) revert ZeroAmount();
```

## Proof of concept

The lens recommendation and write admission predicate are compared in source. No executable empty-batch recommendation/write test is supplied.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
