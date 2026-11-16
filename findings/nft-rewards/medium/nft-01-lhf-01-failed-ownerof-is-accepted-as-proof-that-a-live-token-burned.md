# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Medium ceiling; audit candidate, privileged/admin and external-NFT prerequisites  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Historical local or fork test excerpts; scope and fixture assumptions are described below  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Failed `ownerOf` is accepted as proof that a live token burned

## Description

A default administrator can permanently stop future rewards for a live NFT when its collection temporarily fails the bounded ownerOf query. Restoring readable ownership restores access to already-earned rewards, but does not undo the deactivation cutoff.

Prerequisites and limits: An ordinary holder cannot invoke the deactivation. The local test deliberately deploys a configurable malicious NFT, funds and activates synthetic tokens, and invokes the administrator path. The census confirms proxy/clone integrations, not hostile ownership of their upgrade authority.

### Mechanism step by step

#### Step 1 — An active token becomes temporarily unreadable

The bounded staticcall accepts only a successful, exactly 32-byte, nonzero owner result. A revert, exhausted stipend or malformed response leaves live=false.

Source: `contracts/contracts/core/[project]Storage.sol:229-243`.

```solidity
    function _tryOwnerOf(address collection, uint256 tokenId) internal view returns (bool burned) {
        uint256 stipend = _getCoreStorage().collections[collection].ownerOfGas;
        bytes memory callData = abi.encodeWithSelector(IERC721.ownerOf.selector, tokenId);
        bool live;
        assembly ("memory-safe") {
            let success := staticcall(stipend, collection, add(callData, 0x20), mload(callData), 0, 0)
            if and(success, eq(returndatasize(), 32)) {
                returndatacopy(0, 0, 32)
                let word := mload(0)
                if and(iszero(shr(160, word)), iszero(iszero(word))) {
                    live := 1
                }
            }
        }
        burned = !live;
```

#### Step 2 — The administrator classifies failure as a burn

The irreversible lifecycle transition is role-gated. The token must be activated, not already deactivated and eligible at the cutoff.

Source: `contracts/contracts/core/[project]Deactivation.sol:14-34`.

```solidity
    function deactivateBurned(address collection, uint256[] calldata tokenIds)
        external
        onlyRole(DEFAULT_ADMIN_ROLE)
        nonReentrant
    {
        CoreStorage storage $ = _getCoreStorage();

        Collection storage c = _collection(collection);

        _requireNotRoundDue(c.vaultId);

        uint256 n = tokenIds.length;
        if (n == 0) revert EmptyList();
        if (n > MAX_ACTIVATE_TOKENS) revert TooManyTokens(n, MAX_ACTIVATE_TOKENS);

        Vault storage dv = _getCoreStorage().vaults[c.vaultId];
        uint40 anchor = dv.roundAnchor;
        uint40 interval = dv.roundInterval;
        uint40 delay = dv.eligibilityDelay;
        uint40 effectiveFrom = _nextGridCutoff(anchor, interval, SafeCast.toUint40(block.timestamp));

```

#### Step 3 — Failure sets the permanent cutoff

The failed owner lookup permits deactivatedFrom to be written. Pre-cutoff rewards are computed separately and remain claimable when owner lookup recovers.

Source: `contracts/contracts/core/[project]Deactivation.sol:55-85`.

```solidity
        uint40 anchor,
        uint40 interval,
        uint40 delay
    ) private returns (uint128 unclaimed) {
        NftState storage nft = $.nfts[collection][tokenId];

        if (nft.activatedAt == 0) revert NotActivated();

        if (nft.deactivatedFrom != 0) revert AlreadyDeactivated(collection, tokenId);

        uint40 eligibleAt = nft.activatedAt + delay;
        if (eligibleAt > effectiveFrom) {
            revert TokenNotYetEligible(collection, tokenId, _nextGridCutoff(anchor, interval, eligibleAt - 1));
        }

        if (!_tryOwnerOf(collection, tokenId)) revert TokenStillOwned(collection, tokenId);

        nft.deactivatedFrom = effectiveFrom;

        uint32 first = nft.firstRound;
        if (first == 0) {
            first = [project]Math.lowerBoundCutoff($.collectionRounds[collection], roundCount, eligibleAt);
            if (first != 0) nft.firstRound = first;
        }

        if (first != 0) {
            uint32 base = nft.lastClaimedRound;
            if (base < first - 1) base = first - 1;
            unclaimed = $.collectionRounds[collection][roundCount].cumRewardPerNft
                - $.collectionRounds[collection][base].cumRewardPerNft;
        }
```

## Proof of concept

