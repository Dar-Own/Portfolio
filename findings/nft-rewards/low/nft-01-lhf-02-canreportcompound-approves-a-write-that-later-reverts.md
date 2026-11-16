# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Low ceiling; confirmed can-check/write inconsistency  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Historical local or fork test excerpts; scope and fixture assumptions are described below  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# `canReportCompound()` approves a write that later reverts

## Description

The named canReportCompound preflight can return success although reportCompound will deterministically reject the same positive amount with NoWeights. The pending conversion duty remains and blocks subsequent rounds until a permitted recovery is used.

Prerequisites and limits: The reporting operation is privileged. The dashboard preview already flags BLOCK_NO_PRINCIPAL; only consumers that rely on the narrower Core preflight miss this condition.

### Mechanism step by step

#### Step 1 — An operator queries the can-check

The module delegates the read to _checkReport without running the later attribution.

Source: `contracts/contracts/[project]CompoundModule.sol:19-25`.

```solidity
    function canReportCompound(uint32 vaultId, uint32 roundId, uint128 xphar, bool confirmZero)
        external
        view
        returns (uint256 toConvert)
    {
        toConvert = _checkReport(vaultId, roundId, xphar, confirmZero);
    }
```

#### Step 2 — The write adds attribution feasibility

A nonzero report loads frozen weights and invokes the split calculation; noWeights reverts before marking the duty reported.

Source: `contracts/contracts/core/[project]CompoundReporting.sol:69-90`.

```solidity
    function _attribute(uint32 vaultId, uint32 roundId, uint128 xphar, uint256 toConvert) private {
        CoreStorage storage $ = _getCoreStorage();

        Snapshot memory s = _loadSnapshot($, vaultId, roundId);

        (
            uint256[] memory divertedParts,
            uint256[] memory capitalShares,
            uint256 divertedTotal,
            uint256 rest,
            bool noWeights
        ) = [project]Math.splitConversion(xphar, toConvert, s.divertedAvax, s.weights);
        if (noWeights) revert NoWeights();
        _creditDivertedParts($, vaultId, roundId, s, divertedParts);
        _creditCapitalShares($, vaultId, roundId, s, capitalShares);

        VaultRound storage vr = $.vaultRounds[vaultId][roundId];
        vr.divertedXphar += SafeCast.toUint128(divertedTotal);
        vr.compoundXphar += SafeCast.toUint128(rest);
        VaultStats storage vs = $.vaultStats[vaultId];
        vs.totalDivertedXphar += SafeCast.toUint128(divertedTotal);
        vs.totalCompoundXphar += SafeCast.toUint128(rest);
```

#### Step 3 — A pending duty blocks the next round

The round gate requires all conversion duties to be settled.

Source: `contracts/contracts/core/[project]Rounds.sol:34-41`.

```solidity
        _requireFlagClear(PAUSE_REPORTING);
        Vault storage v = _vault(vaultId);
        if (v.paused) revert VaultPaused();
        _requireNotShort(v);
        if (v.pendingCompoundRounds != 0) revert CompoundReportDue(vaultId, v.pendingCompoundRounds);
        RoundResult memory r = _computeRound(vaultId, input);
        if (msg.value != r.holderFunded) revert WrongFunding(r.holderFunded, msg.value);
        _recordRound(vaultId, v, r);
```

## Proof of concept

The targeted test calls canReportCompound successfully, then expects reportCompound to revert with NoWeights. The dashboard preview reports BLOCK_NO_PRINCIPAL and pendingCompoundRounds remains 1. This proves the preflight/write mismatch and retained pending duty in the constructed zero-weight state.

**Proof limits:** Privileged reporting and constructed zero-weight state; related to LHF-03 rather than an independent extraction root.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `contracts/audit-tests/test/unit/edge-reporting.test.ts:211-230`.

```typescript
describe("canReportCompound: the NoWeights exception, and the boundary of it", function () {
  it("ACCEPTS a report that reportCompound refuses with NoWeights - and previewCompound catches what canReportCompound cannot", async function () {
    const f = await networkHelpers.loadFixture(noWeightsWorld);
    const figure = 100n * E18;

    expect(await f.core.canReportCompound(VAULT_1, f.roundId, figure, false), "canReportCompound says yes").to.equal(f.toConvert);

    await expect(reportCompound(f, VAULT_1, f.roundId, figure, false, "doomed"), "the writer says no").to.be.revertedWithCustomError(
      f.core,
      "NoWeights",
    );

    const preview = await f.dashboardLens.previewCompound(VAULT_1, f.roundId, figure);
    expect(preview.ready, "the preview refuses what canReportCompound accepted").to.equal(false);
    expect(hasBit(BigInt(preview.blockers), BLOCK_NO_PRINCIPAL), "BLOCK_NO_PRINCIPAL").to.equal(true);
    expect(preview.failure, "and names the very error the transaction raises").to.equal(NO_WEIGHTS_SELECTOR);

    expect((await f.lens.getVault(VAULT_1)).pendingCompoundRounds, "the round is still pending").to.equal(1n);
    expect(await f.lens.isCompoundReported(VAULT_1, f.roundId), "and unreported").to.equal(false);
  });
```
