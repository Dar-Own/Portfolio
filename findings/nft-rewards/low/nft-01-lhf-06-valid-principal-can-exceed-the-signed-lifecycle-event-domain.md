# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Low ceiling; privileged lifecycle denial at an extreme integer range  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Partial test or synthetic model; does not establish the complete deployed impact  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Valid principal can exceed the signed lifecycle-event domain

## Description

Repeated valid report supplements can accumulate uint128 principal beyond int128.max. Retirement or movement subsequently attempts to emit/log the full release as a signed negative int128 and reverts at the conversion.

Prerequisites and limits: Existing tests establish legal accumulation to uint128.max, not a separately executed retire/move failure at that boundary. The lifecycle consequence is a source deduction. DEFAULT_ADMIN can reduce principal through a manual adjustment before the lifecycle operation.

### Mechanism step by step

#### Step 1 — Valid supplements increase unsigned principal

The supplement route adds individually representable slices to unsigned collection and vault totals.

Source: `contracts/contracts/core/[project]BatchReporting.sol:84-115`.

```solidity
        (address[] memory contributors, uint256[] memory shares) = _contributorShares(batchId, xphar);

        uint32 vaultId = batch.vaultId;
        Vault storage v = $.vaults[vaultId];
        uint256 count = contributors.length;
        for (uint256 i; i < count; ++i) {
            address contributor = contributors[i];
            uint128 slice = SafeCast.toUint128(shares[i]);
            Collection storage col = $.collections[contributor];
            CollectionStats storage stats = $.collectionStats[contributor];

            stats.batchXphar += slice;

            if ([project]Math.isCurrentMember(col.vaultId, col.status, vaultId)) {
                uint128 newXphar = col.xphar + slice;
                col.xphar = newXphar;
                v.xpharTotal += slice;
                emit XpharChanged(
                    contributor, vaultId, XpharSource.BATCH, SafeCast.toInt128(int256(shares[i])), batchId, newXphar
                );
            } else {
                v.releasedXphar += slice;
                stats.releasedXphar += slice;
                _releaseForWithdrawal(vaultId, contributor, XpharSource.BATCH, slice, batchId);
            }
        }

        $.vaultStats[vaultId].batchXphar += xphar;
        uint128 totalReported = batch.xpharReported + xphar;
        batch.xpharReported = totalReported;

        emit BatchSupplemented(vaultId, batchId, xphar, totalReported, memo);
```

#### Step 2 — A lifecycle operation releases the full accumulated value

The release path narrows the entire released principal to int128 before negating it; a value above int128.max cannot be represented.

Source: `contracts/contracts/core/[project]Lifecycle.sol:177-197`.

```solidity
    function _releaseXphar(
        CoreStorage storage $,
        address collection,
        uint32 vaultId,
        Reason reason,
        uint32 refId
    ) private returns (uint128 released) {
        Collection storage c = $.collections[collection];
        Vault storage v = $.vaults[vaultId];

        released = c.xphar;
        c.xphar = 0;

        v.xpharTotal -= released;
        v.releasedXphar += released;
        $.collectionStats[collection].releasedXphar += released;

        int128 delta = -SafeCast.toInt128(int256(uint256(released)));
        uint32 adjustmentId = _logLifecycle(collection, vaultId, reason, delta, refId, 0);

        emit XpharChanged(collection, vaultId, XpharSource.ADJUSTMENT, delta, adjustmentId, 0);
```

#### Step 3 — The preflight omits the signed-domain check

The retirement can-check returns the lifecycle guard result, which validates operational state rather than the release conversion.

Source: `contracts/contracts/[project]AdminModule.sol:24-26`.

```solidity
    function canRetireCollection(address collection) external view returns (uint32 vaultId) {
        (, vaultId) = _checkRetire(collection);
    }
```

## Proof of concept

The boundary test begins with principal 1 and adds int128.max twice, reaching uint128.max. One further unit panics and leaves state unchanged. These assertions establish the accumulation prerequisite only. The later signed lifecycle-event failure follows from the source cast; an end-to-end lifecycle test is not supplied.

**Proof limits:** Artificial extreme-value boundary; prerequisite test does not reproduce the full lifecycle failure.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `contracts/audit-tests/test/unit/edge-reporting.test.ts:473-486`.

```typescript
  it("accumulates to exactly uint128.max and panics on the unit after that", async function () {
    const f = await networkHelpers.loadFixture(soloContributorWorld);
    await mined(supplement(f, BATCH_1, INT128_MAX, "first half"));
    await mined(supplement(f, BATCH_1, INT128_MAX, "second half"));

    expect((await f.lens.getCollection(f.col.address)).xphar, "the principal sits on the field's ceiling").to.equal(UINT128_MAX);
    expect((await f.lens.getVault(VAULT_1)).xpharTotal, "and so does the vault's total").to.equal(UINT128_MAX);
    expect((await f.lens.getBatch(BATCH_1)).xpharReported, "and the batch's running total").to.equal(UINT128_MAX);
    expect((await f.lens.getVaultStats(VAULT_1)).batchXphar, "and the lifetime counter").to.equal(UINT128_MAX);
    await expectBooks(f.lens, f.col.address, "at the uint128 ceiling");

    await expect(supplement(f, BATCH_1, 1n, "one unit too many")).to.be.revertedWithPanic(PANIC_ARITHMETIC);
    expect((await f.lens.getCollection(f.col.address)).xphar, "and nothing moved").to.equal(UINT128_MAX);
  });
```