The test deploys a configurable NFT, synthetically funds and activates tokens, then makes ownerOf temporarily revert while the administrator calls deactivation. Restoring ownership readability does not restore post-cutoff accrual. Historical assertions preserve 1.8 AVAX pre-outage claimable, deny 1.8 AVAX of later reward to the affected token, and pay that later reward to a control token. No natural outage or production loss is established.

**Proof limits:** Administrator action and a failing external NFT are required; local fixture only.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `contracts/audit-tests/test/unit/deactivation-temporary-outage.test.ts:24-42`.

```typescript
async function temporaryOutageFixture() {
  const f = await deployCoreFixture();
  const firstActivationWindow = f.anchor + WEEK;
  const firstRewardCutoff = firstActivationWindow + 2n * WEEK;
  const secondRewardCutoff = firstRewardCutoff + WEEK;

  await networkHelpers.time.increaseTo(firstActivationWindow + 1n);
  const nft = await ethers.deployContract("MaliciousERC721");
  const collection = await nft.getAddress();

  await mined(f.core.createVault(NO_COMPOUND_SPLIT, await f.wallet.getAddress(), 0n, WEEK, WEEK, f.anchor));
  await mined(f.core.addCollection(collection, VAULT_ID, PRICE, true, 0n, 0n));
  await mined(nft.mint(f.holder.address, DEACTIVATED_TOKEN));
  await mined(nft.mint(f.holder.address, CONTROL_TOKEN));
  await mined(as(f.core, f.holder).activate(collection, [DEACTIVATED_TOKEN, CONTROL_TOKEN], { value: PRICE * 2n }));
  await mined(as(f.core, f.operator).reportBatch(VAULT_ID, XPHAR, false, "temporary-outage principal"));

  await advanceAndProcess(f, firstRewardCutoff, ROUND_ONE_REWARD, 1n, ROUND_ONE_FUNDING);
  return { ...f, nft, collection, firstRewardCutoff, secondRewardCutoff };
```

Source: `contracts/audit-tests/test/unit/deactivation-temporary-outage.test.ts:45-76`.

```typescript
describe("deactivation-temporary-outage", function () {
  it("permanently cuts off post-outage rewards when a temporary ownerOf revert is treated as a burn", async function () {
    const f = await loadFixture(temporaryOutageFixture);

    await mined(f.nft.setMode(MODE_REVERT_OWNER_OF));
    await expect(f.core.deactivateBurned(f.collection, [DEACTIVATED_TOKEN]))
      .to.emit(f.core, "TokenDeactivated")
      .withArgs(f.collection, DEACTIVATED_TOKEN, f.secondRewardCutoff, PER_TOKEN_REWARD);
    expect(await f.nft.realOwnerOf(DEACTIVATED_TOKEN), "the token was never burned").to.equal(f.holder.address);

    await mined(f.nft.setMode(0n));
    await advanceAndProcess(f, f.secondRewardCutoff, ROUND_TWO_REWARD, 2n, ROUND_TWO_FUNDING);

    const deactivated = await f.core.claimableOf(f.collection, [DEACTIVATED_TOKEN]);
    const control = await f.core.claimableOf(f.collection, [CONTROL_TOKEN]);
    const preOutageClaimableWei = BigInt(deactivated.infos[0].claimable);
    const controlEarnedAfterWei = BigInt(control.infos[0].claimable) - preOutageClaimableWei;
    const recoveredAfterWei = deactivated.infos[0].claimable - preOutageClaimableWei;
    const postRestoreDeniedWei = controlEarnedAfterWei - recoveredAfterWei;
    const recoveredAfterRestore = recoveredAfterWei !== 0n;

    expect(preOutageClaimableWei, "pre-outage reward remains claimable after ownerOf recovers").to.equal(PER_TOKEN_REWARD);
    expect(controlEarnedAfterWei, "the live control earns the post-cutoff round").to.equal(PER_TOKEN_REWARD);
    expect(recoveredAfterWei, "the restored token recovers no post-cutoff reward").to.equal(0n);
    expect(postRestoreDeniedWei, "the outage deactivation's per-token loss").to.equal(PER_TOKEN_REWARD);
    expect(recoveredAfterRestore, "post-cutoff rewards are not restored").to.equal(false);

    await expect(as(f.core, f.holder).claim(f.collection, [DEACTIVATED_TOKEN]))
      .to.emit(f.core, "TokenClaimed")
      .withArgs(f.collection, DEACTIVATED_TOKEN, PER_TOKEN_REWARD, 1n, 1n);
    await expect(as(f.core, f.holder).claim(f.collection, [CONTROL_TOKEN]))
      .to.emit(f.core, "TokenClaimed")
```
